# Lección 12 — Lab: Container Security and Architecture Analysis

Este lab de cierre encadena cuatro ataques que espejan lo que pasa en breaches cloud-native reales: un escape por socket de Docker, una misconfiguration de Kubernetes explotada desde un pod, un patrón de supply chain de CI/CD vía un workflow envenenado de GitHub Actions, y un recon de arquitectura completo sobre un stack chico de varios servicios.

Cada desafío viene con notas de detección. No te las saltes — el punto de este módulo no es solo correr el escape, sino poder escribir la regla de Falco, la config de Sysmon o la policy de OPA/Gatekeeper que te habría cazado haciéndolo.

## Diagrama del módulo

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

El Módulo 04 pasó las once lecciones anteriores desarmando el stack de containers y cloud-native capa por capa: namespaces y cgroups de Linux, la especificación del runtime OCI, supply chains de imágenes, internals de red de Docker, objetos del control plane de Kubernetes, service accounts y RBAC, admission controllers, modelos de amenaza de pipelines de CI/CD, y las topologías de reverse-proxy / API gateway / data-store sobre las que se paran la mayoría de los productos SaaS modernos. Cada lección enseñó una rebanada — esta las pega.

Este lab consolida el módulo. Vas a correr cuatro desafíos de punta a punta que encadenan debilidades de container, cluster, pipeline y capa de aplicación como lo haría un adversario real, contra targets controlados que armas vos. Cuando termines, tu repositorio `~/pentest-tools/` va a contener scripts de ataque reproducibles, re-corridas endurecidas que prueban que cada finding está arreglado, y notas de detección que le puedes entregar a un blue team. Desde el Módulo 05 en adelante, cada lección de enumeración y explotación asume que tienes esta memoria muscular.

Los cuatro desafíos son:

- **Escape de Docker vía un socket montado** — abusa de `/var/run/docker.sock` dentro de un container para spawnear un sibling privilegiado y leer el `/etc/shadow` del host.
- **Abuso de RBAC de Kubernetes** — desde un pod con un ServiceAccount de scope excesivo, lista y exfiltra cada Secret de cada namespace.
- **Inyección en pipeline de CI/CD** — arma un workflow de GitHub Actions `pull_request_target` con un sink sin sanitizar `${{ github.event.pull_request.title }}` para robar el `GITHUB_TOKEN`.
- **Recon de arquitectura sobre un stack vivo** — desde afuera, haz fingerprint de un stack nginx → API → Postgres → Redis → Elasticsearch, encuentra el índice de Elasticsearch sin authentication que filtra credenciales y loguéate en la API.

Cada desafío sigue la misma forma: setup, objetivo, walkthrough completo con comandos exactos y output esperado, notas de detección, y un stretch goal que endurece la misconfiguration y te pide re-verificar que el exploit falla. Lee el objetivo, intenta el exploit vos, y después contrasta con la referencia. Tipea los comandos — no los pegues sin leer. Rompe cosas a propósito y vuelve a armarlas.

## 1. Setup del entorno de lab

### 1.1 Requisitos del host

- VM Kali Linux 2026.1 (se prefiere un snapshot del Módulo 01)
- 8 GB de RAM como mínimo, 40 GB de disco (el cluster de kind es glotón)
- Virtualización de CPU habilitada en el hypervisor; no hace falta virt anidada
- Alcanzabilidad de red saliente para pulls de imágenes
- Un usuario en el grupo `docker` y sudo sin password para levantar el cluster

### 1.2 Instalación del toolchain

Corre esto una vez como tu usuario normal (no root):

```bash
sudo apt update
sudo apt install -y \
    docker.io docker-compose-plugin \
    curl wget jq git tmux \
    apt-transport-https ca-certificates gnupg \
    python3 python3-venv python3-pip \
    nmap whatweb dnsutils \
    build-essential

# Habilita Docker sin sudo
sudo usermod -aG docker "$USER"
newgrp docker

# Instala kubectl 1.31.x desde el repo apt upstream
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubectl

# kind (Kubernetes in Docker) v0.24.0
curl -Lo /tmp/kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
sudo install -m 0755 /tmp/kind /usr/local/bin/kind

# trivy (scanner de imágenes y de cluster)
sudo apt install -y trivy

# kube-hunter (scanner de misconfigurations de cluster)
pipx install kube-hunter || pip install --user kube-hunter

# jwt_tool (helper del desafío y desafío independiente 3)
git clone https://github.com/ticarpi/jwt_tool /opt/jwt_tool
sudo ln -sf /opt/jwt_tool/jwt_tool.py /usr/local/bin/jwt_tool
pip install --user termcolor pycryptodomex requests

# elasticdump (Desafío 4)
sudo apt install -y npm
sudo npm install -g elasticdump

# nektos/act para corridas locales de GitHub Actions (Desafío 3)
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash -s -- -b /usr/local/bin
```

