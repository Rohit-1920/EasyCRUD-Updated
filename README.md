# 🚀 EasyCRUD -- Full Stack Deployment on AWS EKS

EasyCRUD is a full-stack CRUD application deployed using:

-   Docker
-   Kubernetes (Amazon EKS)
-   AWS RDS (MariaDB)
-   NGINX Ingress Controller
-   AWS Route53
-   Hostinger Domain

This documentation explains the complete deployment process from
infrastructure setup to DNS configuration.

------------------------------------------------------------------------

# 1️⃣ EKS Cluster Setup

Follow the complete EKS cluster setup from this repository:

👉 https://github.com/Rohit-1920/Kubernetes.git

After setup, verify:

``` bash
kubectl get nodes
```

Nodes should be in `Ready` state.

------------------------------------------------------------------------

# 2️⃣ AWS RDS (MariaDB) Setup

## Step 1 -- Create Database

Go to:

AWS Console → RDS → Create Database

### Configuration:

-   Engine: MariaDB\
-   DB Instance Identifier: `easycrud-db`\
-   Database Name: `student_db`\
-   Master Username: `admin`\
-   Public Access: Enabled (for testing)\
-   Security Group: Allow inbound port `3306`

After creation, copy the **RDS Endpoint**.

------------------------------------------------------------------------

## Step 2 -- Install MySQL Client

``` bash
sudo apt update
sudo apt install mysql-client -y
```

------------------------------------------------------------------------

## Step 3 -- Connect to RDS

``` bash
mysql -h <rds-endpoint> -u admin -p
```

Verify:

``` sql
SHOW DATABASES;
```

------------------------------------------------------------------------

# 3️⃣ Backend Deployment

## Step 1 -- Clone Repository

``` bash
git clone https://github.com/Rohit-1920/EasyCRUD-Updated.git
cd EasyCRUD-Updated
```

------------------------------------------------------------------------

## Step 2 -- Update `application.properties`

``` properties
spring.datasource.url=jdbc:mysql://<rds-endpoint>:3306/student_db
spring.datasource.username=admin
spring.datasource.password=admin123
spring.jpa.hibernate.ddl-auto=update
```

------------------------------------------------------------------------

## Step 3 -- Build Docker Image

``` bash
docker build -t milindnagne/easycrud-backend:v1 .
```

------------------------------------------------------------------------

## Step 4 -- Docker Login

``` bash
docker login
```

------------------------------------------------------------------------

## Step 5 -- Push Image

``` bash
docker push milindnagne/easycrud-backend:v1
```

------------------------------------------------------------------------

## Step 6 -- Backend Deployment YAML

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: milindnagne/easycrud-backend:v1
          ports:
            - containerPort: 8080
```

------------------------------------------------------------------------

## Backend Service YAML

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
```

------------------------------------------------------------------------

## Deploy Backend

``` bash
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
```

Verify:

``` bash
kubectl get pods
kubectl get svc
```

------------------------------------------------------------------------

# 4️⃣ Frontend Deployment

## Step 1 -- Get Backend LoadBalancer Endpoint

``` bash
kubectl get svc
```

Copy backend `EXTERNAL-IP`.

------------------------------------------------------------------------

## Step 2 -- Update `.env` File

``` env
VITE_API_URL=http://<backend-lb-endpoint>:8080/api
```

------------------------------------------------------------------------

## Step 3 -- Build Docker Image

``` bash
docker build -t milindnagne/easycrud-frontend:v1 .
```

------------------------------------------------------------------------

## Step 4 -- Push Image

``` bash
docker push milindnagne/easycrud-frontend:v1
```

------------------------------------------------------------------------

## Step 5 -- Frontend Deployment YAML

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: milindnagne/easycrud-frontend:v1
          ports:
            - containerPort: 80
```

------------------------------------------------------------------------

## Frontend Service YAML

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

------------------------------------------------------------------------

## Deploy Frontend

``` bash
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

------------------------------------------------------------------------

# 5️⃣ Install Ingress Controller

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

Verify:

``` bash
kubectl get svc -n ingress-nginx
```

Copy the Ingress LoadBalancer EXTERNAL-IP (ELB DNS).

------------------------------------------------------------------------

# 6️⃣ Ingress Configuration (Host-Based Routing)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: easycrud-ingress
spec:
  rules:
    - host: frontend.milindproject.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: api.milindproject.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 8080
```

Apply:

``` bash
kubectl apply -f ingress.yaml
```

------------------------------------------------------------------------

# 7️⃣ Route53 & Hostinger DNS Configuration

## Step 1 -- Create Hosted Zone

AWS Console → Route53 → Hosted Zones → Create Hosted Zone

Domain Name:

milindproject.com

Copy the 4 generated Nameservers.

------------------------------------------------------------------------

## Step 2 -- Configure Nameservers in Hostinger

Login to Hostinger:

Domains → Your Domain → Nameservers → Use Custom Nameservers

Replace existing nameservers with Route53 nameservers.

Save changes.

------------------------------------------------------------------------

## Step 3 -- Create DNS Records in Route53

Inside Hosted Zone → Create Record

Create:

  Record Name   Type        Target
  ------------- ----------- -------------
  frontend      A (Alias)   Ingress ELB
  api           A (Alias)   Ingress ELB

------------------------------------------------------------------------

# 🌐 Final Access URLs

http://frontend.milindproject.com\
http://api.milindproject.com

------------------------------------------------------------------------

# 📌 Version 6.0 -- Complete Deployment

## Infrastructure

-   EKS Cluster setup (referenced repository)
-   AWS RDS (MariaDB) configured

## Backend

-   Dockerized and deployed
-   LoadBalancer service created

## Frontend

-   Dockerized and deployed
-   Connected to backend

## Networking

-   NGINX Ingress Controller installed
-   Host-Based Routing configured
-   Route53 + Hostinger DNS configured

------------------------------------------------------------------------

## 👨‍💻 Author

Milind Nagne\
DevOps & Cloud Enthusiast 🚀
