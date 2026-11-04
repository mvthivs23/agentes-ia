# Lección 02 — Docker Compose and Lab Orchestration

    Module 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Todo pentester serio necesita un laboratorio, y todo laboratorio serio necesita orquestación. Ejecutar un único container vulnerable con `docker run` vale para un experimento de cinco minutos, pero en el momento en que necesitas una aplicación web vulnerable hablando con una base de datos vulnerable, ambas aisladas en una red privada, con un container atacante acoplado y un sidecar sniffer grabando el tráfico, necesitas Docker Compose.

Compose es la diferencia entre un laboratorio que puedes reconstruir en treinta segundos y un laboratorio que reconstruyes recordando diecisiete flags de `docker run` a las dos de la mañana. Te da reproducibilidad (el laboratorio de tu repositorio git es el laboratorio que ejecuta todo el mundo), aislamiento (cada proyecto recibe su propia red por defecto) y desmontaje (`docker compose down -v` destruye todo limpiamente, sin recursos filtrados en tu host).

Esta lección cubre Compose v2 en profundidad, desde el formato de archivo hasta la CLI de runtime, y después construye entornos de laboratorio de pentest cada vez más grandes: empieza con un único objetivo vulnerable y termina con un laboratorio de varias capas que incluye un container atacante, una aplicación web vulnerable, una base de datos vulnerable, un servidor SSH vulnerable y un sidecar sniffer, todos cableados con un orden de arranque gobernado por healthchecks.

Al terminar vas a poder diseñar, publicar y desmontar un laboratorio de pentest autocontenido en un único archivo YAML.

## 1. Por qué todo pentester serio ejecuta los objetivos en Compose

Antes de entrar en la sintaxis, entiende qué problema resuelve Compose.

### 1.1 Reproducibilidad

Un `docker-compose.yml` en un repositorio git es una descripción completa y ejecutable de un entorno de laboratorio. Cualquiera con Docker instalado puede clonar el repositorio, ejecutar `docker compose up` y obtener exactamente el mismo laboratorio que tú tienes: las mismas images, las mismas redes, los mismos volumes, las mismas variables de entorno. Cuando escribes un walkthrough de una vulnerabilidad, puedes publicar el laboratorio junto a él y quien lee puede reproducir tus hallazgos sin una guía de instalación de cincuenta pasos.

### 1.2 Aislamiento

Cada proyecto de Compose recibe su propia red bridge por defecto. Los containers del proyecto A no pueden alcanzar los del proyecto B salvo que los conectes a propósito. Esto importa cuando ejecutas servicios vulnerables a propósito en tu estación de trabajo: no quieres DVWA en el puerto 80 expuesto por accidente a tu red de casa, y no quieres un container Metasploitable escuchando en puertos que también usa tu stack de desarrollo legítimo.

### 1.3 Desmontaje

`docker compose down -v` detiene cada container, elimina cada red y destruye cada named volume que creó el proyecto. No hay un problema de «¿qué containers arranqué esta mañana?», ni bridges huérfanos ensuciando `docker network ls`, ni volumes consumiendo disco en silencio. La limpieza es un solo comando y es completa.

### 1.4 El coste de no usar Compose

```bash
# La forma incorrecta de ejecutar un laboratorio de dos containers:
docker network create lab-net
docker run -d --name db --network lab-net -e MYSQL_ROOT_PASSWORD=root mysql:5.7
docker run -d --name web --network lab-net -p 8080:80 -e DB_HOST=db vulnerables/web-dvwa

# Tres comandos, tres oportunidades de olvidar un flag, sin control de versiones,
# y no hay forma de decirle a un colaborador "ejecuta esto" salvo pegando todo el bloque.
# Y cuando terminas:
docker stop web db
docker rm web db
docker network rm lab-net
# ¿También creaste un volume? Espera acordarte.
```

Compose colapsa todo eso en `docker compose up -d` y `docker compose down -v`.

## 2. CLI de Compose v2

Compose v2 no es la misma herramienta que Compose v1. Entiende la diferencia antes de copiar un tutorial viejo.

### 2.1 v1 frente a v2

Compose v1 era un programa Python independiente que se invocaba como `docker-compose` (con guion). Quedó obsoleto en 2023 y salió del soporte activo. Compose v2 es un plugin en Go de la CLI de Docker, que se invoca como `docker compose` (con un espacio).

```bash
# Viejo (v1, obsoleto):
docker-compose up -d

# Nuevo (v2, actual):
docker compose up -d

# Comprueba cuál tienes:
docker compose version
# Docker Compose version v2.29.7

# Si esto falla, estás en v1 o Compose no está instalado:
docker-compose --version
# docker-compose version 1.29.2 (legacy)
```

### 2.2 Arquitectura de plugins

Compose v2 vive en `/usr/libexec/docker/cli-plugins/docker-compose` (o en `~/.docker/cli-plugins/`) y la CLI de Docker lo descubre en el momento de la invocación. Cualquier binario llamado `docker-<algo>` en un directorio de plugins se convierte en `docker <algo>`. Así funcionan `docker buildx`, `docker scan` y `docker compose`.

```bash
# Lista los plugins de CLI instalados:
docker info --format '{{range .ClientInfo.Plugins}}{{.Name}} {{.Version}}{{"\n"}}{{end}}'
# buildx v0.16.2
# compose v2.29.7
# scan 0.23.0

# ¿Dónde está el binario de compose?
ls -l /usr/libexec/docker/cli-plugins/docker-compose
# -rwxr-xr-x 1 root root 58M Sep  1 12:00 docker-compose
```

### 2.3 Notas de migración

