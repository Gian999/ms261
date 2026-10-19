# ms261: Microservicios con Spring Boot

Práctica de arquitectura de microservicios con Java y Spring Boot. Es la base de un sistema de prácticas preprofesionales que busca reemplazar el manejo en Excel que hoy se usa en la universidad.

## Qué incluye
- `services/catalogo`: microservicio de catálogo.
- `services/producto`: microservicio de producto.
- `infra/`: configuración centralizada con **Spring Cloud Config Server**.
- Despliegue con **Docker**.
- `GUIA_CONEXION_DOCKER_PASO_A_PASO.md`: guía para levantar Config Server y los microservicios en Docker, explicando cómo se conectan `infra` y `services`.

## Stack
Java 17 · Spring Boot · Spring Cloud Config · Docker · Maven

## Estado
En progreso. Primera fase terminada: configuración centralizada, Docker y los dos microservicios base.

## Autor
Gianluck Pastor, estudiante de Ingeniería de Sistemas (UPeU).
