---
layout: posts
title:  "Building a Docker image - DOs and DON'Ts"
date:   2018-11-16 20:27:37 +0100
categories: docker
authors: 
  - "Martin Jurča"
---

So it happened. You have been tasked with moving your existing (or a new) app into the cloud and you need to dockerize
it. Why, there are some DOs and DON'Ts.

Just to be clear, these are just general recommendations when it comes to packaging an app into a docker image. You do
not have to follow any of them and it is likely your app will work "just fine", however, you might be able to get
better result by adhering to the here-mentioned practices.

## Packing multiple components of your application

Let's say you app is a web app that consists of a database, an admin interface, a frontend, an admin backend and a
frontend's backend. It might be tempting to just pack the whole thing into a single container to "keep things simple":

```Dockerfile
FROM ...
COPY frontend-api/ frontend/ admin/ admin-api/ .
...
```

You probably have already heard the "there should be only one thing in a docker image," and most likely it was never
followed by an explanation why. Don't worry, we're here to fix that.

There are, unfortunatelly, sever down sides to the approach depicted above:

* Running heterogenous processes inside an isolated environment (a docker container) may result in performance issues,
  with the processes sharing a single execution context. This is not a design flaw - you are meant to scale containers,
  not processes. Worker sub-processes and threads are fine, but you should scale your app by adding more containers
  instead of forking the main process. This is good for reliability of your service as well.
* This results in a bloated (unnecessarily large) image. Not only you image has to include the every component (or
  microservice if you will) of your app, it also has to include all the dependencies of every component. Large images
  as distributed slowly over the network to cloud nodes, are slower to resolve and start up. This results in a much
  slower scaling of your app.
* By running multiple services inside a container, you have to re-implement management of these services' processes -
  starting, monitoring and restarting them. Docker monitors only the entrypoint process's lifetime, and if that process
  dies, docker stops the entire container. Splitting services into separate containers allows you to have docker (and
  whatever orchestration tool you use with it) handle the lifecycle of your services.
* Updating a single component requires re-building and re-deploying all components since they all occupy the same
  image.
* You will get less scaling flexibility. Your admin interface, for example, does not have to handle such a traffic as
  your public frontend does. Packing it together requires you to have the same amount of admin interface instances as
  frontend instances. This way hosting your app can get pretty expensive once the load goes up.

All right, puthing everything into a single image is bad, so how do we fix it? It's actually quite simple: you build a
separate docker image for each of your components, e.g.:

```Dockerfile
FROM ...
COPY frontend/
...
```

```Dockerfile
FROM ...
COPY admin/
...
```

...etc.

Now you might be wondering - how will I ever wire this up? There is no guarantee these containers will get deplyoed to
the same worker node! Is everything lost?

Luckily, no. These components need to be wired over the network, the "hard" part is service discovery. Unfortunately
there is no easy "one size fits all" solution here, so we'll provide at least some examples:

