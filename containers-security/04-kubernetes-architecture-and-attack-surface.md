# Lección 04 — Kubernetes Architecture and Attack Surface

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Kubernetes es ahora el Linux moderno. Hace diez años, una prueba de penetración interna significaba dominios Windows, recursos compartidos de archivos y, de vez en cuando, una aplicación web. Hoy, cada empresa mediana opera al menos un clúster, cada startup despliega en uno y cada proveedor cloud ofrece una distribución gestionada. Si en 2026 estás haciendo una evaluación y no te has topado con un API server de Kubernetes en el puerto 6443, no has mirado con suficiente atención.

Esta lección trata Kubernetes como un sistema operativo para cargas de trabajo distribuidas, y no como una herramienta de devops: un sistema operativo con su propio kernel (el API server), su propio sistema de archivos (etcd), su propio proceso init (el controller-manager), su propio scheduler, su propio IPC (los Services) y su propio modelo de privilegios (RBAC y las ServiceAccount). Cada una de estas capas tiene una superficie de ataque documentada, y cada una ha sido explotada en el mundo real.

Al terminar esta lección vas a poder entrar en una evaluación, identificar los componentes del clúster a partir de sondeos de red sin autenticar, enumerar la superficie de RBAC desde un pod comprometido y encadenar malas configuraciones hasta cluster-admin. También vas a poder explicarle a un blue team exactamente qué admission controller te habría bloqueado.

## 1. Kubernetes es el Linux moderno

Todo pentester que trabaja desde 2018 ha visto la misma transición:

- El perímetro se disolvió en service meshes.
- El dominio Windows dejó paso a un plano de control en el puerto 6443.
- La escalada de privilegios local en una máquina fue reemplazada por el escape de un contenedor hacia un nodo.
- Domain Admin fue reemplazado por cluster-admin.

La curva de aprendizaje es real. La buena noticia es que Kubernetes filtra información: la API es descubrible, los objetos se pueden enumerar y los valores por defecto son permisivos. Un pentester con un kubectl que funciona y un buen modelo mental del grafo de objetos puede lograr en una tarde lo que antes llevaba una semana de enumeración de Active Directory.

> **Idea del pentester:** En cada evaluación cloud desde 2022, el hallazgo de mayor impacto ha salido de Kubernetes, no del propio proveedor cloud. IAM suele estar bien cerrado. RBAC, en general, no. Busca primero el clúster.

## 2. Los componentes del clúster en un diagrama

Un clúster de Kubernetes tiene dos planos: el plano de control (el cerebro) y el plano de datos (los workers). Todo camino de ataque pasa por uno de los dos, o por ambos.

```text
                    ┌────────────────────────────────────────────────────┐
                    │                   CONTROL PLANE                    │
                    │                                                    │
   kubectl ────────►│  ┌──────────────┐   ┌──────────┐   ┌────────────┐  │
   (port 6443)      │  │  API Server  │◄─►│   etcd   │   │ Scheduler  │  │
                    │  │  kube-apisrv │   │  :2379   │   │            │  │
                    │  └──────┬───────┘   └──────────┘   └────────────┘  │
                    │         │                                          │
                    │         │           ┌────────────────────────────┐ │
                    │         └──────────►│ Controller-Manager         │ │
                    │                     │ (deployments, endpoints,   │ │
                    │                     │  service accounts, ...)    │ │
                    │                     └────────────────────────────┘ │
                    └─────────────────────┬──────────────────────────────┘
                                          │  TLS on 6443
                    ┌─────────────────────┴──────────────────────────────┐
                    │                                                    │
              ┌─────▼──────┐  ┌────────────┐  ┌────────────┐             │
              │  Node 1    │  │  Node 2    │  │  Node N    │             │
              │  kubelet   │  │  kubelet   │  │  kubelet   │  :10250     │
              │  :10250    │  │  :10250    │  │  :10250    │             │
              │  kube-proxy│  │  kube-proxy│  │  kube-proxy│             │
              │  containerd│  │  containerd│  │  containerd│             │
              │  ┌───────┐ │  │  ┌───────┐ │  │  ┌───────┐ │             │
              │  │ pods  │ │  │  │ pods  │ │  │  │ pods  │ │             │
              │  └───────┘ │  │  └───────┘ │  │  └───────┘ │             │
              └────────────┘  └────────────┘  └────────────┘             │
                                                                         │
                    ┌────────────────────────────────────────────────────┘
                    │                    DATA PLANE
                    └────────────────────────────────────────────────────
```

### 2.1 Componentes del plano de control

