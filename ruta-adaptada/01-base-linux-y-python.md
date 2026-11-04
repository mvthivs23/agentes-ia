# 01 — Base Linux y Python

Sirve para abrir un repo de automatización de seguridad sin pelearte con el intérprete, los permisos ni el servicio que va a correr el job. El propio curso separa esto del trabajo ofensivo: la lección 01 de Python termina en un enumerador de headers con la biblioteca estándar, y las lecciones de Linux cubren el sistema donde ese job vive.

## Lee esto, en este orden

Linux, en `linux-security/demo.txt` (aún no partido):

| Lección | Título | Para qué la usas |
|---|---|---|
| 01 | Introduction to Linux and Kali 2026 Setup | La VM de lab donde pruebas el control antes de llevarlo al pipeline |
| 02 | Linux Filesystem Hierarchy and Navigation | Dónde quedan binarios, unidades y logs |
| 03 | Users, Groups, and the Permission Model | El usuario del runner y el del servicio, no root por costumbre |
| 04 | Process Management and Job Control | Qué proceso quedó colgado después de un job |
| 05 | Shell Mastery: Bash Internals and Power Usage | El shell del workflow |
| 06 | Text Processing Pipeline | `jq` y filtros sobre el JSON que ya emiten las tools del curso |
| 08 | Package Management and Tool Installation | Fijar lo que el job instala |
| 10 | Linux Security Fundamentals | La base de endurecimiento del host |

La 07 (red desde la línea de comandos) entra cuando el job tiene que hablar con un registry o con la API del cluster. La 09 (servicios y persistencia) entra para leer cómo queda un unit de systemd. La 11 y la 12 arman la plataforma de ataque: no son el camino de esta ruta.

Python, cuando terminen de escribirse:

- `python-for-security/01-python-environment-and-language-fundamentals.md`
- `python-for-security/05-python-tool-development-building-real-pentest-tools.md` solo si el control no cabe en un script de la 01

Del repo, el mismo criterio de entorno ya está en `02-entorno-de-trabajo.md` y `02-entorno-paso-a-paso-rapido.md`.

## Qué tomar de la lección 01 de Python

El texto lo dice así: intérprete 3.12+, `venv`, `pipx` o `uv`, tipos, `with` para sockets y archivos, excepciones concretas, `ruff` y `mypy`. La tabla de ecosistema nombra `requests`/`httpx` para HTTP. Eso alcanza para un chequeo de headers y para fusionar dos JSON de un scanner.

La 05 construye tools de pentest. Úsala cuando el script de la 01 se quede corto. No arranques por ahí.

## Ejercicios

Son los de la propia lección 01, con el artefacto cambiado a un repo de automatización. Los objetivos están en la sección de ejercicios de esa lección (ejercicios 3, 4, 5 y 6, más el enumerador de headers del cierre).

1. **Proyecto aislado.** Crea el virtualenv como indica la lección (`uv` si está, `venv` si no) dentro de un repo `security-automations`. El entregable es el mismo que pide la lección: un proyecto que otro operador puede leer, con tipos y sin paquetes globales.

2. **Headers.** El cierre de la lección construye un enumerador de headers HTTP solo con la biblioteca estándar. Apúntalo a un servicio que el equipo ya despliega en el lab. El fallo que buscas es el que la lección de arquitectura web y la de API gateway tratan del lado del control: headers de seguridad ausentes. No agregues payloads. Compara la lista de headers que el script imprime contra los que esas lecciones nombran.

3. **Fusionar reportes.** El ejercicio 3 (`merge_scans.py`) une dos JSON `{"host", "open_ports"}`. Úsalo igual, con dos salidas de un scanner que ya corras en CI. El resultado es un solo reporte por host.

4. **Tiempo de un paso.** El ejercicio 4 extiende el decorator `@timed` con un umbral. Mide un paso del pipeline (el scan de imagen o el chequeo de headers). Si pasa el umbral, el warning va a stderr, como dice el ejercicio.

5. **Auth header de un job.** El ejercicio 6 arma un header `Authorization: Basic`. Sirve para un cliente de la API interna del lab. La lección ya pide UTF-8 antes de base64. No guardes el secreto en el repo: el patrón de secretos está en la lección 02 de contenedores (secret injection) y en la 06 de CI/CD.
