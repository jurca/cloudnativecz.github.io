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
  consider using multi-staged builds (see below).

To use the dockerfile fragment above as an example, let's improve the situation a little bit:

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