| Componente | Puerto | Función | Por qué le importa al pentester |
| --- | --- | --- | --- |
| kube-apiserver | 6443/tcp | Puerta de entrada de cada lectura y escritura | Aquí viven authn, authz y la admisión |
| etcd | 2379/tcp | Almacén clave-valor de cada objeto | Contiene todos los Secrets en texto plano |
| kube-scheduler | 10259/tcp (metrics) | Asigna los pods a los nodos | Casi nunca es alcanzable; prioridad baja |
| kube-controller-manager | 10257/tcp (metrics) | Ejecuta los bucles de reconciliación | Guarda la clave de firma de las ServiceAccount |

### 2.2 Componentes del plano de datos

| Componente | Puerto | Función | Por qué le importa al pentester |
| --- | --- | --- | --- |
| kubelet | 10250/tcp | Ejecuta y vigila los pods del nodo | `/exec`, `/pods`, `/run` = ejecución de código |
| kube-proxy | (iptables/IPVS) | Programa la red del nodo | Camino hacia el movimiento lateral |
| container runtime | (unix socket) | Ejecuta de verdad los contenedores | Objetivo de escape si caes en el nodo |

## 3. Montar un clúster de laboratorio en Kali 2026

Kali 2026 trae `docker.io` y `containerd` en los repositorios por defecto. Para el laboratorio importan tres distribuciones de clúster: kind (Kubernetes-in-Docker, la mejor para experimentos con forma de CI), minikube (respaldada por una VM, la mejor para pruebas a nivel de kernel) y k3s (Rancher, un solo binario, la mejor para labs multinodo realistas).

### 3.1 kind

```bash
# Instala docker (lo exige kind):
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker "$USER"
newgrp docker

# Instala kind:
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Instala kubectl 1.29 (coincide con la imagen de nodo de kind):
curl -LO "https://dl.k8s.io/release/v1.29.4/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Crea un clúster de dos nodos:
cat > kind.yaml <<'YAML'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
YAML
kind create cluster --name pt-lab --config kind.yaml --image kindest/node:v1.29.4

# Verifica:
kubectl get nodes
# NAME                   STATUS   ROLES           AGE   VERSION
# pt-lab-control-plane   Ready    control-plane   40s   v1.29.4
# pt-lab-worker          Ready    <none>          20s   v1.29.4
```

### 3.2 minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker --kubernetes-version=v1.29.4 --nodes=2

kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# minikube       Ready    control-plane   50s   v1.29.4
# minikube-m02   Ready    <none>          30s   v1.29.4
```

### 3.3 k3s

```bash
# Instalación en un solo comando (server + agent en el mismo host):
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.4+k3s1 sh -

# k3s escribe su kubeconfig en un archivo solo de root; cópialo fuera:
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER":"$USER" ~/.kube/config
sed -i 's/127.0.0.1/<your-lab-ip>/' ~/.kube/config

kubectl get nodes
# NAME    STATUS   ROLES                  AGE   VERSION
# kali    Ready    control-plane,master   30s   v1.29.4+k3s1
```

> **Idea del pentester:** Usa kind para trabajo de RBAC y de admission controllers, k3s cuando quieras un clúster mal configurado y realista (sus valores por defecto son más permisivos) y minikube cuando el exploit necesite un kernel Linux real con sus propios módulos cargados.

## 4. El API server: la puerta de entrada

Toda petición que cambia un clúster pasa por kube-apiserver en `https://<master>:6443`. Es el único componente que habla con etcd, así que si no logras pasarlo no puedes cambiar el clúster. A la inversa, si lo pasas como cluster-admin, todo lo demás es un trámite.

El API server ejecuta tres etapas en cada petición, en orden:

```text
   Petición ──► [ AUTHN ] ──► [ AUTHZ ] ──► [ ADMISSION ] ──► etcd
              ¿quién eres?  ¿puedes hacerlo?  ¿está permitido?
```

### 4.1 Autenticación

Kubernetes no tiene usuarios propios. Acepta identidad desde cualquiera de estos plugins:

- Certificados de cliente X.509 (el valor por defecto de kubectl en clústeres kubeadm)
- Bearer tokens (JWT de ServiceAccount, tokens estáticos)
- OIDC (Dex, Okta, Azure AD, Google)
- Webhook token authn (servicios propios)
- Bootstrap tokens (un kubelet que se une a un clúster)

Las peticiones anónimas reciben el nombre de usuario `system:anonymous` y el grupo `system:unauthenticated`. Esto está habilitado por defecto en el API server (`--anonymous-auth=true`) y, de forma desastrosa, a menudo se deja habilitado también en el kubelet.

