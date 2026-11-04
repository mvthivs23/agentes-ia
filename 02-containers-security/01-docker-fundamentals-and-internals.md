# Lección 01 — Docker Fundamentals and Internals

Los objetivos modernos rara vez son un único servidor con un monolito. Son images de Docker unidas por Compose, orquestadas por Kubernetes, delante de un API gateway que verifica JWT, respaldadas por una base de datos particionada y publicadas por un pipeline de CI/CD que tiene más privilegios que cualquier desarrollador. Atacar una capa de esa pila sin entender las demás deja hallazgos en el suelo.

Este módulo recorre cada capa de la arquitectura de aplicaciones moderna desde la perspectiva de quien hace pentest. Vas a aprender cómo los namespaces y los cgroups de Linux definen el aislamiento del container, cómo un pod comprometido hace escape por un path del host montado o por una service account con privilegios de más, cómo un workflow de GitHub Actions envenenado se convierte en un punto de apoyo de supply chain, cómo un archivo de estado de Terraform filtra secretos y cómo los flujos de OAuth2 fallan de formas que los escáneres no ven. Al terminar deberías poder mirar un despliegue completo y enumerar su superficie de ataque capa por capa.

> Topología de containers y Kubernetes: las capas que este módulo ataca en secuencia, desde escapes de namespace hasta deriva de RBAC a escala de cluster.

    Module 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Los containers no son una moda pasajera: son el sustrato de despliegue de internet moderno. Cada producto SaaS que sondées, cada pipeline de CI/CD que audites y cada cluster de Kubernetes que evalúes está hecho de images de container sacadas de un registry y ejecutadas por un daemon en un host Linux. Como pentester, los containers te importan de tres maneras distintas.

Primero, son una superficie de ataque. Un daemon de Docker mal configurado expuesto en TCP 2375, un `/var/run/docker.sock` montado dentro de una aplicación web comprometida, un container en ejecución con `--privileged`, una image publicada en Docker Hub con archivos `.env` horneados en una capa: cada uno de estos es un camino directo desde el primer punto de apoyo hasta el control total del host. Entender cómo funcionan las primitivas es la diferencia entre ver la ruta de escape y quedarte mirando un shell.

Segundo, son una pieza de laboratorio. Casi todos los ejercicios prácticos de este curso —aplicaciones web vulnerables, rangos de Active Directory, sandboxes de malware— se montan más rápido como container que como máquina virtual. Un container arranca en milisegundos, usa megabytes de memoria en lugar de gigabytes y se reconstruye desde un Dockerfile en segundos cuando lo rompes.

Tercero, son una primitiva de ataque. Los red teams empaquetan payloads, infraestructura de C2 y herramientas de post-explotación como images. Una image de container mínima y endurecida con tu implante es más portable, más reproducible y —de forma perversa— más confiable para quienes defienden, que ven un `docker pull` familiar en los logs, que un binario crudo dejado en `/tmp`.

Al terminar esta lección vas a poder construir, inspeccionar y diseccionar una image de Docker a nivel de capa, explicarle a quien defiende exactamente qué función del kernel aísla un container de otro, e identificar a partir de la salida de `docker inspect` si un container en ejecución es una amenaza o una nota al pie.

## 1. Por qué los containers le importan a un pentester

### 1.1 El container como objetivo

Docker es el runtime de containers más común en hosts Linux de internet. Las búsquedas de Shodan de `port:2375` devuelven decenas de miles de daemons expuestos, y cada uno de ellos es un shell de root esperando a que lo reclamen. Incluso donde el socket no está expuesto, los containers introducen modos de fallo que el endurecimiento clásico de Linux no cubre:

- Un container de aplicación web que corre como root UID 0 y comparte su user namespace con el host
- Un container de build montado con el socket de Docker del host para habilitar «Docker-in-Docker»
- Una image sacada de un registry público que contiene un openssl viejo con un CVE conocido
- Un `docker run --privileged` usado para hacer visible un dispositivo, que de paso desactiva cada límite de seguridad que el runtime aporta

### 1.2 El container como laboratorio

A lo largo de este curso vas a levantar decenas de servicios vulnerables. Un solo `docker run` te da una copia fresca, limpia y desechable de DVWA, Juice Shop, Metasploitable3 o de un objetivo a medida que construyas tú. Cuando lo rompas —y lo vas a romper— `docker rm -f` y otro `docker run` lo restauran en menos de un segundo.

### 1.3 El container como arma

La infraestructura moderna de red team vive en containers. Sliver, Mythic, Havoc, team servers de Cobalt Strike, redirectores, landing pages de phishing: todos se publican con archivos de docker compose. Aprender a escribir tus propias images es la diferencia entre ejecutar el `docker-compose up` de otra persona y ser dueño de tu infraestructura desde la image base.

## 2. Container frente a máquina virtual — la comparación honesta

La frase de marketing es «los containers son VMs ligeras». Eso es falso en el sentido que más importa a la seguridad.

### 2.1 De dónde sale el aislamiento

```text
┌───────────────────────────────────────────────────────────┐
│                    Virtual Machine                        │
├───────────────────────────────────────────────────────────┤
│  App A  │  App B  │  App C   ← userland                   │
│  Guest Kernel A   │ Guest Kernel B   ← full OS per VM     │
│  Virtual Hardware │ Virtual Hardware  ← emulated devices  │
│           Hypervisor (KVM, ESXi, Hyper-V)                 │
│                    Host Kernel                            │
│                     Bare Metal                            │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│                        Container                          │
├───────────────────────────────────────────────────────────┤
│  App A  │  App B  │  App C     ← userland only            │
│    Docker / containerd / runc   ← runtime                 │
│              Shared Host Kernel                           │
│                    Bare Metal                             │
└───────────────────────────────────────────────────────────┘
```

