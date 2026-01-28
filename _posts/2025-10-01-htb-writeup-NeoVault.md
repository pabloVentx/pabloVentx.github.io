---
layout: single
title: NeoVault - Hack The Box
excerpt: "Este challenge simula una aplicación bancaria con funciones básicas como registro, transferencias y descarga de extractos. El reto esconde una vulnerabilidad **IDOR** en una versión antigua de su API. Manipulando identificadores de usuario, es posible acceder a archivos de otros clientes. El objetivo es explotar este fallo para obtener la flag y entender riesgos de control de acceso."
date: 2025-10-01
classes: wide
header:
  teaser: /assets/images/htb-writeup-neovault/logo_nv.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hackthebox
  - Pentesting
  - Challenge
  - Web
tags:
  - writeup
  - challenge
  - HackingWeb
  - API
  - IDOR
---

![](/assets/images/htb-writeup-neovault/logo_nv.png)

"Este challenge simula una aplicación bancaria con funciones básicas como registro, transferencias y descarga de extractos. El reto esconde una vulnerabilidad **IDOR** en una versión antigua de su API. 
Manipulando identificadores de usuario, es posible acceder a archivos de otros clientes. El objetivo es explotar este fallo para obtener la flag y entender riesgos de control de acceso."

## WRITE UP
Sesión Doker-> 94.237.55.43


#### ANÁLISIS PÁGINA

Iniciamos la instancia del reto:

![](/assets/images/htb-writeup-neovault/instancia_nv.png)


Nos metemos a la página, para analizarla.
En la interfaz de una aplicación bancaria sobre transferencias bancarias llamada "NeoVault".

![](/assets/images/htb-writeup-neovault/interfaz_nv.png)


Le daremos a "**Create Account**" y nos crearemos una cuenta, para ver como funciona por dentro que funciones a nivel usuario tenemos disponibles.

![](/assets/images/htb-writeup-neovault/creacacc_nv.png)

#### ENUMERACIÓN SITIOS WEB

Tendremos una interfaz gráfica sobre transferencias, el histórico y seguimiento del dinero.

![](/assets/images/htb-writeup-neovault/graf_nv.png)


Entre las opciones que disponemos nos quedamos con:

<span style="color:lightblue">/Transfer</span>  Aquí podremos hacer transacciones a quien queramos.

![](/assets/images/htb-writeup-neovault/transaccion_nv.png)


<span style="color:lightblue">/Transactions</span> sacamos al usuario <span style="color:blue">neo_system</span> porque nos hizo una transacción de 100 dolares.
Nos podemos descargar un PDF con el historial de las transacciones.

![](/assets/images/htb-writeup-neovault/trasachistory_nv.png)


Le damos arriba derecha a "Download".

![](/assets/images/htb-writeup-neovault/pdf_nv.png)


#### INTERCEPTAR PETICIONES
##### CONFIGURAR BURP SUITE

Usaremos *Burp Suite* para interceptar peticiones a nivel web, para analizarlas peticiones donde nos metamos.
Nos ponemos en escucha dándole a "Proxy" > "**Intercept on**".

![](/assets/images/htb-writeup-neovault/burpon_nv.png)


##### ENVIAR TRANSFERENCIA

Como <span style="color:blue">neo_system</span> nos ha enviado dinero, vamos a darle nosotros y, da igual la cantidad, le damos a "Transfer Money".

![](/assets/images/htb-writeup-neovault/trans_nv.png)


Nos sale esta pestaña, pero antes de aceptar, habrá que configurar un Proxy para recibir los paquetes a nuestro BurpSuite.

![](/assets/images/htb-writeup-neovault/trans2_nv.png)

##### CONFIGURAR FOXY PROXY

En la extensión de <span style="color:lightyellow">FoxyProxy</span>, configuramos uno para BurpSuite. 

![](/assets/images/htb-writeup-neovault/burp_nv.png)


Ponemos esta configuración:

*127.0.0.1* (apunta a nuestro localhost)
*8080* (Por este puerto)

![](/assets/images/htb-writeup-neovault/conffp_nv.png)


Le damos a confirmar transacción, y nos saldrá la cuenta de la misma.

![](/assets/images/htb-writeup-neovault/totaltras_nv.png)


#### ANÁLISIS SOLICITUDES

Miramos la petición de **Burp Suite** que ha capturado en el apartado "Proxy", y vemos que nos ha dejado el <span style="color:yellow">ID</span> de <span style="color:blue">neo_system</span>, por lo cual puede ser crítico si sabemos como aprovecharlo.

En -> **POST** <span style="color:lime">/api/</span><span style="color:orange">v2</span><span style="color:lightblue">/transactions</span> | Son los datos que hemos enviado en la información de la transacción; cantidad dinero, razón...

<span style="color:blue">neo_system</span>:<span style="color:yellow">68dd6c1ce4e7643eadc1c73</span>

![](/assets/images/htb-writeup-neovault/idtras_nv.png)


En el **HTTP HISTORY** vemos una respuesta donde podremos ver la información de la transacción.

En -> **GET** <span style="color:lime">/api/</span><span style="color:orange">v2</span><span style="color:lightblue">/transactions</span> | Son los datos que nosotros recibimos, en este caso; la fecha de la transaccion, de quien la hizo  a quien la recibe... Esto se ve el histórico de transacciones.

![](/assets/images/htb-writeup-neovault/httphistory_nv.png)


Aquí concretamente.

![](/assets/images/htb-writeup-neovault/histodown_nv.png)