Verifica cada tool:

```text
docker version         # 25.x or newer
kind version           # v0.24.0
kubectl version --client
trivy --version        # 0.55.x or newer
kube-hunter --help
jwt_tool -h
elasticdump --version
act --version
```

### 1.3 Layout del workspace

```bash
mkdir -p ~/pentest-tools/module04 ~/labs/module04
cd ~/pentest-tools
[ -d .git ] || git init
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip requests pyjwt
pip freeze > module04/requirements.txt
git add module04/requirements.txt
git commit -m "chore(m04): pin module04 lab deps" || true
```

Cada desafío escribe sus artefactos en `~/pentest-tools/module04/<challenge>/` y cada manifiesto de target de lab vive bajo `~/labs/module04/<challenge>/`. Mantén los dos árboles separados: tooling de ataque en el repo de tools, infraestructura del target en el árbol de labs.

## 2. Desafío 1 — Escape de Docker vía socket montado

### 2.1 Modelo de amenaza

Un container de developer en un runner de CI legítimo monta `/var/run/docker.sock` «para que el pipeline pueda construir imágenes». Un atacante con RCE dentro de ese container hereda la capacidad de hablarle al daemon de Docker del host, y el daemon de Docker del host corre como root. Desde ahí, montar el filesystem raíz del host en un container privilegiado fresco es trivial. Este es el escape de container más común que se encuentra en la vida real.

### 2.2 Setup

```bash
mkdir -p ~/labs/module04/c1-docker-sock
cd ~/labs/module04/c1-docker-sock
```

`docker-compose.yml`:

```yaml
version: '3.8'
services:
  dev-runner:
    image: alpine:3.20
    container_name: c1-dev-runner
    command: sh -c "apk add --no-cache docker-cli curl && sleep infinity"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

Levántalo:

```bash
docker compose up -d
docker compose ps
```

Esperado:

```text
NAME            IMAGE          COMMAND                  STATUS
c1-dev-runner   alpine:3.20    "sh -c 'apk add --n…'"   Up 4 seconds
```

Sanity: el container debería tener un cliente docker que funcione y ver el daemon:

```bash
docker exec -it c1-dev-runner sh -c "docker version --format '{{.Server.Version}}'"
# → 25.x.x   (this is the *host* daemon speaking, not a nested one)
```

### 2.3 Objetivo

El objetivo es dejar el compose vulnerable, documentar que el socket está montado y que el grupo `docker` equivale a root, y después quitar el volume y confirmar que el cliente ya no habla con el daemon. No imprimas `/etc/shadow`.

### 2.4 Walkthrough

Abre una shell en el container comprometido:

```bash
docker exec -it c1-dev-runner sh
```

Dentro del container, confirma que el primitivo de escape está presente:

```bash
ls -l /var/run/docker.sock
# srw-rw---- 1 root root 0 Sep  4 12:00 /var/run/docker.sock
```

El socket está montado y el cliente docker del container habla con el daemon del host. El fuente lanza desde ahí un container privilegiado que monta `/` del host y lee la primera línea de `/etc/shadow`. Ese escape no se incluye.

### 2.5 Por qué el socket equivale a root

La CLI de docker en el container le habla al daemon por el socket UNIX montado. El daemon autentica por ownership del socket, y el socket es escribible por el grupo `docker`. Un container nuevo con `--privileged` y un bind de `/` sale del namespace. La frontera del container desaparece. Los flags no se incluyen.

### 2.6 Notas de detección

Regla de Falco (`~/pentest-tools/module04/c1-docker-sock/falco-docker-sock.yaml`):

```yaml
- rule: Docker socket mounted into container
  desc: Detect containers started with /var/run/docker.sock bind-mounted
  condition: >
    container.id != host and
    proc.name = docker and
    proc.args contains "run" and
    (proc.args contains "docker.sock" or proc.args contains "--privileged")
  output: >
    Docker-in-Docker escape attempt
    (user=%user.name container=%container.id cmd=%proc.cmdline)
  priority: CRITICAL
  tags: [container, escape, mitre_privilege_escalation]
