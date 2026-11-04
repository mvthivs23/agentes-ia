# Lección 11 — API Gateway and Authentication Architecture

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 2026 · Red Team Leaders

## Introducción

Las APIs modernas casi nunca están protegidas por un solo control. Delante de la aplicación se sienta un API gateway que impone authentication, delante de eso un Web Application Firewall (WAF) inspecciona payloads, detrás un identity provider emite tokens de OAuth que viajan dentro de JSON Web Tokens (JWTs), y los rate limiters frenan a los callers abusivos por el camino. Cada capa está diseñada para ser fuerte por sí sola, pero las suposiciones que cada capa hace sobre las otras crean las costuras en las que vive un pentester.

Esta lección recorre el stack completo de afuera hacia adentro — gateway, después protocolo de autorización, después formato de token, después session — y para cada capa muestra la clase de bypass, la tool que lo ejercita y la mitigación que por fin lo cierra. Al final vas a poder razonar sobre un request como lo hace un atacante: qué control lo vio, qué control confió a ciegas en el anterior y qué capa contiene la falla que cierra el engagement con un token de admin en tus manos.

## 1. Introducción — una pila, no un muro

Un request de API de producción en 2026 típicamente cruza al menos seis fronteras de confianza distintas antes de llegar a la lógica de negocio.

```text
Client ──▶ CDN/WAF ──▶ Load Balancer ──▶ API Gateway ──▶ Service Mesh ──▶ Microservice ──▶ Database
              │             │                  │                │              │
              │             │                  │                │              └─ row-level auth
              │             │                  │                └─ mTLS peer identity
              │             │                  └─ token introspection, rate limit, transform
              │             └─ TLS termination, header stripping
              └─ pattern matching, bot scoring, geo filtering
```

Cada salto reescribe, tira o agrega headers. Cada salto toma una decisión de autorización o la delega. Cada salto confía en el salto anterior de forma implícita para al menos un dato — casi siempre la identidad del cliente, la IP del cliente o el hecho de que el request ya fue «limpiado». El trabajo de un pentester es encontrar la única frontera donde la confianza está mal y después empujar un request a través de ella que la capa de atrás nunca debía ver.

> **Pentester Insight:** Nunca modeles la seguridad de una API como el control más fuerte. Modélala como el hand-off más débil entre dos controles. Un WAF perfecto delante de un gateway que confía en `X-User-Id` de cualquier origen no está defendido — está instrumentado para el atacante.

## 2. El paisaje del API gateway — el patrón de cuello de botella

Un API gateway es un reverse proxy especializado en tráfico de API. Ofrece un único ingress delante de muchos servicios de backend y centraliza preocupaciones transversales para que cada microservicio no tenga que reimplementarlas.

### 2.1 Productos de gateway habituales

| Gateway | Deployment | Lenguaje principal | Dónde lo encuentras |
|---|---|---|---|
| Kong | Self-hosted, Konnect SaaS | Lua sobre OpenResty/nginx | Empresas con muchos equipos |
| AWS API Gateway | Managed (REST, HTTP, WebSocket) | Cerrado | Shops nativos de AWS, backends de Lambda |
| Google Apigee | Managed | Cerrado | Industrias reguladas, APIs monetizadas |
| Envoy | Self-hosted, service mesh | C++ | Istio, AWS App Mesh, Consul Connect |
| Traefik | Self-hosted, nativo de Kubernetes | Go | Clusters de Docker/K8s, arranques rápidos |
| Netflix Zuul | Self-hosted (legacy) | Java/Groovy | Spring Cloud, estates más viejos |
| Tyk | Self-hosted, SaaS | Go | Mid-market, multi-región |

### 2.2 El patrón de cuello de botella

El gateway es un único punto de ingress donde cae cada request antes de tocar la lógica de negocio. Esa concentración es su valor y su pasivo.

```text
                        ┌─────────────┐
       Public Traffic ──▶│   Gateway   │──▶  users-service (10.0.1.10:8080)
                        │             │──▶  orders-service (10.0.1.11:8080)
                        │  authn      │──▶  billing-service (10.0.1.12:8080)
                        │  authz      │──▶  admin-service  (10.0.1.13:8080)
                        │  rate-limit │
                        │  transform  │
                        └─────────────┘
```

El gateway es el único host al que internet se supone que llega. Cada backend escucha en una dirección RFC 1918 dentro de una VPC o de una red de cluster de Kubernetes. Si un pentester puede hablar directo con cualquier backend, los controles del gateway son irrelevantes para esa llamada.

### 2.3 Enumerar el gateway

```bash
# Fingerprint del gateway desde los headers de response:
curl -sI https://api.example.com/ | grep -Ei 'server|via|x-kong|x-envoy|x-amzn|x-apigee'
# Server: kong/3.7.1
# X-Kong-Proxy-Latency: 3
# Via: kong/3.7.1

# Firma de AWS API Gateway:
# x-amzn-RequestId: e2f4c5b0-1234-5678-9abc-def012345678
# x-amz-apigw-id: aBcDeFgHiJkLmNo=

# Firma de Envoy:
# server: envoy
# x-envoy-upstream-service-time: 12

# Firma de Apigee:
# X-Powered-By: Apigee
# Via: 1.1 google
```

