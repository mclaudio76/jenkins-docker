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
This is a pretty normal docker-compose file. The relevant parts are the following:
- just for sake of avoiding troubles with permission, I required to run the image under root user. 
  This is generally a bad idea;  it's much more safer to create a specific user - with limited privileges - 
  and adding it to docker group (so that it can connect to underlying docker daemon;
- I assigned a container name to the image to be run;
- via ```volumes:``` block I mapped host folders to the corrispondent container folders; I want Jenkins plugins,
  configurations and so on to be persistent after subsequent modifications to the container.
- very important: the line 
  ```/var/run/docker.sock:/var/run/docker.sock```
  is needed to allow the docker client (inside jenkins) to connecto to the running docker daemon on the host machine.
  Otherwise, we couldn't write pipelines able to build docker images inside Jenkins.

So, let's start the container by issuing:

```
docker-compose up -d # -d runs the container in daemon mode.
```
To verify that Jenkins will be able to connect to docker daemon, let's log into the container by issuing:

```
docker exec -it custjenkins  /bin/bash
```
and at the shell prompt , we need to verify that ```docker -v``` is executed without errors.

## Setting up Jenkins to use Maven and a specific JDK.

By default, at the time of writing this document Jenkins docker image is shipped with JDK 8.0. What if we want to use a more recent Java version ? Luckily, it's pretty easy to setup a specific JDK in Jenkins - indeed, you can use a number of different JDKs to build up your projects. Let's see how to install a JDK and how to use it in a pipeline.

### Installing maven.

Installing Maven it's quite simple. Go to Global Tool Configuration Section in Jenkins console and find Maven setup section.
It's enough to provide a symbolic name for Maven installation and tell Jenkins which Maven version  to download from Apache's website.
![Image of Jenkins Maven Section](https://github.com/mclaudio76/jenkins-docker/blob/master/mavensetup.png)
The very first time a pipeline required Maven will be run, Jenkins will download Maven and will run it. We can also have multiple installations of Maven, but for this simple tutorial a recent Maven installation will suffice.

### Installing JDK.

Installing a JDK is slighty tricky, at least for a novice like me. Let's find JDK section in Global Tool Configuration page and let's tell Jenkins we want to add a new JDK. We have to check "Install automatically" option and provide a few information:
![Image of JDK setup](https://github.com/mclaudio76/jenkins-docker/blob/master/jdksetup.png)
I've made some mess here during my experiments. The most important thing I've leart is that you should only specify 'Name' attribute and the URL to download JDK from, leaving 'Subdirectory of extracted archive' and 'Label' inputs blank.



