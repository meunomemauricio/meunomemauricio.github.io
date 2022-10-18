---
layout: post
comments: true
title: Simulação de Pêndulo Simples usando Pyglet e PyMunk
date: 2022-10-17 21:50:00
lang: pt
ref: simple-pendulum
categories: Simulation
tags:
- Python
- Pyglet
- PyMunk
- Simulation
- Physics
---

A biblioteca [PyMunk](http://www.pymunk.org/en/latest/) é uma engine de
simulação Física para Python muito interessante! Ela é perfeita para simular
corpos rígidos em 2D e suas interações, como colisões.

Neste post irei aliar ela à [pyglet](https://pyglet.org/) (biblioteca para
criação de jogos e aplicações visuais) para demonstrar como criar a simulação
de um Pêndulo Simples interativo.

<!--more-->

O primeiro passo é instalar as dependências. Para isso, crio um novo ambiente
virtual (estou utilizando a versão **3.10.8** do Python):

{% highlight terminal %}
$ python -m venv venv --prompt pendulum
$ source venv/bin/activate

(pendulum)$ pip install pyglet==1.5.27 pymunk==6.2.1
{% endhighlight %}

Irei criar um modulo chamado `simple_pendulum.py` e começarei importando as
bibliotecas:

{% highlight python %}
import pymunk
import pyglet

from pyglet.window import key
from pymunk import Vec2d
from pymunk.pyglet_util import DrawOptions
{% endhighlight %}

Como base mínima deste programa, irei declarar uma nova classe que herda de
`pyglet.window.Window` e executá-la como uma app do Pyglet.

A classe `Window` aceita alguns argumentos para ajustar parâmetros da tela.
Como utilizarei parâmetros fixos neste exemplo, sobrescrevo o `__init__` e
invoco o `__init__` do `super()`, passando estes parâmetros que vêm dos
atributos de classe.

{% highlight python %}
class SimulationWindow(pyglet.window.Window):
    CAPTION = "Fixed Pendulum Simulation."

    WIDTH = 720
    HEIGHT = 720

    def __init__(self):
        super().__init__(
            width=self.WIDTH, height=self.HEIGHT, caption=self.CAPTION
        )


if __name__ == "__main__":
    SimulationWindow()
    pyglet.app.run()
{% endhighlight %}

Executar este módulo deve abrir uma nova janela vazia:

<!-- TODO: Add fixed_pendulum/empty_window.jpg -->

Na sequência, expando o `__init__` para criar uma nova instância da classe
`pymunk.Space`.

{% highlight python %}
    def __init__(self):
        super().__init__(
            width=self.WIDTH, height=self.HEIGHT, caption=self.CAPTION
        )

        self.space = pymunk.Space()
        self.space.gravity = Vec2d(0, -9807)  # mm/s²
{% endhighlight %}

O Espaços são a unidade básica de simulação. Os corpos (*bodies*), formas
(*shapes*) e junções (*joints* ou *constraints*) são adicionados ao espaço e
estes todos são simulados em conjunto, ao longo do tempo.

Após instanciar o espaço, aproveito e já defino a gravidade utilizada na
simulação, que é tratada como uma aceleração comum a todos os corpos.

Note que é utilizada uma instância de `Vec2d`. Esta é uma classe da
própria `pymunk` para representar Vetores 2D, no formato `Vec2d(x, y)`. Neste
caso, há somente uma componente vertical para esta aceleração, apontando para
baixo (negativa), com valor de `9807 mm/s²`, equivalente a aceleração da
gravidade na Terra.

> Vale comentar aqui que a biblioteca `pymunk` **é agnostica em relação a
> unidades físicas**! Não interessa para ela qual unidade está utilizando em
> suas medidas; Se passar um valor em `s` para uma função que espera tempo e um
> valor em `mm` para uma função de distância ou posição, então todos os
> cálculos serão feitos nestas unidades.
>
> Unidades derivadas, como velocidade e aceleração, são calculadas a partir da
> combinação das outras unidades.

TBC...