La mayoría de los archivos de Compose v1 funcionan sin cambios en v2, pero hay diferencias de comportamiento que conviene conocer:

| Aspecto | v1 (`docker-compose`) | v2 (`docker compose`) |
| --- | --- | --- |
| Invocación | Comando con guion | Subcomando de `docker` |
| Clave `version:` | Obligatoria (p. ej. `version: '3.8'`) | Opcional e ignorada |
| Nombres de container | `project_service_1` | `project-service-1` |
| Nombres de red | `project_default` | `project_default` (sin cambio) |
| Salida del build | Usa el builder legado | Usa BuildKit por defecto |
| Rendimiento | Arranque del intérprete Python | Binario Go, arranque en frío más rápido |

El cambio del separador en el nombre del container es el que tropieza a la gente en los scripts: si tenías `docker exec myproject_web_1 sh`, ahora es `docker exec myproject-web-1 sh`.

### 2.4 Comandos centrales

```bash
# Levanta todo el proyecto en segundo plano:
docker compose up -d
# [+] Running 3/3
#  ✔ Network lab_default    Created
#  ✔ Container lab-db-1     Started
#  ✔ Container lab-web-1    Started

# Detén y elimina todo lo que creó el proyecto (containers, redes):
docker compose down

# Igual, pero también borra los named volumes:
docker compose down -v

# Sigue los logs de todos los servicios:
docker compose logs -f

# Reconstruye las images antes de arrancar:
docker compose up -d --build

# Reinicia un solo servicio:
docker compose restart web

# Renderiza la config totalmente resuelta (tras la sustitución de entorno y los merges):
docker compose config
```

## 3. Estructura del archivo Compose

Un archivo Compose es YAML con un número pequeño de claves de nivel superior.

### 3.1 Claves de nivel superior

```yaml
# docker-compose.yml — un archivo v2 mínimo. Sin clave version:.

services:      # containers que ejecutar
networks:      # redes que crear (opcional; se crea una por defecto)
volumes:       # named volumes que crear (opcional)
configs:       # archivos de configuración no secretos (orientado a swarm)
secrets:       # archivos de secretos (orientado a swarm, también funciona en local)
x-anything:    # campos de extensión propios, Compose los ignora
```

La única clave que siempre necesitas es `services`. Todo lo demás es opcional.

### 3.2 Se acabó `version:`

En Compose v2 el campo `version:` se ignora. Aun así vas a ver archivos así en la naturaleza:

```yaml
version: '3.8'    # Compose v2: se ignora, sin aviso
services:
  web:
    image: nginx
```

Borra la línea `version:` cuando escribas archivos nuevos. Ya no selecciona una versión de esquema: Compose v2 usa la Compose Specification, que es una única especificación que evoluciona, no un conjunto de esquemas versionados.

### 3.3 Convenciones de nombre de archivo

Compose busca estos nombres en el directorio actual, en este orden:

- `compose.yaml`
- `compose.yml`
- `docker-compose.yaml`
- `docker-compose.yml`

Los proyectos nuevos deberían usar `compose.yaml`. Esta lección usa `docker-compose.yml` porque es lo que todavía publican la mayoría de los repositorios de laboratorios de pentest.

## 4. El bloque services en profundidad

El bloque `services` es donde defines los containers. Cada clave de nivel superior bajo `services:` es un nombre de servicio: el nombre lógico de uno o más containers que ejecutan la misma image.

### 4.1 `image` frente a `build`

Un servicio o saca una image ya construida o la construye desde un Dockerfile. También puedes hacer las dos cosas: construir con un tag concreto para que la image quede en caché.

```yaml
services:
  # Saca una image ya construida de un registry:
  dvwa:
    image: vulnerables/web-dvwa:latest

  # Construye desde un Dockerfile local:
  attacker:
    build:
      context: ./attacker           # directorio que contiene el Dockerfile
      dockerfile: Dockerfile.kali   # opcional, por defecto Dockerfile
      args:                         # ARGs de build
        KALI_VERSION: "2024.3"

  # Construye y etiqueta, para poder reutilizar la image:
  custom-target:
    build: ./target
    image: lab/custom-target:1.0    # tag aplicado a la image construida
```

### 4.2 `ports`

`ports` publica puertos del container en el host. Tiene forma corta y forma larga.

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"                # host 8080 -> container 80, todas las interfaces
      - "127.0.0.1:8081:80"      # host 8081 solo en loopback -> container 80
      - "8443:443/tcp"           # protocolo explícito
      - "5353:53/udp"            # UDP
      - target: 80               # forma larga
        published: 8082
        protocol: tcp
        mode: host
```

> **Nota de pentester:** En el laboratorio, enlaza siempre los servicios vulnerables a `127.0.0.1` salvo que quieras a propósito que sean alcanzables en la LAN. `"127.0.0.1:8080:80"` evita que tu DVWA sea escaneable por cualquiera en el wifi de la misma cafetería.

### 4.3 `environment` y `env_file`

```yaml
services:
  db:
    image: postgres:16
    environment:
      # Forma de mapa:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: hunter2
      POSTGRES_DB: appdata
      # o forma de lista:
      # - POSTGRES_USER=admin
      # - POSTGRES_PASSWORD=hunter2

  web:
    image: myapp
    env_file:
      - .env                # se carga primero
      - .env.local          # pisa los valores de .env