## 3. Policies habituales del gateway

Cada gateway ofrece más o menos el mismo catálogo de policies. Los nombres difieren; la semántica no.

### 3.1 Authentication (authn)

El gateway verifica la identidad del caller antes de reenviar. Mecanismos habituales:

- **API key:** el caller envía `X-API-Key: ak_live_...`; el gateway la busca en un store de keys.
- **Bearer token de OAuth 2.0:** el caller envía `Authorization: Bearer eyJhbGc...`; el gateway hace introspection o valida en local.
- **mTLS:** el caller presenta un certificado de cliente X.509 durante el handshake TLS.
- **Firma HMAC:** el caller firma el request con un secret compartido; el gateway recalcula y compara.

### 3.2 Authorization (authz)

Una vez conocida la identidad, el gateway decide si esa identidad puede realizar la action pedida. Esto suele ser una de:

- **Chequeo de scope:** ¿el token incluye el scope `orders:write`?
- **Chequeo de role:** ¿el caller pertenece a un group como `admin`?
- **Attribute-based access control (ABAC):** un motor de policy (OPA, Cedar) evalúa un conjunto de reglas.

### 3.3 Rate limiting

Acota cuántos requests puede emitir un caller por ventana de tiempo. Ver §14 para la taxonomía de algoritmos y sus bypasses.

### 3.4 Transformación de request / response

El gateway reescribe headers, paths y bodies:

```yaml
# Plugin de ejemplo de Kong: quita un header suministrado por el caller e inyecta uno de confianza.
plugins:
  - name: request-transformer
    config:
      remove:
        headers:
          - X-User-Id            # no confiable, viene del cliente
      add:
        headers:
          - X-User-Id:$(jwt.sub) # de confianza, derivado de un JWT validado
```

La transformación es donde la autenticidad de «quién es este caller» se materializa en un header que el backend puede leer barato. Este mismo patrón es también un vector de bypass — ver §4.

### 3.5 Terminación de mTLS

El gateway típicamente termina TLS desde internet y restablece una session nueva (a menudo mTLS) hacia el backend. El upstream ahora ve la identidad del gateway, no la del caller.

## 4. Bypass del gateway

El gateway solo es un control si cada request que toca un backend de verdad lo cruzó. Tres patrones vuelven falsa esa suposición.

### 4.1 Llegar al backend directo

Si el backend está expuesto por IP o por DNS interno, el gateway se vuelve opcional. El hallazgo clásico es un backend que confía en que el gateway ya autenticó y por eso no autentica. Llegar al servicio directo salta cada policy del gateway. El barrido no se incluye.

### 4.2 Confianza en un header autoritativo

Un gateway a menudo inyecta un header como `X-User-Id`, `X-Consumer-Id` o `X-Authenticated-User` después de un authn exitoso, y el backend usa ese header verbatim para identificar al caller.

Si el backend no verifica de dónde vino el header, quien llega al backend directo, o quien pasa por un gateway que reenvía el header del cliente, se presenta como el usuario que escribió en `X-User-Id`. Las peticiones no se incluyen.

La mitigación es el patrón request-transformer de la sección 3.4: el gateway quita cualquier `X-User-Id` entrante y vuelve a agregar su propio valor. Los backends rechazan el header si no viene de la IP interna del gateway o del certificado de cliente.

### 4.3 Path traversal en el matching de rutas

Los gateways matchean paths contra patrones de ruta. Si el gateway y el backend normalizan `..` y la barra codificada de forma distinta, una ruta que el gateway trata como pública llega al backend como ruta privilegiada. Envoy CVE-2021-29492, issues de path traversal de Zuul y varios CVEs de rutas regex de Kong son de esta clase. Las variantes de codificación no se incluyen.

## 5. Flujos de OAuth 2.0 en profundidad

OAuth 2.0 es un protocolo de delegación de autorización definido en RFC 6749 y refinado por la consolidación «OAuth 2.1». Define cómo un resource owner (el usuario humano) le otorga a un client (una aplicación) acceso limitado a un resource server (la API), mediado por un authorization server (el identity provider).

### 5.1 Authorization Code con PKCE (el flujo default en 2026)

```text
Browser (user)        Client App          Auth Server              Resource Server
     │                    │                    │                          │
     │  1. click login    │                    │                          │
     │───────────────────▶│                    │                          │
     │                    │ 2. redirect w/     │                          │
     │                    │    code_challenge  │                          │
     │◀───────────────────│                    │                          │
     │  3. authorize      │                    │                          │
     │─────────────────────────────────────────▶│                          │
     │  4. redirect w/ authorization_code       │                          │
     │◀─────────────────────────────────────────│                          │
     │  5. code                                  │                          │
     │───────────────────▶│                    │                          │
     │                    │ 6. code + verifier  │                          │
     │                    │────────────────────▶│                          │
     │                    │ 7. access_token +   │                          │
     │                    │    refresh_token    │                          │
     │                    │◀────────────────────│                          │
     │                    │ 8. API call w/ token│                          │
     │                    │─────────────────────────────────────────────▶ │
```

