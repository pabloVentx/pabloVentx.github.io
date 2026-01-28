---
layout: single
title: Lofi - TryHackMe
excerpt: "En esta máquina sencilla aprendemos a como funciona la vulnerabilidad *LFI* **(Local File Inclusion)**, para conseguir la bandera."
date: 2026-01-25
classes: wide
header:
  teaser: /assets/images/thm-writeup-lofi/logo_lof.png
  teaser_home_page: true
  icon: /assets/images/tryhackme.webp
categories:
  - TryHackMe
  - Pentesting
tags:
  - writeup
  - HackingWeb
  - LFI
---

![](/assets/images/thm-writeup-lofi/logo_lof.png)

"En esta máquina sencilla aprendemos a como funciona la vulnerabilidad *LFI* **(Local File Inclusion)**, para conseguir la bandera."

## WRITE UP
Ejemplo: 10.66.128.249 (víctima) TTL=63 Linux


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Lofi</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat**

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/thm-writeup-lofi/which_lof.png)


-Usar el comando *ping* <span style="color:red">10.66.128.249</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, nos muestra el proceso.*

![](/assets/images/thm-writeup-lofi/conectividad_lof.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">10.66.128.249</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span> 
*Veremos porque el formato grepeable, es importante.*

![](/assets/images/thm-writeup-lofi/nmap1_lof.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/thm-writeup-lofi/extractp_lof.png)


**nmap** <span style="color:yellow">-sVC</span>  <span style="color:red">10.66.128.249</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcar lenguaje java para verlo mejor*

![](/assets/images/thm-writeup-lofi/nmap2_lof.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">targeted</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java*

![](/assets/images/thm-writeup-lofi/batcat_lof.png)


#### ANÁLISIS WEB: Lo-Fi Music

Como vemos un *whatweb*, vamos a probar hacer un pequeño reconocimiento a nivel web. Nos muestra la arquitectura y demás, o podemos usar la extensión **wappalyzer**, lo mismo pero por gráfico.

![](/assets/images/thm-writeup-lofi/whatweb_lof.png)


Veremos una página de música, aparentemente no vulnerable, vemos que hay varios enlaces que nos llevan al apartado que le demos: **Games,Relax...**

![](/assets/images/thm-writeup-lofi/pag_log.png)


Si le pinchamos a uno vemos que la URL se ve así -> http://10.66.128.249/?page=Games.php
/**?page=** nos da una grave advertencia sobre  **LFI**.

**page**<span style="color:orange">=</span>Game.php

![](/assets/images/thm-writeup-lofi/pag2_lof.png)

#### EXPLOTACIÓN VULNERABLE: LOCAL FILE INCLUSION

>cuando la aplicación web no valida correctamente la entrada del usuario, permitiendo a un atacante manipular las rutas de archivos mediante técnicas como "path traversal" (navegación por directorios, ej: `../`).

**page**<span style="color:orange">=</span><span style="color:lightblue">../../../../etc/</span><span style="color:grey">passwd</span>

![](/assets/images/thm-writeup-lofi/lfi_lof.png)

"**CTRL + U**" para ver mejor desde el código fuente. Usuarios con bash solo tenemos a <span style="color:blue">root</span>.

![](/assets/images/thm-writeup-lofi/lfimejor_lof.png)


Podríamos intentar robar las *claves de ssh* de los usuarios, pero no se puede, aparte de que root es privilegiado.

![](/assets/images/thm-writeup-lofi/claves_lof.png)


#### BANDERA | RESUELTO

Para leer la bandera **../../../../flag.txt** -> mostrándonos el contenido.

![](/assets/images/thm-writeup-lofi/flag_lof.png)
