---
layout: post
comments: true
title: Building Multi Platform Docker Images using GitHub Actions
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
GitHub Actions to automatically build multi platform images and pushing them to
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

The solution I found is using the [`buildx`][buildx-repo] command, which is
available via the CLI since version `19.03`.

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

## Automatic Builds using GitHub Actions ##

The previous procedure is enough to solve the initial problem, but it's 
still manual and doesn't fit a continuously developed project.

A good way to solve this is by automating it and including it as a 
Continous Integration (CI) routine. [GitHub Actions][gh-actions] is a very 
good solution for that, being very easy to configure and integrating 
seamlessly with the repositories.

GitHub Actions allows us to configure **Workflows**. These are defined 
through `.yml` files and represent a list of actions to be executed 
sequentially. These files should be located in the `.github` directory, in the
root of the repository:

{% highlight terminal %}
$ mkdir -p .github/workflows
$ touch .github/workflows/docker.yml
{% endhighlight %}

This file is defined as:

{% highlight yaml %}
name: Build Docker images

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Setup QEMU
        id: qemu
        uses: docker/setup-qemu-action@v1.0.1
        with:
          platforms: linux/amd64,linux/arm/v6

      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v1.0.4

      - name: Login to Docker Hub
        uses: docker/login-action@v1.8.0
        with:
          username: {% raw %}${{ secrets.DOCKER_HUB_USERNAME }}{% endraw %}
          password: {% raw %}${{ secrets.DOCKER_HUB_TOKEN }}{% endraw %}

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          platforms: linux/amd64,linux/arm/v6
          push: true
          tags: <repo>/<project>:latest

      - name: Image digest
        run: echo {% raw %}${{ steps.docker_build.outputs.digest }}{% endraw %}
{% endhighlight %}

* The `name` directive simply gives the workflow a name.

* `on` defines the conditions in which the action will be executed: 
  * `push` - Defines that the workflow will be executed every time there's 
    a push to the `main` branch (If it's an older repository, it probably 
    uses the `master` branch).
  * `workflow_dispatch` - Allows manual execution of the workflow, in the 
    `Actions` tab, in the project page.

* `jobs` defines the jobs to be executed and it's parameters:
  * `build` - is the name of the job being defined.
  * `runs-on` - defines the distribution used in the virtual machine that 
    will execute the action:
    * I've chosen `ubuntu-18.04`, since it's a mature and stable version, 
      though I might consider updating it soon.
  * `steps` - Steps to be executed.
    * `name` - Step name.
    * `id` - ID of the step. Allows for it to be referenced in other parts 
      of the config file.
    * `uses` - Repository where the action is defined.
    * `with` - Parameters passed to the action.

#### Actions Description

* [`action/checkout`][checkout-action] - Checkouts the repository into the VM.

* [`docker/setup-qemu-action`][setup-qemu] - Setups QEMU:
  * `platforms` defines the platforms to be configured:
    * In this case, I've opted for `linux/amd64` and `linux/arm/v6`.
    * For a complete list of supported platforms, check out 
      [the action repository][setup-qemu].

* [`docker/setup-buildx-action`][setup-buildx] - Setups `buildx`.

* [`docker/login-action`][login-action] - Login on Docker Hub:
  * This action will allow pushing the generated image directly to Docker Hub.
  * Since this config file is commited to the repository, it's not a good 
    idea to store credentials in plaintext, **even if it's a private 
    repository**.
  * To store credentials safely, it's possible to use [GitHub Secrets]
    [gh-secrets] and reference them through variables prefixed with `secrets.`.
  
* [`build-push-action`][build-push-action] - Generates the Docker Image:
  * Once again, platforms are specified in `platforms`.
  * `tags` specifies which tag is applied to generated images:
    * I've chosen only the `:latest` tag, which is overriden every time a 
      new image is generated.
    * I haven't explored much yet, but it's probably possible to use 
      variables to assign the same tag of the commit being processed.

* The `run` directive allows for the execution of a shell command on the VM:
  * In this case, the `echo` prints the hash of the image generated in the 
    previous step. It's an example on how to use the IDs to refer to 
    previous steps (`steps.<step_id>`).
* To discover other available actions, please take a look at the [GitHub
Marketplace][gh-marketplace].

[build-push-action]: https://github.com/docker/build-push-action
[buildx-repo-multi]: https://github.com/docker/buildx/#building-multi-platform-images
[buildx-repo]: https://github.com/docker/buildx/
[checkout-action]: https://github.com/actions/checkout
[docker-hub]: https://hub.docker.com
[finding-actions]: https://docs.github.com/pt/actions/learn-github-actions/finding-and-customizing-actions?learn=getting_started&learnProduct=actions
[gh-actions]: https://github.com/features/actions
[gh-marketplace]: https://github.com/marketplace?type=actions
[gh-secrets]: https://docs.github.com/pt/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository
[login-action]: https://github.com/docker/login-action
[qemu]: https://www.qemu.org/
[rasp-pi-docker]: https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/
[run-syntax]: https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#jobsjob_idstepsrun
[setup-buildx]: https://github.com/docker/setup-buildx-action
[setup-qemu]: https://github.com/docker/setup-qemu-action
