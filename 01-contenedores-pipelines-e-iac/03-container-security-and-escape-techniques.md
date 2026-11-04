# Lección 03 — Container Security and Escape Techniques

    Module 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Los containers son la unidad de empaquetado por defecto de la nube moderna. Cada microservicio, cada job de CI/CD y cada runtime serverless es, al final, un proceso de Linux envuelto en un conjunto de namespaces y cgroups. Ese empaquetado es cómodo y, cuando está mal configurado, es una línea recta desde un usuario de aplicación con pocos privilegios hasta root en el host del container. Quien hace pentest y entiende el modelo de aislamiento puede convertir una primitiva de ejecución remota de código dentro de una aplicación web en control del nodo subyacente, del daemon de Docker y —en Kubernetes— del cluster entero.

Esta lección recorre el marco ético, el modelo de aislamiento, la lista de comprobación de enumeración, ocho familias de escape por nombre e impacto, la ingeniería de detección que usa el equipo azul para pillar cada familia y las medidas de endurecimiento que impiden que el escape funcione. Cada ejemplo está acotado a un laboratorio que montas tú, o a un objetivo que está explícitamente en alcance de un engagement autorizado.

Los procedimientos reproducibles —comandos, payloads y pruebas de concepto que obtienen un shell de root en el host— no se incluyen. Sí se conservan el nombre de cada familia, el impacto, la detección y la mitigación.

## 1. Introducción y alcance ético

Todas las técnicas de esta lección se documentan en clave ofensiva, pero cada ejemplo está acotado a un entorno de laboratorio que levantas tú con los archivos Compose del walkthrough, o a un objetivo que está explícitamente en alcance de un engagement de red team autorizado. Las razones son legales, prácticas y profesionales:

- **Legal.** Escapar de un container que no es tuyo es acceso no autorizado bajo la CFAA (EE. UU.), la Computer Misuse Act (Reino Unido), la Lei 12.737/2012 (Brasil) y estatutos equivalentes en otros sitios. Las primitivas de escape de container suelen cruzar un límite de tenencia, y ese límite es exactamente lo que protege la ley.
- **Práctica.** Un escape de container en producción puede tumbar a otros inquilinos que comparten el mismo nodo, corromper volumes compartidos o disparar una autorremediación que llama a quien está de guardia a las 03:00. Incluso dentro de un engagement autorizado, los pasos de escape entran por escrito en las reglas del engagement antes de ejecutarlos.
- **Profesional.** El valor de un escape de container en un informe no es el shell: es la cadena mapeada desde el punto de apoyo inicial hasta el radio de explosión, con detecciones y mitigaciones en cada salto. La mitad del entregable es el camino de la corrección.

> **Nota de pentester:** La inmensa mayoría de los escapes de container en engagements reales vienen de una mala configuración, no de un 0-day del kernel. `--privileged`, un socket de Docker montado, un volume hostPath sobre `/`, un container que corre como UID 0 con `CAP_SYS_ADMIN`: esos son los hallazgos que se cobran. Trata los escapes por exploit de kernel como el plan de reserva y la caza de malas configuraciones como el plato principal.

### 1.1 Declaración de alcance usada en esta lección

Salvo que se diga otra cosa, cada comando de esta lección asume:

- Una estación de trabajo atacante Kali o Ubuntu 22.04.
- Una VM de laboratorio aparte (Ubuntu 22.04, kernel 5.15+, Docker 24.x, containerd 1.7.x) que sea tuya y que puedas reconstruir.
- Un cluster Kubernetes de un solo nodo, kind o k3s, en la misma VM de laboratorio, para los ejemplos de escape de pod.
- Snapshots tomados antes de cada paso destructivo, para poder volver atrás.

## 2. Repaso del modelo de aislamiento del container

Un container no es una VM ligera. Es uno o más procesos de Linux con un disfraz hecho de seis primitivas del kernel:

| Primitiva | Qué aísla | Función del kernel |
| --- | --- | --- |
| PID namespace | IDs de proceso | `CLONE_NEWPID` |
| Mount namespace | Montajes del filesystem | `CLONE_NEWNS` |
| Network namespace | Interfaces, rutas, sockets | `CLONE_NEWNET` |
| UTS namespace | Hostname, dominio | `CLONE_NEWUTS` |
| IPC namespace | System V IPC, colas de mensajes POSIX | `CLONE_NEWIPC` |
| User namespace | Mapeo de UID/GID | `CLONE_NEWUSER` |
| cgroups v2 | Cuotas de CPU, memoria e I/O | `unified_hierarchy` |
| capabilities | Poderes de root de grano fino | máscara `CAP_*` |
| seccomp | Filtro de syscalls | `SECCOMP_MODE_FILTER` |
| LSM (AppArmor/SELinux) | Control de acceso obligatorio | hooks LSM |

El hecho crucial para el trabajo ofensivo es que todas estas primitivas comparten un solo kernel. El uid 0 del container 1, el uid 0 del container 2 y el uid 0 del host son el mismo entero dentro del mismo kernel; solo el mapeo de namespace y el conjunto de capabilities deciden quién puede hacer qué con él. Si un container tiene `CAP_SYS_ADMIN` y puede tocar un recurso fuera de su mount namespace, root-en-el-container se convierte en root-en-el-host en un salto.

### 2.1 Consecuencias del radio de explosión

