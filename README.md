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

Por último, cada uno de los scripts lanza el escenario de ejemplo asociado a cada una de las partes descritas en este readme. Los scripts se deben ejecutar con permisos de superusuario para que puedan realizar las operaciones necesarias con iproute2. Los scripts realizan de forma automática una serie de pruebas para comprobar el correcto funcionamiento del escenario, pero en algunos casos en los que no sea posible hacer estas comprobaciones desde la misma máquina que ejecuta el script se indicará al usuario lo que se ha de hacer para comprobar que el comportamiento del sistema desplegado es el esperado.

### Parte 1
El objetivo de esta sección es proporcionar una forma de que los contenedores lanzados en un mismo host se pueden comunicar entre sí con protocolos IP. 

La solución se basa en el uso de un bridge virtual al que se conecten los contenedores que queramos que puedan comunicarse entre sí, esto lo haremos mediante el paquete iproute2.

En este ejemplo, se lanzan dos contenedores, por tanto, tendremos tres espacios de nombres de red, dos para los contenedores y uno para el bridge. Se crean enlaces VETH entre cada uno de los contenedores y el bridge, y se asignan direcciones IP (en la misma subred) a las interfaces de los contenedores. Este esquema muestra las conexiones resultantes y dirección IP de cada interfaz:

 <img src="Images/parte_1.jpg" width="500" />

El script que lanza este escenario es launch_1.

### Parte 2
El objetivo de esta sección es ampliar el escenario anterior para permitir comunicaciones entre los contenedores y el host.

La solución parte de lo descrito anteriormente. Los cambios son pocos, ya que lo único que se necesita es crear un enlace VETH entre el host y el bridge, y asignar una dirección IP a la interfaz VETH en el host. En este caso, esta interfaz se crea en el espacio de nombres de red del host, y la subred que se usa para la dirección IP es la misma que para los contenedores (192.168.2.0/24).

A continuación se muestra el sistema descrito:

<img src="Images/parte_2.jpg" width="500" />

El script que lanza este escenario es launch_2.

### Parte 3
El objetivo de esta sección es ampliar el escenario anterior para permitir comunicaciones entre los contenedores y cualquier nodo alcanzable por el host.

Para conseguir esto, propongo un escenario donde el host actúa como un router entre los contenedores y el exterior, para hacer esto, se usa el paquete iptables. 

Se crea una regla en la cadena POSTROUTING de la tabla de nat para que cuando el host reciba una comunicación desde uno de los contenedores, cambie su dirección de orígen a la IP de su interfaz externa, y un puerto concreto, y viceversa cuando reciba un mensaje del exterior en ese puerto, cambiando su dirección de destino a la del contenedor correspondiente.

En la cadena FORWARD de la tabla de filter se añaden dos reglas, una para aceptar paquetes que pertenezcan a una conexión establecida o que estén estableciendo una conexión, y otra para aceptar paquetes que entren desde la interfaz interna del host, la que se conecta con los contenedores, y salgan por la interfaz externa del host.

Las reglas son las siguientes:

`iptables -t nat -A POSTROUTING -o interfaz_externa -j MASQUERADE`

`iptables -t filter -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`

`iptables -t filter -A FORWARD -i veth0 -o interfaz_externa -j ACCEPT`

Se toma como interfaz externa del host la que aparezca como gateway por defecto al usar el comando `ip route`

Además de estas reglas, es necesario establecer al host como gateway por defecto para los contenedores, de forma que cualquier mensaje dirigido a una dirección fuera de su subred se envíe a la interfaz interna del host, desde donde se redirige a su destino.

Una vez hecho esto, los contenedores pueden comunicarse con cualquier nodo alcanzable desde el host si son ellos los que inician la comunicación, sin embargo, desde otras máquinas no es posible alcanzar los contenedores. Una solución para hacer esto posible es el port forwarding, establecemos una serie de puertos en la interfaz externa del host que se redirigirán a los contenedores. 

De nuevo, hacemos esto usando iptables. Para cada puerto del host que queramos redirigir a un contenedor son necesarias dos reglas, una, en la cadena PREROUTING de la tabla de nat, que cambie la dirección de destino de los mensajes dirigidos al puerto en concreto a la ip y puerto del contenedor que queramos:

`iptables -A PREROUTING -t nat -i interfaz_externa -p protocolo --dport puerto_host -j DNAT --to ip_contenedor:puerto_contenedor`

La segunda regla, en la cadena FORWARD de la tabla filter, acepta paquetes dirigidos a la IP y puerto del contenedor:

`iptables -A FORWARD -p protocolo -d ip_contenedor --dport puerto_contenedor -j ACCEPT`

En el escenario que despliega el script, se redirigen los puertos 8000 y 8080 del host al puerto 80 de los contenedores 1 y 2 respectivamente, permitiendo que desde una máquina externa se acceda a la página de inicio del servidor nginx instalado en los contenedores.

El script que lanza este escenario es launch_3.

### Parte 4
El objetivo de esta sección es permitir la comunicación entre contenedores desplegados en distintas máquinas a través de protocolos IP. Se parte de un escenario similar al descrito en la parte 3, aunque sin el port forwarding, ya que no es necesario para este ejemplo.

Una solución no generalizada, como la que despliega el script, implica pocos cambios. Asumiendo que conocemos la subred en la que están los contenedores en cada host, y la IP de todos los hosts que queremos conectar, la solución es tan sencilla como añadir una regla de rutado usando `ip route` en cada uno de los hosts, indicando en cada caso la subred que queremos alcanzar y la IP del host donde están esos contenedores:

`ip route add subred/mascara via host_remoto dev interfaz_externa`

En este ejemplo, la subred de uno de los hosts será la 192.168.2.0/24, y la del otro 192.168.3.0/24. Para conocer la IP del otro host, símplemente se le pide al usuario que introduzca la dirección de forma manual.

Una forma de generalizar esto sería utilizar un nodo maestro, que podría ser uno de los hosts que queremos comunicar u otra máquina, la dirección IP de este nodo tendría que ser conocida por todos. Cuando un host quisiera incorporarse a la red, y permitir que sus contenedores se comunicaran con los de otros hosts, y viceversa, haría una petición al nodo maestro, que le indicaría la subred en la que tiene que lanzar sus contenedores, así como una lista de las direcciones IP del resto de nodos en la red, así como la subred en la que están los contenedores en cada uno de esos nodos.

Esto implicaría que el nodo maestro tuviera que mantener esa lista de direcciones IP de los hosts y las subredes para sus contenedores, también que cada vez que un host se incorpore a la red sería necesario informar a todos los nodos de la IP y subred de los contenedores del nuevo host, con lo cual los nodos también tendrían que permanecer en espera de esas atualizaciones mientras sigan formando parte del sistema.

---

<a name="english"/>

## Description and usage
### Part 1















