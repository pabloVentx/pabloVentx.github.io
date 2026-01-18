---
layout: single
title: File Upload Validation ByPass
excerpt: "En este Post, mostrare una serie de métodos para ByPassear la Validación una subida de archivos cual solo permite cierta extensión o otros factores, llegando en algunos casos, a una ejecución remota de comandos (RCE). A medida que vaya aprendiendo más, lo ire actualizando."
date: 2026-01-18
classes: wide
header:
  teaser: /assets/images/htb-writeup-shocker/
  teaser_home_page: true
  icon: /assets/images/
categories:
  - Web-documentation
  - Pentesting
tags:
  - HackingWeb
  - FileUpload
---

##### 1 FORMA: Null Byte

Cuando una web valida la extensión de un archivo para permitir solo ciertos tipos (como `.jpg` o `.png`), puedes intentar usar un **null byte** (`%00`) para evadir esa validación. 

El null byte es un carácter especial que indica el final de una cadena en muchos lenguajes, por lo que la validación puede detenerse antes de la extensión real del archivo. 

Por ejemplo, nombrar un archivo como `shell.php%00.jpg` puede hacer que la validación acepte `.jpg`, pero el servidor procese el archivo como `.php`. Sin embargo, esta técnica suele estar mitigada en sistemas modernos.

>Ej: Máquina "Jerry" se ve este metodo.

>Está técnica se puede usar para diferentes ataques. Como *PATH TRAVERSAL* o *LFI*


##### 2 FORMA: validación de formato en la cadena del nombre

Interpreta que al comprobar que en la cadena del nombre contiene `.x` (formato deseado), ya no compruebe lo que tiene después.

`shell.jpg.php`


##### 3 FORMA: Validación de los primeros bytes o "Magic Numbers"

Usado para identificar el archivo mediante los primeros bytes, aparte del propio formato.


**JPG**<span style="color:orange">;</span> (En la primera línea del archivo)

<?php system("whoami"); ?>


>Ej: Challenge "Trickster", se ve este metodo.

>Más información: [List of Signatures](https://en.wikipedia.org/wiki/List_of_file_signatures)
