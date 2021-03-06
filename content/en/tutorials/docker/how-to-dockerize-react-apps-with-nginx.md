---
title: "How to Dockerize React Apps With NGINX"
date: 2020-08-20T21:33:32-04:00
draft: true
author: serainville
tags:
    - docker
    - react
    - nginx
description: |
    Learn how to use Docker multistage builds to test, build, and containerize React apps with NGINX for running in production environments.
---

In this tutorial, you will learn how to use Docker's multistage builds in order to create a small, lightweight image for your React applications. The final Docker image will use NGINX to host our React application.

The artifacts generated by a React application's build are relatively tiny in comparison to the project itself. The artifacts are generated by transpiling from a large `node_modules` directory, which stores all of a project dependancies. 

Previously, in order to build a light-weight Docker image for node-based applications, such as React, the application build and Docker image build had to be two separate processes. With Docker's multistage builds we are now able to perform both during a regular Docker build, while creating a very tiny, light-weight image.

## Getting Started
In this tutorial we will cover the following topics. You learn how to create your first React Docker image using a multistage Dockerfile.
* Learn how to use multistage Docker builds
* Learn how to containerize React apps
* Learn how to use NGINX Docker images for React apps

You will need the following:
* Docker installed on your local machine


## Multistage Docker Builds
### Creating Stages
Stages are defined everytime there is a `FROM` command in your `Dockerfile`. Each stage is indexed and can be referenced from its ID, starting from zero. Optionally, to make referencing stages much easier a name can be given to a stage using `AS <stage-name>` in the `FROM` command. 

An example of a stage in a Dockerfile with a name looks like the following:

```dockerfile
FROM <image>:<tag> AS <stage-name>
```

The following bare-bones example of a multistage Dockerfile looks like the following. We can three separate stages named build, tests, and final. The last stage, final, will be what creates our final image. 

```dockerfile
FROM <image>:<tag> AS build
FROM <image>:<tag> AS tests
FROM <image>:<tag> AS final
```

The contents of each stage remains behind and does not get promoted to the next one. However, files can be copied from one stage into another. This allows us to use one stage to install all of our build tools and dependancies, perform a build, and then only copy the output artifacts into a final stage.

For node-based projects such as React, this means that massive `node_modules` directory remains behind in the `build` stage.

### Copying files between stages
Unless a file is copied from an earlier stage into a later stage, it will not be included in the final stage. Contents of earlier stages will be discarded.

In order to copy files from one stage to another you must use the `COPY` command with the `--FROM` flag.

```dockerfile
FROM <image>:<tag> AS build
FROM <image>:<tag> AS final
COPY --from=build output/artifact.js /app
```

## Containering React Application
### Test and Build Stage
The first stage of our Docker build will run units tests and build our application. This stage will contain all of the things that are needed to build and test the application, and those things will remain in this stage. They will not be included in our final stage.

```dockerfile
FROM node:latest AS build
COPY ./src .
RUN npm install \ 
    && npm run-script test \
    && npm run-script build-production
```

### Final Stage
Our second and final stage will generate our final, deployable image for our React application. The image will contain only the files and packages necessary to run our application. This stage will create an image based off of the official NGINX image. NGINX was chosen as it is a very high performant static file web server.

Create a new stage named `final` that copies build artifacts from the `build` stage.

```dockerfile
FROM nginx:latest AS final
COPY --from=build ./build/* /usr/local/nginx/html
```

### Final Dockerfile
Our final Dockerfile should resemble the following example. We have defined two different stages for our React application. 

```dockerfile
FROM node:latest AS build
COPY ./src .
RUN npm install \ 
    && npm run-script test \
    && npm run-script build-production

FROM nginx:latest AS final
COPY --from=build build/ /usr/local/nginx/html  
```