```text
┌─────────────────────────────────────────────────────────┐
│ Host Kernel  (one for everyone)                          │
├───────────────┬───────────────┬────────────────┬────────┤
│ Container A   │ Container B   │ Container C    │  Host  │
│ uid 0 (mapped)│ uid 1000      │ uid 0 privd    │ uid 0  │
│ CAP_NET_BIND  │ (no caps)     │ ALL CAPS       │ ALL    │
│ seccomp on    │ seccomp on    │ seccomp OFF    │ n/a    │
│ apparmor on   │ apparmor on   │ apparmor UNCON │ n/a    │
└───────────────┴───────────────┴────────────────┴────────┘
             │              │              │
             └──────────────┴──────────────┘
                one syscall interface
```

Si comprometes el container C del diagrama, en la práctica ya tienes el host, porque la única línea entre tú y el host es un mapeo de namespace que cualquiera de tus capabilities puede deshacer. Comprometer el container A o el B sigue exigiendo un escape de verdad (bug del kernel, socket montado, mala configuración).

### 2.2 Panorama de runtimes

- **runc** — el runtime de referencia OCI, usado por Docker y containerd.
- **crun** — una reescritura en C de runc, más rápida; es el valor por defecto en Podman y RHEL 9.
- **gVisor** — un kernel en espacio de usuario de Google que reimplementa las syscalls de Linux para que los containers no hablen directamente con el kernel del host.
- **Kata Containers** — cada container va envuelto en una VM ligera (Firecracker/QEMU): aislamiento por hardware en lugar de aislamiento por namespace.
- **Firecracker** — la microVM que usan AWS Lambda y Fargate para aislar inquilinos.

Los escapes de esta lección aplican a runc/crun. gVisor y Kata se tratan como mitigaciones en la sección 12: cambian el juego porque el objetivo del escape deja de ser el kernel del host.

## 3. Enumeración desde dentro de un container — la primera lista del red team

En el momento en que tienes un shell dentro de un container (por RCE en la aplicación, un Jupyter expuesto, un Confluence mal parcheado, sea cual sea el vector inicial), ejecutas una secuencia fija de comprobaciones antes de tocar nada más. El punto de la lista es responder, en menos de sesenta segundos, cuál es el camino más corto desde este shell hasta el host. Las comprobaciones de abajo son de lectura y de diagnóstico. No incluyen las acciones que realizan el escape.

### 3.1 ¿Estoy de verdad en un container?

```bash
# Señal 1 — el path del cgroup contiene docker/containerd/kubepods:
cat /proc/1/cgroup
# 0::/system.slice/docker-a4c8f7....scope       ← Docker
# 0::/kubepods.slice/kubepods-burstable...       ← Kubernetes
# 0::/                                           ← host or systemd-nspawn

# Señal 2 — existe /.dockerenv (específico de Docker):
ls -la /.dockerenv 2>/dev/null && echo "docker container"

# Señal 3 — init no es systemd:
cat /proc/1/comm
# nginx, node, python — you are containerized
# systemd — probably a full host or a systemd-nspawn container

# Señal 4 — variables de entorno específicas de container filtradas:
env | grep -iE 'kubernetes|docker|container'
# KUBERNETES_SERVICE_HOST=10.96.0.1    ← inside a Kubernetes pod
```

La cadena `docker container` la imprime el propio comando: se queda en inglés.

### 3.2 ¿Qué capabilities tengo?

```bash
# Método 1 — capsh (de libcap2-bin), el más legible:
capsh --print
# Current: cap_chown,cap_dac_override,cap_fowner,cap_setgid,cap_setuid,
#          cap_net_bind_service,cap_sys_chroot,cap_kill,cap_audit_write=ep
# Bounding set: same as Current
# Ambient set:
# Securebits: 00/0x0/1'b0

# Método 2 — máscara cruda de /proc:
grep -E '^Cap(Inh|Prm|Eff|Bnd|Amb)' /proc/self/status
# CapInh:    0000000000000000
# CapPrm:    00000000a80425fb   ← the meaningful one
# CapEff:    00000000a80425fb
# CapBnd:    00000000a80425fb
# CapAmb:    0000000000000000

# Método 3 — decodifica la máscara a mano o con capsh:
capsh --decode=00000000a80425fb
# 0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,...
```

Las máscaras que hay que memorizar:

| CapEff en hex | Significado |
| --- | --- |
| `0000003fffffffff` | Todas las capabilities: estás en `--privileged` o equivalente |
| `00000000a80425fb` | Conjunto por defecto de Docker (14 capabilities) |
| `0000000000000000` | Todas retiradas (poco frecuente, y buena señal de endurecimiento) |
| Cualquiera que contenga `cap_sys_admin` | Probablemente se acabó el juego — ver Escape 3 |
| Cualquiera que contenga `cap_sys_ptrace` + PID del host | Ver Escape 6 |
| `cap_sys_module` | Puedes insertar un módulo del kernel. Se acabó el juego. |
| `cap_dac_read_search` | Salta el DAC de lectura de archivos: caza de secretos |

### 3.3 ¿Qué está montado?

```bash
# Lista completa de montajes:
mount
# overlay on / type overlay (rw,relatime,lowerdir=...)
# /dev/sda1 on /etc/hosts type ext4 (rw,relatime)          ← HOST DEVICE INSIDE CONTAINER
# tmpfs on /run/docker.sock type tmpfs                     ← socket path
# /var/run/docker.sock                                     ← DOCKER SOCKET MOUNTED

# La misma vista desde /proc — funciona aunque falte el binario `mount`:
cat /proc/mounts

# Busca montajes de bandera roja:
mount | grep -E 'docker.sock|/proc$|/sys$|/dev$|/host|/root'
```

