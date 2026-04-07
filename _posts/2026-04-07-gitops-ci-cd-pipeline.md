---
layout: post
title: "Understanding a GitOps CI/CD pipeline step by step"
date: 2026-04-07 08:00:00 +0300
categories: [gitops, ci-cd]
tags: [gitops, ci-cd, pipeline, kubernetes]
excerpt: "Dive into a pull‑based GitOps pipeline and learn how to build, publish and deploy applications by treating your manifests as code."
---

GitOps is an extension of DevOps that treats **Git as the single source of truth** for both application code and Kubernetes manifests.  Instead of pushing changes directly to the cluster, a GitOps pipeline updates configuration stored in a Git repository; an agent in the cluster then reconciles the actual state with the desired state in Git.  This pull‑based model improves security and auditability【640607256036994†L116-L129】【87887131674625†L150-L156】.

## Source code and tests

As in a traditional pipeline, development starts with a normal Git workflow.  Developers clone the repository, create branches, make changes and run local tests:

```bash
git clone https://github.com/myorg/myapp.git
cd myapp
git checkout -b feature/login
# run tests
npm install
npm test
```

After merging the feature branch, a CI job builds and publishes a container image.  The CI job itself does **not** deploy to the cluster; instead, it updates manifests stored in a separate Git repository.

## Build and publish a new container image

The first step of the pipeline builds and pushes a new container image.  The Argo CD documentation recommends building an image with a new tag and pushing it to your registry【525239033752059†L265-L272】:

```bash
# build and push the new version
docker build -t mycompany/guestbook:v2.0 .
docker push mycompany/guestbook:v2.0
```

## Update the manifests and push to Git

Next, update the Kubernetes manifests that describe the desired state.  The manifests live in a separate **configuration repository**; this separation avoids accidental triggering loops and keeps application code and config independent【525239033752059†L278-L284】.  Clone the configuration repo and update the image tag using Kustomize or direct YAML patching【525239033752059†L273-L295】:

```bash
# clone the config repository
git clone https://github.com/mycompany/guestbook-config.git
cd guestbook-config

# use kustomize to update the image
kustomize edit set image mycompany/guestbook:v2.0

# alternatively, patch the YAML locally (plain YAML example)
kubectl patch --local -f config-deployment.yaml \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"guestbook","image":"mycompany/guestbook:v2.0"}]}}}}' \
  -o yaml > config-deployment.yaml

# commit and push
git commit -am "Update guestbook to v2.0"
git push
```

These commands commit the desired state change (the new image tag) into Git.  Because Git is the source of truth, every change is versioned and auditable【640607256036994†L116-L129】.

## Synchronize the cluster (optional)

Many GitOps controllers, such as **Argo CD** or **Flux**, automatically detect changes in the configuration repository and sync them to the cluster.  If automated sync is disabled, you can trigger a manual sync using the `argocd` CLI【525239033752059†L296-L308】:

```bash
# download the argocd CLI (if needed)
export ARGOCD_SERVER=argocd.example.com
export ARGOCD_AUTH_TOKEN=<JWT token>
curl -sSL -o /usr/local/bin/argocd \
  https://${ARGOCD_SERVER}/download/argocd-linux-amd64

# trigger a sync and wait until the app is healthy
argocd app sync guestbook
argocd app wait guestbook
```

With automated synchronization enabled, the controller reconciles the cluster automatically: it compares the actual state of the cluster with the desired state in Git and makes the necessary changes【525239033752059†L296-L312】.  This pull‑based model eliminates the need to store cluster credentials in the CI pipeline and ensures that any configuration drift is corrected.

## Conclusion

A GitOps pipeline uses the same build and test stages as a traditional DevOps pipeline but diverges at deployment time.  Instead of using the CI system to push changes to the cluster, GitOps stores all configuration in Git and relies on an agent to pull and reconcile the desired state.  This approach improves security (no cluster credentials in the pipeline), provides an immutable audit trail, and makes rollbacks as simple as reverting a commit【640607256036994†L116-L129】【87887131674625†L150-L156】.  In practice, GitOps pairs well with DevOps: you still need a robust CI pipeline to build and test code, but GitOps handles deployment in a declarative, versioned and automated way.
