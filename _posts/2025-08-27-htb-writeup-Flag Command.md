
![[logo_fc.png]]


Tags: #challenge #web 

2025-08-27
## VIDEOS

<iframe title="Flag Command - Web Challenge | HackTheBox" src="https://www.youtube.com/embed/UybPTPuFkFM?feature=oembed" height="113" width="200" style="aspect-ratio: 4 / 3; width: 100%; height: 100%;" allowfullscreen="" allow="fullscreen"></iframe>


## WRITE UP
docker host: 94.237.61.242:35563

### INTRODUCCIÓN

El objetivo es mediante un juego web, escapar del encantado bosque.


#### ANÁLISIS PÁGINA

Iniciamos la instancia del reto:

![[ins_fc.png]]


Copiamos la IP del host en el navegador.
De primeras, vemos en la página, la introducción del juego y unos simples comandos con "help", que nos dicen como empezarlo...

![[opt_fc.png]]


Si ponemos "**start**", vemos que nos salen 4 opciones y hay que elegir una, pero si nos equivocamos morimos y habrá que poner "**restart**". A medida que avanzamos, llegamos a un punto en el que da igual que opción elijamos que vamos a morir si o si.

![[opt2_fc.png]]


#### ANÁLISIS DEV TOOLS:  NETWORK Y TOMAR BANDERA

Vamos a probar ver las solicitudes en la red que recibe la página mientras que la página funciona.
Apretamos en el teclado "**Ctrl + shift + i**"  > ==Network== y refrescamos la página.

![[net_fc.png]]


Vemos distintas solicitudes, entre ellas, una que destaca llamada  "<span style="color:green">options</span>" y que nos deja acceder,  si le hacemos doble Clic , nos lleva al **endpoint del api**  <span style="color:lightblue">/options</span>  con las opciones que ya conocemos.

![[optshow_fc.png]]

hay una opción secreta, que la copiaremos.
![[secrop_fc.png]]


Ahora, empezamos el juego "**start**" y copiamos dicha opción, así nos devolverá la flag, escapando del bosque.

![[flag_fc.png]]