Montajes de bandera roja que hay que cazar:

- `docker.sock` montado en cualquier sitio → Escape 1
- `/` o cualquier path raíz del host montado → Escape 5
- `/var/log`, `/etc/shadow`, `/root/.ssh`, `/proc`, `/sys` del host → robo de datos o Escape 5
- `/dev` del host con dispositivos de bloque → primitiva del Escape 2 incluso sin `--privileged`
- Volumes hostPath de Kubernetes → lo mismo, con otro nombre

### 3.4 ¿Está montado el socket de Docker?

```bash
# La comprobación más productiva del pentest de containers:
ls -la /var/run/docker.sock /run/docker.sock 2>/dev/null
# srw-rw---- 1 root docker 0 Nov 12 10:11 /var/run/docker.sock

# Confirma que habla con un daemon vivo:
curl --unix-socket /var/run/docker.sock http://localhost/version
# {"Platform":{"Name":"Docker Engine - Community"}, ...}
```

Si la respuesta es sí, el hallazgo ya es de impacto máximo: el socket es una API equivalente a root. El procedimiento que lo convierte en un shell del host no se incluye; la detección y la mitigación están en el Escape 1.

### 3.5 ¿Estoy en privileged?

Casi nunca ves `--privileged` en `/proc/self/status`, pero puedes probarlo intentando operaciones que solo un container privileged puede hacer. La prueba que escribe en `/proc/sysrq-trigger` no se incluye: un sysrq con éxito puede afectar al host.

Señales de solo lectura y de diagnóstico:

- Crear y borrar una interfaz dummy exige `CAP_NET_ADMIN`. Si el runtime está endurecido, la respuesta esperada es `RTNETLINK answers: Operation not permitted`.
- Ver dispositivos de bloque del host (`/dev/sda*`, `/dev/nvme0n1*`, `/dev/mem`) indica acceso a dispositivos del host.
- Poder hacer `mount` dentro del container indica `CAP_SYS_ADMIN`.

Esas tres comprobaciones se repiten en la sección 15.7 como validación de que el endurecimiento las bloquea.

### 3.6 Herramientas automáticas

Tres herramientas hacen toda la lista y producen un informe que se puede triar: linpeas (con comprobaciones de container), deepce (enumeración de Docker y de escapes de container) y amicontained (Jess Frazelle), que sonda capabilities y syscalls.

Una salida de amicontained que muestra `sys_admin` en el bounding set, `Seccomp: disabled` y `user: false` es la firma de un container maduro para un escape:

```text
# Container Runtime: docker
# Has Namespaces:
#   pid: true
#   user: false
# AppArmor Profile: docker-default (enforce)
# Capabilities:
#   BOUNDING -> chown dac_override fowner ... sys_admin ...
# Seccomp: disabled
```

No se incluye el flag de deepce que intenta escapes.

## 4. Escape 1 — socket de Docker montado

**Precondición:** `/var/run/docker.sock` (o cualquier otro path enlazado al socket de Docker del host) está montado dentro del container y el usuario del shell puede leerlo y escribirlo. Es habitual en sistemas de CI, Portainer y agentes de monitorización que «solo necesitan ver los containers».

**Impacto:** el socket de Docker es una API equivalente a root y sin autenticación. Si puedes hablar con él, puedes pedirle al daemon que lance un container nuevo que monte el filesystem del host. A partir de ahí, el control del host es el impacto.

El procedimiento —instalar un cliente, lanzar el container que monta `/` y entrar en el mount namespace del host— no se incluye.

### 4.1 Montaje de laboratorio (autorizado)

La configuración vulnerable, solo en una VM de laboratorio que sea tuya, es un container que monta el socket del host. Esa misma configuración es el «antes» que la sección 15 endurece. No se incluye el paso que convierte el socket en un shell del host.

### 4.3 Detección

- **Falco:** las reglas `Launch Privileged Container` y `Container Started with Sensitive Mount` (integradas) se disparan cuando el container del escape se crea con un montaje de `/` del host.
- **auditd:** `-w /var/run/docker.sock -p rwa -k docker_socket_access` pilla cualquier proceso que abra el socket y no sea el propio `dockerd`.
- **EDR:** CrowdStrike, SentinelOne y Defender for Endpoint traen reglas que alertan de un chroot dentro de un proceso cuyo padre es `dockerd` y cuyo origen de montaje es `/`.

### 4.4 Mitigación

- Nunca montes el socket de Docker en un container de aplicación. Si una carga de verdad necesita lanzar containers, usa el daemon rootless, `podman.socket` activado por socket o Kubernetes con RBAC adecuado.
- Si tienes que exponer el socket, ponlo detrás de un socket-proxy (Tecnativa docker-socket-proxy) que filtre los endpoints de API permitidos (típicamente solo `/containers/json`).

## 5. Escape 2 — container `--privileged`

**Precondición:** el container se arrancó con `--privileged`, lo que le da todas las capabilities, sin seccomp, sin AppArmor y con acceso a todos los dispositivos del host vía `/dev`.

**Impacto:** con acceso directo al dispositivo de bloque de la raíz del host, se puede montar el filesystem del host dentro del container y leerlo o escribirlo. Eso incluye secretos del host y persistencia en el host. El procedimiento que monta el disco y modifica cuentas o claves del host no se incluye.

### 5.1 Montaje de laboratorio

La configuración vulnerable es un container Alpine arrancado con `--privileged`. El endurecimiento que la anula está en las secciones 12.3 y 15.6.

### 5.3 Detección

