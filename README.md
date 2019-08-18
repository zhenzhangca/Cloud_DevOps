# Introduction
## Overview
The purpose of this repository is as follows:
1) Demonstrate how to dockerize a Java Springboot Micro-service Application using Docker for local development environment.  
2) Deploy the Application in AWS Elastic BeanStalk as a Dockerized container using the Elastic Beanstalk deployment service (combination of EC2, RDS, S3).
## Application
You can find the Trading Application that will be deployed at https://github.com/zhenzhangca/Trading_Application.git    
It is a trading platform simulation that allows traders to trade securities. By building REST APIs using Springboot, this application can implement the following functions:
- Manage trader information and accounts.
- Execute security orders (e.g. buy/sell stocks).
- Process data in batch and real-time.   
	
**Architecture diagram**  
	
![trading-app architecture](assets/trading-app%20architecture.png)


# Dockerize trading_app
## Clone project
Clone the Trading_Application project from GitHub using command `git clone https://github.com/zhenzhangca/Trading_Application.git`.
## Commands
```
#start docker
sudo systemctl start docker
#17.05 or higher
sudo docker -v

#create network bridge between SpringBoot app and postgreSQL
sudo docker network create --driver bridge trading-net

#build trading-app image
cd Trading_Application/
sudo docker build -t trading-app .

#build jrvs-psql image
cd psql/
sudo docker build -t jrvs-psql .

#run a jrvs-psql container, attach this container to trading-net network
sudo docker run --rm --name jrvs-psql \
-e POSTGRES_PASSWORD=password \
-e POSTGRES_DB=jrvstrading \
-e POSTGRES_USER=postgres \
--network trading-net \
-d -p 5432:5432 jrvs-psql

#Setup IEX token
IEX_TOKEN='my_IEX_token'
#run a trading-app container
sudo docker run \
-e "PSQL_URL=jdbc:postgresql://jrvs-psql:5432/jrvstrading" \
-e "PSQL_USER=postgres" \
-e "PSQL_PASSWORD=password" \
-e "IEX_PUB_TOKEN=${IEX_TOKEN}" \
--network trading-net \
-p 8080:8080 -t trading-app

#list all the running container
docker container ls

#verify health
curl localhost:8080/health

#verify Swagger UI from the browser
localhost:8080/swagger-ui.html
```
## Swagger-UI page

![swagger](assets/swagger.png)
## Docker Architecture Diagram
![Dockerize trading-app Diagram](assets/Dockerize%20trading-app%20Diagram.png)

## Dockerfiles

  - Create a Dockerfile in the root directory of Trading_Application   
  ![root_dockerfile](assets/root_dockerfile.jpeg)  
   
  `Build stage` commands:
   1) Pull the maven image from Dockerhub, then launch a container (rename it as build) before running the app.  
   2) Copy `src` to `/build/src`.  
   3) Copy `pom.xml` to `/build/`.  
   4) Run `mvn -f /build/pom.xml clean package -DskipTests` to generate artifacts(jar files). 
   Because we just need this container to compile and package the app rather than running it, we will not need it after the artifacts are generated.  
  
  `Package stage` commands:
   1) Pull the open JDK image from Dockerhub, then launch a container.   
   2) Copy jar file to the location of container.    
   3) Run the app.    
   
    
  - Create a Dockerfile in the `psql` directory   
  ![psql_dockerfile](assets/psql_dockerfile.jpeg)  
  1) Database  
  In the commands above, we set up the value of environment variables `POSTGRES_PASSWORD`, `POSTGRES_DB` and `POSTGRES_USER`. Then, a database with the name of `POSTGRES_DB` will be created when the postgres image is first started.  
  2) Tables  
  In the Dockerfile, command `FROM postgres` means regarding the postgres image from Dockerhub as a basic image, command `COPY ./sql_ddl/schema.sql /docker-entrypoint-initdb.d/` means copy `schema.sql` file from localhost to the postgres container.  
  After the entrypoint calls initdb to create the default postgres user and database, it will run `schema.sql` file automatically to create following tables: 
   
**ER diagram**  

   ![ER](assets/ER.png)  
   
# Cloud Architecture Diagram
- trading app diagram
  - use draw.io and aws icons (it's in the draw.io library)
  - include ec2, alb, auto scaling, target group, rds
  - security groups
  - label all important ports(e.g. ALB HTTP, ec2 tpc:5000, RDS tcp:5432)
  
# AWS EB and Jenkins CI/CD Pipeline Diagram
- Please refer to Jenkins guide architecture diagram.