### 4.2 Autorización

Una vez autenticada, la petición se evalúa contra uno o más módulos de authz:

- RBAC: el valor por defecto moderno, cubierto en la sección 9
- Node: un autorizador especial que limita a los kubelets a los objetos de su propio nodo
- Webhook: decisiones de política externas (poco frecuente)
- ABAC: legado; evítalo

### 4.3 Admisión

Si authz dice que sí, los admission controllers se ejecutan en orden. Pueden mutar el objeto (admisión mutante) o rechazarlo (admisión validadora). Aquí viven PodSecurity, OPA/Gatekeeper y Kyverno.

```bash
# Sondea un API server sin credenciales:
curl -sk https://<master>:6443/version
# {
#   "major": "1",
#   "minor": "29",
#   "gitVersion": "v1.29.4",
#   ...
# }

# Listado anónimo de namespaces (será 403 si RBAC está bien):
curl -sk https://<master>:6443/api/v1/namespaces
# {"kind":"Status","status":"Failure","message":"namespaces is forbidden:
#  User \"system:anonymous\" cannot list resource ...","code":403}

# Si aquí ves 200, ya ganaste:
curl -sk https://<master>:6443/api/v1/namespaces/kube-system/secrets
```

## 5. etcd: la joya de la corona

etcd guarda cada objeto de Kubernetes como un valor serializado bajo una clave bien conocida. Los Secrets se guardan del mismo modo y, por defecto, se almacenan sin cifrar en reposo. Quien pueda leer el almacén de etcd puede leer cada Secret del clúster.

etcd escucha en el puerto 2379/tcp (cliente) y 2380/tcp (peer). En un clúster kubeadm escucha solo en `127.0.0.1` y exige mTLS. En un clúster autogestionado a menudo escucha en `0.0.0.0` y los certificados están en `/etc/kubernetes/pki/etcd/` a disposición de cualquiera con acceso al nodo.

### 5.1 Enumerar etcd

```bash
# En el nodo del plano de control:
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -20
# /registry/apiregistration.k8s.io/apiservices/v1.
# /registry/apiregistration.k8s.io/apiservices/v1.admissionregistration.k8s.io
# /registry/clusterrolebindings/cluster-admin
# /registry/configmaps/kube-system/coredns
# /registry/secrets/default/default-token-abcde
# ...
```

### 5.2 Sacar un Secret directamente

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/kube-system/bootstrap-token-abcdef
# k8s
# v1
# Secret
# ...
# token-id
# abcdef
# token-secret
# 0123456789abcdef
```

Ese bootstrap token, si sigue siendo válido, te permite unir un nodo ilegítimo: mira la sección 6.

### 5.3 Cifrado en reposo

La corrección es `EncryptionConfiguration`, que se pasa al API server mediante `--encryption-provider-config`. En 2026 el proveedor recomendado es kms v2 con un KMS externo. Cualquier configuración que use `identity` como primer proveedor no está cifrada, aunque después venga `aescbc`.

> **Idea del pentester:** Cuando consigas una shell en un nodo del plano de control, vuelca siempre `/etc/kubernetes/manifests/kube-apiserver.yaml`. Si referencia `--encryption-provider-config`, lee ese archivo. Si el primer proveedor es `identity`, dile al cliente que cada Secret de etcd está en texto plano.

## 6. El kubelet en cada nodo

El kubelet es el agente que habla con el runtime de contenedores en cada nodo. Escucha en 10250/tcp las peticiones de API del plano de control y, históricamente, de cualquiera que pudiera alcanzarlo. Expone los endpoints de abajo:

| Endpoint | Propósito |
| --- | --- |
| `/pods` | JSON de cada pod del nodo |
| `/exec/{ns}/{pod}/{container}` | Shell interactiva en un contenedor |
| `/run/{ns}/{pod}/{container}` | Comando de disparar y olvidar |
| `/attach/{ns}/{pod}/{container}` | Se engancha al pid 1 en ejecución |
| `/portForward/{ns}/{pod}` | Reenvía un puerto TCP |
| `/metrics` | Métricas de Prometheus |
| `/logs/` | Sistema de archivos del nodo (sí, de verdad, si el webhook está apagado) |

### 6.1 Sondear un kubelet expuesto

```bash
# Listado anónimo de pods (devuelve JSON si --anonymous-auth=true):
curl -sk https://<node>:10250/pods | jq '.items[].metadata.name'
# "kube-proxy-abcde"
# "coredns-76f75df574-xyzab"
# "victim-app-6b7d5b7b5b-qrstu"