PKCE (Proof Key for Code Exchange, RFC 7636) liga el intercambio de token al browser que empezó el flujo, y derrota la intercepción de code en clients públicos.

```bash
# Genera un verifier y un challenge de PKCE en local:
python3 -c "
import base64, hashlib, secrets
v = base64.urlsafe_b64encode(secrets.token_bytes(32)).rstrip(b'=').decode()
c = base64.urlsafe_b64encode(hashlib.sha256(v.encode()).digest()).rstrip(b'=').decode()
print('verifier :', v)
print('challenge:', c)
"
```

### 5.2 Client Credentials

Server-to-server. El client se autentica con sus propias credenciales — no hay usuario de por medio.

```bash
curl -X POST https://auth.example.com/oauth/token \
  -u 'svc_client_id:svc_client_secret' \
  -d 'grant_type=client_credentials&scope=orders:read'
```

### 5.3 Device Code

Para dispositivos con input limitado (TVs, tools de CLI). El dispositivo muestra un code que el usuario tipea en su teléfono.

```bash
# Paso 1 — el dispositivo pide un code:
curl -X POST https://auth.example.com/oauth/device/code \
  -d 'client_id=cli_tool&scope=read'

# Paso 2 — el dispositivo hace poll hasta que el usuario autoriza en otra pantalla:
curl -X POST https://auth.example.com/oauth/token \
  -d 'grant_type=urn:ietf:params:oauth:grant-type:device_code' \
  -d 'device_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS' \
  -d 'client_id=cli_tool'
```

### 5.4 Refresh token

Credencial de larga vida usada para obtener un access token nuevo cuando el actual expira.

```bash
curl -X POST https://auth.example.com/oauth/token \
  -d 'grant_type=refresh_token' \
  -d 'refresh_token=v1.MjE0MDgtY2NmZjItMDA3MDc' \
  -d 'client_id=web_app'
```

### 5.5 Flujos deprecados

- **Implicit flow** (`response_type=token`): el access token vuelve en el fragment de la URL. Lo removió OAuth 2.1 por fuga en Referer y exposición en el historial del browser.
- **Resource Owner Password Credentials** (`grant_type=password`): el client recolecta la password del usuario directo. Lo removieron porque derrota todo el propósito de la auth federada.

Encontrar cualquiera de los dos flujos habilitado en 2026 es en sí un issue digno de reporte.

## 6. Fallas habituales de OAuth

### 6.1 Debilidad en la validación de `redirect_uri`

Si el authorization server hace validación de match exacto, no puedes sustituir el redirect. Si hace matching de prefijo o de substring, sí puedes.

El fuente agrupa cuatro fallos de matching: un subpath bajo el callback registrado, un parámetro que redirige a otro origen, un dominio que solo parece el registrado y una ruta codificada. Las URLs no se incluyen. Un bypass devuelve el authorization code a otro origen, y ese code se cambia por un token.

### 6.2 PKCE ausente

Cualquier client público (SPA, app mobile, CLI) que corre el flujo authorization code sin PKCE es vulnerable a intercepción de code en el handler de loopback o de custom-scheme.

### 6.3 CSRF por validación ausente de `state`

El parámetro `state` es un token CSRF ligado al browser que inició el login. Si el client no lo valida al volver, un atacante puede forzar al browser de la víctima a completar un flujo usando el authorization code del atacante — logueando a la víctima en la cuenta del atacante, desde la cual va a pegar datos bancarios con gusto.

### 6.4 Fuga de tokens en fragments de URL vía Referer

Los returns legacy del implicit flow filtran a headers Referer cuando la landing page carga cualquier recurso de terceros (analytics, fonts, images). Cualquier cosa que loguee referrers loguea tokens.

### 6.5 Authentication de client débil o ausente

Los confidential clients se supone que se autentican contra el token endpoint. Muchos servers mal configurados aceptan un `client_secret` vacío o caen a comportamiento de public client cuando la authentication falla, y eso deja que cualquiera canjee un code que interceptó.

## 7. OpenID Connect encima de OAuth

OpenID Connect (OIDC) es una capa de identidad encima de OAuth 2.0. Agrega:

- **`id_token`:** un JWT firmado que lleva claims sobre el usuario (`sub`, `email`, `email_verified`, `aud`, `iss`, `nonce`, `exp`).
- **Endpoint UserInfo:** `/userinfo` devuelve claims cuando se consulta con un access token válido.
- **Endpoint JWKS:** `/.well-known/jwks.json` publica las public keys que el identity provider usa para firmar tokens.
- **Documento de discovery:** `/.well-known/openid-configuration` publica cada URL relevante, los algoritmos soportados y los tipos de claim.