```

Los valores de `env_file` se fijan dentro del container, pero no se usan para la sustitución `${VAR}` del propio archivo compose: para eso está el archivo `.env` en la raíz del proyecto (ver la sección 7).

### 4.4 `depends_on` con condiciones

El `depends_on` básico solo impone el orden de arranque, no la disponibilidad. Para la disponibilidad, usa la forma larga con condiciones.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: labpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10

  web:
    image: myapp
    depends_on:
      db:
        condition: service_healthy      # espera a que pase el healthcheck de db
        restart: true                   # reinicia web si db se reinicia

  migrator:
    image: myapp
    command: ["./migrate.sh"]
    depends_on:
      db:
        condition: service_healthy

  seed:
    image: myapp
    command: ["./seed.sh"]
    depends_on:
      migrator:
        condition: service_completed_successfully  # espera un exit 0
```

Las tres condiciones válidas son `service_started`, `service_healthy` y `service_completed_successfully`.

### 4.5 `healthcheck`

```yaml
services:
  api:
    image: myapi
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 30s        # periodo de gracia antes de que los fallos cuenten
      start_interval: 2s       # sondeo más rápido durante start_period
```

Usa `CMD` cuando la image ya tiene el binario, y `CMD-SHELL` cuando necesitas funciones del shell:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 3306 || exit 1"]
```

Desactiva un healthcheck heredado poniendo `test: ["NONE"]`.

### 4.6 `restart`

```yaml
services:
  web:
    image: nginx
    restart: unless-stopped
    # Opciones:
    # no             — valor por defecto; no reinicia nunca
    # always         — reinicia en cada salida y al arrancar el daemon de Docker
    # on-failure     — solo si el container salió con código distinto de cero
    # unless-stopped — siempre, excepto si un usuario lo detuvo a mano
```

Para el laboratorio, `unless-stopped` es el valor por defecto correcto. `always` se pelea contigo cuando detienes un servicio a propósito para depurar.

## 5. Redes

Las redes de Compose son donde ocurre gran parte de la orquestación de pentest, porque a menudo quieres un aislamiento muy controlado entre los containers atacante, objetivo y de monitorización.

### 5.1 Bridge por defecto por proyecto

Si no defines ninguna red, Compose crea una red bridge llamada `<proyecto>_default` y engancha cada servicio a ella. Los servicios pueden alcanzarse entre sí por el nombre del servicio.

```yaml
# Sin bloque networks. Todos los servicios comparten <proyecto>_default.
services:
  web:
    image: nginx
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: pw
# Dentro de web: `ping db` funciona, `getent hosts db` devuelve la IP del container.
```

### 5.2 Bridges definidos por el usuario con IPAM

```yaml
services:
  attacker:
    image: kalilinux/kali-rolling
    networks:
      lab_net:
        ipv4_address: 10.99.0.10
    cap_add:
      - NET_ADMIN
      - NET_RAW

  target:
    image: vulnerables/web-dvwa
    networks:
      lab_net:
        ipv4_address: 10.99.0.20

networks:
  lab_net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.99.0.0/24
          gateway: 10.99.0.1
    driver_opts:
      com.docker.network.bridge.name: br-lab
      com.docker.network.bridge.enable_icc: "true"
```

IPAM fija la subred para que tu container atacante tenga siempre un rango de IP de objetivo conocido: útil cuando escribes scripts de explotación que dejan `10.99.0.20` escrito a fuego.

### 5.3 Redes externas para encadenar laboratorios

Si quieres que dos proyectos Compose distintos compartan una red —por ejemplo un proyecto de «infraestructura compartida» que ejecuta un proxy y un proyecto «objetivo» que se engancha a ella— declara la red como externa.

```yaml
# infra/docker-compose.yml — el dueño de la red
services:
  proxy:
    image: nginx
    networks:
      - shared

networks:
  shared:
    name: shared_lab_net
    driver: bridge
    ipam:
      config:
        - subnet: 10.200.0.0/24
```

```yaml
# target/docker-compose.yml — se engancha a la misma red
services:
  target:
    image: vulnerables/web-dvwa
    networks:
      - shared

networks:
  shared:
    external: true
    name: shared_lab_net       # tiene que existir ya
```

Levanta primero infra y después target. El container objetivo queda alcanzable a través del proxy.

### 5.4 `network_mode: host`

`network_mode: host` desactiva el namespacing de red de un container: comparte directamente la pila de red del host. De vez en cuando sirve para un sniffer o un escáner que necesita ver el tráfico a nivel de host, pero las contrapartidas son graves:

```yaml
services:
  sniffer:
    image: nicolaka/netshoot
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    command: ["tcpdump", "-i", "any", "-w", "/pcap/dump.pcap"]
    volumes:
      - ./pcap:/pcap
```

Implicaciones:

- El container ignora todas las directivas `networks:` y `ports:`.
- Los procesos del container se enlazan directamente a puertos del host; los conflictos son reales.
- En Docker Desktop para macOS y Windows, el modo host se comporta distinto: el «host» es la VM Linux, no tu estación de trabajo.
- El aislamiento desaparece. No ejecutes images en las que no confíes con `network_mode: host`.

## 6. Volumes

El estado persistente vive en volumes. Hay dos clases que importan: named volumes y bind mounts.

### 6.1 Named volumes

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
    # No hace falta nada más. Docker gestiona el almacenamiento bajo
    # /var/lib/docker/volumes/<proyecto>_db_data/_data
```

Los named volumes sobreviven a `docker compose down` y solo se borran con `docker compose down -v` o `docker volume rm`. Úsalos para cualquier cosa que el container necesite entre reinicios: archivos de base de datos, artefactos subidos, claves generadas.

### 6.2 Bind mounts

```yaml
services:
  web:
    image: nginx
    volumes:
      - ./html:/usr/share/nginx/html:ro     # bind mount de solo lectura
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - /var/log/host-nginx:/var/log/nginx  # path absoluto del host
```

Los bind mounts son perfectos para desarrollar exploits: editas `./exploit.py` en el host con tu editor habitual y el container atacante ve el cambio al instante.

