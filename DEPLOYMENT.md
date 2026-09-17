# Deploying EasyCRUD to EKS Fargate

This is the step-by-step guide for deploying this app to Amazon EKS on **Fargate**
(no EC2 worker nodes anywhere), with images in **ECR**, traffic routed through a
single **ALB Ingress**, and builds/deploys automated by **GitHub Actions** on
every push to `dev`.

Everything here assumes you're doing the AWS setup through the **AWS Console**
(GUI). The only CLI-ish step is a couple of `kubectl`/`mysql` commands, which you
run from **AWS CloudShell** in the browser — you never need to create or manage
an EC2 instance.

## Architecture

```
                         ┌─────────────────────────┐
 Internet  ───────────▶  │   ALB (Ingress)          │
                         │   one public DNS name     │
                         └───────────┬───────────────┘
                                     │
                      ┌──────────────┴──────────────┐
                      │                              │
               path: /api/*                    path: /*
                      │                              │
                      ▼                              ▼
            ┌──────────────────┐           ┌──────────────────┐
            │ backend-svc       │           │ frontend-svc      │
            │ (ClusterIP :8080) │           │ (ClusterIP :80)   │
            └─────────┬─────────┘           └─────────┬─────────┘
                      │                                │
                      ▼                                ▼
            ┌──────────────────┐           ┌──────────────────┐
            │ backend Deployment │         │ frontend Deployment │
            │ runs on Fargate    │         │ runs on Fargate      │
            └─────────┬─────────┘          └──────────────────┘
                      │
                      ▼
              RDS (MariaDB) — student_db
```

The frontend calls a **relative** `/api` path (not a hardcoded backend URL), so
both apps share one ALB / one origin. No CORS, no editing `.env` after every
backend redeploy.

Files that make this work, already on the `dev` branch:

| File | Purpose |
|---|---|
| `k8s/namespace.yaml` | Creates the `easycrud` namespace (this is also the Fargate profile's selector) |
| `k8s/backend-deployment.yaml` | Backend Deployment, 2 replicas |
| `k8s/backend-service.yaml` | ClusterIP service for backend, port 8080 |
| `k8s/frontend-deployment.yaml` | Frontend Deployment, 2 replicas |
| `k8s/frontend-service.yaml` | ClusterIP service for frontend, port 80 |
| `k8s/ingress.yaml` | ALB Ingress, path-routes `/api` → backend, `/` → frontend |
| `.github/workflows/deploy.yml` | Builds both images, pushes to ECR, applies `k8s/`, waits for rollout |
| `frontend/.env` | `VITE_API_URL=/api` (relative, baked into the frontend build) |
| `backend/src/main/resources/application.properties` | Your RDS endpoint / DB creds — configure this yourself before merging |

---

## Prerequisites checklist

- [ ] AWS account with console access
- [ ] RDS MariaDB instance already running (you're handling this separately)
- [ ] `application.properties` already points at your RDS endpoint
- [ ] This repo's `dev` branch is what you'll deploy from
- [ ] A GitHub account with push access to this repo

---

## Phase 1 — ECR: create your image repositories

1. Console → **ECR** → **Repositories** → **Create repository**.
2. Visibility: Private. Name: `easycrud-backend`. Leave the rest default. **Create repository**.
3. Repeat with name `easycrud-frontend`.
4. Note down your **AWS Account ID** (top-right account menu) and your **region**
   (e.g. `ap-south-1` — match wherever your RDS instance lives).

- [ ] `easycrud-backend` repo created
- [ ] `easycrud-frontend` repo created
- [ ] Account ID and region noted

---

## Phase 2 — Using the Default VPC (no CloudFormation)

You're using the account's **Default VPC** and its existing subnets — no new
VPC. There's one important fact this depends on: **Fargate pods never get a
public IP, even in a subnet that routes straight to an Internet Gateway.** So
regardless of "public" vs "private" labeling, pods need a route to a **NAT
Gateway** to reach the internet (to pull images from ECR). The Default VPC
doesn't have one, so we add just that — everything else stays as-is.

Plan: keep 2 of the default subnets (different AZs) untouched for the **ALB**,
and re-point 2 *other* default subnets (different AZs) to a NAT Gateway for
the **Fargate pods**. No subnets are created or deleted — we're just adding a
NAT Gateway and changing which route table 2 of the existing subnets use.

1. Console → **VPC** → **Your VPCs** → find the one marked "Default VPC: Yes".
   Note its **VPC ID**.
2. **Subnets** (left nav) → filter by that VPC ID. You'll see one subnet per
   AZ, each already with a route to the Internet Gateway and "Auto-assign
   public IPv4" = Yes. Note at least 4 subnet IDs across at least 2 AZs (most
   regions have 3+ AZs, so this is usually already there):
   - 2 subnets → **ALB subnets** (leave these completely alone)
   - 2 different subnets → **Pod subnets** (these get re-routed below)
3. **Elastic IPs** → **Allocate Elastic IP address** → allocate one (no config needed).
4. **NAT Gateways** → **Create NAT gateway** → subnet = one of your **ALB
   subnets** (it needs the IGW route) → Elastic IP allocation = the one from
   step 3 → **Create NAT gateway**. Wait until its status is `Available`.
5. **Route Tables** → **Create route table** → name `easycrud-pod-rt`, VPC =
   the Default VPC → **Create**.
   - Open it → **Routes** tab → **Edit routes** → **Add route** →
     `0.0.0.0/0` → target = the NAT Gateway from step 4 → **Save**.
   - **Subnet associations** tab → **Edit subnet associations** → check your
     2 **Pod subnets** → **Save**. (This is the only thing that changes about
     those subnets — they keep existing, just stop using the main/IGW route
     table.)
6. Tag the subnets so EKS and the ALB controller can auto-discover them
   (Subnets page → select a subnet → **Tags** tab → **Manage tags**):
   - On the 2 **ALB subnets**: add `kubernetes.io/role/elb` = `1`
   - On the 2 **Pod subnets**: add `kubernetes.io/role/internal-elb` = `1`
   - You'll add one more tag (`kubernetes.io/cluster/easycrud-cluster` =
     `shared`) to all 4 subnets after the cluster exists in Phase 4 — EKS adds
     this automatically to subnets you select during cluster creation, so
     there's nothing to do here yet.