```bash
# Empieza cada assessment de OIDC acá:
curl -s https://auth.example.com/.well-known/openid-configuration | jq .

# Trae las signing keys:
curl -s https://auth.example.com/.well-known/jwks.json | jq .
```

El claim `sub` es el identificador estable y único del usuario final entre sessions. Nunca uses `email` como identificador — los emails se pueden reasignar; `sub` no.

## 8. Estructura de un JWT

Un JSON Web Token (RFC 7519) son tres segmentos codificados en base64url unidos por puntos:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhbGljZSIsInJvbGUiOiJ1c2VyIn0.mmS9J-vqZ7ubM7q9U9c9Zk5H_pS8yqAK1SdW7Y1r7oI
└──────────── header ────────────┘.└────────── payload ──────────┘.└──────────── signature ────────────┘
```

### 8.1 Decodificar

```bash
# Decode manual con base64 (agrega padding según haga falta):
python3 -c "
import base64, json
t = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9'
pad = '=' * (-len(t) % 4)
print(json.dumps(json.loads(base64.urlsafe_b64decode(t + pad)), indent=2))
"

# O usa jwt_tool (viene en Kali 2026):
jwt_tool eyJhbGciOi...
```

### 8.2 Algoritmos de uso habitual

| `alg` | Familia | Tipo de key | Notas |
|---|---|---|---|
| HS256 | HMAC-SHA256 | Secret simétrico compartido | Rápido, peligroso si el secret filtra o es débil |
| HS384/512 | HMAC-SHAx | Secret simétrico compartido | Las mismas propiedades, digest más fuerte |
| RS256 | RSA-PKCS1-v1_5 SHA-256 | Private key RSA | La public key se puede distribuir ampliamente |
| RS384/512 | RSA-PKCS1-v1_5 SHAx | Private key RSA | Igual que arriba |
| PS256 | RSASSA-PSS SHA-256 | Private key RSA | Moderno, evita issues de PKCS1-v1_5 |
| ES256 | ECDSA P-256 SHA-256 | Private key EC | Keys y firmas chicas |
| EdDSA | Ed25519 | Private key Ed25519 | Default moderno, determinístico |
| none | (sin firma) | — | Hay que rechazarlo; bypass histórico |

## 9. Ataques clásicos a JWT

Cada ataque de abajo es una falla del verifier en imponer algo que el header afirma.

### 9.1 `alg: none`

El token declara que no hay firma. Un verifier que elige el algoritmo desde el header lo acepta sin firmar. `jwt_tool -X a` genera las variantes de capitalización. La invocación no se incluye.

### 9.2 Confusión de key HS256 ↔ RS256

El token se firmó con RS256. El verifier elige el algoritmo desde el header y verifica con la public key. Si acepta HS256, esa misma public key sirve como secret HMAC. La forja no se incluye.

### 9.3 Header `jku` de JWKS apuntando a otro origen

`jku` o `x5u` le dicen al verifier de dónde traer la key. Un verifier que fetchea cualquier URL acepta una key que no es la suya.

Mitigación: pinea `jku` a una allow-list de URLs exactas, o ignora el header y usa siempre el JWKS configurado.

### 9.4 Path traversal o SQL injection en `kid`

`kid` elige qué key usar. Si el verifier lo pasa sin filtrar a una ruta de filesystem o a una query SQL, el id deja de ser un nombre de key. El fuente muestra un `kid` que apunta a un archivo vacío, con lo que el secret HMAC queda vacío. El comando no se incluye.

### 9.5 Firma vacía

Algunos verifiers recortan whitespace o bytes finales y cortocircuitan si la firma está vacía. Truncar el tercer segmento, dejando el punto, es la prueba que nombra el fuente. El token no se incluye.

### 9.6 Secret HS256 débil

Si se usa HS256 con un secret corto o común (`secret`, `changeme`, `supersecret`), se puede atacar offline. El recorrido con hashcat no se incluye. La higiene está en la sección 10.

## 10. Higiene moderna de JWT

Estos son los defaults actuales de un verifier de JWT endurecido en 2026:

- **Algoritmo pineado del lado del server.** El verifier se instancia con exactamente un algoritmo; el `alg` del header del token se ignora para la selección y solo se cruza para igualdad.
- **JWKS con allow-list de `kid`.** Solo se honran key ids conocidos; los ids desconocidos se rechazan sin ningún fetch de red.
- **`jku` / `x5u` deshabilitados o en allow-list.** Ningún fetch a una URL arbitraria, nunca.
- **TTL corto en access tokens.** Cinco a quince minutos es lo típico; cualquier cosa más larga te mueve al territorio de refresh token.
- **Los refresh tokens rotan.** Cada canje invalida el refresh token anterior; el reuso es una señal de compromiso.
- **Lista de revocación para sessions de alto riesgo.** Una denylist de valores `jti` (JWT ID) con entradas que expiran en el `exp` del token.
- **`aud` e `iss` validados en cada request.** Un token acuñado para `api.internal` no debe aceptarse en `api.public`.
- **Clock skew acotado.** Típicamente ±30 segundos; cualquier cosa más grande acepta tokens demasiado viejos.

## 11. jwt_tool y hashcat — recorrido completo de ataque

La imagen de Kali 2026 trae las dos tools. El fuente monta una app Flask que firma con HS256 y un secret corto, captura el token, lo abre con `jwt_tool`, lo ataca con hashcat modo 16500 y vuelve a firmar los claims `role` y `sub`. Ese recorrido no se incluye. Lo que sí queda es la reemisión con RS256 y el verifier pineado.

### 11.1 Reemitir con RS256 y pinning de JWKS

La app remediada:

```python
import jwt, time
from cryptography.hazmat.primitives import serialization
priv = serialization.load_pem_private_key(open("priv.pem","rb").read(), None)
tok = jwt.encode({"sub":"alice","role":"user","exp":int(time.time())+900,
                  "iss":"https://auth.lab","aud":"api.lab"},
                 priv, algorithm="RS256", headers={"kid":"2026-01"})

