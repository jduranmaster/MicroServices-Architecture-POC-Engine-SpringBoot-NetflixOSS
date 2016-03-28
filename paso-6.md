# Paso 6 : Comunicando microservicios
## Un bus para comunicarles a todos
La configuración centralizada en archaius está muy bien peeeero implementar su actualización en caliente es compleja. también hay otros escenarios de negocio donde es necesario comunicar eventos a todos los servicios o a ciertos grupos, por ejemplo: activación/desactivación de una campaña comercial, cambio en la modalidad de funcionamiento de la aplicación (normal, contingencia, sólo lectura, etc) y cualquier otro cambio de estado del conjunto de microservicios.

Spring-Cloud Bus es la implementación que Spring propone para comunicar los servicios y la tecnología/protocolo por defecto es amqp. El protocolo amqp tiene como implementación de referencia a RabbitMQ.

## Instalando rabbitmq
[RabbitMQ](https://www.rabbitmq.com/download.html) es un servidor de colas muy eficiente y ágil que incluye una consola de administración muy potente.

para instalarlo debemos seguir las instrucciones para cada plataforma como se indican en https://www.rabbitmq.com/download.html pero en macosx es tan simple como:
```sh
$ brew update
$ brew install rabbitmq
```
El instalador crea los scripts en ```/usr/local/sbin``` pero no los incluye automáticamente en el path del .bash_profile o .profile así que nos tocará editarlo manualmente y añadir:
```sh
PATH=$PATH:/usr/local/sbin 
```
Luego podremos iniciar RabbitMQ desde línea de comando simplemente haciendo:
```sh
rabbitmq-server
```
Si abrimos un navegador en http://localhost:15672/ veremos la página de login de la consola donde podremos acceder utilizando ```guest``` como usuario y password.

Ya está... y nada más... es uno de los servidores de colas más simples de instalar.

## Actualizar los microservicios
Para implementar el bus debemos incluir en el servidor de configuraciones y en los microservicios la siguiente dependencia:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```
Y al arrancar se conectarán por defecto al servidor rabbitmq en la url por defecto http://localhost:5672. Si se quiere cambiar por que se esté en cloud se puede añadir al application.yml un bloque del estilo:
```yml
spring:
  rabbitmq:
    host: rabbitmqhost
    port: 5672
    username: guest
    password: guest
```
Al arrancar van a crear una cola “no durable” en el servidor rabbitmq para mandar los mensajes en esta sesión.

En nuestro caso, al arrancar nos ha dejado la traza:
```properties
[cTaskExecutor-8] o.s.amqp.rabbit.core.RabbitAdmin         : Auto-declaring a non-durable Queue (891a9a17-5964-4bf1-8bc4-c93e2d045db9). It will be redeclared if the broker stops and is restarted while the connection factory is alive, but all messages will be lost.
```
Y si vamos a la consola de nuestro servidor de rabbitmq podemos ver la cola recién creada.

## Enviar mensajes

Cada aplicación creará su propia cola, la cual se enganchará al “exchange” spring-cloud-bus, de tal manera que cada mensaje que se envíe mediante /bus/env -d :
```sh
$ curl -X POST http://localhost:8888/bus/env -d nombre=JORGE
$ curl -X POST http://localhost:8888/bus/refresh
```
Se escribirá a todas las colas y llegará a cada una de las aplicaciones, las cuales actualizarán los valores cuando llegue la orden /bus/refresh , por ejemplo:
```sh
curl -X POST http://localhost:8888/bus/refresh
```
