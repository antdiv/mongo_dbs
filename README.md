# MONGO_DBS - EHUB exercise

The exercise requires the creation of a production infrastructure for the mongo_dbs.js application.

The application uses a MongoDB database and needs the two packages mongodb and express. It also needs the two environment variables :
-   MONGO_URI="mongodb://username:password@hostname:port""
-   HTTP_PORT="xxxx";

Once started, it connects to the MongoDB database and displays the list of databases present.
Calling the application with http://hostname_web:port/STOP simulates a crash.
Objective of the exercise is:

-   Automate the creation of the infrastructure
-   Automate the application setup.
-   Implement a method to automatically restart the service in case of a crash.
-   Automate the database backup;

The ouptput of the exercise must be:
-   the code used to produce the automation and configuration
-   a file (README.md) with a description that allows you to understand the code and execute it
-   a drawing of the architecture created with Draw.io (https://app.diagrams.net/)

It is possible to use any public cloud platform and any programming/automation language.
The time for the execution of the exercise is one week starting from the receipt by email.


## _Infrastructure architecture_

The MONGO_DBS architecture is based on a mongodb server that is used by a nodejs application.
In order to support this application we have two different docker containers, one for database layer and another one for the appication.

For more details please see the diagram in "architecture.svg"
![architecture](architecture.svg)

In order to setup this kind of infrastructure we use docker compose, defining the a multi-containers application

### Prerequisites

You need to have Docker installed.
To check if you have Docker installed, open your Command Line and run the following code
```sh
docker -v
```
You should get a version number if you have Docker installed.

### Technology stack

Mongo_dbs uses the following open source projects to work properly:

- [node.js] - evented I/O for the backend
- [Express] - fast node.js network app framework [@tjholowaychuk]
- [MongoDB driver] - MongoDB Node Driver
- [MONGO DB] - source-available cross-platform document-oriented database program

## Database Setup

For the database layer we need a mongodb image, so for this purpose it's possible to use the official mongodb docker image. 
In our docker-compose.yml we use it and set up the ports and default environment variables:

```sh
 mongodb:
    image: mongo
    container_name: mongodb
    restart: unless-stopped
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=example
```

## Application Setup

For application service we need to set up an ad-hoc docker image in order to start the nodejs code, mongo_dbs.js, located in /src.

- in the root of the project we have a package.json file for dependencies definition related to "express framework" and "mongodb driver" as in the following snippet:
  
    ```sh
    {
      "name": "mongo_dbs",
    ....
      "dependencies": {
        "express": "^4.17.2",
        "mongodb": "^4.2.2"
      }
    }
    ```
- an .env file with the environemnt variables required: HTTP_PORT and MONGO_URI 
- 
  ```sh
    HTTP_PORT=3000
    MONGO_URI=mongodb://root:example@mongodb:27017/
  }
  ```
- define a Dockerfile that enables us to create a docker, starting from node:12-alpine and installing the required dependencies as defined in package.json
  ```sh
    FROM node:12-alpine
    ...
    COPY package*.json ./
    ...
    RUN npm install
  ``` 
Then we define a service named "ehub-express" in the docker-compose.yml.
We have a "build" step that uses all the code in the root folder in order to create the docker image named "mongo_dbs_be" and container named "mongo_dbs_be"

```sh
services:
  ehub-express:
    build:
      context: .
      dockerfile: Dockerfile
    image: mongo_dbs_be
    container_name: mongo_dbs_be
```

The application properties are setted using in .env file and the property HTTP_PORT is used also in order to configure container's ports
```sh
    env_file: .env
    ports:
      - "${HTTP_PORT}:${HTTP_PORT}"
```
The "command" will execute the core of the application and the "depends_on" estabilish that this services needs to be started after "mongodb" container

```sh
    depends_on:
      - mongodb
    command: node ./src/mongo_dbs.js
```

The restart config are used in order to have the container restart when the STOP operation is executed

```sh
   restart: unless-stopped
```
## Run Multi-container application

Now we have a docker-compose file that enable us to startup the infrastucture with application service and database with the following command

```sh
docker-compose up
```

After some seconds you will se the creation of the images and containers started:

```sh
[+] Running 2/2
 - Container mongodb       Created                                                                                                                                                                                                         
 - Container mongo_dbs_be  Recreated                                                                                                                                                                                                       
Attaching to mongo_dbs_be, mongodb
mongodb       | {"t":{"$date":"2021-12-22T08:21:35.839+00:00"},"s":"I",  "c":"NETWORK",  "id":4915701, "ctx":"-","msg":"Initialized wire specification","attr":{"spec":{"incomingExternalClient":{"minWireVersion":0,"mlient":{"minWireVersion":0,"maxWireVersion":13},"outgoing":{"minWireVersion":0,"maxWireVersion":13},"isInternalClient":true}}}
...
mongo_dbs_be  | Server listening on port 3000
```
The above log informations confirm that the containers are started and available 


### Test Application 

Open a web browser and run the following URL:
```sh
localhost:3000
```
where "3000" is what we have configured as HTTP_PORT and then you will see the application output:

```sh
Hello MongoDB server!
Databases:
 - admin
 - config
 - local
Bye!
```

### Application parameters 
The environment variables HTTP_PORT and MONGO_URI give us the possibility to manage application parameterization required by the mongo_dbs.js.

### Application restart 
The following property in docker_compose "restart: always" allow the container to be restarted when the service goes down invoking the following service http://hostname_web:port/STOP


### Database backup 
In order to have an automated backup we define an additional docker compose file that will manage this kind of operation.
So please create a docker-compose-backup.yml with the following content

```sh
version: '3.9'
services:
  mongodb_backup:
    image: mongo
    restart: always
    ports:
      - 27017:27017
    volumes:
      - '/tmp/'
    command:
      mongodump --uri mongodb://root:example@mongodb:27017/ --archive=/tmp/dump.archive
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: example
      ME_CONFIG_MONGODB_URL: mongodb://root:example@mongodb:27017/
```

Then run the command:
```sh
docker-compose -f docker-compose-backup.yml run mongodb_backup
```
This command can be executed with a cron expression at defined time interval

The same functionality can be implemented with the following command

```sh
docker container exec -it mongodb bash
mongodump --uri mongodb://root:example@mongodb:27017/ --archive > /tmp/mongo.dump
```

## Docker compose to Azure

###Create an Azure context for Docker

To use Docker commands to run containers in Azure Container Instances, first log into Azure and then create an azure context:
```sh
docker login azure
docker context create aci myacicontext
```
Create an ACI context by running "docker context create aci".

### Deploy application to Azure Container Instances

Next, change to the ACI context. Subsequent Docker commands run in this context.
```sh
docker context use myacicontext
```
Run docker compose up to start the application in Azure Container Instances.
```sh
docker compose up
```
Then we can check the successful infrastructure startup