- **Falco:** las reglas `Mount Launched in Privileged Container` y `Write below root` se disparan de inmediato.
- **auditd:** `-a always,exit -F arch=b64 -S mount -k container_mount` pilla la syscall mount de cualquier proceso dentro de un cgroup de Docker.
- **EDR:** se marca como comportamiento de «manipulación del filesystem del host desde un container».

### 5.4 Mitigación

- Prohíbe `--privileged` en el admission controller de la organización (Kyverno/Gatekeeper, o un filtro simple de proxy del socket de Docker).
- Donde `--privileged` sea inevitable (algunos runners de CI), fíjalo a un node pool dedicado, sin secretos compartidos y sin datos persistentes.

## 6. Escape 3 — `CAP_SYS_ADMIN` mediante release_agent y notify_on_release en cgroup

**Precondición:** el container tiene `CAP_SYS_ADMIN` (presente en `--privileged` y a veces concedida por separado a sidecars de CI o de aprovisionamiento de almacenamiento) y el host tiene cgroup v1 activo. El truco está en gran parte desactivado en sistemas de cgroup v2 puros; Ubuntu 22.04 es v2 por defecto, pero muchas distros de Kubernetes siguen trayendo v1.

**Impacto:** en cgroup v1, `CAP_SYS_ADMIN` permite usar el mecanismo `release_agent` para que el kernel ejecute un programa como root en el host, fuera de los namespaces del container.

El diagrama de pasos y la prueba de concepto de laboratorio no se incluyen.

### 6.3 Detección

- **Falco:** la regla `Detect release_agent File Container Escapes` (reglas de la comunidad) busca un `open_write` sobre cualquier archivo cuyo nombre termina en `release_agent` desde un cgroup de container.
- **auditd:** `-w /sys/fs/cgroup -p wa -k cgroup_write` va a sacar a la luz el montaje y la escritura.
- **Tracee / Tetragon (eBPF):** firmas integradas de abuso de `release_agent` en cgroup. Son fiables y con pocos falsos positivos.

### 6.4 Mitigación

- Pasa a cgroup v2: el mecanismo `release_agent` no existe ahí.
- Quita `CAP_SYS_ADMIN` de los containers que de verdad no la necesitan.
- Mantén activo el perfil por defecto de AppArmor o SELinux: `docker-default` bloquea el montaje.

## 7. Escape 4 — `/proc/sys/kernel/core_pattern` escribible

**Precondición:** `/proc` está montado desde el host (no es el procfs propio del container) y es escribible, o el container es `--privileged`. Cualquier caída de un proceso en cualquier PID namespace que comparta ese sysctl dispara el patrón.

**Impacto:** un `core_pattern` del host escribible permite que el kernel ejecute un programa como root en el host cuando un proceso vuelca un core. El payload y el disparo del volcado no se incluyen.

### 7.2 Detección

- **Falco:** la regla `Write below /proc` pilla la escritura del sysctl desde dentro de un container.
- **auditd:** `-w /proc/sys/kernel/core_pattern -p wa -k core_pattern_write`.
- **Sysmon para Linux:** los eventos de creación o modificación de archivo sobre `/proc/sys/kernel/core_pattern` tienen mucha señal.

### 7.3 Mitigación

- Nunca montes `/proc` del host dentro de un container. El procfs enmascarado que Docker monta por defecto no es escribible y oculta `core_pattern`.
- Aplica el perfil de seccomp por defecto, que bloquea las syscalls necesarias para restablecer montajes enmascarados.

## 8. Escape 5 — paths del host montados

**Precondición:** el container tiene un bind mount `-v /algún/path/del/host:/dentro` sobre un `/algún/path/del/host` sensible. Patrones habituales:

- `-v /etc/shadow:/etc/shadow:ro` (el patrón descuidado de «compartir los usuarios»)
- `-v /root/.ssh:/root/.ssh` (container de build con las claves de quien lo escribió)
- `-v /var/log:/var/log` (agregadores de logs)
- `-v /var/lib/docker:/var/lib/docker` (estilo dind: te da las capas de todos los demás containers)
- Kubernetes `hostPath: { path: / }` (se acabó el juego en ese nodo)

**Impacto:** según el path y si el montaje es de escritura, el efecto va de leer secretos del host a escribir persistencia en cuentas del host, alterar logs o leer layers y volumes de otros containers, incluidos secretos horneados en ellos. Las acciones concretas sobre esos paths no se incluyen.

### 8.2 Detección

- **Falco:** `Read sensitive file untrusted` (coincide con una lectura de `/etc/shadow` desde un container) y `Write below root` (cualquier container que escriba bajo `/root/`).
- **auditd:** watches de archivo sobre `/etc/shadow`, `/root/.ssh` y `/var/log`.
- **osquery / Fleet:** consulta periódica `docker_container_mounts` con una lista de permitidos.

### 8.3 Mitigación

- Aplica una lista de permitidos de bind mounts en tu admission controller.
- Usa montajes de solo lectura (`:ro`) siempre que el container no necesite escribir.
- Prefiere named volumes a paths del host siempre que puedas.

## 9. Escape 6 — pod de Kubernetes con `hostPID: true` o `hostNetwork: true`

**Precondición:** un pod del cluster objetivo se programó con `hostPID: true`, `hostNetwork: true` o `hostIPC: true`. A menudo va junto con `privileged: true`. Es muy habitual en DaemonSets que publican proveedores de observabilidad, almacenamiento y CNI: esos DaemonSets son objetivos de primera.

