---
layout: post
comments: true
title: Configurando um WiFi AccessPoint em um PC com Linux
date: 2014-10-29 12:00:00
lang: pt
ref: wifi-ap
categories: Linux
tags:
- Networking
- Linux
---

O **hostapd** é um daemon que faz toda a funcionalidade de acesspoint e
autenticação WiFi.

É uma ferramenta muito completa que implementa a `IEEE 802.11` para
gerenciamento do AP, autenticação `802.1X/WPA/WPA2/EAP`, entre outros. Para uma
lista completa de features é possível acessar o [site do projeto.][hostapd_page]

É importante dizer que o funcionamento do `hostapd` para nosso propósito vai
depender da placa de rede sem fio utilizada. Esta precisa ser capaz de entrar
em modo `ap`

## Verificando que a placa suporta o modo AP ##

...

## Instalação ##

Numa distribuição baseada em Debian, é possível instalar o `hostapd` através do
`apt-get`.

Além do `hostapd`, também necessitamos de um servidor DHCP

```
sudo apt-get install dhcp3-server hostapd;
```

## Configuração do hostapd ##

Editar o arquivo `/etc/hostapd/hostapd.conf`:

```
interface=wlan0
driver=nl80211
ssid=MyAP
hw_mode=g
channel=11
wpa=1
wpa_passphrase=MyPasswordHere
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_ptk_rekey=600
```

Para testar as configurações, rodamos antes o `hostapd` com debugs habilitados:

```
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

## Configuração do DHCP ##

Editamos o arquivo `/etc/dhcp3/dhcpd.conf`:

```
subnet 10.10.0.0 netmask 255.255.255.0 {
        range 10.10.0.25 10.10.0.50;
        option domain-name-servers 8.8.4.4, 208.67.222.222;
        option routers 10.10.0.1;
}
```

Modify /etc/default/dhcp3-server:

```
INTERFACES="wlan0"
```

Configure /etc/network/interfaces:

```
iface wlan0 inet static
 address 10.10.0.1
 netmask 255.255.255.0
```

Allow ip masquerading: (Run with 'sudo su')

```
echo "1" > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Fazendo tudo iniciar após o boot ##

Editar o arquivo `/etc/default/hostapd` e inserir os parâmetros:

```
RUN_DAEMON="yes"
DAEMON_CONF="/etc/hostapd/hostapd.conf"
DAEMON_OPTS="-dt"
```

Criar script `.sh` contendo os seguintes comandos:

```
- IP Masquerading
```

Editar o arquivo `/etc/network/interfaces` e utilizar o parâmetro `post-up`,
que serve para executar um comando após a interface ficar up:

```
post-up <script>.sh
```

## Conclusão ##

A disvantagem de utilizar o computador como um ponto de acesso é que este só
vai funcionar quando a máquina estiver online. Isto, no entanto, deixa a ideia
de utilizar o `hostapd` em um RaspberryPi com um adaptador USB<->WiFi.

Também é possível enxergar uma aplicação utilizando mais regras de `iptables`,
ou até um proxy, de forma a criar uma rede wireless separada para visitas.
Poderiam ser criadas regras para bloquear acesso a outros dispositivos na rede,
fazer traffic shaping/QoS e até limitar a utilização à alguns poucos sites.

Um outro detalhe a ser ressaltado é a facilidade em monitorar o tráfego dos
dispositivos conectados. Basta dar um wireshark na interface `wlan0` e todo o
tráfego está la.

[hostapd_page]: http://w1.fi/hostapd/
