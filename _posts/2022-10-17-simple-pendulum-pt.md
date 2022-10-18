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

A biblioteca [PyMunk][pymunk-page] é uma engine de simulação Física para Python
muito interessante! Ela é perfeita para simular corpos rígidos em 2D e suas
interações, como colisões.

Neste post irei aliar ela à [pyglet][pyglet-page] (biblioteca para criação de
jogos e aplicações visuais) para demonstrar como criar a simulação de um
Pêndulo Simples interativo.

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

Note que é utilizada uma instância de [`Vec2d`][vec2d-ref]. Esta é uma classe
da própria `pymunk` para representar Vetores 2D, no formato `Vec2d(x, y)`.
Neste caso, há somente uma componente vertical para esta aceleração, apontando
para baixo (negativa), com valor de `9807 mm/s²`, equivalente a aceleração da
gravidade na Terra.

> Vale comentar aqui que a biblioteca `pymunk` **é agnostica em relação a
> unidades físicas**! Não interessa para ela qual unidade está utilizando em
> suas medidas; Se passar um valor em `s` para uma função que espera tempo e um
> valor em `mm` para uma função de distância ou posição, então todos os
> cálculos serão feitos nestas unidades.
>
> Unidades derivadas, como velocidade e aceleração, são calculadas a partir da
> combinação das outras unidades.

A gravidade foi definida em `mm/s²` já que definiremos posições e distâncias em
`mm`.

Na sequência iremos construir nosso modelo e definir os corpos e formas que o
compõem. Começamos por criar uma nova classe:

{% highlight python %}
class Pendulum:
    """Fixed Pendulum PyMunk Model."""

    MASS = 0.100  # g
    ACCELERATION = 100  # mm/s²

    def __init__(self, space: pymunk.Space):
        self.space = space

        self._create_entities()
{% endhighlight %}

Definimos as constantes `MASS` e `ACCELERATION` que utilizaremos na sequência e
no `__init__` recebemos a instância de `Space` utilizada na simulação.

Invocamos o método privado `_create_entities`, onde iremos criar as entidades
do modelo:

{% highlight python %}
    def _create_entities(self) -> None:
        """Create the entities that form the Pendulum."""
        self.static_body = pymunk.Body(body_type=pymunk.Body.STATIC)
        self.static_body.position = (360, 360)

        moment = pymunk.moment_for_circle(
            mass=self.MASS, inner_radius=0, outer_radius=10.0
        )
        self.circle_body = pymunk.Body(mass=self.MASS, moment=moment)
        self.circle_body.position = (360, 50)

        circle_shape = pymunk.Circle(body=self.circle_body, radius=10.0)

        rod_joint = pymunk.constraints.PinJoint(
            a=self.static_body,
            b=self.circle_body,
        )

        self.space.add(
            self.static_body, self.circle_body, circle_shape, rod_joint
        )
{% endhighlight %}

A primeira entidade é chamada `static_body`, ou "corpo estático". Se trata de
um ponto fixo no espaço, onde iremos fixar nosso pêndulo.

> A classe `pymunk.Body` é um dos conceitos básicos da biblioteca. Ela contêm
> todas as propriedades físicas do objetos (massa, posição, rotação,
> velocidade, etc...). No entanto, ela não define uma forma por si só.

Este ponto é definido na posição `(360, 360)`, bem no centro da tela
(considerando as constantes `WIDTH` e `HEIGHT`, definidas em
`SimulationWindow`). Para que tenhamos uma simulação em `mm`, podemos assumir
uma equivalência de 1:1 entre `px` e `mm`.

Na sequência, definimos o corpo para o Círculo que ficará na ponta do pêndulo.
Antes de instanciar [`pymunk.Body`][body-ref] para ele, é necessário calcular o
**momento de inercia**, um dos argumentos necessários para se criar um corpo
dinâmico, através da função `pymunk.moment_for_circle`.

Diferente do corpo estático, não passamos um valor de `body_type` para o
círculo porque, por padrão, os corpos são criados com o tipo
`pymunk.Body.DYNAMIC`.

Desta vez, também criamos umas instância da classe `pymunk.Circle`, subclasse
de [`pymunk.Shape`][shape-ref]. Esta irá criar uma forma de um círculo,
associado ao corpo que criamos anteriormente.

> A principal utilidade das `Shape`s no PyMunk é realizar cálculo de colisões.
> Isso não será tão importante para nossa simulação, porém também é utilizada
> para gerar os gráficos (sprites) que serão utilizados no Pyglet.

E finalmente definimos uma [`pymunk.constraints.PinJoint`][pin-joint-ref] entre
o `static_body` e o `circle_body`.

> Constraints são entidades utilizadas para **restringir** o
> movimento/comportamento dos corpos físicos. Existem diversas constraints,
> cada uma com um comportamento diferente.
>
> A `PinJoint`, em específico, mantêm uma distância constante entre 2 objetos,
> e é utilizada para formar a haste de nosso pêndulo.
>
> Para mais informações, [consultar a referência][constraints-ref].

Com todas as entidades criadas, utilizamos `space.add` para adicionar elas ao
nosso espaço simulado.

[body-ref]: http://www.pymunk.org/en/latest/pymunk.html#pymunk.Body
[constraints-ref]: http://www.pymunk.org/en/latest/pymunk.constraints.html
[pin-joint-ref]: http://www.pymunk.org/en/latest/pymunk.constraints.html#pymunk.constraints.PinJoint
[pyglet-page]: https://pyglet.org/
[pymunk-page]: http://www.pymunk.org/en/latest/
[shape-ref]: http://www.pymunk.org/en/latest/pymunk.html#pymunk.Shape
[vec2d-ref]: http://www.pymunk.org/en/latest/pymunk.html#pymunk.Vec2d