**Impacto:** con `hostPID: true`, el container ve cada proceso del host en su propio `/proc`. Combinado con `CAP_SYS_ADMIN` o con `--privileged`, se puede entrar en los namespaces de mount y de PID del PID 1 del nodo (normalmente systemd o un init gestionado por el kubelet) y obtener un shell en el nodo. En un nodo de control plane, eso es el cluster entero. Sin `--privileged`, pero con `hostPID: true`, todavía se pueden enumerar los procesos del host y leer el entorno de cada uno, lo que filtra secretos, tokens del kubelet, URLs de bases de datos y todo lo que se pasó como variable de entorno a cualquier proceso del nodo.

El manifiesto del pod vulnerable y el comando que entra en los namespaces del nodo no se incluyen.

### 9.3 Detección

- **Falco:** `Launch Privileged Container`, `Launch Sensitive Mount Container` y `Namespace Change (setns) by Unknown Process` se disparan en el paso de cambio de namespace.
- **Log de auditoría de Kubernetes:** la admisión de cualquier pod con `hostPID: true` o `privileged: true` es un evento de mucha señal. Las políticas de Kyverno/OPA deberían bloquearlo, y el rastro de auditoría debería alertar.
- **Falco k8s-audit:** las reglas `Create HostNetwork Pod` y `Create Privileged Pod` vienen en el conjunto de reglas por defecto.

### 9.4 Mitigación

- Aplica Pod Security Standards en el nivel `restricted` en cada namespace que no necesite una excepción concreta.
- Usa políticas de Kyverno u OPA Gatekeeper para rechazar `hostPID`, `hostNetwork`, `hostIPC`, `privileged`, `hostPath` y `capabilities.add` fuera de una lista de permitidos muy pequeña.
- Separa los DaemonSets que de verdad necesitan acceso al host en node pools dedicados y pon network policies para aislarlos.

## 10. Escape 7 — exploits de kernel (solo laboratorio)

Las vulnerabilidades del kernel son el camino de reserva cuando la lista de malas configuraciones vuelve vacía. También son la razón por la que las distros parchean el kernel cada mes. Úsalas solo contra un kernel de laboratorio que arrancaste específicamente para probar el arreglo. No se incluyen pruebas de concepto ni repositorios de exploit.

### 10.1 CVE-2022-0492 — release_agent de cgroups v1 sin `CAP_SYS_ADMIN`

- **Kernel:** anterior a 5.16.14 sin el backport del arreglo.
- **Impacto:** la escritura de `release_agent` no comprobaba la propiedad del namespace. Un container sin `CAP_SYS_ADMIN` en el user namespace inicial podía escribir `release_agent` si se combinaba con un user namespace no privilegiado, y el efecto es el mismo escape que en la sección 6.
- **Mitigación:** actualiza el kernel; corre en cgroup v2; desactiva los user namespaces no privilegiados donde se pueda (`sysctl kernel.unprivileged_userns_clone=0`).

### 10.2 CVE-2022-0847 — Dirty Pipe

- **Kernel:** de 5.8 a 5.16.11 / 5.15.25 / 5.10.102.
- **Impacto:** un fallo en el subsistema de pipes deja que un usuario no privilegiado sobrescriba páginas de archivos de solo lectura, incluidos binarios SUID y capas de images de container. Desde dentro de un container, eso puede alcanzar binarios del runtime en el host y dar ejecución de código la próxima vez que se lance cualquier container.
- **Mitigación:** parchea el kernel. No hay un workaround que valga la pena.

### 10.3 CVE-2019-5736 — sobrescritura del binario de runc en el host (histórico)

- **Runtime:** runc anterior a 1.0-rc7.
- **Impacto:** cuando un container hacía exec dentro de una image comprometida, el descriptor de exec `/proc/self/exe` podía reescribirse para apuntar a código del atacante, que después sustituía el binario de runc del host.
- **Mitigación:** las versiones parcheadas de runc traen una mitigación basada en memfd. Compruébalo con `runc --version`.

### 10.4 Detección

- Los exploits de kernel son difíciles de pillar en la propia superficie de syscalls. Detecta el resultado: monitorización de integridad de archivos sobre `/usr/bin/runc`, `/usr/sbin/*`, `/etc/passwd` y `/etc/shadow`; alertas de EDR ante binarios SUID nuevos; `file_events` de osquery en paths bajo `/usr` y `/etc`.
- La regla de Falco `Modify binary dirs` pilla el resultado de Dirty Pipe sobre runc.

## 11. Escape 8 — socket de Docker por la API cruda

La misma precondición que el Escape 1, pero sin el binario `docker` instalado. Bastan `curl` y la API HTTP-sobre-Unix del daemon.

**Impacto:** enumerar la versión y las images, y pedirle al daemon que cree y arranque un container, es el mismo control del host que en el Escape 1. El cuerpo JSON que crea ese container y el paso que deja un shell con privilegios en el host no se incluyen.

### 11.1 Enumerar

Las lecturas de diagnóstico del socket, ya vistas en la lección 01, son:

```bash
curl --unix-socket /var/run/docker.sock http://localhost/version
curl --unix-socket /var/run/docker.sock http://localhost/images/json
```

### 11.4 Detección

Las mismas detecciones que el Escape 1: el daemon registra la creación del container de todas formas, y Falco y auditd se disparan en la escritura del socket y en el montaje sospechoso del container resultante.

## 12. Visión general de detección y mitigación

### 12.1 Sandboxes de runtime