### 6.3 `driver_opts` para tmpfs y NFS

```yaml
volumes:
  scratch:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: "size=256m,uid=1000"

  shared_nfs:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.10.10.5,rw"
      device: ":/exports/lab"
```

Un named volume respaldado por tmpfs sirve para datos volátiles que no quieres en disco: credenciales capturadas en un laboratorio, por ejemplo.

### 6.4 Patrón de backup

No existe `docker compose backup`. La forma idiomática de hacer un snapshot de un named volume es un container desechable con el volume montado:

```bash
# Respalda el volume db_data a un tarball en el host:
docker run --rm \
  -v myproject_db_data:/data:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/db_data.tgz -C /data .

# Restaura:
docker run --rm \
  -v myproject_db_data:/data \
  -v "$PWD":/backup \
  alpine sh -c "cd /data && tar xzf /backup/db_data.tgz"
```

## 7. Sustitución de entorno

Compose expande `${VAR}` en el propio archivo usando variables del shell y de un archivo `.env` en la raíz del proyecto.

### 7.1 El archivo `.env`

```bash
# .env  (en el mismo directorio que docker-compose.yml)
LAB_SUBNET=10.99.0.0/24
DB_PASSWORD=labpass
DVWA_TAG=latest
```

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  dvwa:
    image: vulnerables/web-dvwa:${DVWA_TAG}

networks:
  lab_net:
    ipam:
      config:
        - subnet: ${LAB_SUBNET}
```

El archivo `.env` se lee solo para la sustitución dentro del archivo compose. Para inyectar variables en un container, usa `environment:` o `env_file:` (sección 4.3).

### 7.2 Valores por defecto y errores

```yaml
services:
  web:
    image: nginx:${NGINX_TAG:-stable}          # "stable" si no está definida
    ports:
      - "${WEB_PORT:-8080}:80"

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
      # Compose se niega a arrancar si DB_PASSWORD no está definida o está vacía.
```

Resumen de la sintaxis:

| Forma | Comportamiento |
| --- | --- |
| `${VAR}` | Cadena vacía si no está definida |
| `${VAR:-default}` | `default` si no está definida o está vacía |
| `${VAR-default}` | `default` solo si no está definida |
| `${VAR:?message}` | Falla con el mensaje si no está definida o está vacía |
| `${VAR?message}` | Falla con el mensaje solo si no está definida |

### 7.3 Verificar la sustitución

```bash
docker compose config
# Renderiza el archivo con todas las sustituciones resueltas.
# Úsalo para pillar erratas antes de `up`.
```

## 8. Profiles para servicios opcionales

Los profiles te permiten tener varios servicios relacionados en un solo archivo y arrancar solo subconjuntos bajo demanda. Es ideal para laboratorios de pentest en los que hay objetivos «siempre encendidos» y containers «atacante» que solo corren cuando estás trabajando.

```yaml
services:
  target-web:
    image: vulnerables/web-dvwa
    profiles: ["target"]

  target-db:
    image: mysql:5.7
    profiles: ["target"]

  attacker:
    image: kalilinux/kali-rolling
    profiles: ["attacker"]
    stdin_open: true
    tty: true
    command: ["bash"]

  monitor:
    image: nicolaka/netshoot
    profiles: ["monitor"]
```

Arranca solo el profile de objetivo:

```bash
docker compose --profile target up -d

# Arranca objetivo y atacante juntos:
docker compose --profile target --profile attacker up -d

# Un servicio sin profile se arranca siempre.
```

Un patrón habitual es dejar el conjunto central de objetivos sin etiqueta (siempre arriba) y meter al atacante y al sniffer en profiles.

## 9. Overrides

Casi nunca quieres un único archivo compose plano. En la práctica quieres un archivo base más overrides específicos del entorno.

### 9.1 Override automático

Si en el mismo directorio existen `docker-compose.yml` y `docker-compose.override.yml`, Compose los fusiona automáticamente.

```yaml
# docker-compose.yml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
```

```yaml
# docker-compose.override.yml
services:
  web:
    volumes:
      - ./html:/usr/share/nginx/html
    environment:
      DEBUG: "true"
```

```bash
docker compose up -d
# Equivale a un único archivo en el que web tiene image, ports,
# volumes y environment, todo fusionado.
```

### 9.2 Merge explícito con `-f`

```bash
# Solo base + overrides de prod, sin override.yml:
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Base + overlay del atacante para el laboratorio:
docker compose -f docker-compose.yml -f docker-compose.attacker.yml up -d
```

Cuando se especifica `-f`, no se aplica el `docker-compose.override.yml` automático. Los archivos se fusionan de izquierda a derecha; los posteriores pisan a los anteriores.

### 9.3 Semántica del merge

- Los escalares (`image`, `restart`) se sobrescriben.
- Las secuencias de primitivos (`ports`, `expose`) se sustituyen enteras: gana la lista del override.
- Los mapas (`environment`, `labels`) se fusionan clave a clave.
- Las secuencias de mapas (volumes, enganches de redes) se fusionan por destino.

Usa `docker compose config` para ver exactamente a qué se fusiona.

## 10. Límites de recursos

Los containers vulnerables a veces se hacen una fork bomb o se bloquean bajo la explotación. Pon techos de recursos para que un container desbocado no deje tu estación de trabajo inservible.

```yaml
services:
  target:
    image: vulnerables/web-dvwa
    cpus: 1.5                      # hasta 1.5 núcleos de CPU
    mem_limit: 512m                # tope duro de memoria
    memswap_limit: 512m            # sin swap más allá de mem_limit
    mem_reservation: 128m          # suelo blando
    pids_limit: 200                # tope de la tabla de procesos

  db:
    image: postgres:16
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1g
        reservations:
          memory: 512m
