---
layout: single
title: Trickster - PicoCTF
excerpt: "En este challenge dispondremos de una simple página web con una posible subida de archivos de solamente formato .png . Que cambiando los primeros bytes del archivo cual contendrá el código malicio será posible su subida, aunque tenga un.php (debe tener el .png antes de este) y asi, coger la bandera mediante una ejecución remota de comandos RCE via webshell ."
date: 2026-01-18
classes: wide
header:
  teaser: /assets/images/pctf-writeup-trickster/logo_trickster.png
  teaser_home_page: true
  icon: /assets/images/picoCTF.webp
categories:
  - PicoCTF
  - Pentesting
  - Challenge
  - Web
tags:
  - writeup
  - challenge
  - HackingWeb
  - Fuzzing
  - FileUpload
  - AbusingFileBytes
  - RemoteAccessRCE
  - WebShell
---

![](/assets/images/pctf-writeup-trickster/logo_trickster.png)

"En este challenge dispondremos de una simple página web con una posible subida de archivos de solamente formato <span style="color:grey">.png</span> . Que cambiando los primeros bytes del archivo cual contendrá el código malicio será posible su subida, aunque tenga un <span style="color:grey">.php</span> (debe tener el <span style="color:grey">.png</span> antes de este) y asi, coger la bandera mediante una ejecución remota de comandos **RCE** via <span style="color:lime">webshell</span>."

## WRITE UP
Ejemplo: atlas:picoctf.net:58900 (víctima) TTL=  63 LINUX


#### ÁNALISIS WEB: RECONOCIMIENTO

De primeras, cuando accedamos a la página vemos que no tiene mucho más que seleccionar una archivo **.png** desde nuestro gestor de archivos y subirlo.

![](/assets/images/pctf-writeup-trickster/1_trickster.png)


Si usamos la extensión <span style="color:lightyellow">Wappalyzer</span>, no nos saldrá nada acerca de las tecnologías o lenguaje de la página.


![](/assets/images/pctf-writeup-trickster/wappalyzer_trickster.png)


Para comprobar el funcionamiento, vamos a probar a subir uno de prueba.

![](/assets/images/pctf-writeup-trickster/subidapng_trickster.png)


(Ignorar el error). Le damos a "**Upload File**".

![](/assets/images/pctf-writeup-trickster/subidapngg_trickster.png)


Nos dirá que se ha subido exitosamente.

![](/assets/images/pctf-writeup-trickster/succupl_trickster.png)


#### FUZZING DIRECTORIOS: NIVEL WEB

Con *ffuf* haremos fuzzing a nivel web para descubrir en que ruta se aloja los archivos que subimos en esta web.

**ffuf** <span style="color:green">-c  -t</span>  200  <span style="color:green">-w</span>  <span style="color:lightblue">/usr/share/wordlists/SecLists/Discovery/Web-Content/</span>big.txt <span style="color:green">-u</span>  http://atlas.picoctf.net:58900/FUZZ/ <span style="color:green">-fc</span> 404 <span style="color:green">-mc </span> 200,302,403,401

Veremos el directorio <span style="color:lightblue">/uploads/</span> que es probablemente donde se aloja dichos archivos.

![](/assets/images/pctf-writeup-trickster/Fuzzpag_trickster.png)


Si entramos entonces a la siguiente ruta, leeremos el archivo:
Aunque no tengamos acceso, podemos leer los archivos.

![](/assets/images/pctf-writeup-trickster/verfil_trickster.png)



#### SUBIDA DE PHP MALICIOSO

Suponiendo que puede leer archivos **php**, vamos a comprobar si metemos este archivo, con un código que si se ejecuta, nos dará la salida el comando "**id**".

**nano** <span style="color:lime">cmd.php</span>

<?php system("id"); ?>

![](/assets/images/pctf-writeup-trickster/php_trickster.png)


En la página, para subirlo filtramos por el campo "All files" y lo seleccionamos.

![](/assets/images/pctf-writeup-trickster/subidaphp_trickster.png)


Al subirlo nos dirá que no contiene la cadena **.png** en el nombre.

![](/assets/images/pctf-writeup-trickster/wrror1_trickster.png)


#### AGREGAR .PNG AL ARCHIVO

Le cambiamos el formato agregando .png antes del .php ya que alomejor puede interpretar que al comprobar que en la cadena del nombre contiene .png, ya no compruebe lo que tiene después.

<span style="color:lime">cmd.png.php</span>

