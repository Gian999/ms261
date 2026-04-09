# GUIA PASO A PASO - CONEXION DOCKER + SPRING CLOUD (INFRA + SERVICES)

## Objetivo
- Levantar correctamente Config Server + microservicios catalogo/producto en Docker.
- Entender como se conectan entre carpetas infra y services.
- Tener una guia clara para repetir el proceso sin errores.

---

## 0) CREAR CONFIG SERVER DESDE CERO (SI SOLO TIENES PRODUCTO Y CATALOGO)

Si aun no tienes Config Server hecho, crea esta estructura minima:

```text
infra/
  config-server/
    pom.xml
    src/main/java/com/example/configserver/ConfigServerApplication.java
    src/main/resources/application.yml
```

### 0.1 Crear proyecto en IntelliJ
1. IntelliJ -> New Project -> Spring Boot.
2. Name: config-server.
3. Group: com.example.
4. Java: 17.
5. Dependencies: Spring Boot Actuator.
6. Crear el proyecto dentro de: D:/clase micros/infra/config-server.

### 0.2 pom.xml (contenido clave que debes tener)

```xml
<properties>
  <java.version>17</java.version>
  <spring-cloud.version>2025.0.2</spring-cloud.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### 0.3 Clase principal
Archivo: src/main/java/com/example/configserver/ConfigServerApplication.java

```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### 0.4 application.yml de Config Server
Archivo: src/main/resources/application.yml

```yaml
server:
  port: 7071

spring:
  application:
    name: config-server
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: file:///D:/clase micros/infra/config-repo
```

### 0.5 Crear repo de configuraciones centralizadas
Carpeta:

```text
infra/config-repo/
```

Archivos minimos:
- catalogo-dev.yml
- producto-dev.yml

Ejemplo minimo para catalogo-dev.yml:

```yaml
server:
  port: 8082

spring:
  datasource:
    url: jdbc:mysql://${CATALOGO_DB_HOST}:${CATALOGO_DB_PORT}/${CATALOGO_DB_NAME}?useSSL=false&serverTimezone=America/Lima&allowPublicKeyRetrieval=true
    username: ${CATALOGO_DB_USERNAME}
    password: ${CATALOGO_DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

Ejemplo minimo para producto-dev.yml:

```yaml
server:
  port: 8084

