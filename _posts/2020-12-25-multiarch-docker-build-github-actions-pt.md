---
layout: post
comments: true
title: Construindo Imagens Docker Multi Plataforma através do GitHub Actions
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
utilizando o GitHub Actions para automatizar o build das imagens em multiplas
plataformas e disponibilizando elas no [Docker Hub][docker-hub].

<!--more-->

## Docker no Raspberry Pi

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
containers é _extremamente_ demorado.

Em um dos casos, passei mais de 20 minutos esperando o comando concluir, apenas
para ver ele sendo cancelado devido a um problema temporário de conexão com os
repositórios do NPM.

## Builds Multi Plataforma

A solução que encontrei para este problema foi atravéz do comando
[`buildx`][buildx-repo], que já vem disponível na CLI a partir da versão
`19.03` do Docker.

Este commando permite a criação de _builders_, capazes de construirem imagens
a partir de um `Dockerfile`. Este commando, aliado ao [QEMU][qemu], é capaz de
[gerar imagens em multiplas plataformas][buildx-repo-multi], incluindo `arm/v6`
e `arm/v7`, que são as arquiteturas suportadas pelo Raspberry Pi.

Testei este método gerando uma imagem e armazenando ela num arquivo `.tar`:

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

## Build automático através do GitHub Actions

O procedimento apresentado até então já é suficiente para cobrir a utilização
básica, no entanto, como ele é manual, fica um pouco chato para um
desenvolvimento contínuo de um projeto.

Uma maneira de resolver isto é automatizar e incluir este procedimento nas
rotinas de Integração Contínua (CI). O [GitHub Actions][gh-actions] é perfeito
para isto, sendo extremamente fácil de configurar e proporcionando uma
excelente integração com o repositório do projeto.

O GitHub Actions nos permite configurar **Workflows**. Estes representam uma
lista de ações que serão executadas em sequência. Estes são definidos através
de arquivos `.yml`, definidos no diretório `.github`, na raíz do repositório:

{% highlight terminal %}
$ mkdir -p .github/workflows
$ touch docker.yml
{% endhighlight %}

Conteúdo do arquivo:

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

* A primeira diretiva `name` atribui um nome ao Workflow.

* A diretiva `on` define as condições nas quais o workflow será executado:
  * `push` - Define que o workflow será executado toda vez que houver push no
    branch `main` (Se o repositório for mais antigo, é bem provável que este
    seja o `master`).
  * `workflow_dispatch` - Permite executar o workflow manualmente, através da
    aba `Actions` na página principal do projeto.

* Diretiva `jobs` define as tarefas a serem executadas e os seus parâmetros:
  * `build` - o nome do job a ser definido.
  * `runs-on` - permite escolher qual distribuição será utilizada na 
    máquina virtual (provisionada automaticamente) que executará as ações.
    * Optei pelo `ubuntu-18.04` pois é uma versão bem madura e estável.
  * `steps` - passos a serem executados.
    * `name` - Nome do passo.
    * `id` - ID do passo. Permite ser referenciado no restante do arquivo de
      configuração. 
    * `uses` - Repositório da ação a ser executada.
    * `with` - Parâmetros a serem passados à ação.
  
#### Descrição das ações:
    
* [`action/checkout`][checkout-action] - Faz um checkout do repositório na VM.

* [`docker/setup-qemu-action`][setup-qemu] - Faz o setup do QEMU:
  * `platforms` define as plataformas a serem configuradas no QEMU.
    * No meu caso, optei por `linux/amd64` e `linux/arm/v6`.
    * Para uma lista completa de plataformas suportadas, consulte o 
      [repositório da ação no GitHub][setup-qemu].

* [`docker/setup-buildx-action`][setup-buildx] - Faz o setup do `buildx`.

* [`docker/login-action`][login-action] - Fazer o login no Docker Hub:
  * Permitirá fazer um _push_ da imagem gerada para o repositório da Docker,
    definido no próximo passo.
  * Como este arquivo estará commitado no nosso repositório, não é uma boa 
    ideia simplesmente colocar credenciais em plain text, **mesmo que o 
    repositório seja privado**!
  * Para fazer isto de maneira segura, é possível adicionar as credenciais 
    como [Segredos (_Secrets_) no GitHub][gh-secrets] e referenciá-las 
    através de variáveis prefixadas em `secrets.`.

* [`build-push-action`][build-push-action] - Gera a imagem:
  * Mais uma vez, as plataformas são específicadas em `platforms`.
  * `tags` especifica uma tag a ser aplicada às imagens geradas:
    * Utilizo apenas a tag `:latest`, que será sempre sobrescrita (Ainda não
      explorei muito, mas imagino que seja possível utilizar variáveis para 
      aplicar a mesma tag definida no repositório Git).

* A diretriz `run` serve para executar um comando diretamente no shell da VM.
  Neste caso, o `echo` serve para imprimir o hash da imagem gerada no passo 
  anterior. Serve como exemplo de como se referir a uma etapa anterior 
  (`steps.<id_do_passo>`).

Para descobrir outras ações disponíveis, é possível consultar o [GitHub 
Marketplace] [gh-marketplace].

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
