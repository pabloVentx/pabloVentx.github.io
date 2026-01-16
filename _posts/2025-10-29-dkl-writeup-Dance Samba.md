---
layout: single
title: Dance Samba - Dockerlabs
excerpt: "Esta máquina tendrá de objetivo de descubrir unas credenciales a nivel local mediante una nota cual nos dará acceso a un share."
date: 2025-10-01
classes: wide
header:
  teaser: /assets/images/dkl-writeup-dancesamba/logo_dancesamba.png
  teaser_home_page: true
  icon: /assets/images/dockerlabs.webp
categories:
  - Dockerlabs
  - Pentesting
  - eJPTv2
tags:
  - writeup
  - eJPTv2
  - SSHclaves
  - hash
  - Sudo
  - EscaladaPrivilegios
---

![](/assets/images/dkl-writeup-dancesamba/logo_dancesamba.png)

"Esta máquina tendrá de objetivo de descubrir unas credenciales a nivel local mediante una nota cual nos dará acceso a un share. En dicho share que es el home del usuario, estará la <span style="color:orange">flag del usuario</span>, e tendremos que subir nuestra llave publica para luego loguearnos en **ssh** usando la privada y no nos pedirá contraseña. Después, encontraremos un usuario secreto cual contiene un hash y lo descifraremos obteniendo la contraseña para el usuario inicial de ssh cual nos va a permitir listar nuestros permisos de sudo. Podremos leer archivos arbitrarios leyendo uno que contiene la contraseña de root.
Finalmente obteniendo la <span style="color:orange">flag de root</span> en su home."

## WRITE UP
IP VÍCITMA-> 172.17.0.2 (TTL 64) LINUX

### INTRODUCCIÓN

Esta máquina tendrá de objetivo de descubrir unas credenciales a nivel dominio mediante una nota cual nos dará acceso a un share.

En dicho share que es el home del usuario, estará la <span style="color:orange">flag del usuario</span>, e tendremos que subir nuestra llave publica para luego loguearnos en **ssh** usando la privada y no nos pedirá contraseña.

Después, encontraremos un usuario secreto cual contiene un hash y lo descifraremos obteniendo la contraseña para el usuario inicial de ssh cual nos va a permitir listar nuestros permisos de sudo.

Podremos leer archivos arbitrarios leyendo uno que contiene la contraseña de root.
Finalmente obteniendo la <span style="color:orange">flag de root</span> en su home.


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">dancesmb</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat** -l python

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*


Al ser dockerlabs, vamos a descargarnos el .zip y ejecutar el contenedor:

![](/assets/images/dkl-writeup-dancesamba/despdock_dancesamba.png)


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/dkl-writeup-dancesamba/whichsystem_dancesamba.png)


-Usar el comando *ping* <span style="color:red">172.17.0.2</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, nos muestra el proceso.* 

