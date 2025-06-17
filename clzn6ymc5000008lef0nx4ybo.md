---
title: "Set up a CI/CD server and Configure for use with Private Github Repositories"
seoTitle: "Set up a CI/CD server and Configure for use with Private Github Reposi"
seoDescription: "An article going into setting up a Jenkins instance and configuring it to work with Private Github Repositories"
datePublished: Fri Aug 09 2024 21:00:42 GMT+0000 (Coordinated Universal Time)
cuid: clzn6ymc5000008lef0nx4ybo
slug: set-up-a-cicd-server-and-configure-for-use-with-private-github-repositories
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/VjWi56AWQ9k/upload/9a5f760c03d8f8c1b9cf7f245345d7a6.jpeg
tags: python, django, jenkins, pipeline, ci-cd

---

Recently I have been exploring more Devops subjects and have been interested in setting up a CI/CD pipeline for a private project I have been working on.

This article covers the basics of setting up a private Jenkins instance (customised with Python) using Docker, setting up a Github PAT, registering that credential within Jenkins and configuring Jenkins to read a private repository.

## Setting up Jenkins on Docker

I will use Docker Compose to set up the Jenkins environment.

I did these steps directly from the host that was running my 'prod' Docker instance, in my case its a Raspberry Pi 5 running Ubuntu.

I want to customise the Jenkins base image to include bash and python as my project is Django based. In order to achieve this I first create a folder on my Raspberry Pi called Jenkins and change directory into that folder.

```bash
mkdir jenkins
cd jenkins
```

Now I want to create a Dockerfile, I achieve this by running

```bash
vi Dockerfile
```

The contents of the that I created are below, as you can see I am creating this from the jenkins/jenkins:lts base image and then I am installing Python3 and associated tools.

```dockerfile
FROM jenkins/jenkins:lts-jdk17

USER root

# Install Python 3 and related tools
RUN apt-get update && \
    apt-get install -y python3 python3-pip python3-venv bash && \
    apt-get clean

# Switch back to the Jenkins user
USER jenkins
```

Now create a file called docker-compose.yaml in a folder on your system. and populate it like the below:

```yaml
version: '3.8'
services:
  jenkins:
    build: . 
    user: "1000:1000"                  # UID:GID that exists on the host
    group_add: ["docker"]              # lets Jenkins run `docker build`
    restart: always
    ports:
      - 8090:8080
      - 50000:50000
    container_name: jenkins
    volumes:
      - /home/${myname}/jenkins_compose/jenkins_configuration:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
```

Security note: The Compose file above used to set privileged: true in an older revision of this article. This hands the Jenkins container full root-level control of the host. That’s convenient for personal experiments, but it is not recommended for production or shared hardware.

In serious environments:

* Run Jenkins as an unprivileged UID/GID that matches a real user on the host.
    

* Give that user access to Docker only by adding it to the docker group (or by using a remote build agent), rather than running the container in --privileged mode.
    

* Mount /var/run/docker.sock read-only if the jobs merely build images; use root-less buildkit or a dedicated DinD sidecar if you need full write access.
    

Worth noting in this instance, I am using the build field rather than specifying a pre-created image. This means that when I run Docker Compose it will run the dockerfile that is in the same directory to build the image before deploying the image to a running container.

Once this file is created you can run the command below to run it. I have added the --build argument to ensure the image is built each time rather than using a cached copy and also the -d arg to detach from the process

```bash
docker-compose up -d --build
```

You can then navigate to http://%your-docker-host%:8090 to start the setup on the Jenkins instance. For the purpose of this activity all the defaults are fine during the setup wizard. Don't forget to set a memorable username and password!

Once that is setup we are good to proceed onto the next step...

## Configuring a PAT in Github and Jenkins

Next we need to create a Personal Access Token that will allow us to view our private repositories. I have opted to use a classic token for this purpose as I want my private Jenkins instance to be to see all my private repositories as I likely will expand the usage of this Jenkins instance.

In order to do this you follow the following steps on Github.com

* Log into your Github account
    
* Click on your profile in the top right hand corner then click settings
    
* Along the left hand menu, click 'developer settings'
    
* Click "personal access token" drop down
    
* Select "Tokens (Classic)"
    
* Click "Generate New Token" and choose the classic option
    
* Give it a name and also ensure "repo" is checked
    
    * Set the expiry time to a value desired
        
* Press Generate Token
    
* On the next screen ensure you copy the token and paste it somewhere safe for now. A text editor window is great for this
    

Once you have the PAT it is time to configure the Credentials with your shiny new Jenkins instance

### Now for Jenkins config

To add this PAT to Jenkins navigate to your instance and login in

From Dashboard, click Manage Jenkins and then select 'credentials' from within in there click "System" &gt; "Global Credentials" &gt; "+ Add Credentials" the default option should be "Username and Password" and this is what we want.

* Keep Scope as default value
    
* username is your Github username
    
* Password is your PAT that you copied earlier
    
* ID can be anything to identify the credential
    
* Description can be filled appropriately
    

Now this is all setup for your first multibranch pipeline.

## Setting up a multibranch pipeline for a private repo

From Jenkins Dashboard select "+ New Item" when the page loads give it a name and select "Multibranch Pipeline" and then click "ok".

* Give the pipeline a descriptive name
    
* Optionally give it a fitting description
    
* Select "Add Source" and Choose "Github"
    
* In credentials, in the drop down select the credential you just created
    
* Get your Repositories HTTPS clone link and paste it in
    
* As strategy I selected "Exclude branches that are filed as PRs"
    
* Under Build Configuration I set mode to 'by jenkinsfile' and set script path to Jenkinsfile - this is configurable depending on your needs as you can embed the script directly in the pipeline. I would opt to include it in your repo as you can change behaviour on an individual branch level
    
* I left the rest of the options as default
    

Once this is done, we should have a working pipeline, in order to test navigate back to "Dashboard" and select the pipeline that should now be visible. Once the pipelines homepage loads you should be able to select "Scan Repository Now" and this should give you some positive responses and may even populate branches and PRs depending on the state of your repo.

This will not work if you don't currently have a jenkinsfile in the repo if you followed the above configuration.

A simple Jenkinsfile example for a python Django project:

```java
pipeline {
    agent any

    environment {
        // Define environment variables here, like your Python version
        VENV_DIR = 'venv'
        DJANGO_SETTINGS_MODULE = 'my-django-project.settings' // Adjust this to your settings module
    }

    stages {
        stage('Setup') {
            steps {
                script {
                    // Create a Python virtual environment
                    if (!fileExists("${VENV_DIR}/bin/activate")) {
                        sh 'python3 -m venv ${VENV_DIR}'
                    }
                }
                // Activate the virtual environment and install dependencies
                sh '''
                    . ${VENV_DIR}/bin/activate
                    pip install -r src/requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                // Run Django tests
                sh '''
                    export $(grep -v '^#' .env | xargs)
                    . ${VENV_DIR}/bin/activate
                    cd src
                    pytest --junitxml=../test-results.xml
                '''
            }
        }
    }

    post {
        always {
            // Archive test results and logs
            archiveArtifacts artifacts: 'test-results.xml', allowEmptyArchive: true
            junit 'test-results.xml'
            cleanWs()
        }
        failure {
            // Notify if the build fails
            echo 'The build failed.'
        }
        success {
            // Notify if the build succeeds
            echo 'The build was successful.'
        }
    }
}
```

This concludes this post, I hope it was of some use to someone!