spring:
  datasource:
    url: jdbc:mysql://${PRODUCTO_DB_HOST}:${PRODUCTO_DB_PORT}/${PRODUCTO_DB_NAME}?useSSL=false&serverTimezone=America/Lima&allowPublicKeyRetrieval=true
    username: ${PRODUCTO_DB_USERNAME}
    password: ${PRODUCTO_DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### 0.6 Empaquetar y ejecutar Config Server

```powershell
cd "D:\clase micros\infra\config-server"
.\mvnw clean package -DskipTests
.\mvnw spring-boot:run
```

### 0.7 Probar que esta funcionando
Debe responder JSON en:
- http://localhost:7071/catalogo/dev
- http://localhost:7071/producto/dev

Si esto responde, recien levanta catalogo y producto en Docker.

---

## 1) ESTRUCTURA QUE TIENES

- infra/
  - config-repo/ (YML centralizados que entrega Config Server)
  - config-server/ (servicio Spring Cloud Config Server)
  - eureka-server/ (opcional, para service discovery)
- services/
  - catalogo/ (microservicio + docker-compose)
  - producto/ (microservicio + docker-compose)

### Idea clave
- Los microservicios no leen su config final solo desde su application.yml interno.
- Los microservicios primero llaman al Config Server (puerto 7071).
- Config Server lee los archivos de infra/config-repo/*.yml y se los entrega.

---

## 2) PROGRAMAS NECESARIOS

### Obligatorios
- Docker Desktop (Windows) con engine Linux activo.
- Docker Compose v2 (comando docker compose).

### Recomendados
- PowerShell (terminal).
- VS Code.
- IntelliJ IDEA (Community o Ultimate) para compilar/ejecutar Spring facilmente.

### Solo si ejecutaras servicios Spring fuera de Docker
- Java 17.
- Maven (o mvnw del proyecto).

### Tip para IntelliJ
- Abrir como proyecto raiz: D:\clase micros
- Hacer Reload All Maven Projects para que reconozca dependencias.
- Verificar SDK del proyecto en Java 17.

### Verificaciones rapidas
```powershell
docker --version
docker compose version
docker info
```

---

## 3) CONFIGURACION MINIMA IMPORTANTE

### 3.1 Config Server (infra/config-server)
- Debe estar en puerto 7071.
- Debe usar perfil native.
- Debe apuntar a:

```text
file:///D:/clase micros/infra/config-repo
```

### 3.2 Microservicios dentro de Docker
- Nunca usar localhost para llegar al Config Server si corre fuera del contenedor.
- Deben usar host.docker.internal:7071.
- En application.yml de cada microservicio:

```yaml
spring:
  config:
    import: "optional:configserver:${CONFIG_SERVER_URL:http://host.docker.internal:7071}"
```

### 3.3 Docker Compose
- Cada servicio debe tener:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

- Esto permite que el contenedor vea tu laptop (host).

### 3.4 Perfil activo
- En .env de catalogo y producto:

```dotenv
SPRING_PROFILES_ACTIVE=dev
CONFIG_SERVER_URL=http://host.docker.internal:7071
```

### 3.5 Config repo (infra/config-repo)
- catalogo-dev.yml y producto-dev.yml deben tener datasource por variables:

```text
jdbc:mysql://${..._DB_HOST}:${..._DB_PORT}/${..._DB_NAME}...
```

- Para actuator completo:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

---

## 4) SOBRE POM Y DOCKERFILE (solo lo necesario)

### 4.1 catalogo/pom.xml y producto/pom.xml
- Deben incluir spring-cloud-starter-config.
- Deben incluir springdoc-openapi-starter-webmvc-ui.
- Mantener version de springdoc alineada para evitar diferencias de comportamiento.

### 4.2 Dockerfile de catalogo/producto
- Multi-stage con Maven + JRE esta correcto.
- Debe exponer puerto correcto:
  - catalogo: 8082
  - producto: 8084

### 4.3 BLOQUES EXACTOS DE DEPENDENCIAS POM (listos para copiar)

#### Config Server (infra/config-server/pom.xml)
Dependencias minimas recomendadas:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

Dependency management requerido:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

#### Catalogo (services/catalogo/pom.xml)
Dependencias clave para Spring Cloud + Swagger:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.16</version>
  </dependency>

  <dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
  </dependency>

  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
  </dependency>

  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
  </dependency>
</dependencies>
```

#### Producto (services/producto/pom.xml)
- Debe usar el mismo bloque clave que catalogo.
- Importante: dejar una sola dependencia springdoc (sin duplicados):

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.8.16</version>
</dependency>
```

Dependency management requerido (catalogo/producto):

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

---

## 5) COMPILAR Y EMPAQUETAR JAR (RECOMENDADO)

### Objetivo
- Confirmar que cada servicio compila bien antes de levantar Docker.

### Config Server
```powershell
cd "D:\clase micros\infra\config-server"
.\mvnw clean package -DskipTests
```
Validar que exista: target\*.jar

### Catalogo
```powershell
cd "D:\clase micros\services\catalogo"
.\mvnw clean package -DskipTests
```
Validar que exista: target\*.jar

### Producto
```powershell
cd "D:\clase micros\services\producto"
.\mvnw clean package -DskipTests
```
Validar que exista: target\*.jar

### Equivalente en IntelliJ
- Abrir panel Maven (derecha) y ejecutar Lifecycle -> clean y luego package en cada modulo.
- O crear Run Configuration tipo Maven con comando: clean package -DskipTests.

### Nota
- Tus Dockerfile ya construyen internamente con Maven (multi-stage).
- Este empaquetado previo es para detectar errores temprano y tener control del build.

---

## 6) ORDEN RECOMENDADO DE ARRANQUE

### PASO 1 - Levantar Config Server (fuera de Docker o en Docker)

#### Opcion A (simple): correr Config Server fuera de Docker
```powershell
cd "D:\clase micros\infra\config-server"
.\mvnw spring-boot:run
```
Probar:
- http://localhost:7071/catalogo/dev
- http://localhost:7071/producto/dev

#### Opcion B: correr Config Server en Docker
- Si haces esto, los microservicios deberian apuntar al nombre del servicio de Docker, no a host.docker.internal.
- Si no tienes todo en un mismo compose, usa Opcion A para evitar complejidad.

### PASO 2 - Crear red compartida una sola vez
```powershell
docker network create ms-net
```
Si sale already exists, es normal. Continuar.

### PASO 3 - Levantar catalogo
```powershell
cd "D:\clase micros\services\catalogo"
docker compose down --remove-orphans
docker compose up -d --build
```

### PASO 4 - Levantar producto
```powershell
cd "D:\clase micros\services\producto"
docker compose down --remove-orphans
docker compose up -d --build
```

---

## 7) URLS DE VERIFICACION (CHECKLIST)

### Config Server
- http://localhost:7071/catalogo/dev
- http://localhost:7071/producto/dev

### Catalogo
- http://localhost:8082/actuator/health
- http://localhost:8082/actuator/env
- http://localhost:8082/v3/api-docs
- http://localhost:8082/swagger-ui/index.html
- http://localhost:8082/api/v1/categorias

### Producto
- http://localhost:8084/actuator/health
- http://localhost:8084/actuator/env
- http://localhost:8084/v3/api-docs
- http://localhost:8084/swagger-ui/index.html
- http://localhost:8084/api/v1/productos

---

## 8) ERRORES TIPICOS Y SOLUCION RAPIDA

### Error A: invalid interpolation format ... ${SPRING_PROFILES_ACTIVE:dev}
Causa:
- Sintaxis incorrecta en docker-compose.

Solucion:
- Usar ${SPRING_PROFILES_ACTIVE:-dev}
- Usar ${CONFIG_SERVER_URL:-http://host.docker.internal:7071}

### Error B: network ms-net already exists
Causa:
- Ya habias creado la red antes.

Solucion:
- No recrearla. Continuar con docker compose up.

### Error C: microservicio no conecta al Config Server
Causa:
- Uso de localhost dentro del contenedor.

Solucion:
- Usar host.docker.internal:7071 + extra_hosts.

### Error D: intenta perfil prod cuando trabajas dev
Causa:
- .env con SPRING_PROFILES_ACTIVE=prod.

Solucion:
- Cambiar a dev en .env.

### Error E: Swagger /v3/api-docs da 500
Causa comun:
- Version springdoc incompatible o desalineada.

Solucion:
- Alinear springdoc entre servicios.
- Revisar logs del contenedor.

### Error F: Eureka connection refused localhost:8761
Causa:
- Eureka no esta levantado o URL incorrecta desde contenedor.

Solucion:
- Levantar eureka-server o ajustar URL para Docker.

### Error G: Base de datos no conecta
Causa:
- En config-repo se dejo localhost/puerto host en lugar de nombre del contenedor.

Solucion:
- Usar variables *_DB_HOST=mysql-xxx y *_DB_PORT=3306 dentro de Docker.

### Error H: falla package en IntelliJ pero en terminal no
Causa:
- IntelliJ usa otro JDK o no recargo Maven.

Solucion:
- Ajustar Project SDK a Java 17.
- Maven -> Reload All Projects.
- Ejecutar clean package nuevamente.

### Error I: Docker build falla en paso Maven
Causa:
- Dependencias no descargadas, red lenta o cache corrupta.

Solucion:
- Reintentar build.
- Probar primero .\mvnw clean package -DskipTests en host para validar.
- Revisar proxy/certificados si estas en red corporativa.

---

## 9) COMANDOS DE DIAGNOSTICO UTILES

Ver contenedores activos:
```powershell
docker ps
```

Ver logs catalogo:
```powershell
docker logs -f catalogo-prod-catalogo-1
```

Ver logs producto:
```powershell
docker logs -f producto-prod-producto-1
```

Ver redes:
```powershell
docker network ls
```

Ver detalle de red:
```powershell
docker network inspect ms-net
```

Limpiar recursos no usados (usar con cuidado):
```powershell
docker system prune
docker network prune
```

Comprobar JAR generado rapidamente:
```powershell
Get-ChildItem .\target\*.jar
```

---

## 10) RESUMEN CORTO (MENTAL MODEL)
- config-repo (infra) = fuente de verdad de propiedades.
- config-server (infra) = servidor que publica esas propiedades.
- catalogo/producto (services) = clientes config que arrancan leyendo config-server.
- Docker de microservicios usa host.docker.internal para hablar con config-server en tu host.
- DB dentro de Docker se resuelve por nombre de servicio de compose.

---

Fin.