# Ejecución anónima de un comando en un contenedor:
curl -sk https://<node>:10250/run/default/victim-app-6b7d5b7b5b-qrstu/app \
  -d "cmd=id"
# uid=0(root) gid=0(root) groups=0(root)
```

### 6.2 TLS bootstrap tokens

Los nodos nuevos se unen al clúster presentando un bootstrap token al API server. El token tiene la forma `abcdef.0123456789abcdef` y vive como un Secret en `kube-system`. Si puedes leer Secrets ahí, puedes enrolar un nodo ilegítimo y recibir credenciales de kubelet.

```bash
# Lista los bootstrap tokens (hace falta lectura de Secret en kube-system):
kubectl -n kube-system get secrets \
  --field-selector type=bootstrap.kubernetes.io/token \
  -o jsonpath='{range .items[*]}{.data.token-id}{"."}{.data.token-secret}{"\n"}{end}' \
  | while read b64; do echo "$b64" | base64 -d; done
# abcdef.0123456789abcdef
```

### 6.3 Corregir el kubelet

La configuración correcta desactiva la autenticación anónima y exige autorización por webhook:

```yaml
# /var/lib/kubelet/config.yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
```

## 7. kube-proxy y la red del clúster

kube-proxy programa la pila de red del nodo para que una petición a la IP de un Service llegue a uno de los pods que lo respaldan. Dominan dos implementaciones:

- Modo iptables: el valor por defecto. kube-proxy escribe cadenas `KUBE-SERVICES`. Simple, lento cuando hay muchos Services.
- Modo IPVS: balanceador de carga a nivel de kernel. Más rápido, necesario a escala.

### 7.1 CIDR

Todo clúster tiene al menos dos CIDR:

- Pod CIDR (por ejemplo `10.244.0.0/16`): una IP por pod, enrutable dentro del clúster.
- Service CIDR (por ejemplo `10.96.0.0/12`): IP virtuales balanceadas por kube-proxy.

```bash
# Descubre los CIDR:
kubectl cluster-info dump | grep -E "service-cluster-ip-range|cluster-cidr"
# "--service-cluster-ip-range=10.96.0.0/12"
# "--cluster-cidr=10.244.0.0/16"

# Desde dentro de cualquier pod:
cat /etc/resolv.conf
# nameserver 10.96.0.10       <-- kube-dns Service IP
# search default.svc.cluster.local svc.cluster.local cluster.local
```

### 7.2 Qué hace el pentester con el CIDR

Desde un pod comprometido, escanear el pod CIDR revela cada uno de los demás pods del clúster. Escanear el service CIDR revela cada Service. Como la mayoría de los clústeres no tienen NetworkPolicy (mira la sección 11), los pods pueden hablar entre sí con libertad.

```bash
# Dentro de un pod en 10.244.0.0/16:
apk add --no-cache nmap
nmap -sT -p 6379,3306,5432,8080,9200 --open 10.244.0.0/16
```

## 8. Objetos: la superficie de ataque

Kubernetes expone unos 50 kinds de objeto integrados, más cada CRD que el clúster tenga instalada. Los que importan en cada evaluación:

| Kind | Qué es | Por qué lo miras |
| --- | --- | --- |
| Namespace | Aislamiento lógico | El reconocimiento empieza aquí |
| Pod | Un grupo de contenedores | Donde corre el código |
| Deployment | Controlador de pods | Objetivo del envenenamiento del pipeline |
| Service | VIP balanceada | Mapa de alcanzabilidad |
| Ingress | Router HTTP externo | Superficie de ataque externa |
| Secret | Blob de credenciales en Base64 | Botín |
| ConfigMap | Configuración no sensible, que a menudo igual tiene secrets | Botín |
| ServiceAccount | Identidad del pod | Pivote |
| Role / ClusterRole | Conjunto de permisos | Entender el poder |
| RoleBinding / ClusterRoleBinding | Ata los permisos a identidades | Camino de escalada |
| NetworkPolicy | Firewall L3/L4 | Poco frecuente en la práctica |

```bash
# El barrido de reconocimiento en un solo comando:
kubectl get all,secrets,cm,ing,networkpolicy,sa,role,rolebinding \
  -A -o wide 2>/dev/null | less