| Capa | Producto | Qué cambia |
| --- | --- | --- |
| Interposición de syscalls | gVisor (runsc) | Los containers hablan con un Sentry en espacio de usuario que reimplementa las syscalls de Linux. Los exploits de kernel contra el kernel del host quedan fuera de alcance. |
| Micro-VM | Kata Containers | Cada pod corre en una VM ligera (Firecracker/QEMU/Cloud Hypervisor). El escape tiene que salir del kernel guest y del hypervisor. |
| Micro-VM | Firecracker | La misma idea, usada por AWS Lambda/Fargate. |
| MAC | AppArmor `docker-default` | Bloquea mount, ptrace y muchas escrituras en `/proc` incluso dentro de `--privileged`. |
| MAC | SELinux `container_t` | Parecido en RHEL / OpenShift. |

### 12.2 Pila de detección en runtime

- **Falco** — la herramienta de seguridad en runtime por defecto para Kubernetes. Trae reglas para cada familia de escape de esta lección. Despliégalo como DaemonSet y manda las alertas a Falcosidekick y de ahí al SIEM.
- **Tetragon (Cilium)** — basado en eBPF; pilla transiciones de namespace, cambios de capabilities y escrituras de archivo a nivel de evento del kernel.
- **Tracee (Aqua)** — motor de firmas eBPF con reglas con nombre para cada escape basado en CVE.
- **auditd** — el clásico. Más lento y más ruidoso, pero universal. Watches de archivo sobre `/proc/sys/kernel/core_pattern`, `/var/run/docker.sock` y `/sys/fs/cgroup/*/release_agent`.

### 12.3 Flags del runtime de containers que matan la mayoría de los escapes

```bash
# Endurecimiento de base que deberías aplicar a cada container:
docker run \
    --user 1000:1000 \                    # no root dentro
    --read-only \                          # el rootfs es inmutable
    --cap-drop=ALL \                       # empieza desde cero capabilities
    --cap-add=NET_BIND_SERVICE \           # añade solo lo que necesitas
    --security-opt=no-new-privileges \     # bloquea la escalada por setuid
    --security-opt=seccomp=default.json \  # conserva el seccomp por defecto
    --pids-limit=100 \                     # protección contra fork bomb
    --tmpfs /tmp:rw,noexec,nosuid,size=64m \
    myimage:tag
```

### 12.4 Docker rootless / Podman

El modo rootless ejecuta el daemon (o, en Podman, no hay daemon) como un usuario no privilegiado, así que un escape de container te deja en un UID no privilegiado del host en lugar de en root. Cambia el radio de explosión de «eres dueño del nodo» a «eres dueño de los archivos de ese usuario».

## 13. Endurecimiento de la image de runtime

La superficie de escape empieza en el build. Una image endurecida le deja mucho menos material al runtime.

### 13.1 Images base distroless

```dockerfile
# MAL — Debian completo, cientos de binarios que quien ataca puede usar:
FROM debian:12
COPY app /app
CMD ["/app"]

# BIEN — distroless: solo la app y sus dependencias de runtime:
FROM gcr.io/distroless/base-debian12:nonroot
COPY app /app
USER nonroot
ENTRYPOINT ["/app"]
```

Las images distroless no tienen shell, ni gestor de paquetes, ni curl, ni wget. Quien ataca y tiene RCE tiene que traer sus propias herramientas: cada comando de enumeración de la sección 3 falla porque los binarios no existen.

### 13.2 `USER` no root

```dockerfile
# Crea un usuario dedicado en el build de la image:
RUN groupadd -r app && useradd -r -g app -u 10001 app
USER 10001:10001
```

Correr como UID 0 dentro de un container es el segundo amplificador de radio de explosión, después de `--privileged`. El UID 10001 con `--cap-drop=ALL` casi no tiene superficie de ataque hacia el kernel.

### 13.3 Filesystem raíz de solo lectura

```yaml
# Kubernetes:
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

Un rootfs de solo lectura significa que no se puede dejar un webshell, no se puede instalar un paquete y no se puede modificar `/etc/passwd`. Combinado con `no-new-privileges`, la mayoría de las primitivas de persistencia desaparecen.

### 13.4 Conjunto mínimo de capabilities

Empieza con `--cap-drop=ALL` y devuelve solo lo que la carga demuestra que necesita:

| Carga | Lo que se suele añadir | Por qué |
| --- | --- | --- |
| Servidor HTTP en el puerto 80/443 | `NET_BIND_SERVICE` | Enlazar un puerto bajo como no root |
| tcpdump / captura de paquetes | `NET_ADMIN`, `NET_RAW` | Sockets crudos |
| Ping | `NET_RAW` | Sockets crudos de ICMP |
| Cambiar el dueño de archivos al arrancar | `CHOWN`, `FOWNER` | Solo el paso de init: plantéate correr como root una vez y después soltar privilegios |
| La mayoría de los containers de aplicación | (ninguna) | No necesitan ninguna de estas |

## 14. Escáneres de seguridad de containers

Detecta images vulnerables y Dockerfiles mal configurados antes de que lleguen a producción.

### 14.1 Trivy

```bash
# Escanea una image en busca de CVEs conocidos:
trivy image myapp:1.2.3

# Escanea un filesystem o un repositorio en busca de secretos y malas configuraciones:
trivy fs .

# Escanea un cluster de Kubernetes:
trivy k8s --report summary cluster
```

### 14.2 Grype + Syft

```bash
# Genera un SBOM (Software Bill of Materials):
syft myapp:1.2.3 -o spdx-json > sbom.json

