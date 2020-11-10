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

La solución se basa en el uso de un bridge virtual al que se conecten los contenedores que queramos que puedan comunicarse entre sí, esto lo haremos mediante el paquete iproute2.

En este ejemplo, se lanzan dos contenedores, por tanto, tendremos tres espacios de nombres de red, dos para los contenedores y uno para el bridge. Se crean enlaces VETH entre cada uno de los contenedores y el bridge, y se asignan direcciones IP (en la misma subred) a las interfaces de los contenedores. Este esquema muestra las conexiones resultantes y dirección IP de cada interfaz:

 <img src="Images/parte_1.jpg" width="500" />

El script que lanza este escenario 

---

<a name="english"/>

## Description and usage
### Part 1
