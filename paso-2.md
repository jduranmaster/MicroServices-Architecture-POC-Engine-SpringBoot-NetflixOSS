# Paso 2 : comenzando con netflix oss
## La problemática de los microservicios
- Configuración centralizada y versionada
- Gestionar el tráfico hacia los servicios
- Exponer y comunicar los servicios entre ellos
- Monitorizar los servicios
- Detectar que un servicio falla y evitar enviarle tráfico
- Mensajes entre los servicios (o a los de un tipo concreto) 

## ¿Qué es netflix oss?
Netflix open source software center: externalización de proyectos de ingeniería software de netflix, ver http://netflix.github.io/ 

## El primer servicio netflix: Archaius
 - Archaius es un servidor de configuraciones
   - configuraciones globales y específicas
   - cambios en caliente
   - versionado
 - se puede definir mediante spring boot
   - para ellos vamos a crear un proyecto spring-boot llamado ```service-archaius```con su ```pom.xml``` donde definiremos el parent para spring cloud y la dependency con el artefacto de servidor de configuraciones:

```xml
<parent>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-parent</artifactId>
  <version>1.0.2.RELEASE</version>
</parent>
        
 <dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
  </dependency>
</dependencies>
```
   - crearemos una clase princpal con las anotaciones:

```java
@SpringBootApplication
@EnableConfigServer
public class ArchaiusServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ArchaiusServiceApplication.class, args);
    }
}
```
   - Crearemos un fichero src/main/resources/application.yml para indicar que pondremos el fichero de configuración en github (podría estar en fichero, en url, etc): 

```yaml
server:
  port: 8888
 
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/jduranmaster/archaius-config
```
   - En ese github ponemos el helloworld.properties o el yaml con la configuración y podemos indicar con sufijos el entorno al que pertenecen.
   - Podemos encriptar valores
   - Podemos configurar usuario y contraseña de acceso
   - Lanzamos archaius:
```sh
$ mvn spring-boot:run
```
   - lo probamos con curl:
```sh
$ curl http://localhost:8888/helloworld/prod
```

## Actualizamos el microservicio para usar configuración distribuida
   - Añadimos en el pom.xml del microservice-helloworld la dependencia:

```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-parent</artifactId>
        <version>1.0.2.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  ...
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter</artifactId>
  </dependency>
```
   - configuramos el ```application.properties``` para añadir la configuración del servidor:
```properties
endpoints.shutdown.enabled=true
endpoints.restart.enabled=true
spring.application.name=helloworld
spring.cloud.config.enabled=true
spring.cloud.config.uri=http://localhost:8888
spring.cloud.config.failFast=true
```

   - modificamos la clase principal para añadir el soporte a configuraciones:
 
```java
@Configuration
@EnableAutoConfiguration
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
   - modificamos el controller para usar la configuración con el clásico @Value:

```java
@RestController
public class MessageController {
  @Value("${nombre}")
  private String nombre;

  @RequestMapping("/")
  ResponseEntity<Message> home() {
    return new ResponseEntity(new Message("Hello " + nombre), HttpStatus.ACCEPTED);
  }
}

```

## cambiar la configuración en caliente a un microservicio
```sh
$ curl http://localhost:8080/env -d nombre=JORGE
{"nombre":"JORGE"}
$ curl http://localhost:8080/refresh -d helloworld
[]
$ curl http://localhost:8080/restart -d helloworld
{"message":"Restarting"}
$ curl http://localhost:8080/
{"message":"Hello JORGE"}
```
## cambiar la configuración en caliente en el repo y los microservicios
 - Añadir @RefreshScope a las clases que usen configuraciones:

```java
@RefreshScope
@RestController
public class MessageController {

  @Value("${nombre}")
  private String nombre;

  @RequestMapping("/")
  ResponseEntity<Message> home() {
    return new ResponseEntity(new Message("Hello " + nombre), HttpStatus.ACCEPTED);
  }
}
```

 - hacemos la prueba

```sh
$ curl http://localhost:8080/
{"message":"Hello Jorge"}
--- hacemos el cambio en el GIT a nombre=JorgeA
$ curl http://localhost:8080/refresh -d helloworld
["nombre"]
$ curl http://localhost:8080/
{"message":"Hello JorgeA"}
```

 