# Del lado del verifier, el algoritmo está pineado y kid está en whitelist:
pub = open("pub.pem","rb").read()
claims = jwt.decode(tok, pub, algorithms=["RS256"],
                    audience="api.lab", issuer="https://auth.lab",
                    options={"require": ["exp","iss","aud","sub"]})
```

El atacante ya no puede firmar tokens — no tiene la private key. HS256 con la public key como secret queda bloqueado porque el algoritmo no está en la lista pineada.

## 12. Patrones de API key

Las API keys son secrets compartidos, y cada lugar que tocan es un riesgo de fuga.

### 12.1 Lugares de transporte

```bash
# Header (recomendado):
curl -H 'X-API-Key: ak_live_...' https://api.example.com/orders

# Header Authorization:
curl -H 'Authorization: Bearer ak_live_...' https://api.example.com/orders

# HTTP Basic (la key como username, password vacía — estilo Stripe):
curl -u 'ak_live_...:' https://api.example.com/orders

# Query string (PELIGROSO):
curl 'https://api.example.com/orders?api_key=ak_live_...'
```

### 12.2 Canales de fuga

- **Access logs.** Las keys en query string caen en logs de nginx/ALB/CloudFront, en pipelines de shipping y en SIEMs aguas abajo. Un solo compromiso del store de logs filtra la key de cada caller histórico.
- **Historial y bookmarks del browser.** Las keys embebidas en URL se guardan del lado del cliente para siempre.
- **Header Referer.** Una página cargada con `?api_key=` en la URL la filtra a cada asset de terceros (fonts, analytics, redes de ads) que renderiza.
- **Commits de Git.** `.env`, `config.yml` y strings hardcodeados; usa `gitleaks detect --source . -v` en cualquier repo objetivo al que tengas acceso.
- **Bundles de JS del lado del cliente.** `curl -s https://app.example.com/main.js | grep -Eo 'ak_[a-z]{4}_[A-Za-z0-9]{20,}'`.
- **Recursos de apps mobile.** `apktool d app.apk && grep -RE 'ak_(live|test)_' app/`.

### 12.3 Higiene de keys

- 128+ bits de entropía random, con un prefijo identificable (`sk_live_`, `AKIA`…).
- Rota ante cualquier sospecha; soporta al menos dos keys activas por client para rotación sin downtime.
- Liga las keys a rangos de IP de origen y a scopes para que una key de solo lectura filtrada de un worker batch no se pueda usar para escribir desde cualquier lado.
- Ofrece revocación y expiración programáticas.

## 13. mTLS entre clients y gateway

Mutual TLS extiende el handshake TLS para exigir que el client presente un certificado, y le da al gateway una identidad criptográfica de cada caller.

### 13.1 Emitir un certificado de cliente

```bash
# Crea una CA privada (lab):
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -out ca.crt -subj "/CN=Lab Root CA"

# Key de cliente + CSR:
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr \
  -subj "/CN=svc-orders-01/OU=orders/O=example"

# Fírmalo con la CA:
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days 365 -sha256
```

### 13.2 Usar el cert

```bash
curl --cert client.crt --key client.key --cacert ca.crt \
     https://api.example.com/orders
```

### 13.3 Revocación

- **CRL (Certificate Revocation List):** la CA publica una lista firmada de seriales revocados; los verifiers la traen de forma periódica.
- **OCSP (Online Certificate Status Protocol):** el verifier consulta a la CA por cert, con un nonce.
- **OCSP stapling:** el presentador adjunta una response OCSP reciente para que el verifier no tenga que llegar a la CA.