* If you're running a [Socker Swarm](https://docs.docker.com/engine/swarm/) cloud, you can describe and deploy your
  app using a [docker-compose](https://docs.docker.com/compose/) file.
* If you are deploying to a
  [Marathon](https://mesosphere.github.io/marathon/)+[Mesos](https://mesos.apache.org/) cloud, you can use a
  [proxy](https://mesosphere.github.io/marathon/docs/service-discovery-load-balancing.html) that will utilize
  marathon's service discovery in the background and transparently forward all traffic to the running containers.
* Are you a [Kubernetes](https://kubernetes.io/) user? Well, there are several options. A naive approach would be to
  deploy all containers as a single pod, however, that would suffer from scaling issues described above. A better
  option is to utilize the kubernetes'
  [service declarations](https://kubernetes.io/docs/concepts/services-networking/service/).
* For [OpenShift](https://www.openshift.com/) users this are pretty similar to the Kubernetes' users since OpenShift is
  built on top of Kubernetes - use services to set up communication between the containers of your application. Note
  that while services are great for establishing communication pathways between your containers, it is preferable to
  use openshift's [routes](https://docs.openshift.com/container-platform/3.11/dev_guide/routes.html) to establish
  communication with the outside world.

## Choosing your base image

All right, so let's say you have split your app into several docker containers, with each service having its own image.
As we already stated, smaller images deloy faster. Ergo, when building your app, you want to use the smallest base
image possible, right? Well, no.

Take, for example, the following Dockerfile:

```Dockerfile
FROM alpine
RUN apk update && apk install perl go bash
COPY app/ /src/
RUN bash /src/prepare.sh && perl build-deps.pl && ...
...
```

There are several issues with this approach:

* You are reinventing the wheel. Whatever problem you may be solving, always check whether someone else has solved it
  already first.
* Your image will have a lots of custom content in its layers. This is bad for image layer caching, updating your
  images, performance (you have to re-do the work already done for you in existing images) and deployment speed (the
  more layers your images share with other images, the less has the docker to download during image fetching).
* By installing all these extra stuff, you increase the size your image considerably. Are you sure you need all of this
  to run your image, or just to build your image? If you only need a lot of these dependencies only build your image,
  consider using multi-staged builds (see **Optimizing the image layers** below).

To use the Dockerfile fragment above as an example, let's improve the situation a little bit:

```Dockerfile
FROM golang:1.11-alpine
RUN apk update && apk install perl bash
COPY app/ /src/
RUN bash /src/prepare.sh && perl build-deps.pl && ...
...
```

Sure, this removes the neccessity to install go, but we can do even better by using a golang image based on the Debian
Stretch, which already has everything we need:

```Dockerfile
FROM golang:1.11-stretch
COPY app/ /src/
RUN bash /src/prepare.sh && perl build-deps.pl && ...
...
```

The lesson here is simple: do not do the work someone has already done for you.

## Optimizing the image layers

The simplest way of packaging a component of your app is building it in the same docker image as the one used for
running it:

```Dockerfile
FROM ...
RUN apt update && apt install ...
RUN /opt/build.sh
CMD /opt/start.sh
ENV TLD=com
ENV ENVIRONMENT=production
```

While this approach is indeed simple in its nature, it results in images with lots of unnecessary bloat, especially
since the amount of dependencies used to build a component is usually larger than the amount of dependencies required
to run a built component.

Of course, we may slighty reduce the number of layers by merging some lines in the Dockerfile, e.g.:

```Dockerfile
FROM ...

RUN apt update && apt install ... && /opt/build.sh
CMD /opt/start.sh
```

This, however, is not good enough, simply because we can do a lot better. What we can do is use one image to build our
app and another to host the built app without the sources or build dependencies. The easiest way to do this is using
a [multi-stage docker build](https://docs.docker.com/develop/develop-images/multistage-build/):

```Dockerfile
FROM node:10 as builder
WORKDIR /app
COPY package.json package-lock.json /app/
RUN npm ci
RUN npm run build

FROM NODE:10-alpine
WORKDIR /app
CMD node /app/dist/app.js
RUN NODE_ENV=production npm ci
COPY --from=builder /app/dist/ /app/dist/
```

In the special case that the builder and runtime image of your app has the same base image and there are steps that are
common to both the builder and the runtime image, you may create a common base image within your Dockerfile without
having to manage a separately built and distributed image:

```Dockerfile
FROM node:10 as base
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json /app/

FROM base as builder
RUN NODE_ENV=development npm ci && npm run build

FROM base
COPY --from=builder /app/dist/ /app/dist/
```

When using multi-stage docker builds, the last image will be the result of the `docker build` command and will be
tagged by the tag specified in the command's options (unless the [`--target`]() option is used, but that stops the
build process after the specified target). This, however, does not stop us from creating other images for other uses
(e.g. running end-to-end tests) after the runtime image has been created. All we have to do is "recreate" our runtime
image by using the previously built runtime image as a base image:

```Dockerfile
...

FROM base as runtime
COPY --from=builder /app/dist/ /app/dist/

FROM builder
RUN npm run setup-e2e-env && npm run test:e2e

FROM runtime
```

As stated above, the last built image will be the tagged (if tags were specified) result of the `docker build` command,
however, every built image, including the intermediates, will be localy available. Now, you may frown at the "waste of
space" we create, however, this is another thing we may use to our advantage.

One huge disadvantage of end-to-end tests is how unreliable they are in general, producing false negatives here and
there. This is especially unpleasant if you ran your e2e tests as a part of a single `RUN` directive in your Dockerfile
along with dependencies installation and building the app - rerunning a CI build pipeline gets quite expensive.

To help the situation, it might be beneficial not to optimize the number of layers of your `builder` image but running
each command in a separate `RUN` directive in your Dockerfile. Also, if possible, set up your CI to retry a failed CI
pipeline job on the same worker it ran previously (so that we may utilize the succesfully built layers of the `builder`
image) for the first re-try only (in case there is an issue with the worker itself **and** you have multiple CI workers
available).

This way, should our end-to-end tests (or any build command) fail, we may just hit the "retry" button the CI's UI and
enjoy a huge time savings since no successfully completed work will have to be re-done.

Oh, and the "waste of space" caused by intermediate images? I would recommend setting up a crop script that would
automatically delete all untagged images older than 2 hours (or any other reasonable amount of time), or clear all the
untagged images from your CI worker in the final CI stage.

Run the following command to list **all** the untagged docker images:

```bash
docker images -f “dangling=true” -q
```

To delete all the untagged images, regardless how old or fresh they are, you may use the following command:

```bash
docker rmi $(docker images -f “dangling=true” -q)
```

## Removing unnecessary layers of your docker image

Docker provides us with a plethora of Dockerfile directives that are used while building your app, some are used to
include files into the image, some run commands, some modify the runtime configuration, while other provide various
meta-data.

The thing is that every directive in your Dockerfile will add an additional layer to the resulting image. While modern
docker image filesystem drivers, such as overlay2, handle layers a lot better than their predecessors, it is preferable
to keep the number of layers of your resulting image to a minimum.

Firstly, take a better look at the environment you deploy your image to. It is usually reasonable to assume that it
allows you to maintan a persistent configuration for running your docker image that is kept and applied with every new
version of your image as well. Such configuration overrides the defaults specified by the Dockerfile directives used to
specify runtime configuration:

* [`CMD`](https://docs.docker.com/engine/reference/builder/#cmd) - this specifies the default executable run when
  starting a container based on your image.
* [`EXPOSE`]() - when running a docker container, the platform may choose to expose any container's ports it chooses,
  or none at all. No ports are exposed by default, even if specified in an `EXPOSE` directive.
* [`ENV`]() - this is used to configure the ENV variables available after the `ENV` directive in the subsequent `RUN`
  directives or during the lifetime of a container created from the built image. While these may be useful during build
  time, the runtime ENV variables can be managed by the platform running the containers, ergo the `ENV` directive does
  not have to be used outside the builder image (see [multi-stage docker builds0]()).
* [`ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#entrypoint) - this, along with the `CMD` directive,
  specifies the default executable run when starting a container based on your image (see the docs). Since entrypoints
  are overridable, just as start commands, you may not need this.
* [`VOLUME`](https://docs.docker.com/engine/reference/builder/#volume) - Unless you want a certain directory in your
  container to be a volume regardless of the options passed to the `docker run` command, you do not need this. Any
  directory within a container can be mounted to the host's filesystem regardless of being specified in a `VOLUME`
  directive.
* [`USER`](https://docs.docker.com/engine/reference/builder/#user) - while it is preferrable not to run your docker
  container processes as root for security reasons, and preferably run the processes as nobody, this can be overridden
  when starting a container.
* [`STOPSIGNAL`](https://docs.docker.com/engine/reference/builder/#stopsignal) - docker will send your app a SIGTERM
  to the root process when `docker stop` is executed. Why not just use it?
* [`HEALTHCHECK`]() - this directive allows to specify a command to execute as a healtcheck and use the command's exit
  status as healthcheck result. It might be worth it to look into healthcheck options provided by your deployment
  platform as it most likely offers more flexible/convenient options.
* [`SHELL`]() - unless you are building an image you will use locally and not in the cloud, there is little use for
  this that acutally adds value.

There are also a couple of metadata diretives you may want to reconsider using:

* [`LABEL`](https://docs.docker.com/engine/reference/builder/#label) - are you sure that the image tag, which can
  reflect a git tag is not enough? The git tag may contain any extra information in its message (which **can** be
  multi-line), and tools such as GitHub and GitLab allow for editable "release notes" accompanying the tags.
* [`MAINTAINER`](https://docs.docker.com/engine/reference/builder/#maintainer-deprecated) - this has been deprecated in
  favor of `LABEL` anyway.

And finally, there are directives that are used to configure the build process:

* [`ARG`]() - this is used to configure the ENV variables during the build process. While useful for builder images
  (see [multi-stage docker builds]()), do not use it for runtime images - just the ENV variables using the deployment
  platform.

All these directives may be tempting to use since they provide documentation value, however, such documentation may be
kept elsewhere. If you can ommit a directive listed in this chapter in favor of a runtime configuration maintained by
the platform you deploy to, you probably should.

In case that there is a good reason (and value) in using any of these directives in your Dockerfile (e.g. a container
you run locally on your machine for development), specify them at the start or your Dockerfile right after the `FROM`
directive, since these usually change the least often, and combine to single use them when possible (see the supported
syntaxes for each directive).

## Testing your application

Our advice here is simple: test in CI, do not test in production. Fail the CI pipeline if tests fail. Run simple tests
first (e.g. unit tests), complex tests later (e.g. integration tests) and end-to-end tests as last. Also, run all the
tests that do not require you to actually build your application before you build your application (fail early). And
finally: do not include the test results in the built app, leave them in the builder image
(see [multi-stage docker builds]()).