```

## 9. RBAC: Roles, ClusterRoles y bindings

RBAC es el modelo de mínimo privilegio de Kubernetes. Cuatro kinds de objeto:

- **Role**: verbos (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`) sobre recursos dentro de un namespace.
- **ClusterRole**: lo mismo, pero a nivel de clúster (nodes, PV, CRD con ámbito de clúster) o aplicado a cualquier namespace mediante un RoleBinding.
- **RoleBinding**: ata un Role o un ClusterRole a un subject dentro de un namespace.
- **ClusterRoleBinding**: ata un ClusterRole a un subject en todo el clúster.

Los subjects son User, Group o ServiceAccount.

### 9.1 Dónde falla RBAC

Los modos de fallo son siempre los mismos:

- Comodines. `resources: ["*"]`, `verbs: ["*"]` es un cluster-admin disfrazado.
- El verbo `escalate` sobre roles/clusterroles permite que una cuenta de privilegio bajo se escriba un rol más grande.
- El verbo `bind` sobre roles/clusterroles permite que la cuenta se ate a sí misma un rol poderoso que ya existe.
- El verbo `impersonate` sobre users, groups y serviceaccounts permite que la cuenta se convierta en cualquiera.
- `create` sobre pods, combinado con hostPaths escribibles en el nodo, permite que la cuenta lea archivos de cualquier nodo.
- `create` sobre `pods/exec` permite que la cuenta ejecute comandos en cualquier pod de un namespace.
- `get` sobre secrets en un namespace que contiene un token de ServiceAccount de un rol más poderoso.

### 9.2 Enumerar tus propios permisos

```bash
# Desde dentro de un pod o con un kubeconfig:
kubectl auth can-i --list
# Resources                                       Non-Resource URLs   Resource Names   Verbs
# selfsubjectreviews.authentication.k8s.io        []                  []               [create]
# selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
# ...                                             [/healthz]          []               [get]
# secrets                                         []                  []               [get list]
# pods                                            []                  []               [get list create]
# pods/exec                                       []                  []               [create]
```

Cualquier línea con `pods/exec [create]` en un namespace que ejecuta cargas privilegiadas es una línea recta hacia cluster-admin.

### 9.3 Enumerar los permisos de los demás

```bash
# Instala kubectl-who-can:
kubectl krew install who-can

kubectl who-can create pods -A
kubectl who-can '*' '*' -A          # subjects con cualquier permiso
kubectl who-can escalate clusterroles
```

## 10. ServiceAccount: cada pod tiene una

Cada pod queda asociado automáticamente a una ServiceAccount. Salvo que el Deployment opte por no hacerlo con `automountServiceAccountToken: false`, el sistema de archivos del pod contiene un JWT firmado que lo autentica ante el API server:

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── ca.crt        <-- certificado de la CA del clúster para confiar en el API server
├── namespace     <-- archivo de texto con el namespace del pod
└── token         <-- JWT, válido ~1 h, rotado automáticamente por el kubelet
```

### 10.1 Usar el token desde un pod

```bash
# Las tres variables que necesita todo exploit de Kubernetes:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

# ¿Quién soy?
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/apis/authentication.k8s.io/v1/tokenreviews \
  -X POST -H 'Content-Type: application/json' \
  -d "{\"kind\":\"TokenReview\",\"apiVersion\":\"authentication.k8s.io/v1\",
       \"spec\":{\"token\":\"$TOKEN\"}}"

# ¿Qué puedo hacer?
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/apis/authorization.k8s.io/v1/selfsubjectrulesreviews \
  -X POST -H 'Content-Type: application/json' \
  -d "{\"kind\":\"SelfSubjectRulesReview\",\"apiVersion\":\"authorization.k8s.io/v1\",
       \"spec\":{\"namespace\":\"$NS\"}}"
```

### 10.2 El token por defecto es peligroso cuando está mal vinculado

El hallazgo crítico más común en evaluaciones reales es un ClusterRoleBinding de `cluster-admin` hacia `system:serviceaccount:default:default`. Quien controle cualquier pod del namespace `default` es cluster-admin. Mira la sección 11.

## 11. Malas configuraciones habituales: el top 10 del pentester

### 11.1 Autenticación anónima en el API server o en el kubelet

Los valores de fábrica del API server permiten el acceso anónimo. RBAC debería detener a `system:anonymous`, pero a menudo no lo hace. `kubelet --anonymous-auth=true` es peor: ejecución de código directa.

```bash
curl -sk https://<master>:6443/api/v1/nodes | head -5
curl -sk https://<node>:10250/pods | head -5
```

### 11.2 ClusterRoleBinding demasiado amplio

```bash
# Busca cluster-admin vinculado a cualquier cosa preocupante:
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") |
         "\(.metadata.name)  ->  \(.subjects // [] | map(.kind+":"+(.namespace//"-")+"/"+.name) | join(", "))"'