```bash
# Chequea un cert contra un responder OCSP:
openssl ocsp -issuer ca.crt -cert client.crt -url http://ocsp.lab \
  -CAfile ca.crt -text
```

### 13.4 Misconfigurations clásicas

- **Confiar en una CA wildcard.** El gateway confía en una CA pública (Let's Encrypt, DigiCert) para authentication de cliente. Cualquiera con un cert de 10 dólares se vuelve un «cliente de confianza».
- **Sin chequeo de revocación.** Un cert de cliente comprometido vive hasta su expiración natural.
- **Sin pinning de SAN.** El gateway acepta cualquier cert de cliente firmado por la CA privada, sin importar Subject/SAN. Cualquier movimiento lateral te da un cert válido.
- **Cert reenviado al backend como header (`X-SSL-Client-Cert`) sin validar quién lo seteó.** La misma clase que §4.2.

## 14. Arquitectura de rate limiting

### 14.1 Algoritmos

| Algoritmo | Comportamiento | Dónde se usa |
|---|---|---|
| Token bucket | Un bucket de tokens se rellena a tasa R; cada request consume uno; vacío ⇒ 429. Permite ráfagas cortas. | AWS API Gateway, Kong, Envoy |
| Leaky bucket | Los requests se encolan y drenan a tasa constante R; overflow ⇒ 429. Suaviza ráfagas. | Apigee, Zuul más viejo |
| Fixed window | N requests por ventana de calendario (por minuto). Abuso de frontera: 2N justo en el rollover. | Limiters simples en memoria |
| Sliding window (log) | Guarda cada timestamp de request; cuenta en los últimos N segundos. Preciso pero caro. | Cloudflare, Wallarm |
| Sliding window (counter) | Mezcla ponderada de la ventana fija anterior y la actual. Aproximación barata. | Limiters basados en Redis |

### 14.2 Elecciones habituales de key

Los rate limiters hacen key por algo. La key es la superficie de bypass:

- **Dirección IP:** bypass con una botnet, rotación de IP de proveedor cloud o rotación de IPv6 `/64`.
- **`X-Forwarded-For`:** el limiter confía en el header sin quitarlo — spooféalo.
- **API key o claim `sub`:** fuerza rotación de cuentas, barato para abuso automatizado.
- **Cookie de session:** límpiala y vuelve a loguearte por request.

### 14.3 Bypass por inyección de header

```bash
# El limiter lee X-Forwarded-For; el gateway no lo quita:
for i in $(seq 1 10000); do
  curl -s -H "X-Forwarded-For: 10.0.0.$((RANDOM % 254))" \
       https://api.example.com/login \
       -d "user=admin&pw=guess$i"
done
```

### 14.4 Inyección de un segundo header

Algunos servers concatenan headers duplicados (`a, b`), otros toman el primero, otros el último. Si el parser del limiter no coincide con el parser del origin, un segundo header ata el límite a una identidad y el origin autentica como otra. La petición no se incluye.

### 14.5 Rotación de IP

Un límite que solo mira la IP de origen se salta cambiando de IP en cada petición. La misma técnica se usa en load testing. El enrutado no se incluye. El control es un límite que no dependa solo de la IP.

## 15. Integración de WAF

Los WAFs se sientan delante del gateway e inspeccionan el contenido del request contra conjuntos de reglas — típicamente OWASP ModSecurity Core Rules, reglas administradas por el vendor y deny lists custom.

### 15.1 WAFs habituales

| WAF | Deployment | Fingerprint |
|---|---|---|
| Cloudflare | Red de edge | `Server: cloudflare`, `CF-Ray:` |
| AWS WAF | ALB / CloudFront / API Gateway | `x-amzn-RequestId`, las responses bloqueadas a menudo van en blanco |
| Akamai | Edge | `Server: AkamaiGHost`, reference ids |
| ModSecurity + CRS | Self-hosted (nginx, Apache, plugin de gateway) | 403 con body `<title>403 Forbidden</title>` |
| Wallarm | Self-hosted o SaaS | `nginx-wallarm`, `wallarm_mode` |
| Imperva | Cloud / on-prem | `X-Iinfo`, cookie `visid_incap_` |

### 15.2 Técnicas de bypass

Los WAFs son pattern matchers. El fuente nombra cuatro familias que una firma se pierde: encoding Unicode, comentarios que parten la consulta, variación de mayúsculas y HTTP request smuggling (CL.TE, TE.CL, TE.TE), donde el WAF ve una petición y el origin ve dos. Las peticiones, los tampers y el clonado de `smuggler.py` no se incluyen. La detección de las dos cabeceras de longitud a la vez está en la sección 17.2.

## 16. Session fixation vs session hijacking en un entorno de gateway

Un gateway que termina una session (sea basada en cookie, bearer token opaco o JWT de corta vida) expone los mismos dos riesgos clásicos con un giro: muchos equipos creen que un token de OAuth no es una session y por eso no aplican higiene de manejo de session.

### 16.1 Session fixation

El atacante fuerza al browser de la víctima a usar un identificador de session que el atacante conoce.

- Fuerza a la víctima a través de un link que contiene un valor `state` elegido por el atacante y que el client va a hacer echo como su session id.
- Settea una cookie en un dominio padre compartido antes de que la víctima se autentique.
- Devuelve un session id estático desde el endpoint de login (una misconfig común en sessions emitidas por el gateway).

Mitigación: rota el session id (y, para OAuth, el refresh token) en cada evento de authentication.

### 16.2 Session hijacking

El atacante roba un identificador de session válido — cookie, bearer token, refresh token — y lo rejuega.

- XSS roba cookies no marcadas `HttpOnly`.
- MITM sobre una conexión TLS degradada captura bearer tokens.
- Fugas de Referrer / log / URL (§12) capturan tokens en tránsito.
- Subdominios comprometidos roban cookies con scope `.example.com`.

Mitigación: cookies `HttpOnly`, `Secure`, `SameSite=Strict`; token binding (DPoP, RFC 9449); TTL corto con rotación; monitoreo de uso concurrente del mismo token desde ubicaciones geográficamente imposibles.

## 17. Detección — lo que ve el blue team

Cada ataque de esta lección tiene una firma sobre la que un blue team competente puede construir una regla.

### 17.1 Logs de gateway y de auth

- Picos de tasa de 401 y 403 desde una sola IP o `sub`.
- Auth exitosa desde un país o ASN nuevo para un usuario existente.
- Varios eventos de reuso de `refresh_token` para el mismo token rotativo.
- Fallas de verificación de JWT con algoritmos anómalos (`none`, `alg` que no coincide).
- Valores de `kid` que no están en la allow-list actual.
- Conexiones directas a hosts de backend desde IPs fuera de la subnet del gateway.

### 17.2 Reglas de anomalía del WAF

- Requests con headers `X-Forwarded-For` o `Host` duplicados.
- `Transfer-Encoding` y `Content-Length` presentes los dos.
- Requests cuyo path normaliza distinto antes y después del decoding (`%2f`, `..;/`).
- Valores de `redirect_uri` de OAuth que no están en la lista registrada del client.

### 17.3 Auditoría cloud

- `ConsoleLogin` de CloudTrail desde regiones inesperadas.
- Eventos `AssumeRole` de IAM con duraciones inusualmente largas.
- Ráfagas de `MethodNotAllowedException` de API Gateway.
- `SampledRequests` de WAF que matchean reglas custom con tendencia al alza.

Alimenta las señales al SIEM como detecciones; pagina sobre la composición de segundo orden (falla de auth + bloqueo de WAF + cambio de geo ⇒ severidad alta), no sobre cada una por separado.

## 18. Hands-on — secret HS256 débil a RS256 con pinning de JWKS

Este es el assessment práctico de la lección. Todo corre en una sola VM Kali.

### 18.1 Provisionar

```bash
mkdir -p /root/lab-11 && cd /root/lab-11
python3 -m venv venv && source venv/bin/activate
pip install flask pyjwt cryptography
```

### 18.2 App vulnerable

```bash
cat > vuln.py <<'PY'
from flask import Flask, request, jsonify
import jwt, time
app = Flask(__name__)
SECRET = "sunshine"   # débil, está en rockyou
@app.post("/login")
def login():
    d = request.get_json(force=True)
    if d.get("user") == "bob" and d.get("pw") == "bob":
        t = jwt.encode({"sub":"bob","role":"user","exp":int(time.time())+3600},
                       SECRET, algorithm="HS256")
        return jsonify(token=t)
    return "no", 401
@app.get("/admin")
def admin():
    tok = request.headers.get("Authorization","").split(" ")[-1]
    c = jwt.decode(tok, SECRET, algorithms=["HS256"])
    if c.get("role") != "admin":
        return "forbidden", 403
    return jsonify(secret="flag{jwt_weak_secret_owned}")
app.run(port=5001)
PY
python3 vuln.py &
sleep 1
```

### 18.3 Qué hace el fuente y qué queda

El fuente inicia sesión, ataca el token con hashcat modo 16500 y vuelve a firmar un claim `role=admin`. Ese recorrido no se incluye. El laboratorio que sí queda es el verifier de la sección 18.4: algoritmo pineado, `kid` en allow-list, `iss` y `aud` obligatorios.

### 18.4 Remediar a RS256 + pinning de JWKS

```bash
# Genera un keypair para el servicio de auth:
openssl genrsa -out priv.pem 3072
openssl rsa -in priv.pem -pubout -out pub.pem

cat > safe.py <<'PY'
from flask import Flask, request, jsonify
import jwt, time
app = Flask(__name__)
PRIV = open("priv.pem","rb").read()
PUB  = open("pub.pem","rb").read()
KID_ALLOW = {"2026-01"}
ALG_ALLOW = {"RS256"}

@app.post("/login")
def login():
    d = request.get_json(force=True)
    if d.get("user") == "bob" and d.get("pw") == "bob":
        t = jwt.encode({"sub":"bob","role":"user",
                        "iss":"https://auth.lab","aud":"api.lab",
                        "exp":int(time.time())+900},
                       PRIV, algorithm="RS256", headers={"kid":"2026-01"})
        return jsonify(token=t)
    return "no", 401

@app.get("/admin")
def admin():
    tok = request.headers.get("Authorization","").split(" ")[-1]
    hdr = jwt.get_unverified_header(tok)
    if hdr.get("alg") not in ALG_ALLOW: return "bad alg", 401
    if hdr.get("kid") not in KID_ALLOW: return "bad kid", 401
    c = jwt.decode(tok, PUB, algorithms=list(ALG_ALLOW),
                   audience="api.lab", issuer="https://auth.lab",
                   options={"require":["exp","iss","aud","sub"]})
    if c.get("role") != "admin": return "forbidden", 403
    return jsonify(secret="flag{never_reached}")
app.run(port=5002)
PY
python3 safe.py &
```

### 18.5 Verificar el fix

El fuente repite contra `safe.py` la confusión HS256 (firma con la public key) y una variante `alg: none`. Las dos respuestas son `bad alg`: el algoritmo no está en `ALG_ALLOW`. Las peticiones no se incluyen.

Captura screenshot tanto del ataque exitoso contra `vuln.py` como del ataque rechazado contra `safe.py`. Los dos van en el reporte.

## 19. Ejercicios

### 19.1 Guiados

#### Ejercicio 1 — Enumerar un gateway

Dada una URL objetivo que sea tuya, corre el flujo de fingerprint de §2.3 y después sonda fuga de backend: enumeración DNS de `api-internal.*` y `*.svc.cluster.local` desde un host comprometido, y port scans de la `/24` interna. Documenta al menos un host que responde en un puerto que no es el del gateway.

#### Ejercicio 2 — Reproducción de fuga por Referrer

Levanta una página que se autentica vía OAuth implicit flow, redirige a tu callback e incluye un tag `<img>` apuntando a un nginx que controlas. Inspecciona el access log de nginx en busca del header Referer que contiene el token del fragment. Escribe por qué se deprecó el implicit flow.

#### Ejercicio 3 — Crack de secret HS256 débil

Rehaz §11 de punta a punta contra un token que generes con el secret `passw0rd!`. Compara el tiempo de reloj de pared del crack usando `rockyou.txt`, rockyou + `best64.rule`, y una máscara de 6 caracteres `?a?a?a?a?a?a`.

### 19.2 Independientes

#### Ejercicio 4 — Key confusion en un framework real

Elige cualquier framework HTTP que conozcas (Express, FastAPI, Spring). Escribe la ruta endurecida: el verifier fija el algoritmo y no lo lee del header. Entrega la ruta y un write-up corto de qué rechaza (`alg` fuera de la lista, `kid` desconocido). No incluyas el ataque.

#### Ejercicio 5 — Estudio de bypass de rate limit

Pon Wallarm o ModSecurity delante de un endpoint de login chico. Implementa uno de: rotación de IP por request, spoofing de `X-Forwarded-For`, inyección de un segundo header. Mide la tasa efectiva que el limiter permite antes y después de tu bypass. Explica qué debería haber hecho el limiter.

#### Ejercicio 6 — Descubrimiento de request smuggling

Despliega nginx delante de gunicorn. Anota si las cabeceras `Transfer-Encoding` y `Content-Length` pueden llegar juntas al origin y qué parser se queda con cada una. No introduzcas un desync y no pegues el payload.

## 20. Resumen y lección siguiente

La API es un sistema con muchos defensores: un WAF de edge, un gateway, un identity provider, un conjunto de chequeos de autorización y cada microservicio detrás. Cada capa tiene una superficie de ataque, y cada capa se apoya en que otra capa haya hecho su trabajo. El camino confiable hacia un finding digno de reporte corre por las costuras: el header en el que confía el gateway, el algoritmo que el verifier no pinea, el path que el WAF normaliza distinto que el origin, el rate limit que hace key sobre un valor spoofeable. Un pentester que lee este stack como lo dibujó un arquitecto es rápido; uno que lo lee como lo compone un atacante es peligroso.

Ahora deberías poder hacer fingerprint de cualquiera de los seis gateways principales, diagramar de memoria un flujo OAuth 2.1 authorization code + PKCE y explicar por qué un verifier pinea el algoritmo, el `kid` y el JWKS.

A continuación, Lección 12 — Service Mesh Security and East-West Traffic, se mete dentro del perímetro. Vamos a mirar Istio y Linkerd, el mTLS emitido por el mesh entre servicios, cómo la filter chain de Envoy impone autorización por servicio, y cómo los atacantes que ya tienen un foothold en un pod pivotan lateralmente a otro con (o sin) una identidad SPIFFE válida.
