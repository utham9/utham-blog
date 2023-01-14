---
title: 'Selenium Grid with Docker'
date: '2022-01-17'
tags: ['github', 'guide', 'docker', 'selenium']
draft: false
summary: 'A quick guide to convert your existing selenium automation script\ framework to work with selenium grid, docker images.'
---

# Selenium Grid with Docker

A quick guide to convert your existing selenium automation script\ framework to work with selenium grid, docker images. There are three ways this can be done

1. Use remote web driver execution (running locally) and connect with docker
2. Complete execution on docker images
3. Using CICD tools like Jenkins 


## AWS EC2 Instance

<aside>
💡 **Note**: AWS EC2 Instance to host the docker is optional, in reality we just need a system where the docker is running.

</aside>

AWS EC2 Instance is used to host the docker and dynamically increase the instance capabilities as and when required. 

1. Create an EC2 Instance. Follow the steps [here](https://docs.aws.amazon.com/efs/latest/ug/gs-step-one-create-ec2-resources.html)
    1. Linux based instance is recommended. And this guide will use a Linux instance
2. SSH to your EC2 instance. Follow the steps [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html) 

## Docker Config

### Docker Installation

1. Install docker with the below command

```bash
sudo yum install docker-ce docker-ce-cli containerd.io
```

<aside>
💡 For installing docker on other platforms, refer [here](https://docs.docker.com/engine/install/)

</aside>

1. Start the docker

```bash
#Start docker engine
sudo systemctl start docker

#Stop docker engine
sudo systemctl stop docker
```

1. Create the below docker-compose file in your project. Typically at project root

<aside>
💡 file-name : selenium-grid.yml

</aside>

Below docker compose file creates 3 docker containers in the order.

1. Container ‘selenium-hub’ : Launches selenium grid in “hub” mode , and waits for the nodes to be connected to the port “4444”
2. Container ‘chrome’ : Launches selenium grid in “node” mode and the image has chrome pre-installed and connects HUB at “selenium-hub:4444”
3. Container ‘firefox’ : Launches selenium grid in “node” mode and the image has firefox pre-installed and connects HUB at “selenium-hub:4444”

More nodes can be added, as required

```yaml
version: "3.3"

services:
  selenium-hub:
    image: selenium/hub:3.141.59-yttrium
    container_name: selenium-hub
    ports:
      - "4444:4444"
    expose:
      - 4444
  chrome:
    image: selenium/node-chrome-debug:3.141.59-yttrium
    container_name: chrome-node
    ports:
      - "5900:5900"
    volumes:
      - /dev/shm:/dev/shm
    depends_on:
      - selenium-hub
    environment:
      - HUB_HOST=selenium-hub
      - HUB_PORT=4444
      - NODE_MAX_SESSION=4
      - NODE_MAX_INSTANCES=4
      - JAVA_OPTS=-Dwebdriver.chrome.whitelistedIps=0.0.0.0
  firefox:
    image: selenium/node-firefox:3.141.59-yttrium
    container_name: firefox
    deploy:
      replicas: 8
    volumes:
      - /dev/shm:/dev/shm
    depends_on:
      - selenium-hub
    environment:
      - HUB_HOST=selenium-hub
      - HUB_PORT=4444
```

Run the below command to start the docker container, and configure the selenium-grid

```bash
docker-compose -d selenium-grid.yml up
```

To stop the selenium gird

```bash
docker-compose -d selenium-grid.yml down
```

# Putting it all together

## Running the test locally

Test can be run, by connecting the remote web driver

```java
FirefoxOptions firefoxOptions = new FirefoxOptions();
WebDriver driver = new RemoteWebDriver(new URL("http://selenium-hub:44444/wd/hub"), firefoxOptions);
```

<aside>
💡 Ensure appropriate network firewall settings are configured to access the selenium-hub

</aside>

## Running the test from another docker container

To run the test from another docker container, we need a custom docker image. This can be defined in a DockerFile

```docker
FROM maven:3.6.0-jdk-11-slim AS build
COPY src /home/app/src
COPY pom.xml /home/app
EXPOSE 4444:4444
RUN mvn -f /home/app/pom.xml clean test -Dtest=StandaloneChrome
```

Run the test, with the below command

```bash
docker build -f /path/to/a/Dockerfile .
```

<aside>
💡 DockerFile should be run after the selenium-hub is up and running

</aside>

## Putting it all together, in a Jenkins Pipeline script

You can, automate all the above steps with a Jenkins pipeline script. Below is an example

```groovy
#!groovy

def gitUrl = "https://github.com/utham9/easy-peasy.git"
def branchName = "master"

pipeline {

    agent any

    options {
        timeout(time: 12, unit: 'HOURS')
        buildDiscarder(
                logRotator(
                        numToKeepStr: '5',
                        daysToKeepStr: '7',
                        artifactDaysToKeepStr: '7',
                        artifactNumToKeepStr: '5'
                )
        )
    }

    stages {
        stage('clone') {
            steps {
                git branch: branchName,
                        credentialsId: '',
                        url: gitUrl
            }
        }

        stage('docker') {
            steps {
                script {
                    sh "sudo systemctl start docker"
                    sh "docker-compose -d selenium-grid.yml up"
                }
            }
        }

        stage('test') {
            steps {
                script {
                    sh "docker build Dockerfile ."
                }
            }
        }

        stage('wind-down') {
            steps {
                script {
                    sh "docker-compose -d selenium-grid.yml down"
                    sh "sudo systemctl stop docker"
                }
            }
        }
    }

    post {
        always {
            echo "Run on success"
        }
        success {
            echo "Run on success"
        }
        failure {
            echo "Run on failure"
        }
    }

}
```