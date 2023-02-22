## Project 20 - Migration to the Сloud with containerization. Part 1 - Docker & Docker Compose

Until now, you have been using VMs (AWS EC2) in Amazon Virtual Private Cloud (AWS VPC) to deploy your web solutions, and it works well in many cases. You have learned how easy to spin up and configure a new EC2 manually or with such tools as Terraform and Ansible to automate provisioning and configuration. You have also deployed two different websites on the same VM; this approach is scalable, but to some extent; imagine what if you need to deploy many small applications (it can be web front-end, web-backend, processing jobs, monitoring, logging solutions, etc.) and some of the applications will require various OS and runtimes of different versions and conflicting dependencies – in such case you would need to spin up serves for each group of applications with the exact OS/runtime/dependencies requirements. When it scales out to tens/hundreds and even thousands of applications (e.g., when we talk of microservice architecture), this approach becomes very tedious and challenging to maintain.

In this project, we will learn how to solve this problem and practice the technology that revolutionized application distribution and deployment back in 2013! We are talking of Containers and imply Docker. Even though there are other application containerization technologies, Docker is the standard and the default choice for shipping your app in a container!

## Understanding why we need docker instead of a VM
As you have already learned – unlike a VM, Docker allocated not the whole guest OS for your application, but only isolated minimal part of it – this isolated container has all that your application needs and at the same time is lighter, faster, and can be shipped as a Docker image to multiple physical or virtual environments, as long as this environment can run Docker engine. This approach also solves the environment incompatibility issue. It is a well-known problem when a developer sends his application to you, you try to deploy it, deployment fails, and the developer replies, "- It works on my machine!". With Docker – if the application is shipped as a container, it has its own environment isolated from the rest of the world, and it will always work the same way on any server that has Docker engine.

1. mysql in container
- Pull MySQL Docker Image from Docker Hub Registry
using this command 
```
docker pull mysql/mysql-server:latest
```

![](/mysql.PNG)

