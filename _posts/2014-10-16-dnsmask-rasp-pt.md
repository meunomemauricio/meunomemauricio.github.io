---
layout: post
comments: true
title: Configurando Servidor DNS e DHCP no Raspberry Pi (dnsmasq)
date: 2014-10-16 12:00:00
lang: pt
ref: dnsmasq-rasp
categories: Linux
tags:
- Raspberry Pi
- Networking
- Linux
excerpt_separator: <!--more-->
---

O [DNSMasq][dnsmasq] é um serviço que combina DNS e DHCP de maneira elegante e
muito fácil de configurar.  O foco dele é consumir poucos recursos de memória e
espaço em disco, portanto é indicado à redes mais simples e com poucos hosts.

Devido à esta característica, é muito comum vê-lo sendo utilizado em
distribuições Linux para roteadores básicos, como é o caso do
[OpenWrt][openwrt] ou o [DD-WRT][ddwrt]. Pelo mesmo motivo, ele é perfeito para
ser executado no [Raspberry Pi][rasppi].

Nesse post eu irei demonstrar e explicar uma configuração básica que fiz para
minha rede de casa.

<!--more-->

O DNSMasq está disponível como um pacote `.deb` para a distribuição
[Raspbian][raspbian] e portanto pode ser instalado através do apt-get:

{% highlight terminal %}
$ sudo apt-get install dnsmasq
{% endhighlight %}

O arquivo de configuração principal fica em `/etc/dnsmasq.conf`. É possível
realizar toda a configuração direto nele, no entanto, recomendo utilizar o
diretório `/etc/dnsmasq.d` por questão de organização. Para isso, basta retirar
o comentário na ultima linha do arquivo `/etc/dnsmasc.conf`:

{% highlight ini %}
# Include a another lot of configuration options.
#conf-file=/etc/dnsmasq.more.conf
conf-dir=/etc/dnsmasq.d
{% endhighlight %}

A partir de então, todo arquivo de texto que for criado no `/etc/dnsmasq.d`
será utilizado como configuração do DNSMasq:

{% highlight terminal %}
$ sudo vi /etc/dnsmasq.d/network.conf
{% endhighlight %}

> Utilizei o nome `network.conf`, mas qualquer um é válido.

O primeiro passo é configurar um nome de domínio. A configuração a ser setada
é a `domain`. Este domínio ficará acessível apenas localmente e através deste
servidor DNS.

{% highlight ini %}
domain=raspberry.local
{% endhighlight %}

Por padrão, o DNSMasq carrega os servidores `nameserver` direto do arquivo
`/etc/resolv.conf` como servidores DNS. No entanto, é possível utilizar a opção
`no-resolv` para alterar este comportamento e poder configurar os servidores
externos de DNS manualmente:

{% highlight ini %}
no-resolv
server=208.67.222.222
server=208.67.220.220
{% endhighlight %}

Utilizei os servidores do [OpenDNS][opendns] ao invés do fornecido pelo meu
provedor de internet. O [GoogleDNS][googledns] também é uma boa opção.

> Para outras configurações do serviço de DNS, [consultar o manual do
> DNSMasq][manual]

Na sequencia, partimos para configurar o serviço de DHCP. Há a opção
`dhcp-range` para setar o range de IPs que serão distribuídos entre os clientes
da rede.

{% highlight ini %}
dhcp-range=10.0.0.50,10.0.0.59,120m
dhcp-option=option:router,10.0.0.254
dhcp-authoritative
{% endhighlight %}

Neste caso, o servidor irá anunciar o range de endereços de `10.0.0.50` até
`10.0.0.59`. É importante ajustar o range conforme a quantidade de dispositivos
que forem ser conectados à rede.

Também é importante reparar o valor *120m* configurado junto com o range, que
é o **lease time** para esse range.

> A configuração de **lease time** costuma ser algo que varia bastante conforme
> a aplicação. O valor mais usual é de 24h, que geralmente já é suficiente.
> Períodos mais curtos geralmente são utilizados em redes WiFi públicas, como
> em lojas/restaurantes, onde há uma grande rotatividade de clientes acessando
> a rede.

