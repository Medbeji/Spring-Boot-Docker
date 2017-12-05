# Dockerizing Micro-Services


The plan of this tutorial : 
 - Project Architecture
 - Building the images of each services:
        - Using DockerFile
        - Using Spotify Plugin
  - Running the containers:
        - Manually
        - Using docker-compose
  - Testing the services

# Project Architecture


  [![N|Solid](https://liliasfaxi.github.io/TP-eServices/img/tp4/archi.png)](https://nodesource.com/products/nsolid)
  
- Product Service : Main service, which offers a REST API to list the  products.
- Config Service :  It's the a Configuration service, where we centralize all the configuration files of the other microservices in a single place.
- Proxy Service : A gateway that handles the routing of a request to one of the instances of a service, automatically manage the load distribution.
- Discovery Service: Service for registering instances of services for discovery by other services.

# Building Docker images 

There are 2 ways of building images in Docker with spring-boot: 
1/ Setup up manually the Dockerfile
2/ Using Maven Plugin
### Using DockerFile
 
   We create a new file named Dockerfile in the root project of the service.
   
  ##### Example of DockerFile for config-service :
  &nbsp;    
   ```sh
   
FROM openjdk:8-jdk-alpine
VOLUME /tmp
EXPOSE 8888
ADD /target/config-service-0.0.1-SNAPSHOT.jar //
CMD	java -Dfile.encoding=UTF-8 -Djava.security.egd=file:/dev/./urandom -jar /config-service-0.0.1-SNAPSHOT.jar

   ```
   
  Build the image 
```sh
     docker build -f Dockerfile -t config-service .
```
### Using Maven Plugin
  The plugin that we used for building the images is "docker-maven-plugin" from spotify 
#### Installation

We add to pom.xml for each project this plugin description bellow where we also did the setup of an execution rule to  build the image  after "mvn package" in the project.


```sh
	<plugin>
				<groupId>com.spotify</groupId>
				<artifactId>docker-maven-plugin</artifactId>
				<configuration>
					<imageName>${project.artifactId}</imageName>
					<baseImage>java:8</baseImage>
					<entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
					<!-- copy the service's jar file from target into the root directory of the image -->
					<resources>
						<resource>
							<targetPath>/</targetPath>
							<directory>${project.build.directory}</directory>
							<include>${project.build.finalName}.jar</include>
						</resource>
					</resources>

				</configuration>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>build</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
```
 

# Running the containers
  in order to create the containers from the images, we can use  also 2 differents ways:
  1 - By launching separetly each container and link them together from CMD.
  2 - By Setting up a docker-compose file.
  
#### Manually
 - We verify the names of the diffents images created from the last step 
```sh
  docker images
```
![N|Solid](https://img15.hostingpics.net/pics/644218Capturedecran20171205a21527AM.png)

- First we launch the config-container
 ```sh
docker run -it -p 8888:8888 \                      
      config-service \              
      --spring.cloud.config.server.git.uri=https://github.com/Medbeji/spring-cloud-config \
 ```
  Note: We've pushed to config-files to a github repository in ordre to be accessible from the container
- We get the ip-address of config-container by using this command
```sh
CONFIG_CONTAINER_ID=`docker ps -a | grep config-service | cut -d " " -f1`
CONFIG_IP_ADDRESS=`docker inspect -f "{{ .NetworkSettings.IPAddress }}" $CONFIG_CONTAINER_ID`
```
    
NB:  We use that IP-ADDRESS to let other services fetch config files from this ip address.
 - Launch discovery-service 
```sh
docker run -it -p 8761:8761 discovery-service \ -e --spring.cloud.config.uri=http://$CONFIG_IP_ADDRESS:8888
```
 - Launch proxy-service
```sh
docker run -it -p 9999:9999 proxy-service
```
 - Launch instance of product-service 
```sh
docker run -it -p 8080:8080 product-service -e --spring.cloud.config.uri=http://$CONFIG_IP_ADDRESS:8888
```

## IMPORTANT
in order prevent to prevent errors while opening the different containers simultaneously, we've added to each service a retry dependency from spring in pom.xml.
```sh
<dependency>
			<groupId>org.springframework.retry</groupId>
			<artifactId>spring-retry</artifactId>
			<version>${spring-retry.version}</version>
</dependency>
```
and then add a retry rule in bootstrap.yml of each service that's need to wait for the config-service to be lanched first.

```sh
spring:
  cloud:
    config:
      failFast: true
      retry:
        initialInterval: 3000
        multiplier: 1.3
        maxInterval: 5000
        maxAttempts: 20
```
#### Using docker-compose

we've setup the docker-compose.yml in order to establish the linking automatically inside the Docker network.
```sh
version: '2.0'
services:
 config-service:
    image: config-service
    ports:
        - "8888:8888"
    networks:
        - my-network
 
 discovery-service:
    image: discovery-service
    ports: 
        - "8761:8761"
    depends_on: 
        - config-service
    networks:
        - my-network

 product-service:
    image: product-service
    ports: 
        - "8080:8080"
    depends_on: 
        - config-service
    networks:
        - my-network
proxy-service:
    image: proxy-service
    ports:
        - "8888:8888"
    depends_on:
        - config-service
    networks:
        - my-network
networks:
    my-network:
       driver: bridge
```
and then 
```sh
 docker-compose up
```
#### Result :

  ![N|Solid](https://img15.hostingpics.net/pics/191981Capturedecran20171205a23928AM.png)