![](/assets/images/dkl-writeup-dancesamba/conectividad_dancesamba.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">172.17.0.2</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span>
*Veremos porque el formato grapeable, es importante.*

![](/assets/images/dkl-writeup-dancesamba/nmap1_dancesamba.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/dkl-writeup-dancesamba/extractports_dancesamba.png)


**nmap** <span style="color:yellow">-sVC</span>  <span style="color:yellow">-p</span><span style="color:purple">21,22,139,445</span> <span style="color:red">172.17.0.2</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcat lenguaje java para verlo mejor*
*Nos mostrará la versión de los servicios que están corriendo*
*Usará scripts defaults*

![](/assets/images/dkl-writeup-dancesamba/nmap2_dancesamba.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">target</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java

![](/assets/images/dkl-writeup-dancesamba/batcat_dancesamba.png)


#### ANÁLISIS FTP: RECONOCIMIENTO

En el escaneo de **nmap** gracias al uso de los scripts que usa, nos detectó un login en <span style="color:blue">anoymous</span>, por ende no necesita contraseña.

Vemos que tiene una nota.

![](/assets/images/dkl-writeup-dancesamba/foundanonymousftp_dancesamba.png)


**locate** .nse | **grep** <span style="color:green">-i</span> "ftp"
*En dicha ruta guarda los scripts, programados en LUA*

![](/assets/images/dkl-writeup-dancesamba/scriptsnmapftp_dancesamba.png)


Podríamos usar *searchsploit* para que nos busque alguna vulnerabilidad para la versión <span style="color:yellow">v3.0.5</span> de este servicio pero no nos va a encontrar nada.

![](/assets/images/dkl-writeup-dancesamba/searchespftp_dancesamba.png)


#### ACCESO FTP: ANONYMOUS

Antes de entrar, vamos a crear un fichero vacío para ver si tenemos privilegios de subida de archivos.

**touch** <span style="color:grey">prueba.txt</span>

![](/assets/images/dkl-writeup-dancesamba/ficherovacio_dancesamba.png)


**ftp** <span style="color:red">10.10.10.5</span> :<span style="color:blue">anonymous</span>
Nos descargamos la nota.

![](/assets/images/dkl-writeup-dancesamba/ftplogin_dancesamba.png)


Probamos la subida de archivos pero nos faltan permisos.

![](/assets/images/dkl-writeup-dancesamba/subidaftpfvac_dancesamba.png)


#### ANÁLISIS SSH: RECONOCIMIENTO

Con *searchsploit* comprobamos si existen vulnerabilidades de esta versión pero no nos encontrara nada.

![](/assets/images/dkl-writeup-dancesamba/searchespssh_dancesamba.png)


#### LECTURA DE ARCHIVO DESCARGADO

Leemos la nota que nos hemos descargado en busca información relevante.
Dos usuarios potenciales: <span style="color:blue">Macarena</span> y <span style="color:blue">donald</span>.

![](/assets/images/dkl-writeup-dancesamba/credftp_dancesamba.png)


#### ENUMERACIÓN: SMB

De primeras probaremos con <span style="color:blue">null sesión</span> para entrar a *rpcclient* y tener una posible enumeración de usuarios, grupos y información de usuarios.

![](/assets/images/dkl-writeup-dancesamba/rpccli_dancesamba.png)


Enumeramos los usuarios y nos listara <span style="color:blue">Macarena</span> con el **rid**: <span style="color:lime">0x3e8</span>
**enumdomusers**
![](/assets/images/dkl-writeup-dancesamba/rpcenumusers_dancesamba.png)

Haremos una consulta al rid del usuario para entre otras cosas, ver si guarda alguna información en su descripción. (nada)
**queryuser** <span style="color:lime">0x3e8</span>

![](/assets/images/dkl-writeup-dancesamba/riduser_dancesamba.png)


#### ANÁLISIS: SMB

Con *searchsploit* buscaremos si la versión <span style="color:yellow">v4</span> de **smb** tiene alguna vulnerabilidad, aparentemente no.

![](/assets/images/dkl-writeup-dancesamba/searchespsmb_dancesamba.png)


#### LISTAR SHARES

Listaremos los shares con *netexec* para ver cual tenemos permisos de lectura e incluso escritura con distintos usuarios, probaremos con <span style="color:blue">null sesión</span> primero.

Lo cual nos deja listarlos pero no nos dice nada de que podamos acceder o parecido.
Aunque vemos un share <span style="color:lightblue">macarena</span>, ya nos dice que seguramente sea el home de dicho usuario.

![](/assets/images/dkl-writeup-dancesamba/listshares1_dancesamba.png)


Usaremos *smbmap* y por alguna casualidad probaremos las siguientes credenciales haber si nos deja listarlos y así mismo permisos de lectura o escritura. 

**smbmap** <span style="color:green">-H</span>  <span style="color:red">172.17.0.2</span> <span style="color:green">-u</span> <span style="color:gold">"macarena"</span> <span style="color:green">-p</span> <span style="color:gold">"donald"</span>

Nos deja listarlos e nos dice que podemos escribir y leer el share <span style="color:lightblue">macarena</span>.
Esto nos da a entender de que el usuario del principio y con el que cual hemos listado los shares es válido dentro del dominio y ya sabemos su contraseña.

![](/assets/images/dkl-writeup-dancesamba/listshares2_dancesamba.png)


Lo comprobamos con **netexec** logueandonos y nos lo válida.

![](/assets/images/dkl-writeup-dancesamba/validuser_dancesamba.png)


#### ACCESO A SHARE Y BANDERA USER

Accederemos al recurso compartido con *smbclient*:

**smbclient**  -U "<span style="color:blue">macarena</span>"  \\\\<span style="color:red"><span style="color:red">172.17.0.2</span>  </span>\\<span style="color:lightblue">macarena</span> :<span style="color:darkred">donald</span>

Dentro del share que es el home de este usuario, cual como podemos escribir podemos subir archivos, crear carpetas... y asi mismo ver los archivos ocultos. Nos bajamos la <span style="color:orange">flag de user</span>.

![](/assets/images/dkl-writeup-dancesamba/smbcli_dancesamba.png)


Leemos la bandera usuario que nos hemos bajado.

![](/assets/images/dkl-writeup-dancesamba/flaguser_dancesamba.png)


#### GENERACIÓN DE LLAVES Y ACCESO A SSH 

Si intentamos entrar a **ssh** del usuario <span style="color:blue">macarena</span> nos dira que su identificación ha cambiado.

![](/assets/images/dkl-writeup-dancesamba/sshidch_dancesamba.png)


Para solucionar esto entraremos a su share y crearemos el directorio <span style="color:lightblue">/.ssh/</span>.

**mkdir** <span style="color:lightblue">/.ssh/</span>

![](/assets/images/dkl-writeup-dancesamba/mkdirssh_dancesamba.png)


Después en nuestra máquina eliminaremos el Host de confianza (conocido) que es la víctima en <span style="color:grey">known_hosts</span>:

**ssh-keygen** <span style="color:green">-f</span>  <span style="color:orange">'/home/pablo/.ssh/</span><span style="color:darkorange">known_hosts</span><span style="color:orange">'</span>  <span style="color:green">-R</span>  <span style="color:orange">'172.17.0.2'</span>

![](/assets/images/dkl-writeup-dancesamba/delhosts_dancesamba.png)


Ahora crearemos unas claves:

**ssh-keygen**  <span style="color:green">-t</span>  <span style="color:purple">rsa</span>  <span style="color:green">-b</span>  <span style="color:cyan">4096</span>

1) <span style="color:green">-t</span>  <span style="color:purple">rsa</span> (Este tipo es porque es compatible y cifrado)
2) <span style="color:green">-b</span>  <span style="color:cyan">4096</span> (Usaremos este tamaño en bits debido a que es el más usado en servidores)