Una VM ejecuta su propio kernel sobre hardware virtualizado. Un container corre directamente sobre el kernel del host y usa funciones de Linux —namespaces y cgroups— para que sus procesos crean que están solos en el sistema.

### 2.2 Comparación función por función

| Propiedad | Máquina virtual | Container |
| --- | --- | --- |
| Kernel | El suyo | El del host, compartido con cada container |
| Tiempo de arranque | 30 s a minutos | 50–500 ms |
| Huella en disco | GB | MB |
| Overhead de memoria | Cientos de MB por VM | ~10 MB por container |
| Primitiva de aislamiento | Hypervisor + virtualización por hardware | Namespaces + cgroups + capabilities |
| Un exploit de kernel cruza el límite | No (salvo CVEs de escape de VM) | Sí — un LPE de kernel = escape de container |
| Densidad en un host | Decenas | Miles |

### 2.3 La consecuencia en la superficie de ataque

Un LPE de kernel como CVE-2022-0847 («Dirty Pipe») en el host de una VM es grave: deja que un usuario haga root de ese guest. El mismo LPE dentro de un container da root en el container y, como el kernel es el del host, ese root queda directamente sobre el host, con solo las capabilities y los drops de seccomp que el runtime haya aplicado. Esta es la frase más importante del módulo: los containers comparten el kernel; un bug del kernel es un escape de container.

Cada técnica de endurecimiento del resto del curso —capabilities retiradas, perfiles de seccomp, user namespaces, filesystems raíz de solo lectura, UIDs no root— existe para estrechar esta superficie de ataque. Ninguna hace que el kernel deje de estar compartido.

## 3. Instalar Docker Engine en Kali 2026

Kali trae `docker.io` en sus repositorios, pero va por detrás del upstream. Para trabajo de pentest quieres el Docker Engine actual del repositorio APT oficial de Docker, para tener funciones como buildx, compose v2 y el containerd vigente.

### 3.1 Añadir el repositorio APT de Docker

```bash
# Actualiza e instala los prerequisitos:
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Añade la clave GPG de Docker:
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Kali está basado en Debian; fija el codename estable de Debian para el que Docker construye:
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian bookworm stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

# Instala el engine, la CLI, containerd, el plugin buildx y el plugin compose:
sudo apt install -y docker-ce docker-ce-cli containerd.io \
                    docker-buildx-plugin docker-compose-plugin
```

> **Nota de pentester:** Kali no tiene soporte oficial de Docker. Fijar bookworm (Debian 12) en lugar de kali-rolling es el truco que te da el engine actual. Si te saltas esto y usas `apt install docker.io`, espera ir uno o dos años por detrás en funciones y en correcciones de CVE.

### 3.2 Post-instalación: el grupo docker

```bash
# Añade tu usuario al grupo docker para ejecutar docker sin sudo:
sudo usermod -aG docker "$USER"

# Cierra sesión y vuelve a entrar, o activa el grupo en el shell actual:
newgrp docker

# Comprueba la pertenencia al grupo:
id
# uid=1000(kali) gid=1000(kali) groups=1000(kali),27(sudo),988(docker)
```

> **Nota de pentester:** Pertenecer al grupo `docker` equivale en la práctica a root. Quien está en ese grupo puede pedirle al daemon que arranque un container con el filesystem del host montado y obtener root en el host. Cuando audites un host Linux, trata `getent group docker` con la misma sensibilidad que `/etc/sudoers`.

### 3.3 Verificar la instalación

```bash
# La prueba de humo canónica:
docker run --rm hello-world
# Unable to find image 'hello-world:latest' locally
# latest: Pulling from library/hello-world
# 719385e32844: Pull complete
# Digest: sha256:c41088499908a59aae84b0a49c70e86f4731e588a737f1637e73c8c09d995654
# Status: Downloaded newer image for hello-world:latest
#
# Hello from Docker!
# This message shows that your installation appears to be working correctly.

# Confirma la versión del engine:
docker version
# Client: Docker Engine - Community
#  Version:           25.0.3
#  API version:       1.44
# Server: Docker Engine - Community
#  Engine:
#   Version:          25.0.3
#   API version:      1.44 (minimum version 1.24)
#  containerd:
#   Version:          1.7.13

# Muestra la información del sistema — fíjate en la sección de seguridad:
docker info | grep -A5 "Security Options"
# Security Options:
#  apparmor
#  seccomp
#   Profile: builtin
#  cgroupns
```

La línea de opciones de seguridad te dice qué aplica el daemon por defecto. Si seccomp no muestra `builtin` o falta apparmor, alguien ha debilitado los valores por defecto.

## 4. Linux namespaces — la primitiva de aislamiento

Un namespace es una función del kernel que le da a un proceso su propia vista de un recurso global del sistema. Cuando Docker arranca un container, no virtualiza hardware y no crea un kernel nuevo: crea un conjunto de namespaces y arranca el primer proceso del container (PID 1 dentro) con esos namespaces activos.

### 4.1 Los siete tipos de namespace

| Namespace | Qué aísla | Flag del comando |
| --- | --- | --- |
| PID | Números de ID de proceso | `unshare --pid` |
| MNT | Puntos de montaje | `unshare --mount` |
| NET | Pila de red (interfaces, rutas, sockets, iptables) | `unshare --net` |
| UTS | Hostname y nombre de dominio | `unshare --uts` |
| IPC | System V IPC, colas de mensajes POSIX | `unshare --ipc` |
| USER | UIDs y GIDs | `unshare --user` |
| CGROUP | Directorio raíz del cgroup | `unshare --cgroup` |

