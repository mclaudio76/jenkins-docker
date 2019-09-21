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
    - grab Spring boot project' code from GitHub
    - compile it via maven
    - build a docker image of the Spring boot project
  


