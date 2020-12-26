---
layout: post
comments: true
title: Construindo Imagens Docker Multi Plataforma através do Github Actions
date: 2020-12-25 23:25:00
lang: pt
ref: multiarch-docker-gh-actions
categories: Docker
tags:
- CI
- Docker
- GitHub Actions
- Raspberry Pi  
excerpt_separator: <!--more-->
---

Tentando configurar um serviço utilizando Docker em um Raspberry Pi, deparei-me
com tempos de build astronómicos (parei de contar após 20 minutos), muitas
vezes resultando em erros por incompatibilidade ou indisponibilidade de rede.

Este post apresenta uma solução completa e automatizada para este problema,
utilizando o Github Actions para automatizar o build das imagens em multiplas
plataformas e disponibilizando elas no [Docker Hub][docker-hub].

<!--more-->

## Docker no Raspberry Pi ##

O Docker é uma excelente ferramenta para criação e gerenciamento de containers,
simplificando muito o processo de setup de uma topologia de um projeto e todas
suas dependencias (DB, cache, server de arquivos estáticos, etc...).

Em 2016, foi anunciado no [Blog do Raspberry Pi][rasp-pi-docker] de que a nova
versão do `raspbian/jessie` contaria com suporte oficial ao Docker, permitindo
então que este fosse instalado com relativa facilidade:

{% highlight terminal %}
$ curl -sSL https://get.docker.com | sh
{% endhighlight %}

No entanto, devido à capacidade de processamento do Raspberry Pi, especialmente
nas versões de hardware mais antigas, o processo de construir as imagens dos
containers é *extremamente* demorado.

Em um dos casos, passei mais de 20 minutos esperando o comando concluir, apenas
para ver ele sendo cancelado devido a um problema temporário de conexão com os 
repositórios do NPM.

## Builds Multi Plataforma ##

A solução que encontrei para este problema foi atravéz do comando
[`buildx`][buildx-repo], que já vem disponível na CLI a partir da versão
`19.03` do Docker.

Este commando permite a criação de *builders*, capazes de construirem imagens
a partir de um `Dockerfile`. Este commando, aliado ao [QEMU][qemu], é capaz de
[gerar imagens em multiplas plataformas][buildx-repo-multi], incluindo `arm/v6`
e `arm/v7`, que são as arquiteturas suportadas pelo Raspberry Pi.

Testei este método gerando uma imagem e armazenando ela em um arquivo `.tar`:

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

Depois de pronta, a imagem pode ser copiada para o Raspberry Pi e carregada
através dos comandos:

{% highlight terminal %}
$ docker load -i myproject.tar

# Tag the image to make it easier to reference
$ docker image tag <SHA digest> myproject:latest
{% endhighlight %}

## Build automático através do Github Actions ##

TBC...

## Referencias ##

[buildx-repo]: https://github.com/docker/buildx/
[buildx-repo-multi]: https://github.com/docker/buildx/#building-multi-platform-images
[docker-hub]: https://hub.docker.com
[qemu]: https://www.qemu.org/
[rasp-pi-docker]: https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/