- [ ] Default VPC ID noted
- [ ] 2 ALB subnet IDs and 2 Pod subnet IDs noted
- [ ] NAT Gateway `Available`
- [ ] `easycrud-pod-rt` created, routes to NAT Gateway, associated with the 2 Pod subnets
- [ ] `kubernetes.io/role/elb=1` tag on the 2 ALB subnets

> **Cost note:** a NAT Gateway runs about $0.045/hr (~$32/month) plus a small
> per-GB data charge — this is the one recurring cost this setup adds beyond
> EKS, RDS, and the ALB itself. If you want to avoid it entirely, the
> alternative is several VPC Interface Endpoints (ECR API, ECR DKR, STS,
> EC2, ELB) plus the free S3 Gateway endpoint — more setup, but no NAT
> Gateway charge. Stick with the NAT Gateway above unless that trade-off
> matters to you.

---

## Phase 3 — IAM roles/users you need up front

IAM → **Roles** → **Create role**, three times:

**a) EKS cluster role**
- Trusted entity: AWS service → **EKS** → **EKS - Cluster**
- Attach: `AmazonEKSClusterPolicy`
- Name: `easycrud-eks-cluster-role`

**b) Fargate pod execution role**
- Trusted entity: AWS service → **EKS** → **EKS - Fargate pod**
- Attach: `AmazonEKSFargatePodExecutionRolePolicy`
- Name: `easycrud-eks-fargate-role`

