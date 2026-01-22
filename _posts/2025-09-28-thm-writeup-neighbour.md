---
layout: single
title: Neighbour - TryHackMe
excerpt: "En esta máquina tendremos un login, cual no dispondremos de ninguna credencial. En el código fuente de la página habra unas credenciales de invitado, accediendo. Miraremos el código fuente nuevamente, donde nos dira que el admin puede ser vulnerable, y en la URL tendremos =guest, acontenciendo la vulnerabilidad IDOR (Insecure Direct Object Reference) cambiando en el parámetro de la URL el usuario para convertirnos en otro usuario."
date: 2025-09-28
classes: wide
header:
  teaser: /assets/images/thm-writeup-neighbour/logo_neigh.png
  teaser_home_page: true
  icon: /assets/images/tryhackme.webp
categories:
  - TryHackMe
  - Pentesting
tags:
  - writeup
  - HackingWeb
  - InformationLekeage
  - IDOR
---

![](/assets/images/thm-writeup-neighbour/logo_neigh.png)

"En esta máquina tendremos un login, cual no dispondremos de ninguna credencial. En el código fuente de la página habra unas credenciales de invitado, accediendo. Miraremos el código fuente nuevamente, donde nos dira que el admin puede ser vulnerable, y en la URL tendremos =guest, acontenciendo la vulnerabilidad IDOR (Insecure Direct Object Reference) cambiando en el parámetro de la URL el usuario para convertirnos en otro usuario."

## WRITE UP
IP VÍCITMA-> 10.10.208.50 (TTL 63) LINUX


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">neighbour</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat** -l python

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/thm-writeup-neighbour/which_neigh.png)


-Usar el comando *ping* <span style="color:red">10.10.208.50</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, nos muestra el proceso.* 

![](/assets/images/thm-writeup-neighbour/conectividad_neigh.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">10.10.208.50</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span>
*Veremos porque el formato grapeable, es importante.*

![](/assets/images/thm-writeup-neighbour/nmap1_neigh.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/thm-writeup-neighbour/extractpor_neigh.png)


**nmap** <span style="color:yellow">-sVC</span>  <span style="color:yellow">-p</span><span style="color:purple">22,80</span> <span style="color:red">10.10.208.50</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcat lenguaje java para verlo mejor*
*Nos mostrará la versión de los servicios que están corriendo*
*Usará scripts defaults*

![](/assets/images/thm-writeup-neighbour/nmap2_neigh.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">target</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java

![](/assets/images/thm-writeup-neighbour/batcat_neigh.png)


#### ANÁLISIS WEB: RECONOCIMIENTO

Haremos un pequeño análisis a nivel web con *whatweb*.
Nos dirá que hay un formulario y una cookie de sesión.

**whatweb** http://10.10.208.50/

![](/assets/images/thm-writeup-neighbour/whatweb_neigh.png)


Entramos a la página web y estaremos en un formulario. Vamos a revisar el código fuente.
"**CTRL + U**"

![](/assets/images/thm-writeup-neighbour/log_neigh.png)


Nos dice que podemos entrar como invitados <span style="color:blue">guest</span>:<span style="color:darkred">guest</span>  y nos dan el nombre <span style="color:blue">admin</span>.

![](/assets/images/thm-writeup-neighbour/amdinme_neigh.png)


Probamos a entrar como invitado.

![](/assets/images/thm-writeup-neighbour/logges_neigh.png)



Nos deja y revisamos el código fuente.
Dice que la cuenta admin puede ser vulnerable.

![](/assets/images/thm-writeup-neighbour/admin_neigh.png)


#### VULNERABILIDAD IDOR (Insecure Direct Object Reference)

Vemos en la URL el nombre <span style="color:orange">=</span><span style="color:blue">guest</span>, pues lo cambiamos a <span style="color:orange">=</span><span style="color:blue">admin</span>.
Nos convertiremos en el, obteniendo la bandera.

> cuando una aplicación web permite acceder directamente a objetos internos (IDs, archivos, registros) sin verificar correctamente la autorización del usuario. Esto permite que un atacante manipule parámetros y acceda a datos o recursos de otros usuarios.
> ARTÍCULO: [Rinku IDOR](https://rinku.tech/idor/)

![](/assets/images/thm-writeup-neighbour/flag_neigh.png)