### 4.2 Inspeccionar tus propios namespaces

Cada proceso tiene un directorio `/proc/<pid>/ns/` que indica en qué namespace está para cada tipo. El valor es un número de inodo; dos procesos con el mismo inodo comparten el namespace.

```bash
# Namespaces de tu shell:
ls -la /proc/self/ns/
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 ipc    -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 mnt    -> 'mnt:[4026531841]'
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 net    -> 'net:[4026531840]'
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 pid    -> 'pid:[4026531836]'
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 user   -> 'user:[4026531837]'
# lrwxrwxrwx 1 kali kali 0 Sep  4 10:00 uts    -> 'uts:[4026531838]'
```

Arranca un container y compara:

```bash
docker run -d --name ns-demo alpine sleep 3600
docker exec ns-demo ls -la /proc/self/ns/
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 cgroup -> 'cgroup:[4026532567]'
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 ipc    -> 'ipc:[4026532565]'
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 mnt    -> 'mnt:[4026532563]'
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 net    -> 'net:[4026532568]'
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 pid    -> 'pid:[4026532566]'
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 user   -> 'user:[4026531837]'  ← shared!
# lrwxrwxrwx 1 root root 0 Sep  4 10:01 uts    -> 'uts:[4026532564]'
```

Fíjate en que cada inodo es distinto del de tu host excepto `user`. Docker no activa el user namespace por defecto: dentro del container, root (UID 0) es el mismo UID 0 que en el host. Por eso un escape de container te deja como root completo, no como un usuario no privilegiado mapeado.

### 4.3 Listar namespaces con lsns

```bash
sudo lsns -t pid
#         NS TYPE NPROCS   PID USER   COMMAND
# 4026531836 pid     412     1 root   /sbin/init splash
# 4026532566 pid       1  8934 root   sleep 3600
```

`lsns` es el mapa de «quién es dueño de qué namespace». Por cada proceso de container que corre en el host hay una entrada por tipo de namespace.

### 4.4 Construir un container a mano con unshare

Para convencerte de que Docker no es magia, construye un mini-container a mano:

```bash
sudo unshare --pid --mount --uts --ipc --net --fork --mount-proc \
  chroot /var/lib/docker/overlay2/$(docker inspect ns-demo \
    --format '{{.GraphDriver.Data.MergedDir}}' | \
    sed 's|.*/overlay2/||;s|/merged||') /bin/sh

# Dentro:
hostname
# (empty — new UTS namespace)
ps auxf
# PID 1  /bin/sh
# PID 2  ps auxf
ip link show
# 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
```

Acabas de construir la esencia de un container con tres comandos de coreutils: `chroot` para la vista del filesystem, `unshare` para los namespaces y `--mount-proc` para remontar `/proc` y que el aislamiento de PID funcione.

> **Nota de pentester:** Cuando caes dentro de un container y quieres saber si el daemon dejó el user namespace activo o no, `cat /proc/self/uid_map` te lo dice. `0 0 4294967295` significa que no hay mapeo: el root del container es el root del host. Cualquier otra cosa significa que los user namespaces están activos y un escape te da el UID mapeado, no root.

## 5. cgroups v2 — la primitiva de recursos

Los namespaces aíslan lo que un proceso puede ver. Los control groups (cgroups) limitan lo que un proceso puede usar: ciclos de CPU, memoria, I/O de bloque, PIDs, ancho de banda de red. Kali moderna corre cgroups v2, una jerarquía unificada bajo `/sys/fs/cgroup/`.

### 5.1 La jerarquía unificada

```bash
# Confirma que la versión activa es v2:
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)

# Mira qué controladores hay disponibles:
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory hugetlb pids rdma misc
```

### 5.2 Encontrar el cgroup de un container

Cada container recibe su propio cgroup bajo `/sys/fs/cgroup/system.slice/`:

```bash
docker run -d --name limits-demo --memory 128m --cpus 0.5 alpine sleep 3600

# Encuentra el path de su cgroup:
CID=$(docker inspect --format '{{.Id}}' limits-demo)
ls /sys/fs/cgroup/system.slice/docker-${CID}.scope/
# cgroup.controllers  cpu.max  memory.max  pids.max  ...

# Lee el límite de memoria aplicado:
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/memory.max
# 134217728  (128 MiB in bytes)

# Lee el límite de CPU aplicado — formato: "quota period":
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/cpu.max
# 50000 100000   (50 ms of CPU per 100 ms window = 0.5 core)
```

### 5.3 Ver cómo muerden los límites

```bash
# Dispara el OOM killer al superar el tope de memoria:
docker run --rm --memory 64m alpine sh -c \
  'dd if=/dev/zero of=/dev/shm/fill bs=1M count=200'
# dd: writing '/dev/shm/fill': No space left on device
# (or the container is OOM-killed and exits with code 137)

# Revisa dmesg en el host:
sudo dmesg | tail -5 | grep -i oom
# [12345.678] Memory cgroup out of memory: Killed process 9012 (dd)
```

> **Nota de pentester:** El flag `--memory` es un límite duro aplicado por el kernel. `--cpus` es un límite más blando: un container puede rebasarlo un momento. Si encuentras un container sin ningún límite de cgroup, una sola fork bomb dentro mata el host. Eso es lo primero que hay que mirar en una auditoría de configuración del runtime de containers.

### 5.4 El controlador pids — defensa contra fork bomb

```bash
docker run --rm --pids-limit 20 alpine sh -c ':(){ :|:& };:'
# sh: fork: retry: Resource temporarily unavailable
# (fork bomb blocked at 20 processes instead of exhausting the host)
```

## 6. Capabilities y seccomp — el filtro de syscalls