![](/assets/images/dkl-writeup-dancesamba/genkeys_dancesamba.png)


clave privada -> <span style="color:purple">yes</span>
clave pública -> <span style="color:purple">yes.pub</span>


#### PASAR LLAVES A SHARE Y USARLA 

Dentro del share > <span style="color:lightblue">/.ssh/</span> llevaremos la clave pública y la llamaremos authorized_keys.

**put** <span style="color:purple">yes.pub</span> <span style="color:pink">authorized_keys</span>

![](/assets/images/dkl-writeup-dancesamba/llevarllaves_dancesamba.png)


Ahora nos queda usar en nuestra máquina nuestra clave privada y no nos pedirá contraseña.

**ssh** <span style="color:blue">macarena</span><span style="color:lime">@172.17.0.2</span>  <span style="color:green">-i</span>  <span style="color:purple">yes</span>

![](/assets/images/dkl-writeup-dancesamba/usarllaves_dancesamba.png)


#### CRACKEAR HASH 

Si investigamos en el <span style="color:lightblue">/home</span> listando los demás usuarios veremos 2 nuevos:

1. <span style="color:blue">ftp</span> (De donde descargamos la nota del principio)
2. <span style="color:blue">secret</span> (Contiene un <span style="color:gold">hash</span>, cual tenemos que descifrar)

![](/assets/images/dkl-writeup-dancesamba/foundhash_dancesamba.png)

``MMZVM522LBFHUWSXJYYWG3KW05MVQTT2MQZDS6K2IE6T2==

Usaremos la siguiente página para descifrarlo: https://gchq.github.io/CyberChef/

Dentro pondremos primero de <span style="color:cyan">Base32</span> y que lo pase por último por un <span style="color:cyan">Base64</span>.
La salida: <span style="color:darkred">supersecurepassword</span>

![](/assets/images/dkl-writeup-dancesamba/chefhash_dancesamba.png)


Si lo hacemos por consola sería asi:

1)  Lo pasamos por base32
**echo** "<span style="color:gold">MMZVM522LBFHUWSXJYYWG3KW05MVQTT2MQZDS6K2IE6T2= =</span>" |  *base32* <span style="color:orange">-d</span>

2) Lo pasamos por base64
**echo** "<span style="color:gold">c3VwZXJzZWN1cmVwYXNzd29yZA= =</span>" |   *base64* <span style="color:orange">-d</span>

![](/assets/images/dkl-writeup-dancesamba/crackpass_dancesamba.png)


Obteniendo la contraseña <span style="color:darkred">supersecurepassword</span>, finalmente.

#### ESCALADA DE PRIVILEGIOS

Listaremos nuestros permisos de *sudo* a ver si podemos aprovecharnos de alguno que nos permita escalar privilegios.

**sudo -l** :<span style="color:darkred">supersecurepassword</span>

Tendremos el permiso sudo sobre el binario <span style="color:lightblue">/usr/bin/</span><span style="color:grey">file</span> cual podemos deducir que podremos leer archivos arbitrarios cual requieren ciertos permisos de root.

![](/assets/images/dkl-writeup-dancesamba/listsudopriv_dancesamba.png)


Si nos vamos a la siguiente ruta <span style="color:lightblue">/opt</span> veremos un fichero "**password.txt**".
Cual no tendremos permisos ya que solo lo puede leer el usuario <span style="color:blue">root</span>.

![](/assets/images/dkl-writeup-dancesamba/secretfile_dancesamba.png)


Nos iremos al siguiente repositorio online, donde buscaremos el binario <span style="color:lime">file</span> y veremos como podemos leer:
https://gtfobins.github.io/gtfobins/file/#sudo

![](/assets/images/dkl-writeup-dancesamba/gtfsudo_dancesamba.png)


Con la siguiente sintaxis, leeremos dicho archivo arbitrario: 
**sudo** <span style="color:lightblue">/usr/bin/</span><span style="color:grey">file</span>  <span style="color:yellow">-f</span>  <span style="color:lightblue">/opt/</span><span style="color:grey">password.txt</span>

![](/assets/images/dkl-writeup-dancesamba/leerarcharb_dancesamba.png)


Nos dará las credenciales privilegiadas -> <span style="color:blue">root</span>:<span style="color:darkred">rooteable2</span> 
Ahora ya seremos usuario con permisos privilegiados. 

**"id"** para confirmar.

![](/assets/images/dkl-writeup-dancesamba/shellroot_dancesamba.png)


#### BANDERA ROOT

Para coger la bandera <span style="color:blue">root</span> nos vamos a su directorio <span style="color:lightblue">/root</span>.

![](/assets/images/dkl-writeup-dancesamba/flagroot_dancesamba.png)