If you are interested in a particular version of MySQL, replace latest with the version number. Visit Docker Hub to check other tags [here](https://hub.docker.com/r/mysql/mysql-cluster/tags)

List the images to check that you have downloaded them successfully:

```
docker image ls
```
![](/prj20_002.PNG)

### Step 2: Deploy the MySQL Container to your Docker Engine

1. Once you have the image, move on to deploying a new MySQL container with:

```
docker run --name <container_name> -e MYSQL_ROOT_PASSWORD=<my-secret-pw> -d mysql/mysql-server:latest
```

![](/prj20_003.PNG)

- Replace <container_name> with the name of your choice. If you do not provide a name, Docker will generate a random one
- The -d option instructs Docker to run the container as a service in the background
- Replace <my-secret-pw> with your chosen password
- In the command above, we used the latest version tag. This tag may differ according to the image you downloaded

2. Then, check to see if the MySQL container is running: Assuming the container name specified is mysql-server

```
docker ps -a
```
![](/prj20_004.PNG)

## CONNECTING TO THE MYSQL DOCKER CONTAINER
### step 3: connecting to the mySQL Docker Container


We can either connect directly to the container running the MySQL server or use a second container as a MySQL client. Let us see what the first option looks like.

```
 docker exec -it mysql bash

or

$ docker exec -it mysql mysql -uroot -p

```
![](/prj20_005.PNG)

## Approach 2

At this stage you are now able to create a docker container but we will need to add a network. So, stop and remove the previous mysql docker container.

```
docker ps -a
docker stop mysql 
docker rm mysql or <container ID> 04a34f46fb98
```
![](/prj20_006.PNG)

## first, create a network 
```
docker network create --subnet=172.18.0.0/24 tooling_app_network 
```
Creating a custom network is not necessary because even if we do not create a network, Docker will use the default network for all the containers you run. By default, the network we created above is of DRIVER Bridge. So, also, it is the default network. You can verify this by running the docker network ls command.

But there are use cases where this is necessary. For example, if there is a requirement to control the cidr range of the containers running the entire application stack. This will be an ideal situation to create a network and specify the --subnet

For clarity’s sake, we will create a network with a subnet dedicated for our project and use it for both MySQL and the application so that they can connect.

Run the MySQL Server container using the created network.

First, let us create an environment variable to store the root password:

```
 $ export MYSQL_PW= 
```
verify the environment variable is created

```
echo $MYSQL_PW
```
![](/prj20_007.PNG)

Then, pull the image and run the container, all in one command like below:

```
 docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW  -d mysql/mysql-server:latest
 ```
 Flags used

- -d runs the container in detached mode
- --network connects a container to a network
- -h specifies a hostname

If the image is not found locally, it will be downloaded from the registry.

Verify the container is running:

```
docker ps -a 
```
![](/prj20_009.PNG)

As you already know, it is best practice not to connect to the MySQL server remotely using the root user. Therefore, we will create an SQL script that will create a user we can use to connect remotely.

Create a file and name it create_user.sql and add the below code in the file:

Run script on server and connect from client

![](/prj20_010.PNG)
## 2. Prepare DB Schema
1. Clone tooling repo `$ git clone https://github.com/darey-devops/tooling.git`


2. Export the location of the SQL file `$ export tooling_db_schema=/tooling_db_schema.sql`
3. Use the SQL script to create the database and prepare the schema. With the docker exec command, you can execute a command in a running container. `$ docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema`
4. Update the .env file with connection details to the database

5. Run the Tooling App 

3. Containerize Tooling App

1. Inside the tooling dir, build container `docker build -t tooling:0.0.1` .

2. Run container `docker run --network tooling_app_network -p 8085:80 -it tooling:0.0.1`

![](/prj20_011.PNG)


# Practice Task

 1. Implement a POC to migrate the PHP-Todo app into a containerized application

 ![](/prj20_012.PNG)
 

 2. Run both database and app on your laptop Docker Engine
 3. Access the application from the browser
![](/prj20_013.PNG)


## part 2

1. Create an account in Docker Hub


2. Create a new Docker Hub repository
3. Push the docker images from your PC to the repository

![](/prj20_014.PNG)

## part 3 

1. Write a Jenkinsfile that will simulate a Docker Build and a Docker Push to the registry
2. Connect your repo to Jenkins
3. Create a multi-branch pipeline
4. Simulate a CI pipeline from a feature and master branch using previously created Jenkinsfile

![](/prj20_015.PNG)
5. Ensure that the tagged images from your Jenkinsfile have a prefix that suggests which branch the image was pushed from. For example, feature-0.0.1.
6. Verify that the images pushed from the CI can be found at the registry.

![](/prj20_016.PNG)


## Deployment with Docker Compose


All we have done until now required quite a lot of effort to create an image and launch an application inside it. We should not have to always run Docker commands on the terminal to get our applications up and running. There are solutions that make it easy to write declarative code in YAML, and get all the applications and dependencies up and running with minimal effort by launching a single command.

In this section, we will refactor the Tooling app POC so that we can leverage the power of Docker Compose.

1. First, install Docker Compose on your workstation from here
2. Create a file, name it tooling.yaml
3. Begin to write the Docker Compose definitions with YAML syntax. The YAML file is used for defining services, networks, and volumes:

```
version: "3.9"
services:
  tooling_frontend:
    build: .
    ports:
      - "5000:80"
    volumes:
      - tooling_frontend:/var/www/html
```
The YAML file has declarative fields, and it is vital to understand what they are used for.

- version: Is used to specify the version of Docker Compose API that the Docker Compose engine will connect to. This field is optional from docker compose version v1.27.0. You can verify your installed version with:
docker-compose --version
docker-compose version 1.28.5, build c4eb3a1f
- service: A service definition contains a configuration that is applied to each container started for that service. In the snippet above, the only service listed there is tooling_frontend. So, every other field under the tooling_frontend service will execute some commands that relate only to that service. Therefore, all the below-listed fields relate to the tooling_frontend service.
- build
- port
- volumes
- links

visit this this site for more info:

https://www.balena.io/docs/reference/supervisor/docker-compose/

Let us fill up the entire file and test our application:


```
version: "3.9"
services:
  tooling_frontend:
    build: .
    ports:
      - "5000:80"
    volumes:
      - tooling_frontend:/var/www/html
    links:
      - db
  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: <The database name required by Tooling app >
      MYSQL_USER: <The user required by Tooling app >
      MYSQL_PASSWORD: <The password required by Tooling app >
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql
volumes:
  tooling_frontend:
  db:
```
Run the command to start the containers


```
docker-compose -f tooling.yaml  up -d 
```
![](/prj20_017.PNG)

Verify that the compose is in the running status:

```
docker compose ls
```
![](/prj20_018.PNG)