**c) GitHub Actions deployer (IAM user, not a role)**
- IAM → **Users** → **Create user** → name `github-actions-deployer` → no console access
- Attach this inline policy (replace `ACCOUNT_ID`):
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      { "Effect": "Allow", "Action": ["ecr:GetAuthorizationToken"], "Resource": "*" },
      { "Effect": "Allow", "Action": [
          "ecr:BatchCheckLayerAvailability","ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage","ecr:PutImage","ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart","ecr:CompleteLayerUpload"
        ], "Resource": "arn:aws:ecr:*:ACCOUNT_ID:repository/easycrud-*" },
      { "Effect": "Allow", "Action": ["eks:DescribeCluster"], "Resource": "*" }
    ]
  }
  ```
- **Security credentials** tab → **Create access key** → "Application running
  outside AWS" → save the Access Key ID and Secret (you'll need these in
  Phase 9 — you won't be able to see the secret again).

- [ ] `easycrud-eks-cluster-role` created
- [ ] `easycrud-eks-fargate-role` created
- [ ] `github-actions-deployer` user created, access key saved

---

## Phase 4 — Create the EKS cluster

1. Console → **EKS** → **Clusters** → **Create cluster**.
2. Name: `easycrud-cluster`. Kubernetes version: latest available. Cluster
   service role: `easycrud-eks-cluster-role`.
3. **Networking** step: VPC = your Default VPC; select **all 4** subnets from
   Phase 2 (the 2 ALB subnets + the 2 Pod subnets). Cluster endpoint access:
   **Public and private**.
4. Leave add-ons at their defaults (CoreDNS, kube-proxy, VPC CNI).
5. **Create**. Takes ~10-15 minutes.

- [ ] Cluster status: `Active`

---

## Phase 5 — Fargate profiles (this is what removes EC2 entirely)

Once the cluster is `Active`, cluster page → **Compute** tab → **Add Fargate profile**:

**Profile 1 — your app**
- Name: `easycrud-profile`
- Pod execution role: `easycrud-eks-fargate-role`
- Subnets: only the 2 **Pod subnets** from Phase 2 (the ones routed to the NAT Gateway)
- Namespace selector: `easycrud`

**Profile 2 — CoreDNS (required, easy to miss)**
- Subnets: same 2 **Pod subnets** as Profile 1
- Namespace: `kube-system`
- Labels: `k8s-app: kube-dns`

Without Profile 2, CoreDNS has nowhere to run — no EC2 nodes means no default
scheduling target — and nothing in your cluster will resolve DNS or reach RDS.

CoreDNS also ships with an annotation telling it "only run on EC2 nodes." Clear
it from **AWS CloudShell** (Console → CloudShell icon, top nav) once the
profiles exist:

```sh
aws eks update-kubeconfig --name easycrud-cluster --region <your-region>
kubectl patch deployment coredns -n kube-system --type json \
  -p='[{"op": "remove", "path": "/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