# cluster-admin      ->  Group:-/system:masters
# ci-admin           ->  ServiceAccount:default/default   <-- CRITICAL
# oops               ->  Group:-/system:anonymous          <-- CRITICAL
```

### 11.3 Dashboard expuesto sin autenticación

El Kubernetes Dashboard a menudo se despliega con `--enable-skip-login` o con un ClusterRoleBinding de `cluster-admin` hacia su ServiceAccount. Si puedes llegar a `https://<host>:<port>/#/overview` y pulsar Skip, el clúster es tuyo.

### 11.4 Pods privilegiados, hostPID, hostNetwork

```yaml
# El pod de pesadilla:
spec:
  hostPID: true
  hostNetwork: true
  hostIPC: true
  containers:
    - name: shell
      image: alpine
      securityContext:
        privileged: true
      volumeMounts:
        - name: host
          mountPath: /host
  volumes:
    - name: host
      hostPath:
        path: /
```

Cualquier pod que coincida con ese spec = root en el nodo = los tokens de todos los demás pods.

### 11.5 hostPath montados en las cargas de trabajo

Busca `/`, `/var/run/docker.sock`, `/var/run/containerd/containerd.sock`, `/etc/kubernetes` y `/var/lib/kubelet` montados dentro de un contenedor. Cualquiera de ellos es un camino directo para salir del pod.

```bash
kubectl get pods -A -o json | jq -r '
  .items[] | . as $p |
  (.spec.volumes // [])[] |
  select(.hostPath) |
  "\($p.metadata.namespace)/\($p.metadata.name)  hostPath=\(.hostPath.path)"'
```

### 11.6 etcd sin cifrar

Cubierto en la sección 5. Confírmalo con `grep encryption-provider-config /etc/kubernetes/manifests/kube-apiserver.yaml` en un nodo del plano de control.

### 11.7 NetworkPolicy débil o ausente

Kubernetes no trae ninguna NetworkPolicy por defecto. Cada pod puede alcanzar a cada otro pod en cada puerto. Pídele al cliente el recuento de objetos NetworkPolicy:

```bash
kubectl get networkpolicy -A --no-headers | wc -l
# 0
```

### 11.8 Secrets guardados como variables de entorno en claro

`env.valueFrom.secretKeyRef` está bien; `env.value: "supersecret"` no. Cualquier descripción del pod lo filtra.

```bash
kubectl get pods -A -o json | jq -r '
  .items[] | . as $p | (.spec.containers[]?) |
  (.env // [])[] |
  select(.value != null and (.name | test("(?i)pass|token|key|secret"))) |
  "\($p.metadata.namespace)/\($p.metadata.name)  \(.name)=\(.value)"'
```

### 11.9 Imágenes vulnerables

Imágenes base viejas con glibc, log4j u openssl sin parchear. Trivy o Grype en el pipeline; del lado del pentest, `kubectl get pods -A -o json | jq '.items[].spec.containers[].image'` y crúzalo.

### 11.10 IMDS del cloud alcanzable desde los pods

En EKS, GKE y AKS, el Instance Metadata Service del proveedor cloud (`169.254.169.254`) es alcanzable desde cada pod por defecto, salvo que se bloquee de forma explícita. Un pod que pueda hablar con el IMDS puede robar el rol IAM del nodo. La consulta al servicio no se incluye.

## 12. Herramientas de enumeración

### 12.1 kubectl auth can-i

```bash
kubectl auth can-i --list -n kube-system
kubectl auth can-i create pods -n dev
# yes
kubectl auth can-i '*' '*' -A
# no
```

### 12.2 kube-hunter

```bash
pipx install kube-hunter
kube-hunter --remote <master-ip>
# Kubernetes Hunter found the following vulnerabilities:
# ID    LOCATION                             CATEGORY               VULNERABILITY
# KHV002  <master>:6443     Information Disclosure   K8s Version Disclosure
# KHV005  <node>:10250      Remote Code Execution    Anonymous Authentication
# KHV050  <master>:6443     Access Risk              Read access to K8s secrets
```

### 12.3 kube-bench