```

Las claves de nivel superior `cpus` y `mem_limit` son las claves clásicas de Compose y funcionan con `docker compose up`. La forma `deploy.resources` es portable a Swarm y a herramientas al estilo Kubernetes; también funciona en local en las versiones recientes de Compose.

## 11. Construir entornos de laboratorio — un ejemplo completo que funciona

El laboratorio inicial clásico: DVWA como objetivo, un container Kali como atacante, ambos en una red bridge aislada. Nada expuesto a la LAN.

```yaml
# docker-compose.yml — DVWA + atacante Kali en un bridge aislado

services:
  dvwa:
    image: vulnerables/web-dvwa:latest
    container_name: lab-dvwa
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"        # solo loopback desde el host
    networks:
      lab_net:
        ipv4_address: 10.99.0.20
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost/ >/dev/null || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s

  attacker:
    image: kalilinux/kali-rolling
    container_name: lab-attacker
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
      - NET_RAW
    networks:
      lab_net:
        ipv4_address: 10.99.0.10
    depends_on:
      dvwa:
        condition: service_healthy
    volumes:
      - ./work:/root/work
    command: >
      bash -c "apt-get update -qq &&
               apt-get install -yqq nmap curl netcat-openbsd &&
               exec bash"

networks:
  lab_net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.99.0.0/24
          gateway: 10.99.0.1
```

Levántalo y entra:

```bash
docker compose up -d
# [+] Running 2/2
#  ✔ Container lab-dvwa      Healthy
#  ✔ Container lab-attacker  Started

docker compose exec attacker bash
# root@lab-attacker:/# nmap -sS 10.99.0.20
# Starting Nmap 7.94 ...
# 80/tcp open  http

# Desde el host, DVWA está en http://127.0.0.1:8080/
# Desde cualquier otra máquina de la LAN, es inalcanzable.

docker compose down -v
# [+] Running 3/3
#  ✔ Container lab-attacker  Removed
#  ✔ Container lab-dvwa      Removed
#  ✔ Network lab_lab_net     Removed
```

## 12. Compose de laboratorio con varios objetivos

Un laboratorio más realista: una aplicación web vulnerable que habla con una base de datos vulnerable, un servidor SSH vulnerable y un container atacante que puede pivotar a través de todos ellos.

```yaml
# docker-compose.yml — laboratorio de pentest con varios objetivos

services:
  vuln-web:
    image: vulnerables/web-dvwa:latest
    container_name: mt-web
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"
    networks:
      dmz:
        ipv4_address: 10.10.10.20
      internal:
        ipv4_address: 10.10.20.20
    depends_on:
      vuln-db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost/ >/dev/null || exit 1"]
      interval: 10s
      retries: 5
      start_period: 20s

  vuln-db:
    image: mysql:5.7
    container_name: mt-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: dvwa
      MYSQL_USER: dvwa
      MYSQL_PASSWORD: p@ssw0rd
    networks:
      internal:
        ipv4_address: 10.10.20.30
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -proot"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 30s

  vuln-ssh:
    image: rastasheep/ubuntu-sshd:14.04
    container_name: mt-ssh
    restart: unless-stopped
    networks:
      internal:
        ipv4_address: 10.10.20.40
    # Credenciales por defecto root:root — esto es un laboratorio, no lo expongas.

  attacker:
    image: kalilinux/kali-rolling
    container_name: mt-attacker
    stdin_open: true
    tty: true
    cap_add:
      - NET_ADMIN
      - NET_RAW
    networks:
      dmz:
        ipv4_address: 10.10.10.10
      # a propósito NO está en internal — el atacante tiene que pivotar por vuln-web
    volumes:
      - ./work:/root/work
    depends_on:
      vuln-web:
        condition: service_healthy
    command: >
      bash -c "apt-get update -qq &&
               apt-get install -yqq nmap curl netcat-openbsd hydra sshpass &&
               exec bash"

networks:
  dmz:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.10.0/24
  internal:
    driver: bridge
    internal: true                # sin gateway de salida; simula una LAN de back-end
    ipam:
      config:
        - subnet: 10.10.20.0/24

volumes:
  db_data:
```

Disposición de la red:

```text
                +------------------------+
   host <-----> |  dmz  10.10.10.0/24   |
                |                        |
                |  attacker  .10         |
                |  vuln-web  .20         |
                +------------+-----------+
                             |
                +------------+-----------+
                |  internal  10.10.20.0/24  (internal: true, no egress)
                |                        |
                |  vuln-web  .20         |
                |  vuln-db   .30         |
                |  vuln-ssh  .40         |
                +------------------------+
```

Flujo de ataque:

```bash
docker compose up -d
docker compose exec attacker bash

# Escanea la DMZ:
root@mt-attacker:~# nmap -sS 10.10.10.0/24
# 10.10.10.20 open 80

# Explota vuln-web (SQLi, ejecución de comandos, lo que sea) para obtener RCE dentro de vuln-web.
# Desde vuln-web, pivota hacia internal:
root@mt-web:/# ping -c 1 10.10.20.30    # vuln-db alcanzable
root@mt-web:/# nc -zv 10.10.20.40 22    # vuln-ssh alcanzable