# Escanea el SBOM en busca de CVEs:
grype sbom:./sbom.json
```

### 14.3 `docker scan` / `docker scout`

```bash
docker scout cves myapp:1.2.3
docker scout recommendations myapp:1.2.3
```

### 14.4 Linting de Dockerfile

```bash
# hadolint — linter de buenas prácticas de Dockerfile:
hadolint Dockerfile
# DL3002 warning: Last USER should not be root
# DL3020 error: Use COPY instead of ADD for files and folders

# checkov — escáner de IaC que también trata Dockerfiles y YAML de Kubernetes:
checkov -f Dockerfile
checkov -d ./k8s/
```

La salida de hadolint se queda en inglés: es stdout literal de la herramienta.

### 14.5 Mete los escáneres en CI

El escaneo solo sirve si hace fallar el build. En un pipeline de GitHub Actions o de GitLab CI, añade:

```yaml
- name: Trivy image scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ steps.build.outputs.image }}'
    severity: 'HIGH,CRITICAL'
    exit-code: '1'
```

## 15. Laboratorio completo — del container privileged al host, y después bloqueado

Este es el ejercicio que ejecutas de punta a punta en tu VM de laboratorio. El recorrido ofensivo (ejecutar el escape de `release_agent` y obtener un shell de root en el host montando el disco) no se incluye. Sí se incluyen los prerequisitos, la configuración vulnerable que vas a corregir, la configuración endurecida y las comprobaciones que demuestran que el escape quedó anulado.

### 15.1 Prerequisitos del laboratorio

```bash
# En una VM de laboratorio Ubuntu 22.04 recién instalada, que sea tuya:
sudo apt update
sudo apt install -y docker.io docker-compose-v2 auditd git
sudo usermod -aG docker $USER
newgrp docker

# Verifica:
docker version
docker compose version
```

Pertenecer al grupo `docker` equivale a root en esa VM. Es aceptable aquí porque la VM es desechable y tuya. No copies este paso a un host compartido.

### 15.2 Compose vulnerable

Guárdalo como `~/escape-lab/docker-compose.yml`. Es el «antes»: la mala configuración que vas a detectar y a sustituir.

```yaml
services:
  vulnerable:
    image: alpine:3.19
    container_name: vuln-priv
    privileged: true              # <-- el mando vulnerable
    command: sleep infinity
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # primitiva de escape adicional
```

Levántalo solo para enumerar y para comparar con la versión endurecida:

```bash
cd ~/escape-lab
docker compose up -d
docker compose exec vulnerable sh
```

### 15.3 Enumerar

Dentro del container:

```bash
cat /proc/1/cgroup
ls /.dockerenv
grep -E '^Cap(Prm|Eff)' /proc/self/status
capsh --print | head -20
mount | grep -E 'docker.sock|/dev/sda'
ls /dev/sda* /var/run/docker.sock
```

Resultado esperado en esta configuración vulnerable: conjunto completo de capabilities, socket de docker montado, dispositivo de bloque del host visible, `/proc` del host escribible. No continúes con el escape: pasa a la mitigación.

### 15.6 Aplicar mitigaciones

Reescribe el archivo Compose:

```yaml
services:
  hardened:
    image: alpine:3.19
    container_name: hardened
    command: sleep infinity
    user: "10001:10001"
    read_only: true
    cap_drop: [ "ALL" ]
    security_opt:
      - no-new-privileges:true
      - seccomp=default
    pids_limit: 100
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    # SIN privileged, SIN montaje de docker.sock, SIN hostPath.
```

### 15.7 Volver a comprobar, ahora bloqueado

Recrea el container y repite la enumeración de la sección 15.3:

```bash
docker compose down
docker compose up -d
docker compose exec hardened sh
```

Observaciones que deberías anotar. Son la validación de que el arreglo aguanta:

- `capsh --print` muestra un conjunto current vacío.
- `ip link add dummy0 type dummy` devuelve `RTNETLINK answers: Operation not permitted`.
- `ls /dev/sda*` no muestra dispositivos de bloque del host.
- `mount -t cgroup -o rdma cgroup /tmp/cgrp` falla con `Operation not permitted`.
- Escribir en `/proc/sys/kernel/core_pattern` falla: el montaje es de solo lectura.
- No hay `/var/run/docker.sock`.

El escape queda anulado de punta a punta. Captura pantallas de cada paso bloqueado para el informe. Las cadenas `Operation not permitted` y `RTNETLINK answers: Operation not permitted` son stdout literal: se quedan en inglés.

### 15.8 Capa de detección

Con una segunda terminal abierta mientras comparas la configuración vulnerable y la endurecida en el laboratorio:

```bash
# Activa reglas de auditd que cubren el escape:
sudo tee -a /etc/audit/rules.d/container.rules <<'EOF'
-w /var/run/docker.sock -p rwa -k docker_socket
-w /proc/sys/kernel/core_pattern -p wa -k core_pattern
-a always,exit -F arch=b64 -S mount -k mount_syscall
EOF
sudo augenrules --load
sudo auditctl -l