A opção `dhcp-option` permite configurar os options do pacote DHCP diretamente.
Utilizo ela para anunciar o endereço Gateway para os clientes, já que o DNSMasq
não possui uma opção específica para configurar o Gateway que será anunciado
aos clientes.

O primeiro argumento recebe o valor da opção [conforme a RFC 2132][rfc2132] (ou
essa [página da IANA][ianaopts]) diretamente ou o "enum" `option:router`. O
segundo argumento é o valor que será anunciado, no caso `10.0.0.254`, que é o
IP do modem/roteador.

Finalmente, a linha `dhcp-authoritative` serve para indicar que este é o
servidores principal da rede.

> Para mais opções de como configurar o DHCP, [consultar o manual do
> DNSMasq][manual]

Ao completar a edição dos arquivos de configuração, basta reiniciar o serviço
e está tudo pronto para ser utilizado:

{% highlight terminal %}
$ sudo service dnsmasq restart
{% endhighlight %}

### Hosts com endereço Fixo ###

Como é disponibilizado um range de IPs e estes tem um tempo limitado de lease,
é inevitável que mais cedo ou mais tarde o endereço fornecido a um dispositivo
seja diferente.

Na maioria dos casos isto não é um problema, porém em algumas situações é
importante estabelecer endereços fixos de IP para alguns elementos. Os casos
mais comuns são servidores de arquivos ou de impressão mas as vezes até caso
seja necessário realizar Port Forwarding no NAT.

Para fazer isto, basta criar uma entrada do tipo `dhcp-host`, por exemplo:

{% highlight ini %}
# My Computer
dhcp-host=AA:BB:CC:DD:EE:FF,mycomputer,10.0.0.1,infinite

# Print Server
dhcp-host=00:11:22:33:44:55,printer,10.0.0.2,infinite
{% endhighlight %}

Neste caso, o endereço `10.0.0.1` fica reservado para o dispositivo com
endereço MAC `AA:BB:CC:DD:EE:FF`. É possível fazer com que estes hosts fixos
possuam leases temporários também, no entanto é mais usual manter este lease
como **infinite**, já que será sempre o mesmo host.

> Notar que os endereços não estão dentro do outro range.

Outra característica muito interessante do DNSMasq é que ao especificar um nome
para o dispositivo, este já é incluso automaticamente no DNS. Então, das
proximas vezes que eu precisar acessar a impressora a partir de um PC, ao invés
de apontar para `10.0.0.2`, é possível acessar `printer.raspberry.local` (nome
de domínio local, configurado no início).

{% highlight terminal %}
~$ host printer.raspberry.local
printer.raspberry.local has address 10.0.0.2

~$ ping printer.raspberry.local
PING printer (10.0.0.2) 56(84) bytes of data.
64 bytes from printer (10.0.0.2): icmp_seq=1 ttl=64 time=1.45 ms
64 bytes from printer (10.0.0.2): icmp_seq=2 ttl=64 time=0.144 ms
 --- printer ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.144/0.799/1.454/0.655 ms
{% endhighlight %}

## Referências ##

1. [Man Page de configuração do DNSMasq](http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)
2. [Tutorial em ingles sobre como configurar o DNSMasq](http://www.heystephenwood.com/2013/06/use-your-raspberry-pi-as-dns-cache-to.html) (Focado em DNS Caching)
3. [Postagem no fórum do Raspberry Pi](http://www.raspberrypi.org/forums/viewtopic.php?t=46154)

[dnsmasq]: http://www.thekelleys.org.uk/dnsmasq/doc.html
[manual]: http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html
[rasppi]: http://www.raspberrypi.org/
[openwrt]: https://openwrt.org/
[ddwrt]: https://www.dd-wrt.com/site/index
[raspbian]: http://www.raspbian.org/
[opendns]: http://www.opendns.com/home-internet-security/opendns-ip-addresses/
[googledns]: https://developers.google.com/speed/public-dns/
[rfc2132]: https://www.ietf.org/rfc/rfc2132.txt
[ianaopts]: http://www.iana.org/assignments/bootp-dhcp-parameters/bootp-dhcp-parameters.xhtml