# El container atacante nunca puede alcanzar 10.10.20.0/24 directamente porque
# no está enganchado a la red internal. Todo el acceso L3 al
# back-end tiene que transitar por vuln-web — igual que en los entornos reales.
```

El flag `internal: true` de la red internal es la clave: quita el gateway por defecto dentro de esa red, así que los containers de internal no pueden llegar a internet salvo que otro container haga de proxy. Esto modela un back-end segmentado de forma realista.

## 13. Health checks que importan

Un healthcheck mal hecho es peor que no tenerlo: da una confianza falsa y hace que `depends_on: service_healthy` tenga éxito antes de que la dependencia esté de verdad lista. Hazlos bien.

### 13.1 Disponibilidad de PostgreSQL

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: labpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d postgres -h 127.0.0.1"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 30s
```

`pg_isready` es la sonda correcta porque devuelve éxito solo cuando Postgres acepta conexiones, no solo cuando el proceso está en marcha. El `start_period` le da a Postgres tiempo para inicializarse en el primer arranque (ejecutar initdb, crear el cluster) sin que las sondas fallidas cuenten contra el presupuesto de reintentos.

### 13.2 Disponibilidad de MySQL

```yaml
services:
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
    healthcheck:
      test:
        - CMD-SHELL
        - mysqladmin ping -h 127.0.0.1 -uroot -p"$$MYSQL_ROOT_PASSWORD" || exit 1
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 30s
```

Fíjate en el `$$` doble: Compose interpreta un solo `$` como sustitución, así que para pasar un `$` literal al shell del container lo doblas.

### 13.3 Salud de un endpoint HTTP

```yaml
services:
  api:
    image: myapi:latest
    healthcheck:
      test: ["CMD", "curl", "-fsS", "-o", "/dev/null", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
```

Requiere `curl` en la image. Para images mínimas sin curl:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/healthz >/dev/null || exit 1"]
```

O, si no hay ninguno de los dos:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "exec 3<>/dev/tcp/localhost/8080 && printf 'GET /healthz HTTP/1.0\\r\\n\\r\\n' >&3 && grep -q '200 OK' <&3"]
```

La cadena que el script compara, `200 OK`, se queda en inglés.

### 13.4 Comprobación de un puerto TCP a pelo

```yaml
services:
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep -q PONG"]
      interval: 5s
      retries: 10

  raw-tcp-service:
    image: alpine
    healthcheck:
      test: ["CMD-SHELL", "nc -z 127.0.0.1 4444 || exit 1"]
      interval: 5s
      retries: 10
```

`PONG` es la cadena que compara `grep`: se queda en inglés.

### 13.5 Inspeccionar el estado de salud

```bash
docker compose ps
# NAME       IMAGE           STATUS
# mt-db      mysql:5.7       Up 12 seconds (healthy)
# mt-web     dvwa:latest     Up 5 seconds (health: starting)

docker inspect --format '{{json .State.Health}}' mt-db | jq
# {
#   "Status": "healthy",
#   "FailingStreak": 0,
#   "Log": [ ... last 5 probe results ... ]
# }
```

## 14. Inyección de secretos

No hagas commit de secretos en los archivos compose. Hay tres estrategias habituales, de peor a mejor.

### 14.1 Variables de entorno

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD required}
```

Simple, y suficiente para el laboratorio, donde el «secreto» es `labpass`. La pega: las variables de entorno se ven en `docker inspect`, en `/proc/<pid>/environ` y para cualquier proceso de dentro del container.

### 14.2 Config montada por bind

```yaml
services:
  app:
    image: myapp
    volumes:
      - ./secrets/config.yaml:/etc/myapp/config.yaml:ro
```

Mejor que las variables de entorno: `docker inspect` no las expone, y solo se pueden leer dentro del container. Mantén el directorio `./secrets/` en `.gitignore`.

### 14.3 Docker Secrets

Docker Secrets nació en Swarm, pero también funciona en archivos Compose que no son de Swarm, respaldado por bind mounts por debajo. Montan archivos en `/run/secrets/<nombre>` con modo `0400`.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

La convención `_FILE` (usada por las images oficiales de Postgres, MySQL y muchas otras) lee la contraseña de un archivo en el arranque en lugar de una variable de entorno. Combinada con `secrets:`, es el mejor valor por defecto fuera de Swarm.

En modo Swarm de verdad usarías `external: true` y gestionarías el valor con `docker secret create`: queda fuera del alcance de este laboratorio.

## 15. Disciplina de limpieza

Los containers, las redes y los volumes que se filtran no son solo desorden: son un riesgo de higiene del laboratorio. Un container vulnerable viejo que sigue escuchando en el puerto 3306 la próxima vez que te conectas al wifi de una cafetería es un riesgo real.

### 15.1 Desmontaje estándar

```bash
# Detén y elimina containers + redes, conserva los named volumes:
docker compose down

# Elimina también los named volumes:
docker compose down -v

# Elimina también las images que este proyecto construyó (no las que sacó):
docker compose down --rmi local

# Nuclear:
docker compose down -v --rmi all --remove-orphans
```

### 15.2 Podar la morralla que no es de este proyecto

```bash
# Elimina containers parados, redes sin uso, images colgantes y caché de build:
docker system prune

# Igual, más todas las images sin uso (no solo las colgantes):
docker system prune -a

# Elimina también los named volumes sin uso (VA A destruir datos):
docker system prune -a --volumes
```

Nunca ejecutes `docker system prune -a --volumes` dentro de un directorio de trabajo que todavía tenga proyectos activos, salvo que hayas verificado con `docker compose ls` que están abajo.

### 15.3 Auditar qué está en ejecución

```bash
# Lista cada proyecto Compose del host:
docker compose ls
# NAME             STATUS              CONFIG FILES
# lab              running(3)          /opt/labs/lab/docker-compose.yml
# multi-target     exited(4)           /opt/labs/mt/docker-compose.yml

# Lista cada container, sea del proyecto que sea:
docker ps -a