En -> **GET** <span style="color:lime">/api/</span><span style="color:orange">v2</span><span style="color:lightblue">/auth/inquire?</span>username=<span style="color:blue">neo_system</span> |  **Verifica o consulta el estado del usuario** dentro del sistema de autenticación

![](/assets/images/htb-writeup-neovault/otherhisot_nv.png)


#### RUTAS API ENDPOINTS

Nos vamos a las herramientas de desarrollador. "**CTRL + SHIFT + I**" 

"Debugger" > "app" > <span style="color:cyan">layout.js</span> > Filtramos por **API** y **{ }**

Encontraremos todos los <span style="color:lime">API endpoints</span> <span style="color:orange">v1</span> y <span style="color:orange">v2</span>.\

![](/assets/images/htb-writeup-neovault/rutasapi_nv.png)


#### MANIPULAR API ENDPOINTS

Cogemos la petición de cuando descargamos el PDF para ver nuestras transacciones.
La llevamos al **Repeater**.

![](/assets/images/htb-writeup-neovault/repeat_nv.png)


En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v2</span><span style="color:lightblue">/transactions/download-transactions</span> | Si ponemos los datos del neo_system, no estando en la v2 de la API, no saldrá nada.

```
{
   "_id":68dd6c1ce4e7643eadc1c7"
   "username":"neo_system"
}
```


![](/assets/images/htb-writeup-neovault/trabsidd_nv.png)


Si cambiamos del <span style="color:lime">API endpoint</span> <span style="color:orange">v2</span> al <span style="color:orange">v1</span>, y borramos el campo id y username, así pasándolo al **Response**:

En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v1</span><span style="color:lightblue">/transactions/download-transactions</span>

Dice que no hay id establecido.

![](/assets/images/htb-writeup-neovault/idisnotp_nv.png)


Ponemos el campo "**id**" con lo que sea y nos dice que no ha relacionado ninguna id para el modulo usuario, dando un *Internal server error*.

En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v1</span><span style="color:lightblue">/transactions/download-transactions</span>

```
{
      "_id":"test"
}
```

![](/assets/images/htb-writeup-neovault/testid_nv.png)


Ponemos el de <span style="color:blue">neo_system</span> y nos cargará un PDF de las transacciones de dicho usuario cual no podemos ver.

En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v1</span><span style="color:lightblue">/transactions/download-transactions</span>

```
}
    "_id":"68dd6c1ce4e7643eadc1c73"
{
```

![](/assets/images/htb-writeup-neovault/idprop_nv.png)


Para ver el PDF de sus transacciones, nos ponemos en escucha para capturar peticiones.
Nos vamos al apartado transacciones para descargarnos el PDF.

![](/assets/images/htb-writeup-neovault/downloadpdf_nv.png)


En la petición recibida hacemos lo de antes. Ponemos el <span style="color:yellow">ID del usuario</span> de <span style="color:blue">neo_system</span> y le damos a **Forward** para enviarla. Y ponemos la versión de la <span style="color:lime">API</span> a <span style="color:orange">v1</span>

En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v1</span><span style="color:lightblue">/transactions/download-transactions</span>

```
}
    "_id":"68dd6c1ce4e7643eadc1c73"
{
```

![](/assets/images/htb-writeup-neovault/id_nv.png)


##### IDOR (Insecure Direct Object Reference) -> HISTORIAL NEO_SYSTEM

Vemos que aparte de las transacciones que ya sabemos, hay una nueva del usuario <span style="color:blue">user_with_flag</span>.

>Aconteciéndose así la vulnerabilidad *IDOR* (**Insecure Direct Object Reference**). La versión legada (`v1`) no valida correctamente los parámetros de usuario, permitiendonos (aunque sea con una cuenta normal) manipule el identificador interno de usuario (`_id`) para descargar archivos de transacciones de otros usuarios.

![](/assets/images/htb-writeup-neovault/userflag_nv.png)

##### VER HISTORIAL DEL NUEVO USUARIO

Vamos a coger su ID enviándole una transacción, así ver su historial de transacciones.

![](/assets/images/htb-writeup-neovault/donflagus_nv.png)


En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v2</span><span style="color:lightblue">/transactions/</span>

La tenemos -> <span style="color:blue">user_with_flag</span>:<span style="color:yellow">68dd6c1ce4e76430eadc1c78</span>

![](/assets/images/htb-writeup-neovault/idot_nv.png)


En -> **GET** <span style="color:lime">/api/</span><span style="color:orange">v2</span><span style="color:lightblue">/auth/inquire?</span>username=<span style="color:blue">user_with_flag</span> |  **Verifica o consulta el estado del usuario** dentro del sistema de autenticación

![](/assets/images/htb-writeup-neovault/iduserwf_nv.png)


Hacemos lo mismo, le damos al botón de descargar PDF mientras estamos en escucha, recibimos la petición, y insertamos el <span style="color:yellow">ID del usuario</span> <span style="color:blue">user_with_flag</span>. Le damos a **Forward** para enviar la petición


En -> **GET**  <span style="color:lime">/api/</span><span style="color:orange">v1</span><span style="color:lightblue">/transactions/download-transactions</span>

```
}
    "_id":"68dd6c1ce4e76430eadc1c78"
{
```

![](/assets/images/htb-writeup-neovault/idnewuse_nv.png)


#### BANDERA USUARIO

Nos mostrara el PDF del usuario con la bandera.

![](/assets/images/htb-writeup-neovault/flag_nv.png)