Los namespaces separan vistas; los cgroups ponen techo a los recursos; las capabilities y seccomp estrechan lo que el kernel le permite hacer a un container.

### 6.1 Linux capabilities

El poder de root en Linux está partido en unas 40 capabilities distintas (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_SYS_PTRACE`, etc.). Por defecto Docker conserva un subconjunto pequeño y descarta el resto.

```bash
# ¿Qué capabilities tiene un container por defecto?
docker run --rm alpine sh -c 'apk add -q libcap; capsh --print' | \
  grep "Current:"
# Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,
#   cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,
#   cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=ep
```

Catorce capabilities conservadas, más de veinte descartadas. Compara con `--privileged`:

```bash
docker run --rm --privileged alpine sh -c 'apk add -q libcap; capsh --print' | \
  grep "Current:"
# Current: =ep   ← every capability
```

Ese único flag es la diferencia entre un container endurecido y un shell de root en el host.

### 6.2 Añadir una sola capability

```bash
# Dale a un container la capacidad de cambiar la hora del sistema — nada más:
docker run --rm --cap-drop ALL --cap-add SYS_TIME alpine sh -c \
  'apk add -q libcap; capsh --print' | grep Current
# Current: cap_sys_time=ep
```

### 6.3 Seccomp — filtrar syscalls individuales

Docker trae un perfil de seccomp por defecto que bloquea unas 60 syscalls peligrosas (`kexec_load`, `keyctl`, `create_module`, `finit_module`, etc.).

```bash
# Prueba una syscall bloqueada:
docker run --rm alpine unshare --user echo hi
# unshare: unshare failed: Operation not permitted

# Desactiva seccomp:
docker run --rm --security-opt seccomp=unconfined \
  alpine unshare --user echo hi
# hi
```

> **Nota de pentester:** `--security-opt seccomp=unconfined`, `--privileged` y montar `/var/run/docker.sock` son los tres flags que hay que cazar en cualquier evaluación de containers. Cualquiera de ellos suele bastar para tomar el host por completo.

## 7. Filesystem overlay — capas, image frente a container

Una image es una pila de capas de solo lectura más un manifiesto que describe cómo apilarlas. Cuando arranca un container, el runtime añade encima una «capa de container» escribible. Todo esto lo pega OverlayFS.

### 7.1 Anatomía de una pila de capas

```text
┌────────────────────────────────────────┐
│  Container Writable Layer  (rw)        │ ← disappears on `docker rm`
├────────────────────────────────────────┤
│  Layer N   (created by `COPY app/ .`)  │ (ro)
├────────────────────────────────────────┤
│  Layer N-1 (created by `RUN apt ...`)  │ (ro)
├────────────────────────────────────────┤
│  Layer N-2 (created by `RUN apt-get`)  │ (ro)
├────────────────────────────────────────┤
│  Base image layers (e.g. debian)       │ (ro)
└────────────────────────────────────────┘
```

Cada capa es un directorio de archivos que representa el delta respecto de la capa de abajo.

### 7.2 Recorrer docker inspect

```bash
docker pull nginx:1.27-alpine
docker create --name nginx-inspect nginx:1.27-alpine
docker inspect nginx-inspect --format '{{json .GraphDriver}}' | jq
# {
#   "Data": {
#     "LowerDir": "/var/lib/docker/overlay2/abc.../diff:...",
#     "MergedDir": "/var/lib/docker/overlay2/xyz.../merged",
#     "UpperDir":  "/var/lib/docker/overlay2/xyz.../diff",
#     "WorkDir":   "/var/lib/docker/overlay2/xyz.../work"
#   },
#   "Name": "overlay2"
# }
```

- `LowerDir` — lista separada por dos puntos de las capas de solo lectura de la image
- `UpperDir` — la capa escribible del container
- `MergedDir` — la vista unificada que el container ve como `/`
- `WorkDir` — espacio de trabajo del kernel para operaciones atómicas

### 7.3 Ver un copy-up de una escritura

```bash
docker run -d --name copyup nginx:1.27-alpine
UPPER=$(docker inspect copyup --format '{{.GraphDriver.Data.UpperDir}}')

# Antes de cualquier escritura, el UpperDir del container está casi vacío:
sudo find "$UPPER" -type f | wc -l
# 3

# Escribe algo dentro del container:
docker exec copyup sh -c 'echo owned > /etc/hostname.new'

# El archivo aparece en UpperDir — esto es el "copy-up":
sudo find "$UPPER" -type f | wc -l
# 4
sudo cat "$UPPER/etc/hostname.new"
# owned
```

Cada modificación, por pequeña que sea, promociona el archivo afectado a la capa escribible. Por eso las images deberían construirse para no necesitar escrituras en runtime: la capa escribible cuesta disco y complica la forense.

### 7.4 La herramienta dive

`dive` muestra exactamente qué cambió en cada capa:

```bash
# Instálala:
wget -q https://github.com/wagoodman/dive/releases/download/v0.12.0/\
dive_0.12.0_linux_amd64.deb
sudo apt install -y ./dive_0.12.0_linux_amd64.deb

# Analiza una image:
dive nginx:1.27-alpine
#  ├── LAYER 1 — added Alpine base   (7.3 MB)
#  ├── LAYER 2 — added nginx binary  (2.1 MB, 84% efficient)
#  ├── LAYER 3 — copied conf files   (12 KB)
#  └── LAYER 4 — entrypoint script   (3 KB)
```

`dive` también marca espacio desperdiciado: los archivos añadidos en una capa y borrados en una posterior siguen ocupando espacio en la capa anterior.

> **Nota de pentester:** Las images públicas de Docker Hub suelen contener secretos residuales en capas anteriores. `dive` (o `docker history --no-trunc` combinado con `docker save` y la extracción de capas) te va a mostrar archivos `.env`, claves privadas, credenciales de nube y commits internos de git borrados en una capa de «limpieza» pero todavía recuperables en la capa de abajo.

## 8. Images de Docker y registries

Una image se identifica por un repositorio, un tag y —de forma canónica— un digest.

### 8.1 El formato de referencia de una image

```text
[registry/][namespace/]repository[:tag][@digest]

  docker.io/library/nginx:1.27-alpine
  ghcr.io/rtl/pentest-lab:latest
  registry.company.internal:5000/team/api@sha256:abc123...
```

Si omites el registry, el valor por defecto es `docker.io`. Si omites el namespace, se asume `library/`. Si omites el tag, se asume `:latest` — pero `latest` es solo un nombre convencional, no una garantía.

### 8.2 Tags frente a digests

Los tags son mutables: quien mantiene `nginx:1.27-alpine` puede publicar un build nuevo sobre ese tag mañana. Los digests son hashes inmutables y direccionables por contenido.

```bash
docker pull nginx:1.27-alpine
docker inspect nginx:1.27-alpine --format '{{.RepoDigests}}'
# [nginx@sha256:a5c57a...]

# Fija por digest para pentests reproducibles:
docker pull nginx@sha256:a5c57a97b3f2b81a01c7e5c3d2f7...
```

### 8.3 El manifiesto

Un manifiesto de image es un documento JSON que lista las capas, la config y la plataforma:

```bash
docker manifest inspect nginx:1.27-alpine
# {
#   "schemaVersion": 2,
#   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
#   "manifests": [
#     { "digest": "sha256:...", "platform": { "architecture": "amd64", ... } },
#     { "digest": "sha256:...", "platform": { "architecture": "arm64", ... } }
#   ]
# }
```

### 8.4 Trabajar con registries sin Docker: crane y skopeo

```bash
# Instálalos:
sudo apt install -y skopeo
GOBIN=/usr/local/bin sudo go install github.com/google/go-containerregistry/cmd/crane@latest

# Lista todos los tags de una image:
crane ls nginx | tail -5
# 1.27-alpine
# 1.27-alpine-slim
# 1.27-perl

# Saca una image sin un daemon de Docker:
skopeo copy docker://nginx:1.27-alpine dir:/tmp/nginx-image

# Inspecciona la config de una image sin bajar las capas:
crane config nginx:1.27-alpine | jq '.config.Env'
```

> **Nota de pentester:** `crane` y `skopeo` son esenciales cuando el objetivo no tiene daemon de Docker (o no quieres dejar un `docker pull` en el log de auditoría). Ambos hablan directamente con la API HTTP del registry. Ambos pueden bajar, inspeccionar y volver a publicar images de forma anónima si el registry lo permite.

## 9. Dockerfile en profundidad

Un Dockerfile es una receta de build. Cada instrucción se convierte en una capa.

### 9.1 Las instrucciones que importan

```dockerfile
# syntax=docker/dockerfile:1.7

# FROM — image base (primera instrucción, obligatoria)
FROM debian:12-slim

# ARG — variable de build (no está presente en el container en ejecución)
ARG APP_VERSION=1.0.0

# ENV — variable de entorno de runtime (persiste en la image)
ENV APP_HOME=/opt/app
ENV PATH=$APP_HOME/bin:$PATH

# RUN — ejecuta un comando durante el build; cada RUN crea una capa
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# COPY — copia archivos del contexto de build a la image
COPY --chown=1000:1000 app/ ${APP_HOME}/

# ADD — como COPY, pero también desempaqueta archivos y descarga URLs (una trampa; ver abajo)
# ADD https://example.com/data.tgz /opt/  ← evítalo

# WORKDIR — fija el directorio de trabajo de las instrucciones siguientes
WORKDIR ${APP_HOME}

# USER — baja privilegios a un UID no root
USER 1000:1000

# EXPOSE — documenta (solo documenta) en qué puertos escucha el container
EXPOSE 8080

# HEALTHCHECK — cómo decide el runtime si el container está sano
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1

# ENTRYPOINT — el ejecutable; no se puede sustituir con `docker run img <cmd>`
ENTRYPOINT ["/opt/app/bin/server"]

# CMD — argumentos por defecto; se pueden sustituir con `docker run img <cmd>`
CMD ["--config", "/opt/app/conf/prod.yaml"]
```

### 9.2 La trampa de ADD

`ADD` hace dos cosas peligrosas en silencio:

- Si el origen es un `.tar` o `.tar.gz`, lo extrae solo. Un tarball malicioso puede escribir en paths inesperados.
- Si el origen es una URL, lo descarga durante el build. La invalidación de caché de las URLs es poco fiable y no hay comprobación de firma.

Regla: usa `COPY` para archivos locales y un `RUN curl ... | sha256sum -c ...` para descargas remotas. Reserva `ADD` para el caso raro en el que de verdad quieres extraer un tar.

### 9.3 ENTRYPOINT frente a CMD

| Forma | ¿La sustituye `docker run img foo bar`? |
| --- | --- |
| `ENTRYPOINT ["a", "b"]` | No — `foo bar` se añaden como argumentos |
| `CMD ["a", "b"]` | Sí — los sustituye `foo bar` |
| Ambos | `CMD` es la lista de argumentos por defecto de `ENTRYPOINT` |

Prefiere la forma exec (`["cmd", "arg"]`) a la forma shell (`cmd arg`). La forma shell envuelve tu comando en `/bin/sh -c`, lo que rompe el manejo de señales (Ctrl-C no detiene el container) y añade un proceso de shell inútil como PID 1.

### 9.4 ARG frente a ENV

`ARG` solo es visible durante `docker build`. `ENV` queda horneada en la image y se ve en runtime con `docker exec ... env`.

Nunca uses `ARG` para secretos. `ARG SECRET=hunter2` es visible en `docker history --no-trunc`.

## 10. Builds multi-stage

El entorno de build y el de runtime tienen objetivos opuestos. El build necesita compiladores, headers y frameworks de test. El runtime necesita el mínimo absoluto para ejecutar el artefacto compilado.

### 10.1 Un build multi-stage de Go

```dockerfile
# syntax=docker/dockerfile:1.7

# ---- stage 1: build ----
FROM golang:1.22-bookworm AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server ./cmd/server

# ---- stage 2: runtime ----
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

Construye la image de dos stages:

```bash
docker build -t server:distroless .
docker images server
# REPOSITORY   TAG          SIZE
# server       distroless   8.7MB   ← runtime image
# <none>       <none>       1.2GB   ← build stage, discarded
```

### 10.2 Caché de build

Docker cachea cada capa con una clave formada por la instrucción y (para `COPY`/`ADD`) el checksum de los archivos copiados. Ordena las instrucciones de la que menos cambia a la que más cambia:

```dockerfile
FROM node:20-alpine
WORKDIR /app
# 1. Copia solo los manifiestos — se invalida solo cuando cambian las dependencias
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
# 2. Copia el código — se invalida en cada cambio de código
COPY src ./src
CMD ["node", "src/index.js"]
```

Al revés, cada edición de código vuelve a ejecutar `npm ci` y el rebuild pasa de segundos a minutos.

### 10.3 Distroless

Las images `gcr.io/distroless/*` traen un runtime de lenguaje y nada más: ni shell, ni apt, ni busybox, ni wget. Cuando tu container cae, quien ataca no tiene shell que ejecutar ni gestor de paquetes para instalar uno.

```bash
docker run --rm gcr.io/distroless/static-debian12 /bin/sh
# exec /bin/sh: no such file or directory
```

Alternativas con la misma filosofía: `alpine` (7 MB, tiene `sh` y busybox) y `chainguard/wolfi-base` (distro mínima basada en glibc, con procedencia firmada).

## 11. Redes de Docker

Docker crea un bridge de Linux, cuelga pares veth de él y reescribe iptables para que el tráfico fluya.

### 11.1 Los cuatro drivers integrados

| Driver | Comportamiento | Caso de uso |
| --- | --- | --- |
| bridge | Por defecto. El container recibe una IP privada en `docker0`, con NAT hacia el host | Redes de aplicaciones aisladas |
| host | El container comparte el network namespace del host | Aplicaciones de alto rendimiento, monitorización |
| none | El container solo tiene `lo` y no tiene conectividad externa | Cargas aisladas de la red |
| macvlan | El container obtiene su propia MAC en la red física | Aplicaciones legacy que necesitan L2 |

### 11.2 Listar e inspeccionar redes

```bash
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# 0f2a3b1c4d5e   bridge    bridge    local
# 6a7b8c9d0e1f   host      host      local
# 2f3e4d5c6b7a   none      null      local

docker network inspect bridge --format '{{json .IPAM.Config}}' | jq
# [{ "Subnet": "172.17.0.0/16", "Gateway": "172.17.0.1" }]

# El bridge en el host:
ip addr show docker0
# 3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500
#     inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```

### 11.3 Mapeo de puertos

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine

# Desde el host:
curl -s http://localhost:8080/ | head -1
# <!DOCTYPE html>

# ¿Qué le hizo Docker a iptables?
sudo iptables -L -t nat -n --line-numbers | grep 8080
# 1  DNAT  tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80

# Y en la cadena FORWARD:
sudo iptables -L FORWARD -n
# ACCEPT  all  --  0.0.0.0/0  172.17.0.0/16  ctstate RELATED,ESTABLISHED
# ACCEPT  all  --  172.17.0.0/16  0.0.0.0/0
```

Cada flag `-p` añade una regla DNAT. Cada red definida por el usuario añade una regla en la cadena forward.

### 11.4 Bridges definidos por el usuario

La red bridge por defecto no da DNS entre containers. Los bridges definidos por el usuario sí:

```bash
docker network create labnet
docker run -d --name db --network labnet postgres:16-alpine
docker run --rm --network labnet alpine sh -c 'nslookup db'
# Name:      db
# Address 1: 172.18.0.2
```

Crea siempre un bridge definido por el usuario para laboratorios de varios containers.

### 11.5 Modo host — sin aislamiento

```bash
docker run --rm --network host alpine ip addr show
# 1: lo: ...
# 2: eth0: 10.0.2.15/24  ← the host's interface, shared
```

`--network host` es rápido, pero quita el aislamiento del network namespace. Desde dentro, el container puede enlazar cualquier puerto del host y ver todas las interfaces del host.

## 12. Volumes y bind mounts

### 12.1 Las tres opciones de almacenamiento

| Tipo | Sintaxis | Respaldo |
| --- | --- | --- |
| Named volume | `-v mydata:/var/lib/mysql` | `/var/lib/docker/volumes/mydata/_data` |
| Bind mount | `-v /host/path:/container/path` | Cualquier path del host |
| tmpfs | `--tmpfs /tmp` | RAM |

### 12.2 Named volumes

```bash
docker volume create pgdata
docker run -d --name pg -v pgdata:/var/lib/postgresql/data postgres:16-alpine
docker volume inspect pgdata --format '{{.Mountpoint}}'
# /var/lib/docker/volumes/pgdata/_data
```

Los named volumes sobreviven a `docker rm`; úsalos para servicios con estado.

### 12.3 Bind mounts

```bash
# Monta un directorio del host en solo lectura:
docker run --rm -v /etc/os-release:/tmp/host-os:ro alpine cat /tmp/host-os
# PRETTY_NAME="Kali GNU/Linux Rolling"
```

### 12.4 tmpfs

```bash
docker run --rm --tmpfs /scratch:size=64m alpine \
  sh -c 'mount | grep scratch'
# tmpfs on /scratch type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
```

tmpfs desaparece cuando el container se detiene: sirve para datos temporales o para secretos que no quieres en disco.

### 12.5 El escape `-v /:/host`

Si puedes alcanzar un daemon de Docker —por el socket, por TCP o desde un container hermano— puedes hacer que arranque un container con la raíz del host montada y obtener root sobre ese filesystem. Ese es el escape completo. Por eso el grupo `docker` equivale a root.

El comando concreto que monta `/` del host y lee secretos del host no se reproduce aquí. La detección y el endurecimiento están en las secciones 13 y 15, y en la lección 03.

## 13. El daemon de Docker y el socket

El daemon de Docker, `dockerd`, corre como root y escucha por defecto en `/var/run/docker.sock` (un socket Unix). Cada comando de la CLI de docker es una petición HTTP a este socket.

### 13.1 Ver la API en acción

```bash
# Habla con el daemon directamente:
curl --unix-socket /var/run/docker.sock http://localhost/version | jq .Version
# "25.0.3"

curl --unix-socket /var/run/docker.sock http://localhost/containers/json | \
  jq '.[].Names'
# ["/nginx-inspect"]
# ["/limits-demo"]
```

### 13.2 Por qué el socket es root

El daemon es root, arranca containers arbitrarios y controla todos los namespaces. Quien puede escribir en el socket puede:

- Pedirle al daemon que arranque containers que monten el filesystem del host y obtener root en el host
- Hacer attach a cualquier container en ejecución y leer su memoria
- Leer cualquier image en disco, incluidas las que tienen credenciales embebidas

### 13.3 El escape del «socket montado»

Los dashboards web, los runners de CI y las herramientas de monitorización a menudo montan el socket dentro de un container:

```bash
docker run -d -v /var/run/docker.sock:/var/run/docker.sock some/dashboard
```

Si ese dashboard se compromete alguna vez, quien ataca tiene el socket y, por tanto, el host. Durante las evaluaciones, `ls -la /var/run/docker.sock` dentro de cualquier container es la comprobación más valiosa.

### 13.4 Docker rootless

Docker Engine puede correr como un usuario no root con `dockerd-rootless-setuptool.sh install`. Esto mitiga el problema de que el socket equivalga a root, a costa de perder algunas funciones (sin puertos &lt;1024, algunas limitaciones de redes overlay). Rootless no es el valor por defecto en Kali; activarlo es una tarea de endurecimiento, no la norma.

## 14. Alternativas de runtime

Docker no es el único runtime de containers. Entender el ecosistema importa cuando el objetivo no es «Docker».

### 14.1 containerd

El daemon que Docker usa por debajo. Se distribuye como runtime independiente con la CLI `ctr`. Los nodos worker de Kubernetes desde 2022 suelen usar containerd directamente, sin Docker.

```bash
sudo ctr images list
sudo ctr containers list
```

### 14.2 CRI-O

Un runtime compatible con OCI diseñado específicamente para Kubernetes, preferido por OpenShift. No tiene CLI de Docker.

### 14.3 Podman — rootless por defecto

```bash
sudo apt install -y podman
podman run --rm hello-world
```

Podman no tiene daemon: cada invocación de `podman` hace fork del container directamente. Por defecto corre rootless mediante user namespaces. La CLI es un reemplazo directo (`alias docker=podman` funciona para la mayoría de los flujos).

Para trabajo de laboratorio de red team, Podman es el valor por defecto más seguro: un escape desde un container rootless de Podman te deja como tu propio usuario no privilegiado, no como root.

## 15. Laboratorio práctico — una image de Nginx endurecida

Ahora vas a construir una image de Nginx endurecida desde el código fuente, con multi-stage, sobre una base distroless, vas a quitar cada capability que no haga falta y vas a demostrar el endurecimiento desde dentro del container.

### 15.1 El Dockerfile

Crea `hardened-nginx.Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7

# ---- stage 1: construir nginx desde el código fuente ----
FROM debian:12-slim AS build

ARG NGINX_VERSION=1.27.0
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      build-essential curl ca-certificates \
      libpcre3-dev libssl-dev zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN curl -fsSL https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz \
      -o nginx.tgz && \
    tar xzf nginx.tgz && \
    cd nginx-${NGINX_VERSION} && \
    ./configure --prefix=/opt/nginx \
                --sbin-path=/opt/nginx/sbin/nginx \
                --conf-path=/opt/nginx/conf/nginx.conf \
                --error-log-path=/dev/stderr \
                --http-log-path=/dev/stdout \
                --pid-path=/tmp/nginx.pid \
                --http-client-body-temp-path=/tmp/client_body \
                --http-proxy-temp-path=/tmp/proxy \
                --http-fastcgi-temp-path=/tmp/fastcgi \
                --http-uwsgi-temp-path=/tmp/uwsgi \
                --http-scgi-temp-path=/tmp/scgi \
                --with-http_ssl_module \
                --with-http_v2_module \
                --without-http_rewrite_module && \
    make -j"$(nproc)" && \
    make install

# Config mínima
RUN cat > /opt/nginx/conf/nginx.conf <<'EOF'
worker_processes 1;
events { worker_connections 512; }
http {
  server {
    listen 8080;
    location / { return 200 "hardened\n"; }
  }
}
EOF

# ---- stage 2: runtime distroless ----
FROM gcr.io/distroless/base-debian12:nonroot

COPY --from=build /opt/nginx /opt/nginx

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/opt/nginx/sbin/nginx", "-g", "daemon off;"]
```

### 15.2 Build y ejecución

```bash
docker build -f hardened-nginx.Dockerfile -t hardened-nginx:lab .

docker images hardened-nginx
# REPOSITORY       TAG   SIZE
# hardened-nginx   lab   26.4MB

docker run -d --name hnginx \
  --read-only \
  --tmpfs /tmp:size=8m,mode=1777 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -p 8080:8080 \
  hardened-nginx:lab

curl -s http://localhost:8080/
# hardened
```

### 15.3 Verificar el endurecimiento

```bash
# ¿Capabilities retiradas?
docker inspect hnginx --format '{{.HostConfig.CapDrop}}'
# [ALL]

# ¿Raíz de solo lectura?
docker inspect hnginx --format '{{.HostConfig.ReadonlyRootfs}}'
# true

# ¿no-new-privileges?
docker inspect hnginx --format '{{.HostConfig.SecurityOpt}}'
# [no-new-privileges]

# ¿Corre como no root?
docker exec hnginx cat /proc/1/status | grep -E '^(Uid|CapEff)'
# Uid:  65532 65532 65532 65532
# CapEff:  0000000000000000    ← literally zero effective capabilities
```

Distroless no tiene shell, así que `docker exec hnginx sh` no va a funcionar. Para inspeccionar desde dentro, ejecuta un sidecar que comparta los namespaces:

```bash
docker run --rm -it \
  --pid container:hnginx \
  --network container:hnginx \
  --cap-add SYS_PTRACE \
  alpine sh -c 'apk add -q libcap; capsh --print'
# Current: =
# Bounding set =
# ...
```

Has construido y verificado un container que corre como UID 65532, tiene cero capabilities, un filesystem raíz de solo lectura, sin shell, sin gestor de paquetes y 26 MB en disco. Esta es la línea de base de cómo debería verse un container de producción.

## 16. Ejercicios

### Guiados

1. **Mapeo de namespaces.** Arranca cualquier container y, desde el host, usa `lsns` para listar los siete namespaces en los que está el PID 1 del container. Después encuentra el inodo correspondiente en `/proc/<host-pid>/ns/` y confirma que coinciden.

2. **Aplicación de cgroups.** Ejecuta `docker run --memory 32m --pids-limit 10 alpine sh -c ':(){ :|:& };:'` y observa el resultado. Después quita `--pids-limit` y vuelve a ejecutarlo en una VM desechable — no lo hagas en tu estación de trabajo. Escribe un párrafo que explique qué cambió.

3. **Arqueología de capas.** Elige cualquier image etiquetada `latest` en Docker Hub. Usa `docker history --no-trunc` y `dive` para identificar una capa que añade archivos que una capa posterior elimina. Informa de los bytes desperdiciados.

### Independientes

4. **Reconocimiento de registry.** Usando solo `crane ls` y `crane manifest` (sin `docker pull`), enumera todos los tags de `library/postgres` en Docker Hub e identifica el digest del build `16-alpine` más reciente. Documenta tus comandos.

5. **Rust multi-stage.** Escribe un Dockerfile que construya un binario de Rust «Hello, world» en un stage y lo copie a un stage de runtime `distroless/cc-debian12:nonroot`. La image final debe pesar menos de 20 MB y correr como no root.

6. **Auditoría del socket.** Escribe un script de shell que, dado el nombre de un container, imprima un aviso si se cumple cualquiera de estas condiciones: el container monta `/var/run/docker.sock`; el container se arrancó con `--privileged`; las capabilities efectivas del container incluyen `CAP_SYS_ADMIN`; el usuario del container es UID 0. Pruébalo contra el container `hnginx` de la sección 15 y contra un control mal configurado a propósito.

## 17. Resumen y siguientes pasos

Ahora sabes que:

- Los containers son procesos del kernel del host aislados por namespaces y limitados por cgroups, capabilities y seccomp: no son VMs ligeras.
- Los siete namespaces de Linux (PID, MNT, NET, UTS, IPC, USER, CGROUP) aíslan cada uno un recurso global concreto; Docker activa seis por defecto y se salta USER.
- cgroups v2 pone techo a CPU, memoria, PIDs e I/O por container; si faltan límites, un solo container puede dejar al host sin recursos.
- Las capabilities y seccomp reducen lo que un proceso de container comprometido puede pedirle al kernel; `--privileged` y `seccomp=unconfined` tiran ambos por la borda.
- Las capas de OverlayFS presentan una image como diffs de solo lectura con una capa escribible por container encima; `dive` y `docker inspect` revelan qué hay de verdad en cada capa.
- La higiene del Dockerfile —digests fijados, nada de URLs en `ADD`, `COPY --chown`, `USER` no root, `ENTRYPOINT` en forma exec, `ARG` nunca para secretos— es la diferencia entre una image cuidada y una mina.
- Los builds multi-stage junto con bases distroless producen images de runtime pequeñas, sin shell y no root.
- La red de Docker es un bridge de Linux más NAT de iptables; `docker network` e `iptables -t nat -L` son los dos comandos que más vas a ejecutar.
- El socket de docker equivale en la práctica a root; cada escape de container del mundo real empieza por encontrarlo.

Siguiente lección: Lección 02 — Docker Compose, volumes y la construcción de tu primer laboratorio vulnerable. Vas a usar Compose para unir varios containers en un único entorno objetivo reproducible —una aplicación web, una base de datos y un reverse proxy— y a dar tus primeros pasos contra él.