```

- [ ] `easycrud-profile` created (namespace `easycrud`)
- [ ] CoreDNS Fargate profile created (namespace `kube-system`, label `k8s-app=kube-dns`)
- [ ] CoreDNS annotation patched, `kubectl get pods -n kube-system` shows CoreDNS `Running`

---

## Phase 6 — Grant cluster access (EKS Access Entries)

Cluster page → **Access** tab → **Create access entry**:

1. IAM principal: your own IAM user/role → type Standard → policy
   `AmazonEKSClusterAdminPolicy` → scope: cluster-wide.
2. Repeat for the `github-actions-deployer` user from Phase 3c — same policy,
   same scope. This is what lets the GitHub Actions workflow's `kubectl apply`
   succeed.

- [ ] Your own user has an access entry
- [ ] `github-actions-deployer` has an access entry

---

## Phase 7 — AWS Load Balancer Controller (turns Ingress into a real ALB)

1. Cluster → **Access** tab → associate an **OIDC provider** if not already
   associated (there's a button for this; if not, grab the OIDC URL from the
   cluster Overview page and add it under **IAM → Identity providers → Add
   provider**, type OpenID Connect, audience `sts.amazonaws.com`).
2. IAM → **Policies** → **Create policy** → JSON tab → paste the contents of:
   ```
   https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.9.0/docs/install/iam_policy.json
   ```
   Name it `AWSLoadBalancerControllerIAMPolicy`.
3. IAM → **Roles** → **Create role** → trusted entity **Web identity** → select
   your cluster's OIDC provider → audience `sts.amazonaws.com` → condition:
   restrict to service account
   `system:serviceaccount:kube-system:aws-load-balancer-controller`. Attach the
   policy from step 2. Name: `AmazonEKSLoadBalancerControllerRole`.
4. Cluster → **Add-ons** tab → **Get more add-ons** → **AWS Load Balancer
   Controller** → select it → IAM role = the role from step 3 → **Create**.

- [ ] OIDC provider associated
- [ ] `AWSLoadBalancerControllerIAMPolicy` created
- [ ] `AmazonEKSLoadBalancerControllerRole` created
- [ ] AWS Load Balancer Controller add-on installed and `Active`

---

## Phase 8 — Reach RDS without EC2 (CloudShell VPC environment)

This replaces "spin up an EC2 box just to touch the database":

1. Console → **CloudShell** → **Actions → Create VPC environment**.
2. Pick the Default VPC from Phase 2, one of the **Pod subnets** (it has NAT
   Gateway internet access, so CloudShell can still reach the package
   repos), and a security group that's allowed into your RDS security group
   on port 3306 (add an inbound rule on the RDS SG for that CloudShell SG, or
   temporarily allow the VPC CIDR on 3306).
3. In that CloudShell session:
   ```sh
   sudo yum install -y mariadb105
   mysql -h <your-rds-endpoint> -u admin -p
   ```
   ```sql
   CREATE DATABASE student_db;
   ```
   You don't need to create the `students` table by hand — `application.properties`
   already sets `spring.jpa.hibernate.ddl-auto=update`, so Hibernate creates it
   the first time the backend pod starts successfully.
4. Open the RDS security group's inbound rule to also allow port 3306 from
   your **EKS VPC CIDR** (or the Fargate pods' security group) — the backend
   pod itself needs this too, not just CloudShell.

- [ ] `student_db` database created on RDS
- [ ] RDS security group allows inbound 3306 from the EKS VPC

---

## Phase 9 — GitHub repo secrets

In this repo on GitHub → **Settings → Secrets and variables → Actions → New
repository secret**, add:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | from Phase 3c |
| `AWS_SECRET_ACCESS_KEY` | from Phase 3c |
| `AWS_REGION` | e.g. `ap-south-1` |
| `AWS_ACCOUNT_ID` | your 12-digit account ID |
| `EKS_CLUSTER_NAME` | `easycrud-cluster` |

- [ ] All five secrets added

---

## Phase 10 — Deploy and verify

1. Push/merge your changes into `dev`.
2. Watch the **Actions** tab in GitHub — `deploy.yml` builds both images,
   pushes to ECR, applies everything in `k8s/`, and waits for rollout.
3. Once it's green, get the public address from CloudShell:
   ```sh
   kubectl get ingress easycrud-ingress -n easycrud
   ```
   Grab the `ADDRESS` column — that's the ALB DNS name, one URL for both
   frontend and backend.
4. Open it in a browser. Frontend loads at `/`; its calls to `/api/...` route
   through the same ALB to the backend.

- [ ] GitHub Actions run green
- [ ] `kubectl get ingress` shows an `ADDRESS`
- [ ] App loads in browser and CRUD operations work end to end

---

## Troubleshooting

- **Pods stuck `Pending`** — usually means the Fargate profile's namespace
  selector doesn't match, or `resources.requests` are missing/too large for
  any Fargate size. Check `kubectl describe pod <name> -n easycrud`.
- **CoreDNS stuck `Pending`** — you're missing the `kube-system` Fargate
  profile from Phase 5, or haven't patched the compute-type annotation yet.
- **Ingress has no `ADDRESS`** — the AWS Load Balancer Controller isn't
  running or its IAM role/OIDC trust is wrong. `kubectl get pods -n kube-system`
  should show a controller pod; `kubectl describe ingress easycrud-ingress -n easycrud`
  shows events with the actual error.
- **Backend `CrashLoopBackOff` / can't reach DB** — check the RDS security
  group inbound rule (Phase 8, step 4) and that `application.properties` has
  the right endpoint/credentials.
- **GitHub Actions fails on `kubectl apply`** — the `github-actions-deployer`
  IAM user is missing an EKS access entry (Phase 6) or the secrets in Phase 9
  are wrong/missing.
- **`ImagePullBackOff`** — double-check `AWS_ACCOUNT_ID` and `AWS_REGION`
  secrets match your actual ECR repo URIs. Also confirm the Fargate profile's
  subnets are actually the 2 **Pod subnets** associated with `easycrud-pod-rt`
  (Phase 2) — if a pod lands in an ALB subnet instead, it has no route to the
  NAT Gateway and can't reach ECR at all.

## Cleaning up (avoid ongoing charges)

When you're done testing, delete in this order to avoid dangling resources:
1. `kubectl delete -f k8s/` (removes the Ingress first, so AWS deletes the ALB)
2. Delete the Fargate profiles, then the EKS cluster
3. Delete the NAT Gateway from Phase 2, then release its Elastic IP, then
   delete the `easycrud-pod-rt` route table (the Default VPC and its subnets
   stay — you only added these three things to it)
4. Delete the ECR repositories (or just the images in them)
5. Delete/stop the RDS instance if you no longer need it