CIS Kubernetes Benchmark. Ejecútalo en cada nodo del clúster:

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -f job/kube-bench
# [FAIL] 1.2.16 Ensure that the --profiling argument is set to false
# [FAIL] 1.2.19 Ensure that the --audit-log-path argument is set
# [WARN] 4.2.6 Ensure that the --protect-kernel-defaults argument is set to true
```

### 12.4 peirates

Menú interactivo de post-explotación: corre dentro de un pod comprometido y ofrece dump-secrets, ADFS-token-steal, kubelet-abuse e IMDS-abuse. Perfecto para demos.

### 12.5 botb (Break out the Box)

Suite de escape de contenedores. Detecta montajes hostPath, flags privileged, sockets de docker, la clase CVE-2022-0492 y las clases CVE-2024-XXXX.

## 13. Camino de ataque 1: de un pod comprometido a cluster-admin

```text
pod comprometido  ──► kubelet en el nodo  ──► cada pod del nodo  ──► API server ──► cluster-admin
```

### 13.1 Qué consigue el camino

Si el kubelet del nodo acepta peticiones anónimas, quien ya está dentro de un pod puede listar los demás pods de ese nodo y pedir la ejecución de un comando en uno que monte un token de ServiceAccount más potente. Con ese token, el API server responde como cluster-admin. El procedimiento no se incluye. La detección está en la sección 16 y el bloqueo, en el laboratorio de la sección 17.

## 14. Camino de ataque 2: de un kubeconfig filtrado a la toma del cloud

```text
kubeconfig filtrado ──► API server ──► secret de etcd ──► IAM cloud ──► admin de la cuenta cloud
```

### 14.1 Qué consigue el camino

Un kubeconfig de desarrollador que aparece en un bucket público, en un log de GitHub Actions o en un repositorio suele poder leer Secrets. Esos Secrets a veces contienen credenciales de IAM del cloud, y con ellas la cuenta cloud queda al alcance. El volcado y el uso de esas credenciales no se incluyen. El fallo habitual es el token de desarrollador con `get secrets` en todos los namespaces.

## 15. Camino de ataque 3: envenenamiento de CI/CD

```text
commit git de bajo privilegio ──► la ServiceAccount de CI/CD tiene cluster-admin ──► manifiesto envenenado ──► cluster-admin
```

### 15.1 Montaje habitual

El sistema de CI/CD despliega con una ServiceAccount que tiene cluster-admin, porque el equipo de plataforma no quiso pelearse con RBAC. El token de esa ServiceAccount vive en un Secret. Quien pueda disparar un build puede hacer `kubectl apply` de lo que sea.

### 15.2 Qué consigue el camino

Un commit de bajo privilegio cambia un archivo de values de Helm. El pipeline, que aplica manifiestos con una ServiceAccount de cluster-admin, crea un ClusterRoleBinding permanente. No hace falta ejecutar código dentro de una carga de trabajo. El manifiesto no se incluye.

### 15.3 Corrección

- Fija la ServiceAccount de CI en `edit` dentro de namespaces concretos, no en cluster-admin para todo el clúster.
- Exige commits firmados y políticas de Kyverno/OPA que bloqueen la creación de ClusterRoleBinding desde CI.
- Aplica una política de imágenes: solo se admiten imágenes del registry de la organización.

## 16. Detección

### 16.1 Audit logs

El API server registra cada decisión de authz si está configurado:

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
```

Apunta el API server hacia esa política:

```text
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
```

### 16.2 Falco

Detección en tiempo de ejecución basada en reglas de syscall. Trae políticas para:

- Shell dentro de un contenedor
- Escritura por debajo de `/etc`
- Lectura de un token de ServiceAccount desde un pod inesperado
- Contacto con el IMDS

```bash
helm install falco falcosecurity/falco -n falco --create-namespace
kubectl -n falco logs -l app.kubernetes.io/name=falco -f
# 22:14:03.000000000: Warning Sensitive file opened for reading by non-trusted program
```

### 16.3 kubectl-who-can

Introspección de RBAC rápida y enfocada: apuntada a cualquier verbo/recurso, te dice cada subject que puede hacerlo. Los blue teams deberían tener un informe nocturno de los 20 verbos peligrosos principales.

### 16.4 Admission controllers

OPA/Gatekeeper y Kyverno son las dos opciones de producción en 2026. Detienen los objetos malos antes de que lleguen a etcd.

```yaml
# Kyverno: bloquea pods privilegiados
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: disallow-privileged
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

## 17. Laboratorio práctico

**Objetivo:** levanta un clúster kind, deja a propósito un ClusterRoleBinding de cluster-admin hacia la ServiceAccount `default`, instala una política de Kyverno que bloquee ese patrón y demuestra que recrear el binding ya no funciona. La escalada desde dentro del pod y el volcado de Secrets no se incluyen.

### 17.1 Clúster y carga de trabajo

```bash
kind create cluster --name attack-lab --image kindest/node:v1.29.4

