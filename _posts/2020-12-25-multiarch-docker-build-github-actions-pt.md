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

O procedimento apresentado até então já é suficiente para cobrir a utilização básica, no entanto, como ele é manual, fica um pouco chato para um desenvolvimento contínuo de um projeto.

Uma maneira de resolver isto é automatizar e incluir este procedimento nas rotinas de Integração Contínua (CI). O [Github Actions][gh-actions] é perfeito para isto, sendo extremamente fácil de configurar e proporcionando uma excelente integração com o repositório do projeto.

O Github Actions nos permite configurar `Workflows`. Estes representam um lista de ações que serão executadas em sequência. Estes são definidos através de arquivos `.yml`. No diretório raiz do repositório, execute:


{% highlight terminal %}
$ mkdir -p .github/workflows
$ touch docker.yml
{% endhighlight %}

Abra este novo arquivo criado em um editor de texto. Iniciamos este arquivo dando um nome para este Workflow e definindo as condições em que ele será executado:

{% highlight yaml %}
name: Build Docker images

on:
  push:
    branches: [ main ]
  workflow_dispatch:
{% endhighlight %}

Aqui definimos duas condições, `push` e `workflow_dispatch`. A primeira diz que o workflow será executado toda vez que for feito um push no branch `main` (novos projetos no Github são criados com o `main`. Se seu repositório é um pouco mais antigo, é bem provável que este branch seja o `master`). O `workflow_dispatch` nos permitirá executar este Workflow manualmente, através da aba `Actions` na pagina principal do Projeto.

Na sequencia, começamos a definir o procedimento de build:

{% highlight yaml %}
jobs:
  build:
    runs-on: ubuntu-18.04
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2
{% endhighlight %}

A instrução `runs-on` nos permite escolher qual distribuição será utilizada para executar nosso Workflow. Optei pelo `ubuntu-18.04` pois é uma versão bem madura e estável. Logo em seguinte, na seção `steps`, começamos definir os passos que serão executados. O primeiro deles é fazer um checkout de nosso repositório na máquina virtual.

TBC...

## Referencias ##

[buildx-repo]: https://github.com/docker/buildx/
[buildx-repo-multi]: https://github.com/docker/buildx/#building-multi-platform-images
[docker-hub]: https://hub.docker.com
[gh-actions]: https://github.com/features/actions
[qemu]: https://www.qemu.org/
[rasp-pi-docker]: https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/

