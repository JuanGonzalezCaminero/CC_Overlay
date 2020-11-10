# Table of Contents
[Spanish](#spanish)  
[English](#english)

---

<a name="spanish"/>

## Descripción y uso
### Introducción
Todos los scripts de este repositorio usan la imágen juangonzalezcaminero/ubuntu_practica alojada en https://hub.docker.com/. Los scripts descargan la imágen automáticamente, aunque en caso de que esto fallara por algún motivo, sería necesario descargarla de forma manual y etiquetarla como ubuntu_practica para poder ejecutar correctamente los diferentes scripts.

Esta imágen parte de la imágen de ubuntu, y añade los paquetes iputils-ping, iproute2, y un servidor nginx, necesarios para realizar las comprobaciones de que todo funciona correctamente.

Todos los contenedores que se lanzan en estos scripts utilizan la opción --network none, es decir, en el momento de su creación docker no crea una interfaz virtual para el contenedor, por lo que no tienen ninguna forma de comunicarse con el exterior a través de protocolos IP.


### Parte 1
El objetivo de esta sección es proporcionar una forma de que los contenedores lanzados en un mismo host se pueden comunicar entre sí con protocolos IP.

La solución parte del uso de un bridge virtual al que se conecten los contenedores que queramos

---

<a name="english"/>

## Description and usage
### Part 1
