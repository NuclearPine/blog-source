+++
title = "CI/CD basics - deploying a containerized static website automatically with GitHub Actions and Kubernetes"
date = "2025-12-20T17:26:25-07:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Jackson Pyrah"
authorTwitter = "" #do not include @
cover = ""
tags = ["CI/CD", "Kubernetes", "GitHub Actions"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

As part of my ongoing effort to improve my DevOps skillset, I decided to re-architect how this blog was deployed.

Previously, I had simply pushed the static site files to a GitHub respository and manually pulled down the new files to a webserver whenever I made an update to the site.
For starters, I wanted to develop a workflow which would automatically deploy the website when I `git push` to my remote repository. For sake of learning, I decided to re-model the entire deployment after an enterprise Kubernetes application. The new deployment pipeline looks like this:

1. New changes to Hugo source files are pushed to a GitHub remote respository (no compiled static files are commited, we will build the website with Hugo later).
2. A GitHub actions workflow runs with two parts:
  - Running Hugo to compile the source markdown files into HTML/CSS, and packaging the resulting static files into a container image with a webserver (Nginx, in this case).
  - Issuing a command to the Kubernetes cluster which runs the webserver application to update the application with the latest container image.
  
In this blogpost, I'll be detailing how you can implement a similar pipeline in your own projects.

# First steps: creating the Dockerfile

The first thing we need is a Dockerfile for building a complete, self-contained web server image that contains the exact content of our static site. Most major webservers provide an officially supported container image for this purpose. Here I have a sample Dockerfile that uses an Nginx container, but the process for most webservers will likely be similar.

```Dockerfile
FROM nginx:alpine

COPY ./public /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Whenever you run `hugo` to build a site, the resulting output files are put in the `public` top-level directory of your project. We simply `COPY` the files into the directory that the Nginx container is configured by default to serve.

In this instance, I'm setting up the container to only serve HTTP traffic, as the server itself will sit behind a reverse proxy I have running on my Kubernetes node to handle HTTPS.

# Configuring GitHub actions to build the site upon commit

Next, we need to create some GitHub actions YAML to instruct the site to be built whenever we push with commits to the `master` branch. Create a new YAML file in the `.github/workflows` directory of your repository. In this case it's simply `build-image.yml'.

```YAML
name: Build and push static site container image

on:
  push:
    branches:
      - master
    paths-ignore: # If the only changed files are those that don't have anything to do with the site or build process, don't run the workflow"
      - ".gitignore"
      - "README.md"

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Setup Hugo
      - name: Setup hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.131.0"
          extended: true

      # Build the Site
      - name: Build site with Hugo
        run: hugo --minify

      # ./public directory with built site should now exist on the runner

      - name: Login to container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest
            type=sha,format=short

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/arm64,linux/amd64
```

This file is fairly easy to follow along with to explain the process. We tell our GH Actions runner to checkout our repository, use Hugo to build the site, and then have Docker build and push the image to the container registry according to the instructions in our Dockerfile.

Later, we will be adding an additional job to this workflow for automatically updating the site - but for now we will commit this workflow as is and get our first container image published to the container registry. 

# Kubernetes configuration

Now we configure our deployment. For the sake of this tutorial, I'll assume you already have a working Kubernetes cluster. In my case I'm using a simple single-node cluster I installed on an Oracle Cloud VM. Let's create a new manfiest for a deployment and associated service.

```YAML
---
# Blog web server deployment manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blogserver-deployment
  labels:
    app: blogserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blogserver
  template:
    metadata:
      labels:
        app: blogserver
    spec:
      imagePullSecrets:
        - name: ghcr-secret
      containers:
        - name: blogserver-nginx
          image: ghcr.io/nuclearpine/blog:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80

---
# Service manifest
apiVersion: v1
kind: Service
metadata:
  name: blogserver-service
spec:
  selector:
    app: blogserver
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30081
  type: NodePort
```

You can take this sample manifest and adjust it for your needs. Just change `spec.template.spec.containers` appropriately. I opt for a `NodePort` service as this is a single-node cluster.

I also created a `docker-registry` secret with information needed to login to GHCR, but this is optional if your repository is public. Logging in may help avoid things like rate-limits, however. If you want to do this, just make a secret like this with your GitHub credentials.

```Bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_PERSONAL_AUTHENTICATION_TOKEN \
  --docker-email=YOUR_EMAIL # You can use a fake email here, as this is mostly a legacy syntax feature, and does not affect your ability to log in to GHCR
```

Now that the manifests are ready, go ahead and run a `kubectl apply -f my-manifest.yml` and your website should now be live!

# Enabling automatic site updates

Our last step is to set up a mechanism for automatically deploying a new version of our website when one becomes available. There are a couple approaches I could have taken here. I first considered the [Keel](https://keel.sh/) Kubernetes operator. This is probably the cleanest solution, but unfortunately Keel does not appear to offer ARM compatible images. I decided to take a push based approached, with GitHub actions connecting to my K8s node over SSH and running a specific `kubectl` command to update the image. As we set `imagePullPolicy: always` in our K8s manifest, our cluster will always deploy the latest available version of the website whenever a new pod is created, so we can automate this entire process by simply automating a way to call ```kubectl rollout restart deployment/your_deployment``` whenever we build a new version of the site.

Add a new job to the `jobs` section of our `build-image.yml` GH Actions workflow:

```YAML
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Connect to Tailnet
        uses: tailscale/github-action@v4
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci
          ping: ${{ secrets.SERVER_SSH_HOST }}
      - name: Trigger kubernetes rollout via SSH
        id: ssh
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_SSH_HOST }}
          username: ${{ secrets.SERVER_SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          fingerprint: ${{ secrets.SERVER_SSH_FINGERPRINT }}
          capture_stdout: true
          script: sudo /usr/local/bin/k0s kubectl rollout restart deployment/blogserver-deployment
      - name: Print captured output from SSH
        run: echo "SSH output was; ${{ steps.ssh.outputs.stdout }}"
```

To avoid exposing SSH publicy on my node, I connect the GH actions runner to my [Tailscale](https://tailscale.com) mesh network with the GH actions runner the Tailscale team has conveniently provided. Then I use the `appleboy/ssh-action` runner to connect to my K8s node over SSH and execute `kubectl rollout`


# Wrapping up

At this point, we're all done! Now you're just a `git push` away from automatically updating content on your static site. This whole solution is quite overkill for the task at hand - using a serverless solution like GitHub pages or Cloudflare pages is probably more pratical and simpler to implement for just hosting a static website. However, this basic deployment pipeline can be ported to a variety of applications you might want to run on kubernetes, and mimics a common type of deployment workflow you might find in enterprise environments.

Thanks for reading!
