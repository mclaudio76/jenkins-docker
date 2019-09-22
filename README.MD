# Jenkins short tutorial

This tutorial shows how to use Jenkins, deployed as docker container on a docker host, to build a Spring Boot project and how to build a docker image of the Spring project.

This tutorial is far to be optimal, please consider it just as a bunch of notes written by a docker and jenkins novice; anyway, it contains some useful hints to get the work done.

## What is needed (and what you're going to install)
To complete this tutorial, you need to have a recent release of docker installed on your pc / workstation. Personally I tested the steps described here on a Ubuntu 18.04 distro, with docker 18.9 installed; when I'll have enough spare-time, I'll try to replicate the same steps on another host S.O.

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
I've made some mess while setting up JDK during my experiments. The most important thing I've leart is that you should only specify 'Name' attribute and the URL to download JDK from, leaving 'Subdirectory of extracted archive' and 'Label' inputs blank.

### Where do downloaded tools go ?
Short answer: it depends on the tool. For example, JDK is installed in the following folder (within the container):
```/var/jenkins_home/tools/hudson.model.JDK/<JDK-NAME>```
So, in this case, we'll get our OpenJDK 13 installed within ```/var/jenkins_home/tools/hudson.model.JDK/JDK13``` folder. 

Similarly, Jenkins will store Maven packages under ```/var/jenkins_home/tools/hudson.tasks.Maven_MavenInstallation/``` folder.
I presume that different runtimes / tools (i.e, Node) will be extracted elsewhere.

I found useful to know that for configuring the pipeline used to build and dockerize the Spring project.

## Running a simple pipeline.

Here is the pipeline I used to buid my simple project, let's discuss some relevant stuff.
The pipeline itself it's really simple, and I'm sure that any expert Jenkins user will easily find a better way to accomplish the whole stuff. Anyway, in the 'learn by doing' approach, I think it's better to explore as much as one can.
So, I created different 'stages' for the pipeline (a single stage would have been enough, tough).

-   The 'Setup Enviroment' stage defines via ```tool``` directive the path where Jenkins will find named tools. In this case, we 
    need JDK13 and Maven path.
    
-   The  'Get Source' stage uses git (which is provided out of the box within Jenkins installation) to grab all source code from 
    a very simple Spring Boot project I've published on GitHub. The only relevant thing to note here is that project git folder    
    must contain a valid Dockerfile that Jenkins will use in the last stage to build a container.
    
-   The 'Build Java code' section is the tricky part. I had hard times figuring out how to let Maven know which Java runtime to 
    use, because by default Maven runs with the very first java env it encounters in the PATH, and this is definitely not what
    we want. After reading on web here and there, I found a working (and very likely perfectible) solution.
    The ```withEnv``` statement is used to invoke, via shell, Maven after having set:
        -  the JDK to use (via JAVA_HOME).
        -  the whole PATH, taking care to put JAVA_HOME before any other path (otherwise, Maven will pick up the first one that 
           encounters)
    The nasty thing I've done here is that I needed - sorry, no other way until now found - to specify that JAVA_HOME is given by
    the path of JDK13 (automatically resolved via ```tool``` instruction) plus the folder ```jdk-13+33``` that's really specific 
    for the actual JDK. If I wanted to change the JDK implementation - for example, using Amazon Corretto JDK - I would needed to 
    change not only the symbolic jdk tool path, but also the JAVA_HOME manually.

```
node {
   def mvnHome
   def jdk
   stage('Setup Environment') {
      mvnHome = tool 'M3'
      jdk     = tool 'JDK13'
   }
   stage('Get Source') { // for display purposes
      git 'https://github.com/mclaudio76/dockerized.git'
   }
   stage('Build Java') {
      
      withEnv(["JAVA_HOME=$jdk/jdk-13+33", "PATH=${env.JAVA_HOME}/bin:${env.PATH}:$mvnHome/bin"]) {
         if (isUnix()) {
            sh 'mvn -Dmaven.test.failure.ignore clean package'
         } else {
            bat(/"%MVN_HOME%\bin\mvn" -Dmaven.test.failure.ignore clean package/)
         }
      }
   }
   stage('Build docker image') {
        /* This builds the actual image; synonymous to
         * docker build on the command line */
        app = docker.build("spring-automatic")
    }
}

```
## Conclusions.

The notes presented here show how to setup jenkins, using a specifing JDK, and how to write a simple pipeline to build a Java project ad dockerize it. Of course this is NOT suited for production-ready continuos delivery scenarios.