# A propósito mal configurado: vincula cluster-admin a default:default (NO HAGAS ESTO EN PRODUCCIÓN)
kubectl create clusterrolebinding oops \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default

```

El fuente sigue con un pod que monta el token de esa ServiceAccount y, desde dentro, confirma `allowed: true`, vuelca los Secrets y crea otro ClusterRoleBinding. Esa escalada no se incluye. El laboratorio que sí queda es el control: la misma mala configuración tiene que ser rechazada por Kyverno.

### 17.2 Instalar Kyverno y bloquear el patrón

```bash
# De vuelta en el host:
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl -n kyverno wait --for=condition=Available deploy --all --timeout=180s

# Política: prohíbe ClusterRoleBinding cuyos subjects sean ServiceAccount default hacia roles poderosos.
cat <<'YAML' | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: forbid-default-sa-cluster-admin
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: forbid-default-sa-cluster-admin
      match:
        any:
          - resources:
              kinds: ["ClusterRoleBinding"]
      validate:
        message: "Do not bind cluster-admin to a default ServiceAccount."
        deny:
          conditions:
            all:
              - key: "{{ request.object.roleRef.name }}"
                operator: Equals
                value: "cluster-admin"
              - key: "default"
                operator: AnyIn
                value: "{{ request.object.subjects[?kind=='ServiceAccount'].name }}"
YAML
```

### 17.3 Confirma que el binding ahora está bloqueado

```bash
# Borra primero el binding malicioso:
kubectl delete clusterrolebinding oops

# Intenta recrearlo: Kyverno lo rechaza:
kubectl create clusterrolebinding oops \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
# Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
# resource ClusterRoleBinding/oops was blocked due to the following policies:
# forbid-default-sa-cluster-admin:
#   forbid-default-sa-cluster-admin: Do not bind cluster-admin to a default ServiceAccount.
```

### 17.5 Desmontar

```bash
kind delete cluster --name attack-lab
```

## 18. Ejercicios

### 18.1 Guiados

- **Enumera un clúster ajeno.** Arranca un nodo k3s y, desde tu estación de trabajo, ejecuta `kube-hunter --remote <ip>` en modo pasivo. Explica cada hallazgo y mapéalo a una sección de esta lección.
- **Lee etcd desde disco.** En una instalación de laboratorio kubeadm, detén el static pod de etcd, copia `/var/lib/etcd/member/snap/db` fuera del nodo y usa `etcd-dump-db` para listar las claves. Confirma si los Secrets están cifrados.
- **Encadena un montaje HostPath.** Despliega un pod con `hostPath: /var/lib/kubelet` montado en lectura-escritura y, desde dentro, vuelca cada token de ServiceAccount cacheado en el nodo.

### 18.2 Independientes

- Construye un RBAC mínimo que permita a un administrador de namespace hacer de todo excepto crear ClusterRoleBindings, montar hostPaths o poner `privileged: true`. Verifícalo con `kubectl auth can-i`.
- Escribe una política de Kyverno que bloquee cualquier pod que intente montar `169.254.169.254` mediante `hostAliases` o un canal del lado de `hostNetwork`, y demuestra que un pod ingenuo de abuso de IMDS es rechazado.
- Dado un clúster en marcha y solo acceso anónimo a `https://<master>:6443/version`, redacta un informe de cada conclusión que puedas sacar sobre el clúster (distribución, versión, nivel de parche, cloud probable, CNI probable) y de cómo la sacaste. No uses kubectl.

## 19. Resumen y lección siguiente

Kubernetes es un sistema operativo distribuido y, como cualquier SO, tiene una ABI (el API server), un sistema de archivos (etcd), un init (el controller-manager), un scheduler y un modelo de privilegios (RBAC). Todo camino de ataque tiene una de tres formas:

- Llegar a un pod y escalar a través de su ServiceAccount.
- Llegar al plano de control y leer etcd.
- Llegar al pipeline que llega al plano de control.

Toda defensa tiene una de cuatro formas:

- Apagar lo anónimo.
- Apretar RBAC.
- Cifrar etcd, restringir los montajes del host y hacer cumplir NetworkPolicy.
- Desplegar un admission controller que diga que no.

Ya tienes un modelo mental del clúster, un laboratorio que puedes reconstruir en tres comandos y un script de enumeración repetible.

Lección siguiente: Lección 05 — Container Escapes and Kernel Boundaries, donde bajamos del clúster al runtime de contenedores, cubrimos las familias de CVE (carreras de symlink de runc, `release_agent` de cgroups v1, gid de overlayfs) y salimos de un pod endurecido hacia el nodo.