```

Regla de auditd (`/etc/audit/rules.d/docker-sock.rules`):

```text
-w /var/run/docker.sock -p rwxa -k docker_sock_access
```

Recarga las reglas y mira:

```bash
sudo augenrules --load
sudo auditctl -l
sudo ausearch -k docker_sock_access --start recent | head -n 20
```

Deberías ver un registro `SYSCALL` por cada acceso del container al socket, incluido el pid, uid y comm del caller — suficiente para correlacionar con el pod o el id de workload ofensor.

### 2.7 Stretch — quita el mount del socket y confirma que el escape se rompe

Edita el archivo compose, borrando el bloque `volumes:`, y recrea:

```bash
sed -i '/volumes:/,/docker.sock/d' docker-compose.yml
docker compose up -d --force-recreate
```

Vuelve a comprobar el socket desde dentro del container:

```bash
docker exec -it c1-dev-runner sh -c 'docker ps 2>&1'
```

Output de falla esperado:

```text
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

El primitivo de escape se fue. Commitea la transcripción del test negativo:

```bash
docker exec c1-dev-runner sh -c 'docker ps 2>&1' \
  > ~/pentest-tools/module04/c1-docker-sock/hardened-negative.txt
git -C ~/pentest-tools add module04/c1-docker-sock
git -C ~/pentest-tools commit -m "c1: docker socket escape proof + hardened negative"
```

Derriba:

```bash
docker compose down
```

## 3. Desafío 2 — Explotación de una misconfiguration de Kubernetes

### 3.1 Modelo de amenaza

Un ingeniero de plataforma bien intencionado liga el ServiceAccount de una aplicación a un ClusterRole que otorga `get` y `list` de secrets a nivel de cluster — tal vez para dejar que un operador lea material TLS. Cualquiera que caiga con RCE en ese pod hereda el token, y el token es una credencial de primera clase contra el API server. Lectura de Secret a nivel de cluster es game over: expone cada ImagePullSecret, cada password de aplicación, cada signing key JWT del service mesh.

### 3.2 Setup

```bash
mkdir -p ~/labs/module04/c2-k8s-rbac
cd ~/labs/module04/c2-k8s-rbac
```

`kind-cluster.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: m04-lab
nodes:
  - role: control-plane
  - role: worker
```

Levanta el cluster:

```bash
kind create cluster --config kind-cluster.yaml
kubectl cluster-info --context kind-m04-lab
kubectl get nodes
```

Esperado:

```text
NAME                    STATUS   ROLES           AGE   VERSION
m04-lab-control-plane   Ready    control-plane   30s   v1.31.0
m04-lab-worker          Ready    <none>          20s   v1.31.0
```

Siembra un puñado de Secrets interesantes en tres namespaces:

```bash
for ns in dev staging prod; do
  kubectl create namespace "$ns"
  kubectl -n "$ns" create secret generic app-db \
      --from-literal=username=svc_"$ns" \
      --from-literal=password="p@ss_${ns}_$(openssl rand -hex 4)"
done
kubectl -n prod create secret generic stripe-api \
    --from-literal=key=sk_live_$(openssl rand -hex 12)
```

Ahora el RBAC deliberadamente sobre-scoped y el pod víctima. `victim.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: victim-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-all-secrets
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: victim-sa-read-secrets
subjects:
  - kind: ServiceAccount
    name: victim-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: read-all-secrets
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: victim
  namespace: default
spec:
  serviceAccountName: victim-sa
  containers:
    - name: shell
      image: alpine:3.20
      command: ["sh", "-c", "apk add --no-cache curl jq && sleep infinity"]
```

Aplica:

```bash
kubectl apply -f victim.yaml
kubectl wait --for=condition=Ready pod/victim --timeout=90s
```

### 3.3 Objetivo

Desde una shell dentro del pod `victim` — sin binario kubectl, sin kubeconfig — enumera cada Secret de cada namespace y dumpea el valor de `prod/stripe-api`.

### 3.4 Walkthrough

El fuente entra al pod, presenta el token al API server, lista los Secrets de todos los namespaces y decodifica `prod/stripe-api`. Ese volcado no se incluye. Lo que sí queda es la detección y la policy que niega el binding.

### 3.5 Notas de detección

Todo lo que hiciste aparece en el audit log del API server. En kind, la audit policy no está prendida por defecto — habilítala para la lección:

