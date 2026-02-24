FROM maven:3.8.3-openjdk-17
COPY . /opt/
WORKDIR /opt
RUN mvn clean package 
WORKDIR target/
EXPOSE 8080
ENTRYPOINT ["java","-jar"]
CMD ["student-registration-backend-0.0.1-SNAPSHOT.jar"]