# Cada red, marcando las que gestiona compose:
docker network ls --filter "label=com.docker.compose.project"

# Cada volume creado por cualquier proyecto compose:
docker volume ls --filter "label=com.docker.compose.project"
```

Ejecuta esto antes de cerrar el portátil.

## 16. Técnicas de depuración

Cuando un stack de compose no hace lo que esperas, estos son los primeros cuatro movimientos.

### 16.1 `docker compose config`

```bash
docker compose config
# Imprime el YAML totalmente resuelto después de:
#  - fusionar todos los archivos -f
#  - aplicar docker-compose.override.yml si existe
#  - resolver cada sustitución ${VAR}
#  - normalizar las formas cortas a formas largas
```

Si la config renderizada no coincide con lo que esperas, ninguna otra depuración va a ayudar.

### 16.2 `docker compose logs`

```bash
# Todos los logs, modo follow:
docker compose logs -f

# Un servicio, modo follow, últimas 100 líneas:
docker compose logs -f --tail=100 db

# Marcas de tiempo:
docker compose logs -f -t web

# Desde un momento:
docker compose logs --since 5m web
```

### 16.3 `docker compose exec`

```bash
# Shell interactivo dentro de un servicio en ejecución:
docker compose exec attacker bash

# Como root aunque la image corra como un usuario:
docker compose exec -u 0 web sh

# Comando puntual:
docker compose exec db psql -U postgres -c 'select version();'
```

`exec` exige que el servicio esté arriba. Para un servicio parado, usa `docker compose run --rm <servicio> <cmd>`.

### 16.4 `docker compose ps` e inspect

```bash
docker compose ps
docker compose ps -a           # incluye los que salieron

docker compose top             # ps -ef dentro de cada container

docker compose port web 80     # ¿en qué puerto del host está publicado web:80?
# 127.0.0.1:8080

docker compose events          # flujo de eventos del ciclo de vida

# Inspect completo del container de un servicio:
docker inspect $(docker compose ps -q web) | jq
```

### 16.5 Un sidecar de depuración

Engancha `nicolaka/netshoot` a cualquier red para tener un container con tcpdump, curl, dig, nmap, mtr y docenas más:

```yaml
  debug:
    image: nicolaka/netshoot
    profiles: ["debug"]
    networks:
      - internal
    command: ["sleep", "infinity"]
    cap_add: [NET_ADMIN, NET_RAW]
```

```bash
docker compose --profile debug up -d debug
docker compose exec debug bash
# Ahora tienes todas las herramientas de red en la red internal.
```

## 17. Compose frente a Kubernetes — cuándo graduarse

Compose es una herramienta de estación de trabajo y de un solo host. Es la herramienta correcta cuando:

- Ejecutas todo en una máquina (tu portátil, una VM de laboratorio, un jump box).
- Tu laboratorio tiene menos de unos veinte servicios.
- Quieres un archivo que puedas hacer commit y compartir, y que cualquier compañero pueda levantar con un comando.
- El uptime no es un requisito real: que el laboratorio caiga durante un `down && up` está bien.

Gradúate a Kubernetes (o Nomad, o modo Swarm, o ECS) cuando:

- Los servicios tienen que correr en varios hosts físicos o virtuales.
- Necesitas reprogramación automática cuando muere un host.
- Necesitas escalado horizontal de servicios individuales bajo carga real.
- Hay tráfico de producción y necesitas rolling updates, readiness gates y despliegue canary.

Para trabajo de laboratorio de pentest —incluso rangos grandes de simulación empresarial— Compose es casi siempre la respuesta correcta. Cuando al final necesites Kubernetes para una simulación de red team de un objetivo cloud-native, herramientas como kompose pueden traducir un archivo Compose a manifiestos de Kubernetes como punto de partida, pero el YAML resultante va a necesitar ajuste a mano.

El error más común es graduarse demasiado pronto. Si tu laboratorio cabe en un host, usa Compose.

## 18. Ejercicios

### 18.1 Ejercicio guiado 1 — aplicación vulnerable de tres capas

Escribe un archivo compose para un laboratorio de tres capas:

- `web`: `vulnerables/web-dvwa:latest`, publicado en `127.0.0.1:8080`.
- `db`: `mysql:5.7`, contraseña de root `root`, base de datos `dvwa`.
- `cache`: `redis:7-alpine`.

Requisitos:

- Los tres servicios en una red bridge definida por el usuario `app_net` con subred `10.50.0.0/24`.
- `web` depende de que `db` esté sano y de que `cache` haya arrancado.
- `db` usa un named volume para la persistencia.
- `db` tiene un healthcheck de `mysqladmin ping`.
- `cache` tiene un healthcheck de `redis-cli ping`.
- Nada más expuesto al host.

Esqueleto de la solución:

```yaml
services:
  web:
    image: vulnerables/web-dvwa:latest
    ports:
      - "127.0.0.1:8080:80"
    networks:
      app_net:
        ipv4_address: 10.50.0.10
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: dvwa
    volumes:
      - db_data:/var/lib/mysql
    networks:
      app_net:
        ipv4_address: 10.50.0.20
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -proot"]
      interval: 5s
      retries: 20
      start_period: 30s

  cache:
    image: redis:7-alpine
    networks:
      app_net:
        ipv4_address: 10.50.0.30
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep -q PONG"]
      interval: 5s
      retries: 10

networks:
  app_net:
    driver: bridge
    ipam:
      config:
        - subnet: 10.50.0.0/24

volumes:
  db_data:
```

### 18.2 Ejercicio guiado 2 — sidecar sniffer

Extiende el archivo compose anterior con un servicio sniffer que use `nicolaka/netshoot` y que:

- Se enganche a `app_net`.
- Ejecute `tcpdump -i any -w /pcap/lab.pcap`.
- Escriba el pcap en `./pcaps` del host mediante un bind mount.
- Solo arranque bajo el profile `monitor`.

```yaml
  sniffer:
    image: nicolaka/netshoot
    profiles: ["monitor"]
    networks:
      - app_net
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./pcaps:/pcap
    command: ["tcpdump", "-i", "any", "-w", "/pcap/lab.pcap"]
```

Ejecuta: `docker compose --profile monitor up -d`.

Verifica: después de algo de tráfico, `ls -lh ./pcaps/lab.pcap` en el host y ábrelo en Wireshark.

### 18.3 Ejercicio guiado 3 — orden de arranque gobernado por healthcheck

Diseña un stack con una cadena de arranque estricta:

- `db` (postgres) tiene que quedar sano.
- `migrator` (un container de un solo uso) tiene que ejecutarse hasta el final después de que `db` esté sano.
- `seed` (otro de un solo uso) tiene que ejecutarse hasta el final después de que `migrator` salga con 0.
- `app` no debe arrancar hasta que `seed` salga con 0.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: pw
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 20
      start_period: 20s

  migrator:
    image: alpine
    command: ["sh", "-c", "echo migrating && sleep 2 && echo done"]
    depends_on:
      db:
        condition: service_healthy
    restart: "no"

  seed:
    image: alpine
    command: ["sh", "-c", "echo seeding && sleep 2 && echo done"]
    depends_on:
      migrator:
        condition: service_completed_successfully
    restart: "no"

  app:
    image: nginx
    depends_on:
      seed:
        condition: service_completed_successfully
    ports:
      - "127.0.0.1:8090:80"
```

Verifica:

```bash
docker compose up
# db arranca y queda sano
# migrator corre y sale con 0
# seed corre y sale con 0
# app arranca
```

Las cadenas que imprime el comando (`migrating`, `done`, `seeding`) se quedan en inglés: son la salida literal del script.

### 18.4 Ejercicio independiente 1 — stack vulnerable de WordPress + MariaDB

Diseña un archivo compose para un WordPress viejo a propósito (`wordpress:4.7.5-apache`) más `mariadb:10.3`, publicado en `127.0.0.1:8888`, en una subred definida por ti. Añade healthchecks a los dos servicios. Añade un container atacante Kali en la misma red. Levántalo, explota al menos una vulnerabilidad de WordPress desde tu host y desmóntalo limpiamente.

### 18.5 Ejercicio independiente 2 — laboratorio de pivote con dos redes

Diseña un laboratorio con dos redes (`dmz` e `internal`). Coloca un container Tomcat vulnerable en las dos, un Redis vulnerable solo en `internal` y un atacante solo en `dmz`. Configura las redes para que el atacante tenga que explotar Tomcat para alcanzar Redis. Verifica el aislamiento intentando `nc -zv <redis-ip> 6379` desde el container atacante: debe fallar antes de la explotación y tener éxito después del pivote.

### 18.6 Ejercicio independiente 3 — overrides en capas

Parte un archivo compose base en `docker-compose.yml` (servicios y puertos por defecto) y `docker-compose.attacker.yml` (añade el container atacante y habilita la capability `NET_ADMIN`). Confirma que `docker compose -f docker-compose.yml up -d` arranca solo los objetivos y que `docker compose -f docker-compose.yml -f docker-compose.attacker.yml up -d` arranca todo. Usa `docker compose config` en las dos invocaciones para verificar el resultado fusionado.

## 19. Resumen y lección siguiente

Compose v2 le da a quien hace pentest las herramientas para construir, publicar y desmontar entornos de laboratorio reproducibles con un único archivo YAML y un único comando. En esta lección aprendiste:

- La distinción entre v1 y v2, y por qué `docker compose` (espacio) sustituye a `docker-compose` (guion).
- La estructura actual del archivo Compose, sin la clave `version:` obsoleta.
- Cada clave importante bajo `services:`: `image` frente a `build`, `ports`, `environment`, `env_file`, `depends_on` con condiciones, `healthcheck` y políticas de `restart`.
- Redes bridge definidas por el usuario con IPAM, redes externas para encadenar proyectos y las contrapartidas de `network_mode: host`.
- Named volumes, bind mounts, drivers tmpfs y NFS, y el patrón de backup de named volumes.
- Las reglas de sustitución de entorno para `${VAR}`, los valores por defecto y la exigencia de variables obligatorias.
- Profiles para servicios opcionales de atacante y de monitorización.
- Archivos de override, merge explícito con `-f` y la semántica del merge.
- Límites de recursos para contener containers vulnerables que se portan mal.
- Un laboratorio mínimo de DVWA más atacante y un laboratorio de varias capas con una red internal segmentada.
- Healthchecks que de verdad verifican la disponibilidad de Postgres, MySQL, HTTP y TCP a pelo.
- Tres estrategias de inyección de secretos, desde variables de entorno hasta Docker Secrets.
- Disciplina de limpieza: `down -v`, poda y auditoría de lo que sigue en ejecución.
- Depuración con `config`, `logs`, `exec` y un sidecar `nicolaka/netshoot`.
- Cuándo Compose deja de bastar y Kubernetes empieza a tener sentido.

En la lección siguiente vas a usar estas habilidades para construir desde cero una image de container vulnerable a propósito: escribir un Dockerfile que capture malas configuraciones del mundo real (correr como root, directorios escribibles por todo el mundo, secretos embebidos, endpoints de metadatos expuestos) y empaquetarla en un archivo compose para practicar.

Siguiente lección: Lección 03 — Dockerfile en profundidad y construcción de images vulnerables.