```bash
docker exec m04-lab-control-plane sh -c '
  grep -q audit-log-path /etc/kubernetes/manifests/kube-apiserver.yaml \
    || echo "audit not configured"
'
```

Pídele al kubelet los últimos 200 eventos que tocan Secrets desde una cuenta que no es de sistema:

```bash
kubectl get events -A --field-selector reason=SecretRead 2>/dev/null | head
```

Un cluster de producción con audit habilitado matchearía por patrón esta secuencia de llamadas: muchos `list` sobre `secrets` desde `system:serviceaccount:default:victim-sa`, repartidos entre namespaces, desde una IP de pod — una regla `k8s_audit` de Falco lo caza en un minuto.

### 3.6 Stretch — deniega el binding con OPA Gatekeeper

Instala Gatekeeper 3.17:

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.17.0/deploy/gatekeeper.yaml
kubectl -n gatekeeper-system wait --for=condition=Available deploy --all --timeout=180s
```

Constraint template que prohíbe ServiceAccounts de Pod ligados a `read-all-secrets`:

`no-cluster-secrets.yaml`:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoclustersecretsbinding
spec:
  crd:
    spec:
      names:
        kind: K8sNoClusterSecretsBinding
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoclustersecretsbinding

        forbidden_role := "read-all-secrets"

        violation[{"msg": msg}] {
          input.review.kind.kind == "Pod"
          sa := input.review.object.spec.serviceAccountName
          crb := data.inventory.cluster["rbac.authorization.k8s.io/v1"].ClusterRoleBinding[_]
          crb.roleRef.name == forbidden_role
          some i
          crb.subjects[i].kind == "ServiceAccount"
          crb.subjects[i].name == sa
          crb.subjects[i].namespace == input.review.object.metadata.namespace
          msg := sprintf("Pod SA %q is bound to forbidden ClusterRole %q",
                         [sa, forbidden_role])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoClusterSecretsBinding
metadata:
  name: block-cluster-secrets
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

Aplica y re-verifica:

```bash
kubectl apply -f no-cluster-secrets.yaml
kubectl delete pod victim
kubectl apply -f victim.yaml
```

Rechazo de admisión esperado:

```text
Error from server (Forbidden): error when creating "victim.yaml":
admission webhook "validation.gatekeeper.sh" denied the request:
[block-cluster-secrets] Pod SA "victim-sa" is bound to forbidden ClusterRole "read-all-secrets"
```

Commitea el diff y derriba:

```bash
git -C ~/pentest-tools add module04/c2-k8s-rbac
git -C ~/pentest-tools commit -m "c2: k8s rbac secret dump + gatekeeper denial"
kind delete cluster --name m04-lab
```

## 4. Desafío 3 — Simulación de ataque a un pipeline de CI/CD

### 4.1 Modelo de amenaza

`pull_request_target` corre un workflow en el contexto del repositorio base con acceso de escritura a secrets, pero con la metadata no confiable del repositorio head. Cualquier expresión como `${{ github.event.pull_request.title }}` interpolada directo en un bloque `run:` le da a un atacante externo una shell en un entorno de CI de confianza. Desde ahí el `GITHUB_TOKEN` — y típicamente cualquier otra cosa en `secrets.*` — está a un curl de distancia.

### 4.2 Setup

```bash
mkdir -p ~/labs/module04/c3-cicd/.github/workflows
cd ~/labs/module04/c3-cicd
git init -q
```

Workflow vulnerable — `.github/workflows/welcome.yml`:

```yaml
name: welcome-contributor
on:
  pull_request_target:
    types: [opened]
jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Say hi
        run: |
          echo "New PR titled: ${{ github.event.pull_request.title }}"
```

El workflow vulnerable interpola el título del pull request dentro de `run:`. El fuente arma un `event.json` cuyo título cierra la comilla y envía `GITHUB_TOKEN` a un listener, y lo corre con `act`. El payload, el listener y la corrida no se incluyen.

### 4.3 Detección

- **Scanner de Actions de CodeQL** — `github/codeql-action` trae un query pack `actions/actions-injection` que marca cualquier expresión `${{ github.event.* }}` que llega a un script `run:`. Agrégalo a `.github/workflows/codeql.yml`:

```yaml
    - uses: github/codeql-action/init@v3
      with:
        languages: actions
    - uses: github/codeql-action/analyze@v3
```

Finding esperado:

```text
actions/injection: 'github.event.pull_request.title' flows into a run
  step (welcome.yml:9). Severity: high.
