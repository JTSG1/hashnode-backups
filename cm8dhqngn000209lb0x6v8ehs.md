---
title: "K8s, Helm, and Container Orchestration"
datePublished: Mon Mar 17 2025 20:02:06 GMT+0000 (Coordinated Universal Time)
cuid: cm8dhqngn000209lb0x6v8ehs
slug: k8s-helm-and-container-orchestration
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/GSiEeoHcNTQ/upload/7efd018704618a07e487765a57c0b835.jpeg
tags: kubernetes, helm, container-orchestration

---

## Introduction

Over the past year, my role has increasingly involved containerization technology, particularly Kubernetes. Kubernetes is a fantastic tool for deploying applications and enabling them to scale according to demand. Once you're comfortable with these technologies, you can really start to leverage the power of elastic computing.

A supporting repo can be found here: [https://github.com/JTSG1/helm-kubernetes-tutorial](https://github.com/JTSG1/helm-kubernetes-tutorial)

## Kubernetes

Kubernetes is a container orchestration platform. It allows you to take your Docker images and build production-ready workloads using YAML files. These files can be version-controlled and deployed through automated CI/CD pipelines. Sounds great, right?

Throughout this article, I'll provide some practical examples. Most of these will center around a service dashboard software called Homer. I've chosen Homer because it's relatively straightforward to get up and running, and I'm a big fan of the software for self-hosting.

## Helm

Helm is essentially a templating system built on top of Kubernetes objects. It allows you to parameterize and manage releases centrally, offering greater control over your Kubernetes deployments. A great use case for Helm is the ability to tailor your application configuration to specific environments.

Imagine you've just developed the next big thing that's going to go viral and make you a fortune! If you're following good practices, you'll have several different environments set up to progress your deployment through various testing stages. Helm helps with this by allowing you to have a base configuration that you can then override for specific environments.

## A Simple Kubernetes Deployment

We're going to look at a simplified set of Kubernetes manifests. Typically, in a production environment, you'd have more, but let's keep things simple for now. To deploy Homer, I want to define the following Kubernetes objects:

* **deployment**: This contains the actual service.
    
* **namespace**: This is used to segregate objects, as Kubernetes clusters rarely host just one workload.
    
* **service**: This defines how we expose the application to the outside world.
    
* **service account**: This specifies the 'user' the service will run under, which is especially useful when deploying to services like GCP that offer workload federation.
    

The file structure I'm aiming for will look like this:

```yaml
./homer-sample-no-helm
    - default-deployment.yaml
    - default-namespace.yaml
    - default-service.yaml
    - default-serviceaccount.yaml
```

Here's the content of each of these files:

**default-deployment.yaml**

This object contains the actual container and its related settings.

YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homer-deployment
  namespace: ns-homer
  labels:
    app: homer-deployment
spec:
  strategy:
    type: Recreate
  replicas: 1
  selector:
    matchLabels:
      app: kafka-sample
  template:
    metadata:
      annotations:
        sampleAnnotation: "a-sample-annotation"
      labels:
        app: kafka-sample
    spec:
      serviceAccountName: homer-service-account
      containers:
      # hello world container
      - name: hello-world
        image: docker.io/b4bz/homer
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 128Mi
        env:
        - name: MY_ENV_VAR
          value: "my-env-var-value"
        - name: MY_ENV_VAR_2
          value: "my-env-var-value-2"
      restartPolicy: Always
```

**default-namespace.yaml**

This is the Kubernetes namespace definition for this deployment.

YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns-homer
  labels:
    app: ns-homer
```

**default-service.yaml**

This service exposes the deployment externally using a LoadBalancer.

YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: homer-deployment
  namespace: ns-homer
  labels:
    app: homer-deployment
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: kafka-sample
  type: LoadBalancer
```

**default-serviceaccount.yaml**

This is the Kubernetes service account that the deployments will run under.

YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-homer-sample
  namespace: ns-homer
  labels:
    app: home-sample
```

As you can already see, even for a relatively simple deployment, there are quite a few files. Maintaining these files separately for each of your RTL (Route to Live) environments would quickly become cumbersome. This is where Helm comes in handy.

Another thing you might notice is the amount of repetition across these files. A simple typo in one of these fields can lead to hours of debugging! Trust me, I've been there.

## Helm to the Rescue

Now, let's look at the same set of files but within a Helm setup.

```yaml
./homer-sample
    - templates/
        - default-deployment.yaml
        - default-namespace.yaml
        - default-service.yaml
        - default-serviceaccount.yaml
    - Chart.yaml
    - values.yaml
```

Inside the `templates` folder, we have the same set of files, but their content is different. We'll review that in a moment. Additionally, we also have two new files: `Chart.yaml` and `values.yaml`.

`Chart.yaml` contains metadata about the Helm deployment.

`values.yaml` is a central store for variables that we can reference in our Helm-templated Kubernetes manifests.

Now, let's re-examine the four manifest templates:

**default-deployment.yaml**

This file now contains references to `Values.*` variables, which are defined in the `values.yaml` file. You can immediately see the benefits of this approach to structuring these files. We are essentially templating the Helm files and injecting values defined elsewhere into them.

Also of note are the conditional statements included here.

YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deployment.name }}
  namespace: {{ $.Values.namespace.name }}
  labels:
    app: {{ .Values.deployment.name }}
