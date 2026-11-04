# 02 — Contenedores, supply chain y runtime

La imagen es el artefacto que el pipeline publica y el runtime es donde el producto de IA corre. Las lecciones 01 a 03 de `containers-security` separan el mecanismo (namespaces, cgroups, capabilities, seccomp, overlay, socket) del endurecimiento y de los scanners.

## Lee esto

Cuando existan los Markdown:

- `containers-security/01-docker-fundamentals-and-internals.md`
- `containers-security/02-docker-compose-and-lab-orchestration.md`
- `containers-security/03-container-security-and-escape-techniques.md`

Hasta entonces, las mismas secciones están en `containers-security/demo.txt`.

## Qué es trabajo tuyo dentro de esas lecciones

De la 01:

- Secciones 4 a 7: namespaces, cgroups v2, capabilities, seccomp, overlay. Son el modelo de aislamiento. Las lees para saber qué está roto cuando un manifiesto pide de más.
- Secciones 8 a 10: imágenes, registries, Dockerfile, multi-stage. Es el supply chain del servicio.
- Sección 13: el daemon y el socket. Es el control que después prohíbes.
- Sección 15: el lab "A Hardened Nginx Image". Ese es el patrón de imagen base.
- Sección 16: los ejercicios de esa lección, tal cual, sobre esa imagen.

De la 02:

- Estructura del Compose, networks, volumes, limits, health checks, profiles.
- Sección de secret injection y la de cleanup. El secreto del job no va en el YAML commiteado.
- El ejemplo de lab completo y el multi-target. Te sirven para levantar el producto de IA en local antes de hablar del cluster.
- Sección 17: cuándo el Compose ya no alcanza y toca Kubernetes. Ahí pasas al archivo 03 de esta ruta.

De la 03, en este orden y sin rehacer los escapes:

- Sección 1, el alcance ético, y la 2, el modelo de aislamiento.
- Sección 12, Detection and Mitigation Overview.
- Sección 13, Runtime Image Hardening.
- Sección 14, Container Security Scanners.
- Sección 15 solo en la parte "Then Blocked": el lab ya demuestra el mismo hallazgo bloqueado. Tu entregable es esa segunda corrida y la nota de detección, no una copia nueva del escape.
- Las secciones 4 a 11 (socket, privileged, capabilities, core_pattern, mounts, hostPID, kernel, API cruda) se leen para reconocer la condición en un manifiesto. El procedimiento de escape queda en el archivo original.

## Ejercicios

1. **Imagen endurecida.** Haz el lab de la sección 15 de la lección 01 y los ejercicios de la sección 16. La imagen resultante es la base del servicio, no una imagen de ataque.

2. **Manifiesto.** Sobre el Compose de la lección 02, marca con el texto de la lección 03 cada sitio donde aparecería un socket montado, `privileged`, un capability de más, o un bind mount del host. El arreglo es el que la sección 13 ya describe. Vuelve a levantar el Compose y confirma que el servicio sigue healthy con el health check de la lección 02.

3. **Scanner de imagen.** Corre el scanner que la sección 14 de la lección 03 nombra, contra la imagen del ejercicio 1. El informe es el input del gate de CI del archivo 04 de esta ruta. No hace falta un scanner que la lección no mencione.

4. **Secretos.** Sustituye cualquier secreto en claro del Compose por el mecanismo de la sección de secret injection de la lección 02. El archivo que se commitea no contiene el valor.
