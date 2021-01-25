Oracle WebCenter Content Container Image
========================================

## Contents

### 1. [Introduction](#1-introduction-1)
### 2. [Hardware and Software Requirements](#2-hardware-and-software-requirements-1)
### 3. [Pre-requisites](#3-pre-requisites-1)
### 4. [Building Oracle WebCenter Content Image](#4-building-oracle-webcenter-content-image-1)
### 5. [Running Oracle WebCenter Content Container](#5-running-oracle-webcenter-content-container-1)
### 6. [License](#6-license-1)
### 7. [Copyright](#7-copyright-1)
# 1. Introduction
This project offers scripts to build an Oracle WebCenter Content container image based on 12.2.1.4.0 release. Use this documentation to facilitate installation, configuration, and environment setup for DevOps users. For more information about Oracle WebCenter Content, see the [Oracle WebCenter Content 12.2.1.4.0 Online Documentation](https://docs.oracle.com/en/middleware/webcenter/content/12.2.1.4/index.html).

This repository includes quick start Dockerfile for WebCenter Content 12.2.1.4.0 based on Oracle Linux 7, Oracle JRE 8 (Server) and Oracle WebLogic Infrastructure 12.2.1.4.0.

The containers will be connected using a Docker User Defined network.
More information on Docker and its installation in Oracle Linux can be found here: [Oracle Container Runtime for Docker User Guide](https://docs.oracle.com/en/operating-systems/oracle-linux/docker/)

# 2. Hardware and Software Requirements
Oracle WebCenter Portal has been tested and is known to run on the following hardware and software:

## 2.1. Hardware Requirements

| Hardware  | Size  |
| :-------: | :---: |
| RAM       | 16GB  |
| Disk Space| 200GB+|

## 2.2. Software Requirements

|       | Version                        | Command to verify version |
| :---: | :----------------------------: | :-----------------------: |
| OS    | Oracle Linux 7.3 or higher     | more /etc/oracle-release  |
| Docker| Docker version 17.03 or higher | docker version           |

# 3. Pre-requisites

## 3.1. To confirm that userid is part of the Docker group, run the below command:
```
   $ sudo id -Gn <userid>
```
* Run `docker ps -a` command to confirm user is able to connect to Docker engine.
To add/modify user to be part of docker group
```
   $ sudo /sbin/usermod -a -G docker <userid>
```

## 3.2. Set the Proxy if required:

Set up the proxy for docker to connect to the outside world - to access external registries and build a Docker image, set up environment variables for proxy server something like this:
```
   export http_proxy=http://proxy.example.com:80 
   export https_proxy=http://proxy.example.com:80 
   export HTTP_PROXY=http://proxy.example.com:80 
   export HTTPS_PROXY=http://proxy.example.com:80 
   export NO_PROXY=localhost,.example.com 
```

## 3.3. Create a User Defined network

In this configuration creation of a user defined network will enable communication between the containers just using container names. User defined network option was preferred over the container linking option as the latter is now deprecated. For this setup we will use a user defined network using bridge driver.Create a user defined network using the bridge driver by executing the following command:

        $ docker network create -d bridge <some name>

Sample command ...

        $ docker network create -d WCContentNetwork

## 3.4. Mount a host directory as a data volume
You need to mount volumes, which are directories stored outside a container's file system, to store WebLogic domain files and any other configuration files. 

To mount a host directory (`$DATA_VOLUME`) as a data volume, execute the below command.

> The userid can be anything but it must belong to uid:guid as 1000:1000, which is same as 'oracle' user running in the container.

> This ensures 'oracle' user has access to shared volume.

```
sudo mkdir -p /<YOUR_HOST_DIRECTTORY_PATH>/wccontent
sudo chown 1000:1000 /<YOUR_HOST_DIRECTTORY_PATH>/wccontent
```

All container operations are performed as **'oracle'** user.

**Note**: If a user already exists with **'-u 1000 -g 1000'** then use the same user. Or modify any existing user to have uid-gid as **'-u 1000 -g 1000'**

## 3.5. Database
You need to have a running database container or a database running on any machine. 
The database connection details are required for creating WebCenter Content specific RCU schemas while configuring WebCenter Content domain. 

The Oracle Database image can be pulled from [Oracle Container Registry](https://container-registry.oracle.com) or build your own using the [Oracle Database Dockerfiles and scripts]((https://github.com/oracle/docker-images/tree/master/OracleDatabase)) in this repo.

## 3.6. Docker Security Configuration

For detailed instructions of security best practices, please refer to this [documenation](https://docs.oracle.com/en/operating-systems/oracle-linux/docker/docker-security.html#docker-security-components).

# 4. Building Oracle WebCenter Content Image

To build/use a WebCenter Content image you should have Oracle FMW Infrastrucure image.

## 4.1. Pulling Oracle FMWInfrastructure Install Image

Get Oracle FMWInfrastructure Image -

> 1. Sign in to Oracle Container Registry. Click the Sign in link which is on the top-right of the Web page.
> 2. Click Middleware and then click Continue for the fmw-infrastructure repository.
> 3. Click Accept to accept the license agreement.
> 4. Use following commands to pull Oracle Fusion Middleware infrastructure base image from repository :
Refer Documentation here: https://github.com/oracle/docker-images/tree/master/OracleFMWInfrastructure
```
docker login container-registry.oracle.com
docker pull container-registry.oracle.com/middleware/fmw-infrastructure:12.2.1.4
docker tag  container-registry.oracle.com/middleware/fmw-infrastructure:12.2.1.4 oracle/fmw-infrastructure:12.2.1.4.0
docker rmi container-registry.oracle.com/middleware/fmw-infrastructure:12.2.1.4

```
Alternatively to build this image yourself, please refer this [link](https://github.com/oracle/docker-images/blob/master/OracleFMWInfrastructure/README.md).

## 4.2. Building Docker Image for WebCenter Content

You have to download the binary for WebCenter Content shiphome and put it in place. The binaries can be downloaded from the [Oracle Software Delivery Cloud](https://edelivery.oracle.com/). Search for "Oracle WebCenter Content" and download the version which is required.
Extract the downloaded zip files and copy `fmw_12.2.1.4.0_wccontent.jar` file under `dockerfiles/12.2.1.4` .
Checksum of shiphome binary needs to be mentioned in this [file](12.2.1.4/Checksum). Set the Proxies in the environment before building the image as required, go to directory located at OracleWebCenterContent/dockerfiles/ and run these commands -

```
#To generate checksum
md5sum fmw_12.2.1.4.0_wccontent.jar

#To build image
sh buildDockerImage.sh -v 12.2.1.4.0

#Verify you now have the image
docker images
```

# 5. Running Oracle WebCenter Content Container
 
To run the Oracle WebCenter Content container, you need to create:
* Container to manage the Admin Server.
* Container to manage the Managed Servers.

## 5.1. Creating containers for WebCenter Content Server

### 5.1.1. Update the environment file 

Create an environment file `webcenter.env.list` file, to define the parameters.

Update the parameters inside `webcenter.env.list` as per your local setup.
Please note: all parameters mentioned below are manadatory and shouldn't be omitted or, left blank. 

```
#Database Configuration
DB_DROP_AND_CREATE=<true or false>
DB_CONNECTION_STRING=<Hostname/ContainerName>:<Database Port>:<Database Service>
DB_RCUPREFIX=<RCU Prefix>
DB_PASSWORD=<Database Password>
DB_SCHEMA_PASSWORD=<Schema Password>

#configure container
ADMIN_SERVER_CONTAINER_NAME=<Admin Server Container Name>
ADMIN_PORT=<Admin Server Port>
ADMIN_PASSWORD=<Admin Server Password>
ADMIN_USERNAME=<Admin Server User Name>

DOMAIN_NAME=<domain directory-name>
UCM_PORT=<port to be used for UCM managed server on container>
IBR_PORT=<port to be used for IBR managed server on container>
UCM_HOST_PORT=<host port to access UCM managed server - this is the port value to be used for -p option (left of the colon) sec. ### 5.1.3>
IBR_HOST_PORT=<host port to access IBR managed server - this is the port value to be used for -p option (left of the colon) sec. ### 5.1.3>
UCM_INTRADOC_PORT=<UCM intradoc port on container>
IBR_INTRADOC_PORT=<IBR intradoc port on container>

#component
component=IPM,Capture,ADFUI

#HOSTNAME
HOSTNAME=<provide your host name>

#Keep Container alive
KEEP_CONTAINER_ALIVE=true
```

### 5.1.2. Admin Container (WCCAdminContainer)
#### A. Creating and Running Admin Container

Run the following command to create the Admin Server container:

```
docker run -it --name WCCAdminContainer --network=WCContentNET -p <Any Free Port>:<ADMIN_PORT> -v $DATA_VOLUME:/u01/oracle/user_projects --env-file <PATH_TO_ENV_FILE>/webcenter.env.list oracle/wccontent:12.2.1.4.0

# A sample command will look ike this -

docker run -it --name WCCAdminContainer --network=WCContentNET -p 7001:7001 -v $DATA_VOLUME:/u01/oracle/user_projects --env-file <PATH_TO_ENV_FILE>/webcenter.env.list oracle/wccontent:12.2.1.4.0
```
**Note:** The above command deletes any previous RCU with the same prefix if **DB_DROP_AND_CREATE=true**

The `docker run` command creates the container as well as starts the Admin Server in sequence given below:

* Node Manager
* Admin Server

When the command is run for the first time, we need to create the domain and configure the content managed servers, so following are done in sequence:

* Loading WebCenter Content schemas into the database
* Creating WebCenter Content domain
* Extending WebCenter Content domain for associated products (e.g. IPM, ADFUI etc.) - based on **component** env varibale. 
* Configuring Node Manager
* Starting Node Manager
* Starting Admin Server

#### B. Stopping Admin Container
```
docker stop container WCCAdminContainer
```

#### C. Starting Admin Container
```
docker start container -i WCCAdminContainer
```

### 5.1.3. Managed Server Container (WCContentContainer)

#### A. Creating and Running WCContent Container
Run the following command to create the WCContent Managed Server container:

```
docker run -it --name WCContentContainer --network=WCContentNET -p <UCM_HOST_PORT>:<UCM_PORT> -p <IBR_HOST_PORT>:<IBR_PORT> -p <UCM_INTRADOC_PORT>:<UCM_INTRADOC_PORT> -p <IBR_INTRADOC_PORT>:<IBR_INTRADOC_PORT> --volumes-from WCCAdminContainer --env-file <PATH_TO_ENV_FILE>/webcenter.env.list oracle/wccontent:12.2.1.4.0 configureOrStartWebCenterContent.sh

# A sample command will look like this -

docker run -it --name WCContentContainer --network=WCContentNET -p 16200:16200 -p 16250:16250 -p 4444:4444 -p 5555:5555 --volumes-from WCCAdminContainer --env-file <PATH_TO_ENV_FILE>/webcenter.env.list oracle/wccontent:12.2.1.4.0 configureOrStartWebCenterContent.sh
```
The docker run command creates the container as well as starts the WebCenter Content managed servers. 

Note: 1. If manged servers need to be accessed through host ports different from container ports, then intended host port values needs to be supplied as part of -p option of the `docker run`
command mentoned above (for ex. -p 16201:16200 and -p 16251:16250). The same port value needs to be updated in the `webcenter.env.list` as `UCM_HOST_PORT` and `IBR_HOST_PORT`.
If managed servers are going to be accessed via same host port number as the container port, then `UCM_PORT` and `UCM_HOST_PORT` values (and `IBR_PORT` and `IBR_HOST_PORT`) should be same in the `webcenter.env.list`.  
      2. Intradoc ports are for internal server communications and not meant for browser access. While, intradoc ports on container are configurable (like other parametres like admin credentials, admin port, domain-name, manged server container ports) 
through `webcenter.env.list`, publishing these to different host ports is not supported. This essentially means one can provide `-p 7777:7777` instead of `-p 4444:4444`, but `-p 6666:7777` is not supported.

#### B. Stopping WCContent Container
```
docker container stop WCContentContainer
```

#### C. Starting WCContent Container
```
docker container start -i WCContentContainer
```

#### D. Getting Shell in WCContent Container
```
docker exec -it WCContentContainer /bin/bash
```
Please wait till all the above all `docker run` commands are successfully completed, before you can start Admin and Managed Servers with WebLogic admin credentials.

WebLogic Admin Server
http://hostname:7001/console/

UCM Server
http://hostname:16200/cs/

IBR Server
http://hostname:16250/ibr/

# 6. License

To download and run Oracle Fusion Middleware, regardless whether inside or outside a container, you must download the binaries from the Oracle website and accept the license indicated at that page.
All scripts and files hosted in this project are, unless otherwise noted, released under UPL 1.0 license.

# 7. Copyright
Copyright (c) 2021, Oracle and/or its affiliates.