spec:
  strategy:
    type: Recreate
  replicas: 1
  selector:
    matchLabels:
      app: kafka-sample
  template:
    metadata:
      annotations:
        sampleAnnotation: "a-sample-annotation"
        {{- if eq .Values.conditionalAnnotations.enabled true}}
        conditionalAnnotation: "a-conditional-annotation"
        {{- end }}
      labels:
        app: kafka-sample
    spec:
      serviceAccountName: {{ .Values.deployment.serviceAccountName }}
      containers:
      # hello world container
      - name: hello-world
        image: docker.io/b4bz/homer
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        resources:
          {{ toYaml .Values.resources | nindent 10 }}
        env:
        - name: MY_ENV_VAR
          value: "my-env-var-value"
        - name: MY_ENV_VAR_2
          value: "my-env-var-value-2"
      restartPolicy: Always
```

**default-namespace.yaml**

Again, you can see the same values referenced in the namespace file, all defined in the `values.yaml` file.

YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace.name }}
  labels:
    app: {{ .Values.namespace.name }}
```

**default-service.yaml**

Thinking about conditional elements, imagine a situation like the one below where we are defining an exposing service. What if your local environment didn't support `LoadBalancer` type services? Docker Desktop, for example, doesn't support it. You could create an overrides file and adjust your template accordingly to allow you to select a different service type.

YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.deployment.serviceName }}
  namespace: {{ $.Values.namespace.name }}
  labels:
    app: {{ .Values.deployment.serviceName }}
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    app: kafka-sample
  type: LoadBalancer
```

**default-serviceaccount.yaml**

YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.deployment.serviceAccountName }}
  namespace: {{ $.Values.namespace.name }}
  labels:
    app: {{ .Values.deployment.serviceAccountName }}
```

**Values.yaml**

The `values.yaml` file contains the values for each of your defined variables.

YAML

```yaml
deployment:
  name: homer-sample
  serviceName: homer-sample-service
  serviceAccountName: sa-homer-sample

namespace:
  name: homer-sample

conditionalAnnotations:
  enabled: true

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

We mentioned overrides earlier. These are supplementary files to the `values.yaml` file. They can be defined elsewhere and injected into the Helm command like this:

Bash

```bash
helm install kafka-sample-release ./homer-sample -f ./homer-sample-overrides/bld-override.yaml
```

In the example above, we are injecting an overrides file into the Helm command. This will take the values defined in that file and apply them on top of the values in `values.yaml`. An example of an overrides file is shown below.

Here, we are altering resource constraints for a lower-provisioned development environment.

YAML

```yaml
resources:
  limits:
    cpu: 50m
    memory: 64Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

## And That's It

This covers everything I wanted to discuss in this post. I've found Kubernetes and Helm to be incredibly useful tools in my work. For smaller-scale applications, it might seem like overkill initially, but the effort invested in deploying an app this way can really pay off if it ever needs to scale.

How are you using Kubernetes and Helm in your hobby projects or even in your professional life? I'd love to hear more real-world examples of how people are using this fantastic technology.