```

- **Secret scanning de GitHub** — la push-protection de GitHub va a rechazar un commit que contenga un token `ghs_` o `ghp_` en el log de exfil si lo pusheas por accidente. Verifica que tu `.gitignore` excluya `exfil.log` y `.secrets`.

- **Control de egress del runner** — un runner endurecido en un entorno corporativo bloquea el tráfico saliente hacia hosts que no están en allow-list. La llamada al listener fallaría con `curl: (7) Failed to connect`.

### 4.6 Stretch — sanitiza el sink

Reescribe el workflow para que el string no confiable pase por una variable de entorno, que bash trata como dato, nunca como código:

```yaml
name: welcome-contributor
on:
  pull_request_target:
    types: [opened]
jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Say hi
        env:
          PR_TITLE: ${{ github.event.pull_request.title }}
        run: |
          printf 'New PR titled: %s\n' "$PR_TITLE"
```

Con el workflow de arriba, el título llega a `printf` por la variable `PR_TITLE`. Bash lo trata como dato: el texto se imprime y no se ejecuta. No hace falta volver a correr el evento del fuente.

## 5. Desafío 4 — Recon de arquitectura sobre un stack de lab vivo

### 5.1 Modelo de amenaza

Un SaaS «de 3 tiers» en el mundo real rara vez son tres tiers. Hay nginx delante, una API REST, un datastore primario, uno o dos caches, un motor de búsqueda, un object store, una cola y un puñado de dashboards de admin. Cada servicio es un punto de entrada potencial, y un eslabón débil — un Elasticsearch ligado a `0.0.0.0` con `xpack.security.enabled=false` — alcanza para desarmar el resto.

### 5.2 Setup

```bash
mkdir -p ~/labs/module04/c4-stack/{nginx,api}
cd ~/labs/module04/c4-stack
```

`docker-compose.yml`:

```yaml
version: '3.8'
networks:
  stack:
    driver: bridge
    ipam:
      config:
        - subnet: 172.24.0.0/24
services:

  nginx:
    image: nginx:1.27
    container_name: c4-nginx
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "8080:80"
    networks: [stack]

  api:
    image: python:3.12-slim
    container_name: c4-api
    working_dir: /app
    volumes:
      - ./api:/app
    command: sh -c "pip install --quiet flask==3.0.3 psycopg2-binary redis && python app.py"
    networks: [stack]

  postgres:
    image: postgres:16
    container_name: c4-postgres
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app_password
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"                       # deliberado: expuesto al host
    networks: [stack]

  redis:
    image: redis:7
    container_name: c4-redis
    command: ["redis-server", "--protected-mode", "no"]   # deliberado
    ports:
      - "6379:6379"
    networks: [stack]

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
    container_name: c4-es
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false      # deliberado
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"                       # deliberado: sin auth en internet
    networks: [stack]