# Observa los eventos:
sudo ausearch -k core_pattern -ts recent
sudo ausearch -k docker_socket -ts recent
```

Para Falco, instálalo desde el repositorio oficial, aplica el conjunto de reglas por defecto y sigue el log mientras haces la enumeración de la configuración vulnerable: los montajes sensibles y el arranque privileged deberían generar una alerta coincidente con las reglas por defecto.

## 16. Ejercicios

### 16.1 Ejercicios guiados

**Ejercicio 1 — Enumerar un container dado.** Levanta `docker run --rm -it --cap-add=SYS_PTRACE -v /etc:/host-etc:ro alpine sh`. Con la lista de la sección 3, escribe un informe que responda: qué capabilities hay y qué montajes son sensibles. Apunta a menos de 90 segundos. El entregable de «camino de escape o de exfiltración más corto» no se incluye: documenta las condiciones peligrosas y la mitigación que las quitaría.

**Ejercicio 2 — Reproducir el escape del socket de Docker.** Este ejercicio pedía ejecutar el Escape 1 con el Compose de la sección 15.2 y entregar un shell con el hostname del host. Esa reproducción no se incluye. El entregable defensivo que sí conserva la lección es la alerta de Falco de la sección 4.3 ante un montaje sensible, observada mientras existe la configuración vulnerable y antes de aplicar la sección 15.6.

**Ejercicio 3 — Aplicar las mitigaciones y verificarlas.** Toma el Compose vulnerable de la sección 15.2, aplica los ajustes endurecidos de la 15.6 y vuelve a ejecutar la lista de enumeración. Entregable: una tabla lado a lado con el «antes» y el «después» de cada ítem de la lista.

### 16.2 Ejercicios independientes

**Ejercicio 4 — Auditar un archivo Compose dado en busca de primitivas de escape.**

Dado el archivo siguiente, identifica cada primitiva de escape que contiene y propone una corrección para cada una:

```yaml
services:
  legacy-runner:
    image: python:3.11
    privileged: true
    network_mode: host
    pid: host
    volumes:
      - /:/host
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.ssh:/root/.ssh
    environment:
      - AWS_SECRET_ACCESS_KEY=hunter2
    command: python /app/server.py
```

Entregable: un escrito de una página que liste cada hallazgo, su camino de escape a nivel de impacto, la regla de Falco que lo detectaría y la línea concreta de una reescritura endurecida que lo corrige. `hunter2` es un secreto de laboratorio falso, metido en el archivo para que lo marques: no lo uses fuera de este ejercicio.

**Ejercicio 5 — Escribir en Bash un comprobador al estilo de deepce.**

Escribe un script `container-triage.sh` que:

- Detecte si está corriendo dentro de un container (usando al menos tres de las señales de la sección 3.1).
- Imprima el conjunto de capabilities con nombres legibles.
- Marque montajes peligrosos (`docker.sock`, `/` del host, `/etc/shadow`, `/root/.ssh`, `/var/lib/docker`).
- Informe si las capabilities permiten operaciones privilegiadas de red, de mount o de dispositivos de bloque del host. No incluyas una escritura a `/proc/sysrq-trigger`.
- Salga con 0 si todas las comprobaciones pasan y con 1 si encuentra cualquier condición peligrosa.

Entregable: el script, más una salida de ejemplo contra el compose vulnerable y contra el endurecido del walkthrough.

**Ejercicio 6 — Endurecer un Dockerfile que corre como root.**

Dado:

```dockerfile
FROM ubuntu:22.04
RUN apt update && apt install -y python3 python3-pip
COPY . /app
WORKDIR /app
RUN pip3 install -r requirements.txt
CMD ["python3", "server.py"]
```

Produce una versión endurecida que:

- Use una base distroless o `python:3.12-slim`.
- Cree un usuario no root y corra como él (UID > 10000).
- Fije `WORKDIR` a un directorio de ese usuario.
- Habilite `--read-only` con un tmpfs explícito para lo que necesite escribir.
- Pase `hadolint` y `trivy image` con cero hallazgos HIGH/CRITICAL.

Entregable: el Dockerfile nuevo, la salida de hadolint y la salida de `trivy image`.

## 17. Resumen y siguientes pasos

Esta lección mapeó el terreno que recorre un red team después de caer dentro de un container: enumerar primero, elegir el más corto de ocho caminos de escape y enmarcar siempre el hallazgo con la detección y la mitigación. La lección más importante es que los escapes de container en la naturaleza son, de forma aplastante, escapes por mala configuración —un socket montado, un sidecar `--privileged`, un volume `hostPath: /`— y no un 0-day del kernel. Caza eso en tus engagements; enséñale a tus compañeros del equipo azul a escribir reglas de admission controller y alertas de Falco contra ello.

Puntos que hay que retener:

- El modelo de aislamiento del container es namespaces + cgroups + capabilities + seccomp + LSM, todo sobre un kernel compartido. El kernel es el límite de confianza que no puedes cruzar sin un sandbox como gVisor o una micro-VM como Kata.
- Los primeros 60 segundos dentro de un container responden cuatro preguntas: ¿estoy en un container?, ¿cuáles son mis capabilities?, ¿qué está montado?, ¿está el socket de Docker?
- `CAP_SYS_ADMIN` + cgroup v1 + un `release_agent` escribible = root en el host. `--privileged` te da las tres cosas por defecto.
- `hostPID: true` + `privileged: true` permite entrar en los namespaces del nodo y obtener un shell ahí.
- Cada familia de escape tiene una regla de Falco, un watch de archivo de auditd y un comportamiento de EDR: incluye siempre la detección en tu informe.
- El endurecimiento es aburrido y eficaz: `USER` no root, `--cap-drop=ALL`, `no-new-privileges`, seccomp por defecto, rootfs de solo lectura, image base distroless, sin montajes del host.

Siguiente lección: Módulo 04 — Lección 04: caminos de ataque a un cluster de Kubernetes y abuso de RBAC. Ahí la primitiva de escape de pod de la sección 9 se encadena hacia el compromiso del cluster mediante tokens de service account, malas configuraciones de RBAC y exfiltración de etcd. Esa lección no forma parte de este archivo.
