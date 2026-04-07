---
layout: post
title: "Understanding a DevOps CI/CD pipeline step by step"
date: 2026-04-07 08:00:00 +0300
categories: [devops, ci-cd]
tags: [devops, ci-cd, pipeline, kubernetes]
excerpt: "Learn how a traditional push‑based DevOps pipeline works by exploring each stage with practical commands and a Jenkins pipeline example."
---

A high‑level diagram of this pipeline typically shows the flow from source code, through testing and packaging, to image build and deployment.  We provide detailed step‑by‑step instructions below.
A traditional DevOps CI/CD pipeline uses a **push‑based model**: the pipeline itself is responsible for building software, pushing artefacts to a registry and then deploying them into the target environment.  Jenkins (or another CI/CD tool) orchestrates each stage and has credentials to access the Kubernetes cluster.  This guide walks through a typical push‑based pipeline and shows a small Jenkins example.

## Source code and version control

Development starts with code committed to a version‑control system such as Git.  Developers clone the repository and push branches:

```bash
# clone the repository
git clone https://github.com/myorg/myapp.git
cd myapp

# make changes, commit and push
git checkout -b feature/login
# (edit files)
git commit -m "Add login page"
git push origin feature/login
```

When a branch is merged into the main branch, it triggers the pipeline to begin the continuous integration steps.

## Unit tests

The pipeline runs unit tests to catch regressions early.  The exact command depends on your language or framework.  For example, a Node.js project would run:

```bash
npm install
npm test
```

For Java projects you might run `./mvnw test` or for Python `pytest`.  These tests ensure that only code that meets quality standards moves forward.

## Build and package the application

After tests pass, the pipeline builds and packages the application.  For a Java project this might look like:

```bash
./mvnw clean package -DskipTests
```

Many modern pipelines create container images directly from the source.  The build step invokes Docker to build an image using the `Dockerfile` in your repository.

## Publish artifacts to an artifact repository

In some pipelines the build step creates binary artefacts—such as JAR files, libraries or packages—and publishes them to an artefact repository like **Artifactory** or **Nexus**.  This provides a central store for versioned artefacts that other services or teams can consume.  For example, the Maven `deploy` goal compiles the project and uploads the resulting `.jar` to your repository:

```bash
./mvnw deploy -DskipTests
```

Most build tools provide a similar publish command (`gradle publish`, `npm publish`, etc.).  When using containers you may skip this step and move straight to building a container image, but many enterprise pipelines still publish artefacts before containerizing to maintain a standard release process.

## Build the container image

Using the artefacts produced by the build, the pipeline builds a container image.  A typical command is:

```bash
# build the Docker image with a unique tag
docker build -t registry.example.com/myapp:${CI_COMMIT_SHA} .
```

The `${CI_COMMIT_SHA}` environment variable (or an equivalent) ensures each build produces a unique version.

## Push the image to a registry

Once built, the image is pushed to a container registry so that it can be deployed by Kubernetes:

```bash
# push the image to the registry
docker push registry.example.com/myapp:${CI_COMMIT_SHA}
```

The pipeline must have credentials to authenticate with the registry.

## Deploy to Kubernetes

The final stage of a push‑based pipeline applies the new image to the Kubernetes cluster.  This is often done with `kubectl` or Helm.  For example:

```bash
# set the new image on the Deployment resource
kubectl set image deployment myapp myapp=registry.example.com/myapp:${CI_COMMIT_SHA} --record

# or apply a manifest
kubectl apply -f k8s/deployment.yaml

# if using Helm
helm upgrade --install myapp ./helm-chart \
  --set image.repository=registry.example.com/myapp \
  --set image.tag=${CI_COMMIT_SHA}
```

Because the CI/CD pipeline has cluster credentials, it can push changes directly to the cluster.  This push‑based approach is efficient but can lead to configuration drift because the desired state is not stored in version control【640607256036994†L116-L129】.

## Example Jenkins pipeline

The following Jenkinsfile snippet shows a declarative pipeline with stages to build, push and deploy a Docker image.  It is similar to examples in SquareOps’ tutorial【29329369205641†L136-L144】【453623821885254†L190-L204】:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/example/repo.git'
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }
        stage('Push') {
            steps {
                sh 'docker push myrepo/myapp:latest'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }
    }
}
```

This pipeline checks out the code, builds a Docker image, pushes it to a registry and then deploys the updated manifest to Kubernetes using `kubectl apply`【453623821885254†L190-L204】.  You can extend it with additional stages for linting, integration testing or Helm releases.

## Conclusion

A push‑based DevOps pipeline automates the path from source code to running software.  The CI/CD system builds, tests, packages and deploys your application, pushing changes directly to the cluster and container registry.  While this approach is widely used, it requires the CI system to manage cluster credentials and can make it harder to track the desired state.  In the companion post on GitOps, you will see how a pull‑based workflow addresses these challenges by shifting deployment logic into Git.