```

`nginx/default.conf`:

```nginx
server {
  listen 80;
  server_name _;
  location / { proxy_pass http://api:5000; }
}
```

`api/app.py`:

```python
from flask import Flask, request, jsonify
import redis

app = Flask(__name__)
USERS = {"admin": "S3cret_lab_pw!"}     # se carga en ES al arrancar
r = redis.Redis(host="redis", port=6379, decode_responses=True)

@app.route("/")
def index():
    return "lab-api v1", 200

@app.route("/login", methods=["POST"])
def login():
    data = request.get_json(force=True)
    if USERS.get(data.get("user")) == data.get("pass"):
        return jsonify(ok=True, role="admin"), 200
    return jsonify(ok=False), 401

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Levanta el stack:

```bash
docker compose up -d
docker compose ps
```

Siembra Elasticsearch con un índice que filtra (corre una vez, después de que ES esté healthy):

```bash
until curl -sf http://127.0.0.1:9200/_cluster/health >/dev/null; do sleep 2; done

curl -s -X POST http://127.0.0.1:9200/app-users/_doc \
    -H 'Content-Type: application/json' \
    -d '{"user":"admin","pass":"S3cret_lab_pw!","note":"prod-mirror seed"}'
curl -s -X POST http://127.0.0.1:9200/app-users/_doc \
    -H 'Content-Type: application/json' \
    -d '{"user":"alice","pass":"alice-2025","note":"customer"}'
curl -s http://127.0.0.1:9200/app-users/_refresh
```

### 5.3 Objetivo

Asume que solo sabes que `http://127.0.0.1:8080/` está expuesto. Haz fingerprint del stack y anota que el 9200 responde sin autenticación y que existe el índice `app-users`. El control es autenticación en Elasticsearch, no publicar el 9200 y no guardar contraseñas en el índice. No vuelques el índice ni uses esas credenciales.

### 5.4 Walkthrough

Paso 1 — fingerprint de la puerta de entrada.

```bash
whatweb -a 3 http://127.0.0.1:8080/
```

Esperado:

```text
http://127.0.0.1:8080/ [200 OK] HTTPServer[nginx/1.27], nginx[1.27]
```

Paso 2 — barre el host en busca de otros listeners. En un engagement real esto es un scan de rango dirigido; acá escaneamos el loopback de la VM de lab.

```bash
sudo nmap -sS -p- --min-rate 2000 -Pn 127.0.0.1 -oN scan.txt
```

Esperado (recortado):

```text
PORT     STATE SERVICE
5432/tcp open  postgresql
6379/tcp open  redis
8080/tcp open  http
9200/tcp open  wap-wsp
```

Reescanea los puertos interesantes con probes de servicio y de script:

```bash
sudo nmap -sV -sC -p 5432,6379,9200 127.0.0.1
```

Esperado:

```text
5432/tcp open  postgresql PostgreSQL DB 16
6379/tcp open  redis      Redis key-value store 7.x
9200/tcp open  http       Elasticsearch REST API 8.15.0
| http-title: Site doesn't have a title (application/json)
```

Paso 3 — verifica que Elasticsearch no tiene authentication.

```bash
curl -s http://127.0.0.1:9200/ | jq .
```

Esperado:

```json
{
  "name": "c4-es",
  "cluster_name": "docker-cluster",
  "version": {"number": "8.15.0", "build_flavor": "default"}
}
```

La ausencia de un 401 en el documento raíz es la pista. El `_cat/indices` muestra `app-users`. El fuente sigue con `elasticdump` de ese índice y un login en la API usando el campo `pass`. El volcado y el login no se incluyen. El control está en el objetivo de la sección 5.3.

### 5.5 Notas de detección

- **Access log de Elasticsearch** — cada `_cat/indices` y `_search` desde una dirección que no es loopback es una señal. En prod, manda `elasticsearch.log` a un SIEM y alerta sobre `path=/_search` desde IPs de cliente inesperadas.
- **Access log de nginx** — el recon pre-auth se ve como muchos `POST /login` chicos con payloads JSON que varían; rate-limit con `limit_req_zone` y alerta sobre ráfagas de 401.
- **Redis** — `MONITOR` desde un nodo de app comprometido es un camino común de exfil; deshabilítalo (`rename-command MONITOR ""`) y loguea `CONFIG GET` del lado del server.
- **Postgres** — habilita `log_connections=on` y alerta sobre conexiones desde un CIDR de cliente inesperado.

### 5.6 Stretch — endurece todo y vuelve a correr el recon

Edita `docker-compose.yml`:

```yaml
  postgres:
    # quita el bloque `ports:` por completo → sin exposición al host
  redis:
    command: ["redis-server", "--requirepass", "R3dis_hard!", "--protected-mode", "yes"]
    # quita `ports:` también
  elasticsearch:
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=Elastic_hard!
    # quita `ports:` — alcánzalo solo desde `api` en la red interna
```

Recrea y vuelve a correr el recon:

```bash
docker compose up -d --force-recreate
sudo nmap -sS -p 5432,6379,9200 127.0.0.1
```

Esperado:

```text
5432/tcp closed postgresql
6379/tcp closed redis
9200/tcp closed wap-wsp
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:9200/
# → 000 (connection refused)
```

Guarda la evidencia del test negativo y derriba:

```bash
{ nmap -sS -p 5432,6379,9200 127.0.0.1; echo; curl -sv http://127.0.0.1:9200/ 2>&1; } \
  > ~/pentest-tools/module04/c4-stack/hardened-negative.txt
git -C ~/pentest-tools add module04/c4-stack
git -C ~/pentest-tools commit -m "c4: stack recon + credential chain + hardened negative"
docker compose down -v
```

## 6. Desafíos independientes

Estos no tienen nota en el sentido de que no se provee una solución de referencia — inventas, ejecutas y documentas. Cada uno cae en `~/pentest-tools/module04/independent-<n>/` con un `README.md`, los artefactos y una nota breve de detección.

### 6.1 Audita un docker-compose real por drift de seguridad

Elige cualquier `docker-compose.yml` open-source que de verdad uses — Nextcloud, Gitea, Grafana + Prometheus + Loki self-hosted, Mastodon, lo que tengas en tu home lab. Corre `trivy config` y `docker-bench-security` contra él. Entregables:

- El archivo compose, sin modificar, en el directorio del writeup
- Output de `trivy config` (`.txt`)
- Una lista markdown priorizada de cada finding calificado HIGH o CRITICAL, con:
  - la key/línea de docker-compose que lo causa,
  - una explicación de una línea del riesgo,
  - un extracto parcheado que lo arregla,
  - un re-scan que prueba que el finding se fue.

Comandos de referencia:

```bash
trivy config --severity HIGH,CRITICAL ~/labs/independent-1/docker-compose.yml \
    | tee ~/pentest-tools/module04/independent-1/trivy.txt
docker run --rm --net host --pid host --userns host --cap-add audit_control \
    -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro \
    -v /usr/lib/systemd:/usr/lib/systemd:ro -v /etc:/etc:ro --label docker_bench_security \
    docker/docker-bench-security \
    | tee ~/pentest-tools/module04/independent-1/bench.txt
```

### 6.2 Enumera capabilities de pods en un namespace

Escribe un script chico de Python (`k8s_caps.py`) que:

- Use las credenciales in-cluster o de `~/.kube/config`.
- Tome un namespace como `--ns`.
- Imprima, por pod, por container: image, `securityContext.privileged`, `runAsUser`, `capabilities.add`, `capabilities.drop`, mounts de `hostPath`, `hostNetwork`.
- Emita JSON con `--json` y una tabla en caso contrario.

Esqueleto de referencia — reemplázalo con tu propia implementación:

```python
#!/usr/bin/env python3
import argparse, json, sys
from kubernetes import client, config

def load():
    try:
        config.load_incluster_config()
    except Exception:
        config.load_kube_config()

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--ns", required=True)
    ap.add_argument("--json", action="store_true")
    args = ap.parse_args()
    load()
    v1 = client.CoreV1Api()
    rows = []
    for p in v1.list_namespaced_pod(args.ns).items:
        for c in p.spec.containers:
            sc = c.security_context or client.V1SecurityContext()
            caps = sc.capabilities or client.V1Capabilities()
            rows.append({
                "pod": p.metadata.name, "container": c.name,
                "image": c.image,
                "privileged": bool(sc.privileged),
                "runAsUser": sc.run_as_user,
                "add": caps.add or [], "drop": caps.drop or [],
                "hostNetwork": bool(p.spec.host_network),
                "hostPaths": [v.host_path.path for v in (p.spec.volumes or [])
                              if v.host_path],
            })
    if args.json:
        json.dump(rows, sys.stdout, indent=2)
    else:
        for r in rows:
            print(r)

if __name__ == "__main__":
    main()
```

Entregables: el script, output de muestra contra el cluster de kind del Desafío 2, y una nota corta sobre qué findings empujarías de vuelta en un cluster real.

### 6.3 Forja un JWT contra un endpoint de lab HS256

Levanta una app Flask chica que acepte un JWT firmado con HS256 y devuelva roles distintos según el claim `role`. Usa el secret débil `changeme`. Tu trabajo:

- Loguéate como `alice`, captura el token.
- Usa `jwt_tool` para (a) crackear el secret con una wordlist chica, (b) forjar un token con `role=admin`, (c) pegarle con éxito al endpoint de admin.
- Documenta el fix — rota a RS256, o usa un secret random de 256 bits y guárdalo fuera del source.

Comandos de referencia:

```bash
jwt_tool -h
jwt_tool "$TOKEN" -C -d rockyou.txt
jwt_tool "$TOKEN" -T                       # tamper interactivo
jwt_tool "$TOKEN" -S hs256 -p 'changeme'   # firma un token forjado
```

Entregable: el source de la app vulnerable, el token original capturado, el token forjado y una variante endurecida de la app que rechaza el token forjado. Incluye una transcripción de curl de los dos.

## 7. Rúbrica de calificación

Cada desafío se puntúa sobre 10 en tres ejes; total 120 puntos para los siete entregables (4 de referencia + 3 independientes).

| Eje | 0 pts | 3 pts | 6 pts | 10 pts |
|---|---|---|---|---|
| Correctitud | El exploit no corre | Corre pero falta la prueba | El exploit funciona; prueba capturada | El exploit funciona, es idempotente y se re-corre limpio |
| Documentación | Sin writeup | Solo README | README + comandos + output esperado | Writeup markdown completo con racional y sección de detección |
| Comprensión de detección | No se aborda | Nombra una señal | Provee una regla/query y el match esperado | Regla/query, match testeado y discusión de falsos positivos |

Puntos bonus (hasta +10 en total en el lab):

- Test negativo endurecido commiteado al lado de cada exploit.
- Screenshots o grabaciones de asciinema del exploit de punta a punta.
- Un job de CI en el repo de tools que corre el exploit contra el stack compose del lab y falla si el exploit tiene éxito inesperado sobre un build endurecido.

## 8. Entregar tu trabajo

Cada artefacto de esta lección va a tu repo personal de tooling. Estructura:

```text
~/pentest-tools/
├── module04/
│   ├── c1-docker-sock/
│   │   ├── README.md
│   │   ├── proof.txt
│   │   ├── falco-docker-sock.yaml
│   │   └── hardened-negative.txt
│   ├── c2-k8s-rbac/
│   │   ├── README.md
│   │   ├── victim.yaml
│   │   ├── prod-stripe.txt
│   │   └── no-cluster-secrets.yaml
│   ├── c3-cicd/
│   │   ├── README.md
│   │   ├── .github/workflows/welcome.yml
│   │   ├── event.json
│   │   └── exfil.log
│   ├── c4-stack/
│   │   ├── README.md
│   │   ├── app-users.json
│   │   ├── scan.txt
│   │   └── hardened-negative.txt
│   ├── independent-1/
│   ├── independent-2/
│   └── independent-3/
└── requirements.txt
```

Cada `README.md` sigue el mismo template:

```markdown
# Desafío <n> — <título>

## Objetivo
<un párrafo>

## Setup
<extractos de compose o de manifiesto>

## Reproducción
<comandos numerados y output esperado>

## Detección
<regla / query / señal>

## Endurecimiento
<cambio hecho + evidencia del test negativo>

## Notas
<qué te sorprendió, qué recordar>
```

Commitea y pushea:

```bash
cd ~/pentest-tools
git add module04
git commit -m "feat(m04): container + cluster + cicd + stack recon lab"
# Si tienes un remote privado (recomendado), pushea:
# git push origin main
```

Si todavía no tienes un remote privado, crea uno en el GitLab de tu organización o en una org privada de GitHub — no pushees cadenas de exploit, passwords crackeadas ni API keys filtradas a un repositorio público, ni siquiera desde un lab. El archivo `.secrets` del Desafío 3 y el `app-users.json` del Desafío 4 disparan los dos la push-protection de GitHub y los dos pertenecen solo a un repo controlado.

Asegúrate de que tu `.gitignore` en la raíz del repo excluya archivos sensibles obvios:

```gitignore
.venv/
*.pem
*.key
.secrets
**/*token*
**/exfil.log
```

## 9. Resumen y puntero al Módulo 05

Ahora:

- Escapaste de un container a root en el host abusando de un socket de Docker montado, y probaste el fix.
- Enumeraste y dumpeaste cada Secret de un cluster de Kubernetes desde un ServiceAccount de scope excesivo, y bloqueaste el patrón con una constraint de OPA Gatekeeper.
- Armaste el sink de expresión no confiable de un workflow `pull_request_target` para exfiltrar un token de CI, y reescribiste el workflow para sanitizar el input vía una variable de entorno.
- Hiciste fingerprint de un stack de aplicación de cinco servicios desde afuera, encontraste un Elasticsearch mal configurado, extrajiste credenciales en texto plano y te autenticaste contra la API — después endureciste cada servicio y confirmaste que la cadena se rompe.

Módulo 05 — Enumeration — toma lo que armaste acá y lo industrializa. Vas a reemplazar las invocaciones ad-hoc de curl y nmap del Desafío 4 por un pipeline de enumeración repetible y scripteable. Vas a alimentar el tooling que produjiste en el Módulo 03 (`portscan.py`, `webfuzz.py`) y el inventario de targets que produjiste acá en un solo workflow que recorre una red desconocida desde una IP inicial hasta un grafo completo de assets. Lleva tu repositorio `~/pentest-tools/` y tus archivos compose de lab.

Antes de seguir:

- Snapshot de la VM Kali (`Module-04-lab-clean`).
- Poda Docker (`docker system prune -af --volumes`) para liberar espacio — el cluster de kind y la imagen de Elasticsearch juntos son ~4 GB.
- Lee el prospecto del Módulo 05 y preinstala el tooling que lista.

Nos vemos en el Módulo 05.
