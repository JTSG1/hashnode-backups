---
title: "Docker, so what is it anyway?"
seoTitle: "Docker, so what is it anyway?"
seoDescription: "An introductory look at what docker is and how it can be used in your own projects. Includes use cases and example Dockerfile for a Python project."
datePublished: Sat Mar 09 2024 21:51:33 GMT+0000 (Coordinated Universal Time)
cuid: cltkmeohn00010al296o36hcn
slug: docker-so-what-is-it-anyway
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710021365352/709eb7c1-55f7-4fb5-8535-6b30c96d1545.webp
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1710021009534/1dd58b6a-81d9-4364-8408-efdb3790201b.webp
tags: microservices, docker, lxc, containers, windows-containers

---

Docker has been around for a while, and my observations you are either exposed to it daily or haven't really touched it and its a bit a mystery. I was in the latter camp up until a few years ago when I had only briefly played with it. Now as someone who is a little more familiar with containerization, I would like to get my thoughts down on it, you know 'for the record'. My eureka moment with containers came when I realized how portable it can make your applications, especially when combined with orchestration technology like Kubernetes.

### Cool, and?

Containers are a way to create a distinct operating environment for your applications by creating a sandbox that is lightweight and efficient. It shares the same kernel as the host system however, it runs in its own contained environment.

It was originally based on LXC containers for its runtime environment. LXC allows creating many different user space sessions using cgroups (control groups). These environments cannot interact directly with each other (they can use networking communication). Docker has moved on from this as a base technology however, the concepts remain the same.

This is how containers differ from virtualisation, they don't attempt to virtualise at the hardware level, instead, they do it at the user space level. This means a container is far lighter resource wise than a virtual machine. One knock on affect of this efficiency is containers are very quick to start up, very useful if you need to scale an application to answer a surge in demand!

### Why should I care about it?

Its a very useful technology, it allows us to create images that can be used to create containers. The behavior of these containers can be a lot more predictable as the environment is defined when the image is created. You can take the same image and run it on any host and expect the same results. (Images are created through the use of a Dockerfile).

Any application can be containerized, and Windows containers are also possible although, typically, Linux is the target platform when containerizing an app.

### Wait, windows can be containerized?

While I've not used it directly myself, I'm aware that Docker Engine - Enterprise allows the creation of Windows containers via namespace isolation technology allowing containers to share the same Windows Kernel.

From what I gather, Windows containers are less flexible than Linux containers; in the sense that they must be running the same OS version as the host. A Linux container can run any number of distributions of Linux. As a side note, you can run Linux containers via WSL.

### How does this benefit in the real world

Containerised workloads can be leveraged to offer many advantages, some examples include:

#### Continuous Integration and Continuous Deployment (CI/CD)

We can make use of CI/CD tools such as Jenkins or Github Actions to create automated deployment pipelines. Typically this might look something like this. A change is made to an application and committed to the repository, commit triggers a Jenkins pipeline that performs unit tests / vulnerability scanning and container image creation (as a basic example). The container is then deployed to a container registry where it can be used to deploy to a production environment.

#### Microservice Architecture

Containers facilitate systems written with microservice architecture. Microservices ensure that functionality that makes up a larger system is decoupled, unlike monolithic applications. Some advantages of adopting a microservice pattern are:

* individual parts of the application can be updated without impacting other parts, reducing risk of introducing regression issues
    
* Individual parts of the application can be scaled independently, if a sales system gets a sudden surge of orders, the part that handles processing those orders could be scaled whereas other parts of the system could be left. This has large cost benefits for companies as its more surgical and less brute force!
    

#### Legacy applications modernisation

Containers provide a convenient way to create environments that allow legacy applications to run on more modern infrastructures. Even if it is monolithic. Whilst I would advocate for looking into refactoring or recreating if this isn't possible, containerisation alone does offer benefits. Containerising an application in this way can be the first step in a large modernisation project.

### A brief look at a docker file

A Dockerfile can be as simple as:

```dockerfile
FROM python:3.10.13-slim-bullseye

# Use apt-get to install system packages (we are using a Debian base image)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libc-dev \
    curl \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy the application code and requirements file from the code repository to a location inside the image
COPY ./app /app
COPY ./requirements.txt /requirements.txt

# Acquire the rust setup script and run it as well as set up the env var required
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Install Python dependencies using pip
RUN pip install --upgrade pip && pip install -r /requirements.txt

# WORKDIR is analogous to cd command
WORKDIR /app

# Run the app
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

This particular file is containerising a FastAPI (Python) application. It demonstrates how you can run commands to install system packages as well as using curl to install Rust runtime (required for one of the dependencies) and adding a path to the PATH env var. The CMD directive at the bottom of the file runs the uvicorn server and serves the application on part 8000. To create an image from this you would simply run:

```bash
docker build . -t name-of-image
```

And that's it, your application is containerized!

### To Conclude

This article was intended to be a brief introduction to containers to anyone who might have heard about the tech before but didn't really have chance to work directly with it or fully appreciate its implications on how we might build systems.

If it has stoked your interest definitely start checking Docker out, it is super-easy to install via Docker Desktop and you can get started with containerizing your applications simply by adding a Dockerfile to your project and adding a few lines to that.

Do you have use cases that can't use containers? What about some unique situations where containers have been used? I'd love to hear about them!