![](/assets/images/pctf-writeup-trickster/cadena_trickster.png)


Lo subimos.

![](/assets/images/pctf-writeup-trickster/subida2_trickster.png)


Nos dice que lo que acabamos de subir no es un tipo de **PNG** válido.
Con esto ya sabemos que si hubiésemos intentado un <span style="color:cyan">null Byte</span>, no nos hubiera funcionado.

Ej: <span style="color:lime">cmd.php%00.png</span>

![](/assets/images/pctf-writeup-trickster/cadenaerror_trickster.png)


#### MAGIC NUMBERS: PRIMEROS BYTES

Bien ahora sabemos que lo esta validando por los primeros bytes del archivos, es decir los "**Magic numbers**".  Esto es útil para identificar un archivo sin tener en cuenta el formato.


Más información: [List of file signatures](https://en.wikipedia.org/wiki/List_of_file_signatures)


Los primeros bytes de un PNG, serían los siguientes

![](/assets/images/pctf-writeup-trickster/bytespng_trickster.png)


Si hacemos un **file** para ver que tipo de archivo es nuestro archivo es, lo que hemos subido, nos detecta que se trata de una PHP.

Esto se debe a que los primeros bytes: ``<?php``.

![](/assets/images/pctf-writeup-trickster/filephp_trickster.png)


#### CAMBIAR BYTES DEL PHP a PNG

(Podemos ponerlo sin ";"). Para que esto sea posible, ponemos en la primera línea: **PNG**<span style="color:orange">;</span> y guardamos.

![](/assets/images/pctf-writeup-trickster/pngbytesphp_trickster.png)

Si tiramos nuevamente de **file**, ya no nos dice que es un *PHP* solo un *texto ASCII*.

![](/assets/images/pctf-writeup-trickster/compphp_trickster.png)


#### CAMBIAR BYTES del PHP a GIF

Vamos a probar probamos con otra cosa, por ejemplo en <span style="color:grey">GIF</span> .

![](/assets/images/pctf-writeup-trickster/bytesgif_trickster.png)

Ponemos en la primera línea del archivo: **GIF8**<span style="color:orange">;</span>


![](/assets/images/pctf-writeup-trickster/gifcon_tricskter.png)

Usamos **file** y nos dirá que es un *GIF* solamente.

![](/assets/images/pctf-writeup-trickster/filegif_trickster.png)


#### SUBIR FAKE PNG CON CÓDIGO PHP

Ahora nos toca pasar los bytes a PNG porque sino va a detectar que tiene los primeros bytes de PHP. Como tiene el <span style="color:grey">.png</span>, tendría que funcionar definitivamente.

**PNG**<span style="color:orange">;</span>

![](/assets/images/pctf-writeup-trickster/pngconff_trickster.png)


Lo seleccionamos y subimos.

![](/assets/images/pctf-writeup-trickster/ssubidapngb_trickster.png)



Se ha subido correctamente.

![](/assets/images/pctf-writeup-trickster/succces_trickster.png)


##### EJECUCIÓN REMOTA DE COMANDOS (RCE) VÍA WEB

Si ejecutamos el archivo, va a interpretar el comando que le pusimos, en este caso "**id**", cual da salida.

![](/assets/images/pctf-writeup-trickster/rce_trickster.png)


Para verlo mejor, en vez de una sola línea, le damos "**CTRL + U**" para ver el código fuente y que sea más cómodo.

![](/assets/images/pctf-writeup-trickster/mejorvis_trickster.png)


##### WEBSHELL

Ahora que tenemos ejecución remota de comandos, nos daremos una <span style="color:lime">webshell</span>*:
Usaremos etiquetas pre formateadas para permitir una visualización como hemos tenido antes más visual.

<?php <"pre"> . system($_GET["cmd"]) . "</pre>"; ?>

![](/assets/images/pctf-writeup-trickster/phpmej_trickster.png)


Lo cargamos en la página, y ya tendremos para ejecutar comandos cómodamente con la <span style="color:lime">webshell</span> mediante el parámetro **cmd**.

![](/assets/images/pctf-writeup-trickster/webshell_trickster.png)


#### BANDERA

(**"CTRL + U"**)
Leeremos la bandera alojada en <span style="color:lightblue">/var/www/html</span>.

**cd** ..<span style="color:orange">;</span> **ls**<span style="color:orange">;</span> **cat** <span style="color:orange">GQ4DOOBVMMYGK.txt</span>

![](/assets/images/pctf-writeup-trickster/flag_trickster.png)
