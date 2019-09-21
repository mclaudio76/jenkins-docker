# Jenkins short tutorial

This tutorial shows how to use Jenkins, deployed as docker container on a docker host, to build a Spring Boot project and how to build a docker image of the Spring project.

This tutorial is far to be optimal, please consider it just as a bunch of notes written by a docker and jenkins novice; anyway, it contains some useful hints to get the work done.

## What is needed (and what you're going to install)
To complete this tutorial, you need to have a recent release of docker installed on your pc / workstation. Personally I tested the steps described here on a Ubuntu 18.04 distro, with docker 18.9 installed; when I'll have enough spare-time, I'll try to replicate the same steps another host S.O.

In this tutorial:
- we are going to build a custom docker image of Jenkins, and provide it with necessary tools to build a docker image after the main build stage (i.e, building the Spring Boot application);
- after that, we'll create a Docker-compose file to start our custom Jenkins image;
- we'll configure Jenkins to download and install latest stable version of Maven;
- we'll configure Jenkins to download and install a recent JDK release, to be used to build the project;
- we'll write a simple Jenkins pipeline to 
    - grab Spring boot project' code from GitHub;
    - compile it via Maven;
    - build a docker image of the Spring boot project (without publishing to Docker Hub repo)

## Building custom Jenkins docker image.

Since we need to build docker images directly in Jenkins - which is in turn deployed as a docker container in the docker host - I realized after several attempts that the simplest way to achieve this is to install docker-client inside Jenkins image. As far as I understood, Jenkins images is a Debian distro provided with OpenJDK 1.8; so , we can install on it a docker client taking care that the release of the docker client we're going to install is compatible with the underlying docker daemon on the host machine.
Also, after having built the docker image, we'll use a compose file to map  Jenkins folders to a persistent volume on the docker host, so that installed plugins, configurations and so on we'll be preserved after container restart. 

### Step 1. Build docker image

Custom Jenkins docker imager we want to build will use official Jenkins image from Docker Hub. We need to add to the base image the DockerCLI package (and related dependencies). Base image is debian-based, so we can simply use apt-get to install required packages. Here following the Dockerfile I used:

```
FROM jenkins/jenkins:lts
USER root
RUN apt-get update -qq \
    && apt-get install -qqy apt-transport-https ca-certificates curl gnupg2 software-properties-common
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
RUN add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
RUN apt-get update  -qq
RUN apt-get install docker-ce-cli=5:18.09.0~3-0~debian-stretch  containerd.io  -y
```
To build the actual image, just run the following command:

```
docker build  --tag custom-jenkins .
```

### Step 2. Prepare to run customized jenkins image via docker-compose

Next step is about setting up a docker-compose.yml file to start new built container. I prefer to user docker compose instead of passing command-line parameters whenever i want to start the container.
Here following docker-compose.yml file:

```
version: '2'
services:
  jenkins:
    user: root
    image: 'custom-jenkins'
    container_name: custjenkins
    restart: always
    ports:
      - '7080:8080'
      - '8443:8443'
      - '50000:50000'
    volumes:
      - /opt/docker/volumes/jenkins/home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock  
volumes:
  jenkins_data:
    driver: local
```




