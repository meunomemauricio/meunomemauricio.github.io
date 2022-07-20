---
layout: post
comments: true
title: Sprinkler Automático - O Sensor de Umidade do Solo
date: 2022-02-01 22:33:00
lang: pt
ref: sprinkler-moisture-sensor
categories: Home Automation
tags:
- Sensors
- Gardening
- Home Automation 
excerpt_separator: <!--more-->
---

Mais de uma vez, já me aconteceu de eu esquecer completamente de minhas plantas
ao sair para uma viagem de 1 ou 2 semanas. Era sempre um momento de muita 
tristeza chegar em casa depois e deparar que muitas delas haviam morrido ou,
mesmo quando recuperavam, nunca voltavam 100%.

Felizmente, da última vez que viajei, consegui lembrar e consegui pedir para a
minha mãe regá-las. No entanto, não pude deixar em pensar em alguma solução
mais autônoma para as próximas vezes.

Ao tentar planejar essa solução, o primeiro desafio foi: Como medir a 
umidade do solo?

<!--more-->

## O Sensor

Idealmente, a propriedade que realmente gostariamos de medir é o volume de água 
presente no solo. Infelizmente, para medir isto *diretamente*, é necessário
remover uma amostra do solo, pesá-lo, secá-lo e pesá-lo novamente, visando 
comparar a diferença para calcular quanta água foi perdida.

Como isto é completamente imprático para este propósito, devemos contar com 
uma medida *indireta*, ou seja, medir outra propriedade física que seja 
proporcional à umidade do solo.

Uma destas propriedades é a **resistividade** do solo.

A princípio, quando o solo está completamente seco, sua resistividade tende 
a ser elevada. Conforme ele vai sendo molhado, quanto maior for a concentração 
de água, sua resistividade tende a cair proporcionalmente.

    Confirmar o sentido da proporcionalidade.

    <grafico?>

Para nossa alegria, resistividade é uma das propriedades mais fáceis de se 
medir eletronicamente. Basta inserir 2 condutores no solo, ligar eles em 
série com um outro resistor de valor conhecido e aplicar uma tensão no 
circuito. Se trata, basicamente, de um circuito divisor de tensão:

    Diagrama do divisor de tensão

Seguindo as formulas, basicamente teremos que a resistividade do solo será:

    Formula divisor tensão

E este é exatamente o princípio de funcionamento deste sensor:

    Foto/Link do sensor resistivo

Basta ligar os terminais dele em um ADC e temos uma medida proporcional à 
umidade do solo.

No entanto, leitores mais atentos já devem ter percebido que esta solução 
tem um problema bem sério, que afeta severamente a longevidade deste 
tipo de sensor!

## Eletrólise

    ... Explicação sobre eletrólise

## Capacitância ao resgate

    ... Explicação sobre sensor capacitivo

