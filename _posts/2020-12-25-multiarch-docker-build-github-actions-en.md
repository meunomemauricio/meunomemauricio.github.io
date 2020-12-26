---
layout: post
comments: true
title: Building Multi Platform Docker Images using Github Actions
date: 2020-12-25 23:25:00
lang: en
ref: multiarch-docker-gh-actions
categories: Docker
tags:
- CI
- Docker
- GitHub Actions
- Raspberry Pi  
excerpt_separator: <!--more-->
---

Trying to setup a service using Docker in a Raspberry Pi, I've found myself
dealing with astronomical build times (I stoped counting after 20 minutes),
many times resulting in errors due to incompatibility or network issues.

This post presents a complete and automated solution for this problem, using
Github Actions to automatically build multi platform images and pushing them to
[Docker Hub][docker-hub].

<!--more-->

## Running Docker on Raspberry Pi ##

Docker is an excelent tool for creating and managing containers, greatly
simplifying the process of setting up a project topology and all its
depencies (DB, cache, static hosting, etc...).

In 2016, it was announced at the [Raspberry Pi Blog][rasp-pi-docker] that the
new `raspbian/jessie` version would count with official support for Docker,
allowing it to be installed with relatively ease:

{% highlight terminal %}
$ curl -sSL https://get.docker.com | sh
{% endhighlight %}

Unfortunately, due to the reduced processing power of the Raspberry Pi,
specially older hardware versions, building a container image is *extremely*
slow.

In one instance, I waited more then 20 minutes for it to finish, just to see it
being cancelled due to a temporary network issue with the NPM repositories.

## Multi Platform Builds ##

The solution I've found is using the [`buildx`][buildx-repo] command, which is
available vie the CLI since version `19.03`.

The command allows the creation of *builders*, capable of building images from
`Dockerfile`s. The command, allied with [QEMU][qemu], is capable of [building
multi platform images][buildx-repo-multi], including `arm/v6` and `arm/v7`,
both supported by the Raspbery Pi.

I tested this method by generating an image and storing it in a `.tar` file:

{% highlight terminal %}
# Create Builder
$ docker buildx create --name mybuilder

# Use QEMU to support ARM archtectures
$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Inspect the builder (the bootstrap flag also initiates its instance)
$ docker buildx inspect --bootstrap

# Build docker image and save it to dist/myproject.tar
$ docker buildx build --platform linux/arm/v6 -t myproject:latest -o type=docker,dest=- . > dist/myproject.tar
{% endhighlight %}

After finished, the image can be copied to the Raspbery Pi and loaded using:

{% highlight terminal %}
$ docker load -i myproject.tar

# Tag the image to make it easier to reference
$ docker image tag <SHA digest> myproject:latest
{% endhighlight %}

## Automatic Builds using Github Actions ##

TBC...

## References ##

[buildx-repo]: https://github.com/docker/buildx/
[buildx-repo-multi]: https://github.com/docker/buildx/#building-multi-platform-images
[docker-hub]: https://hub.docker.com
[qemu]: https://www.qemu.org/
[rasp-pi-docker]: https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/
