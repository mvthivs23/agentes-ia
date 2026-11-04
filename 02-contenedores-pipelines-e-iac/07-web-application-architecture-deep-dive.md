# Lección 07 — Web Application Architecture Deep Dive

Módulo 04: Containers and Software Architecture · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Conocer la arquitectura de una aplicación web objetivo duplica la tasa de acierto del pentester. Dos testers que corren la misma wordlist contra el mismo dominio van a producir informes radicalmente distintos si solo uno de ellos reconoce que la app es un bundle SSR de Next.js sentado detrás de un edge de Cloudflare, con un origen nginx y un almacén de sesión respaldado por Redis. Quien lee la arquitectura elige la clase correcta de bugs al primer intento, se ahorra horas de fuzzing en callejones sin salida y encuentra las malas configuraciones que los escáneres de plantilla nunca ven.

Esta lección disecciona el stack web moderno tal como necesita verlo un pentester: cada modelo de renderizado que entrega JavaScript en 2026, los paradigmas de API que se sientan detrás de esos front-ends, los reverse proxies, los balanceadores de carga y las CDN que abren el tráfico en abanico, y los controles de sesión y de seguridad de contenido que se superponen encima. Cada sección trae un ángulo de pentest: la clase de bug que la arquitectura invita, la huella que usas para detectarla y el comando concreto de Kali 2026 que convierte la teoría en un hallazgo.

Al terminar vas a mirar un alcance nuevo y, solo con cabeceras y un bundle de JavaScript, vas a poder nombrar el framework, el modo de render, el estilo de API, la CDN, el reverse proxy y el modelo de sesión, y saber qué cuatro ataques probar primero.

## 1. Introducción: conocer la arquitectura duplica la tasa de acierto

Un pentester tradicional corre nikto, dirb y sqlmap en ese orden y da el día por terminado. Un pentester web moderno se pasa la primera hora de una evaluación sin hacer otra cosa que fingerprinting arquitectónico, porque la arquitectura dicta todo el plan de prueba.

### 1.1 Por qué la arquitectura importa antes de cualquier payload

Considera el mismo objetivo —`https://app.example.com`— bajo tres arquitecturas distintas:

| Arquitectura | Los primeros 4 bugs que hay que probar |
| --- | --- |
| Rails MPA + Postgres | CSRF, SQLi, mass assignment, fijación de sesión |
| React SPA + API REST | Manipulación de JWT, IDOR, mala configuración de CORS, divulgación de source maps |
| Next.js SSR + GraphQL | SSTI en `getServerSideProps`, introspección de GraphQL, DoS por batch, envenenamiento de caché en el edge de Vercel |

La wordlist que detectaría un bug en la app Rails no encuentra nada en la SPA y se topa con el rate limit en la app SSR. La wordlist no es el problema: lo es la lectura arquitectónica que falta.

### 1.2 El bucle de reconocimiento de arquitectura

Toda evaluación debería empezar por el bucle de abajo antes de enviar un solo payload:

```text
┌────────────────────────────────────────────────────────────────┐
│  1. Huella del front-end (framework, modo de render)           │
│  2. Huella de la API (REST/GraphQL/gRPC-Web/WebSocket)         │
│  3. Huella del edge (CDN, WAF, reverse proxy)                  │
│  4. Huella del modelo de sesión (cookie/JWT/session id)        │
│  5. Huella de la postura de seguridad de contenido (CSP, CORS) │
│  6. Mapea cada URL/endpoint que salió de los pasos 1–5         │
│  7. Elige las 5 clases de bug principales de ESTE stack, y     │
│     solo entonces abre Burp, ffuf, sqlmap, etc.                │
└────────────────────────────────────────────────────────────────┘
```

> **Idea del pentester:** Los pasos 1 a 6 de arriba son reconocimiento puro y no cuestan peticiones que un WAF fuera a limitar por tasa. Házelos desde tu caja Kali con curl, jq, whatweb, wappalyzer-cli, httpx y una pestaña de dev-tools del navegador. Solo cuando conozcas la forma del objetivo pasas a Burp y al fuzzing activo.

## 2. MPA, SPA, SSR, SSG e ISR: cada modelo que entrega JavaScript hoy

Hay cinco modelos de renderizado distintos en producción en 2026, y cada uno deja una huella en la respuesta HTML y cada uno invita a una clase distinta de bug.

### 2.1 MPA: Multi-Page Application

El modelo clásico. Cada navegación es un viaje HTTP completo que devuelve un documento HTML entero renderizado en el servidor. Piensa en Django, Rails, Laravel, Spring MVC, ASP.NET MVC, WordPress.

```bash
# Huella de una MPA:
curl -sI https://target.example.com/ | grep -i 'x-powered-by\|server'
# Server: nginx
# X-Powered-By: PHP/8.2.15

# El código fuente HTML está LLENO y tiene SENTIDO:
curl -s https://target.example.com/ | grep -c '<form\|<h1\|<article'
# 47   <- content is server-rendered
```

Ángulo de pentest: las MPA hablan cookies de sesión y campos ocultos de formulario. Las clases de bug que hay que cazar primero son CSRF (tokens ausentes o reutilizables), inyección de plantillas del lado del servidor (SSTI) en páginas de error y vistas de email, y SQLi clásica en los parámetros del query-string que el servidor, a todas luces, parsea.

### 2.2 SPA: Single-Page Application

El navegador descarga una shell HTML pequeña y un bundle grande de JavaScript que renderiza cada vista posterior en el cliente. Los datos vienen de una API. Piensa en Create-React-App, Vue CLI, Angular CLI.

```bash
# Huella de una SPA: el HTML está casi vacío:
curl -s https://spa.example.com/ | wc -l
# 12   <- twelve lines total

curl -s https://spa.example.com/ | grep -oE 'src="[^"]*\.js"'
# src="/static/js/main.4f3a2b.js"
# src="/static/js/vendors.9c1e.js"

# Descarga el bundle y busca un source map:
curl -sI https://spa.example.com/static/js/main.4f3a2b.js | grep -i sourcemap
# SourceMap: main.4f3a2b.js.map

# Si el archivo .map es público, tienes el código fuente original:
curl -s https://spa.example.com/static/js/main.4f3a2b.js.map -o main.js.map
npx source-map-explorer main.js.map --tsv > components.tsv
```

Ángulo de pentest: source map = objetivo de revisión de código. Un archivo `.map` publicado te da el código fuente original de React/Vue/TypeScript, incluidos comentarios y nombres de variables internas. Esto convierte un pentest de caja negra en una revisión de código de caja blanca. Además, como cada pantalla llama a la API directamente, el bundle de la SPA es un mapa completo de endpoints de API: `grep -oE '"/api/[^"]+"' main.js` enumera toda la superficie de ataque en segundos.

### 2.3 SSR: Server-Side Rendering

El servidor corre el mismo framework de JavaScript que va a correr el navegador y devuelve un documento HTML completamente renderizado, que luego se hidrata en el navegador. Piensa en Next.js (`getServerSideProps`), Nuxt (`asyncData`), SvelteKit (`load`), Remix (`loader`).

```bash
# Huella de SSR: el HTML está lleno Y hay un payload de hidratación:
curl -s https://ssr.example.com/ | grep -c '__NEXT_DATA__\|__NUXT__\|__SVELTEKIT_'
# 1

curl -s https://ssr.example.com/ | grep -oE '__NEXT_DATA__[^<]*' | head -c 200
```

Ángulo de pentest: inyección de plantillas del lado del servidor. Los frameworks SSR pasan entrada de usuario no confiable por un pipeline de renderizado que, bajo malos patrones de desarrollo, puede evaluar expresiones de plantilla en el servidor. Si una página de Next.js pasa `req.query.name` directamente a un componente React renderizado cuyas props se serializan después mediante `dangerouslySetInnerHTML`, obtienes XSS que se dispara en el servidor, y a menudo se salta la CSP porque el script inyectado va inline en el HTML inicial. Busca también path traversal en lecturas de archivo de `getServerSideProps`, SSRF en props que el servidor va a buscar, y prototype pollution en el payload de hidratación.

### 2.4 SSG: Static Site Generation

Cada página se prerenderiza en tiempo de build a HTML plano y se sirve desde una CDN. Piensa en Gatsby, Astro (por defecto), Hugo, Jekyll, Next.js `getStaticProps` sin ISR.

```bash
# Huella de SSG: el HTML está lleno, las cabeceras muestran HIT de caché de CDN, no hay cookie de sesión:
curl -sI https://ssg.example.com/ | grep -i 'cache\|age\|x-vercel\|cf-cache'
# cache-control: public, max-age=31536000
# age: 84213
# cf-cache-status: HIT
```

Ángulo de pentest: céntrate en la API. El HTML estático es un callejón sin salida: se construyó en tiempo de CI y no se puede explotar. Pero un sitio SSG casi siempre llama a una API viva para cualquier cosa dinámica (login, comentarios, búsqueda). Sáltate las páginas, haz grep del JavaScript en busca de endpoints y ataca la API directamente.

### 2.5 ISR: Incremental Static Regeneration

Un híbrido: las páginas son estáticas, pero la CDN las regenera con un temporizador o bajo demanda mediante un endpoint de revalidación. Piensa en `revalidate` de Next.js, `swr` de Nuxt.

```bash
# Huella de ISR: cache-control con s-maxage y stale-while-revalidate:
curl -sI https://isr.example.com/products/42 | grep -i 'cache-control\|x-vercel-cache'
# cache-control: s-maxage=60, stale-while-revalidate=300
# x-vercel-cache: STALE
```

Ángulo de pentest: envenenamiento de caché. ISR cachea por URL, a veces con claves adicionales. Si alguna entrada controlada por el usuario fluye hacia la página renderizada pero NO forma parte de la clave de caché, tienes entrada sin clave: inyecta un payload que aterrice en el HTML y cada visitante posterior recibe tu versión. Entradas sin clave habituales: `X-Forwarded-Host`, `X-Original-URL`, `X-Rewrite-URL`, cookies que el servidor lee pero que no forman parte de la clave de caché.

## 3. Panorama de frameworks de frontend

Hacer fingerprint del framework exacto y de su versión recorta la búsqueda de CVE a segundos.

### 3.1 React

```bash
# Huella de React: busca el atributo reactroot o el marcador de hidratación:
curl -s https://target/ | grep -oE 'data-reactroot\|__reactContainer\$[a-z0-9]+'

# En el bundle de JavaScript:
curl -s https://target/static/js/main.*.js | strings | grep -oE 'react-dom@[0-9.]+' | sort -u
```

### 3.2 Vue

```bash
# Vue deja hashes de atributo data-v-* en los elementos renderizados:
curl -s https://target/ | grep -oE 'data-v-[a-f0-9]{8}' | head -3

# Específico de Nuxt:
curl -s https://target/ | grep -o '__NUXT__'
```

### 3.3 Angular

```bash
# Angular entrega ng-version en el elemento raíz:
curl -s https://target/ | grep -oE 'ng-version="[0-9.]+"'
# ng-version="17.3.2"
```

### 3.4 Next.js

```bash
# Firmas de Next.js:
curl -sI https://target/ | grep -i 'x-powered-by\|x-nextjs'
# x-powered-by: Next.js
# x-nextjs-cache: HIT

# La versión está en el path _next/static:
curl -s https://target/ | grep -oE '/_next/static/[^/]+/'
# /_next/static/chunks/    <- confirms Next.js
```

### 3.5 Nuxt

```bash
curl -s https://target/ | grep -oE '/_nuxt/[^"]+\.js' | head -3
```

### 3.6 SvelteKit

```bash
curl -s https://target/ | grep -oE '__SVELTEKIT_DATA__\|/_app/immutable/'
```

### 3.7 Fingerprinting automatizado de frameworks

```bash
# whatweb es el clásico:
whatweb -a 3 https://target.example.com

# CLI de wappalyzer (paquete npm):
npx wappalyzer https://target.example.com --pretty

# httpx (de projectdiscovery) hace fingerprint en masa:
echo target.example.com | httpx -tech-detect -title -status-code -web-server
```

## 4. Patrones de diseño de API REST y ángulos de ataque

REST es el estilo de API dominante. Su previsibilidad es exactamente lo que lo hace fácil de atacar.

### 4.1 El contrato REST

```text
GET     /api/users              → list all users
GET     /api/users/42           → read user 42
POST    /api/users              → create a user
PUT     /api/users/42           → replace user 42
PATCH   /api/users/42           → modify user 42
DELETE  /api/users/42           → delete user 42
```

Los códigos de estado llevan significado: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests, 500 Server Error.

### 4.2 IDOR: enumera los IDs

Cualquier ID numérico en una URL invita a probar Insecure Direct Object Reference. Si `/api/users/42` devuelve tu propio perfil, prueba `/api/users/41`, `/api/users/43`, `/api/users/1`.

```bash
# Sonda IDOR manual con curl:
for id in $(seq 1 100); do
  status=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer $TOKEN" \
    https://target/api/users/$id)
  echo "$id $status"
done | grep ' 200'

# Automatizado con ffuf:
ffuf -u https://target/api/users/FUZZ \
     -w /usr/share/wordlists/seclists/Fuzzing/numbers.txt \
     -H "Authorization: Bearer $TOKEN" \
     -mc 200 -fs 34
```

### 4.3 Mass assignment

Si un `POST /api/users` acepta un cuerpo JSON `{"name":"joe","email":"j@e.com"}`, prueba a añadir `"role":"admin"` o `"is_admin":true`. Los frameworks que vinculan cuerpos de petición directamente a modelos ORM van a poner el campo extra encantados.

```bash
curl -X POST https://target/api/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"joe","email":"j@e.com","role":"admin","is_verified":true,"balance":999999}'
```

### 4.4 Manipulación de verbos

Los controles de acceso a veces comprueban POST pero se olvidan de PUT o PATCH. Si `POST /api/admin/users` devuelve 403, prueba PUT, PATCH, DELETE, o los exóticos HEAD y OPTIONS.

```bash
for verb in GET POST PUT PATCH DELETE HEAD OPTIONS CONNECT TRACE; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -X $verb https://target/api/admin/users)
  echo "$verb $code"
done
```

### 4.5 Filtraciones en respuestas HATEOAS

Las API que devuelven enlaces hipermedia a menudo comparten de más. Un endpoint de usuario que devuelve `"_links":{"admin":"/api/admin/users/42"}` acaba de decirte que la URL de admin existe, aunque tu token no pueda seguirla.

## 5. Arquitectura GraphQL

GraphQL sustituye muchos endpoints REST por un solo endpoint POST (normalmente `/graphql`) que acepta una cadena de consulta que describe exactamente los campos que quiere el cliente.

### 5.1 La forma de GraphQL

- **Un solo endpoint:** una URL, un verbo (POST), un content-type (`application/json`).
- **Schema:** un contrato fuertemente tipado de cada campo consultable y de cada mutación.
- **Resolvers:** funciones en el servidor que obtienen cada campo.
- **Batching:** se pueden enviar varias consultas en una sola petición HTTP.

### 5.2 Consulta de introspección

Si la introspección está habilitada, el schema entero está disponible con solo pedirlo:

```bash
curl -X POST https://target/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{__schema{types{name fields{name type{name kind ofType{name}}}}}}"}'

# Más bonito con una herramienta:
gograpql -u https://target/graphql -introspection
# o
graphw00f -f -t https://target/graphql

# Convierte el JSON de introspección en un archivo SDL:
get-graphql-schema https://target/graphql > schema.graphql
```

### 5.3 DoS basado en alias

Los alias dejan que un cliente llame al mismo campo caro muchas veces en una sola petición:

```graphql
{
  a1: expensiveReport(id:1) { data }
  a2: expensiveReport(id:2) { data }
  a3: expensiveReport(id:3) { data }
  # ... a1000: expensiveReport(id:1000) { data }
}
```

Una petición HTTP, mil consultas a la base de datos. Si el servidor no tiene análisis de coste de consulta, esto es un DoS instantáneo.

```bash
# Genera un payload de bomba de alias:
python3 -c "
q='{'
for i in range(1000):
  q += f'a{i}: user(id:{i}) {{ id email }} '
q += '}'
import json,sys; print(json.dumps({'query': q}))
" > bomb.json

curl -X POST https://target/graphql \
  -H "Content-Type: application/json" \
  --data @bomb.json -o /dev/null -w "time=%{time_total}s size=%{size_download}\n"
```

### 5.4 Abuso de batch

Los servidores que aceptan un array de consultas en una sola petición se pueden usar para forzar logins sin disparar los rate limits por petición:

```bash
curl -X POST https://target/graphql \
  -H "Content-Type: application/json" \
  -d '[{"query":"mutation{login(user:\"admin\",pass:\"1\"){token}}"},
       {"query":"mutation{login(user:\"admin\",pass:\"2\"){token}}"},
       {"query":"mutation{login(user:\"admin\",pass:\"3\"){token}}"}]'
```

### 5.5 Tooling específico de GraphQL

```bash
# Instala y corre graphql-cop para una auditoría completa:
pipx install graphql-cop
graphql-cop -t https://target/graphql -o json
```

## 6. Arquitectura WebSocket

Los WebSockets elevan una conexión HTTP a un canal bidireccional persistente. Traders, apps de chat, dashboards en vivo y juegos multijugador hablan todos WebSocket.

### 6.1 El handshake de upgrade

```text
Client → GET /ws HTTP/1.1
         Host: target.example.com
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
         Sec-WebSocket-Version: 13
         Origin: https://target.example.com

Server → HTTP/1.1 101 Switching Protocols
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 6.2 Socket.IO: la capa de framing

Socket.IO es una librería encima de WebSocket que añade rooms, namespaces, callbacks de ack y fallback a long-polling. Tiene su propio framing de texto (`42["event",{"data":1}]`) que las herramientas a veces se saltan.

### 6.3 CSWSH: Cross-Site WebSocket Hijacking

Los WebSockets no envían peticiones preflight de CORS. Si el servidor acepta la conexión solo a partir de cookies y no valida la cabecera `Origin`, una página en otro origen puede abrir un WebSocket hacia la app y leer y enviar mensajes con la sesión de quien visita esa página. La comprobación con un `Origin` ajeno no se incluye.

### 6.4 Inyección de mensajes

Incluso los canales WebSocket autenticados suelen reenviar frames sin validar que el emisor tenga permiso para la acción pedida dentro del frame. Un frame `joinRoom` hacia `admin` o un `listUsers` que el servidor acepta es una autorización rota en el canal. Los frames no se incluyen.

## 7. gRPC-Web a través de Envoy

Los navegadores no pueden hablar gRPC nativo: los trailers de HTTP/2 y el framing binario no están expuestos por `fetch`. gRPC-Web es un formato de cable compatible con el navegador que un proxy Envoy traduce a gRPC real para el backend.

### 7.1 La forma

```text
Browser  ──(HTTP/1.1 or HTTP/2 with base64 body)──►  Envoy  ──(HTTP/2 gRPC)──►  Backend
Content-Type: application/grpc-web+proto
```

### 7.2 Huella

```bash
curl -sI -X POST https://target/api.UserService/GetProfile | grep -i grpc
# content-type: application/grpc-web+proto
# grpc-status: 0
```

### 7.3 Ángulos de ataque

- grpcurl sobre el puerto de admin de Envoy en texto plano: si 9901 u 8080 están expuestos, puedes obtener server reflection y una lista completa de servicios.
- Manipulación de mensajes: el payload es un protobuf con prefijo de longitud; decodifícalo con `protoc --decode_raw` y vuelve a codificarlo con un campo modificado.
- Filtración de deadline/backend: Envoy a veces hace eco del host upstream en `grpc-message` cuando hay un error.

```bash
grpcurl -plaintext target:9901 list
grpcurl -plaintext target:9901 describe api.UserService
```

## 8. Reverse proxies: introducción al request smuggling

Toda app seria de producción esconde su origen detrás de un reverse proxy: nginx, HAProxy, Envoy, Apache mod_proxy, Traefik, Caddy.

### 8.1 Por qué los reverse proxies crean bugs

El proxy y el backend son dos parsers HTTP independientes. Siempre que discrepen sobre dónde termina una petición y empieza la siguiente, un atacante puede colar una segunda petición, por delante de todo el control de acceso a nivel de proxy.

### 8.2 Variantes de HTTP request smuggling

- **CL.TE:** el proxy usa `Content-Length`, el backend usa `Transfer-Encoding: chunked`.
- **TE.CL:** el proxy usa `Transfer-Encoding`, el backend usa `Content-Length`.
- **TE.TE:** ambos usan `Transfer-Encoding`, pero a uno se le engaña para que ignore la cabecera mediante una variante ofuscada (`Transfer-Encoding : chunked`, tab, cabecera extra, valor chunked con un casing raro).

### 8.3 Qué consigue CL.TE

En CL.TE el proxy cuenta el cuerpo con `Content-Length` y el backend lo corta con `Transfer-Encoding: chunked`. El resto del cuerpo queda como el inicio de la petición siguiente en la misma conexión, por delante del control de acceso del proxy. El payload no se incluye. El fuente cita `smuggler.py` y la extensión `http-request-smuggler` de Burp como formas de detectar las tres variantes; la invocación no se incluye.

### 8.4 Override de cabeceras

Los reverse proxies a menudo confían en cabeceras internas que vienen de servicios downstream. Si nginx no las elimina, `X-Forwarded-For`, `X-Real-IP`, `X-Original-URL`, `X-Rewrite-URL` o `X-Forwarded-Host` cambian la IP o el path que el backend cree haber recibido. Las peticiones no se incluyen.

## 9. Balanceadores de carga: L4 frente a L7

### 9.1 Capa 4 frente a capa 7

- **Balanceador L4:** enruta solo por IP/puerto. No lee HTTP. Ejemplos: AWS NLB, HAProxy en modo TCP, keepalived.
- **Balanceador L7:** lee cabeceras HTTP, path y cookies. Puede enrutar por host, path o cookie de sesión. Ejemplos: nginx, HAProxy en modo HTTP, AWS ALB, Envoy, F5.

### 9.2 Sesiones sticky

Los balanceadores L7 fijan un cliente a un backend concreto mediante una cookie (`AWSALB`, `JSESSIONID`, un `BACKENDID` a medida) o un hash de IP. Las sesiones sticky filtran dos cosas:

- El número de backends: cookies frescas repetidas producen valores `AWSALB` distintos, uno por backend.
- La identidad del backend: el valor de la cookie a menudo es un hostname interno en base64.

### 9.3 Health checks que filtran paths de backend

Los balanceadores de carga sondean cada backend en una URL de health-check. Si el path se deja en su valor por defecto (`/health`, `/healthz`, `/status`, `/ping`, `/actuator/health`, `/api/health`, `/_status/vars`) y ese endpoint devuelve datos verbosos, el software del backend se enumera desde fuera. El barrido no se incluye.

## 10. Arquitectura de CDN

Cloudflare, Fastly, CloudFront, Akamai y Bunny se sientan entre el usuario y el origen. Cachean, filtran (WAF) y esconden la IP de origen.

### 10.1 Huella de la CDN

```bash
curl -sI https://target/ | grep -iE 'server|cf-ray|x-served-by|x-cache|x-amz-cf-id'
# server: cloudflare               <- Cloudflare
# cf-ray: 8b1c...
# x-served-by: cache-lax-...       <- Fastly
# x-amz-cf-id: ...                 <- CloudFront
```

### 10.2 Envenenamiento de caché mediante cabeceras sin clave

Si una cabecera se refleja en la respuesta cacheada pero no forma parte de la clave de caché, una sola respuesta envenenada se sirve después a cada visitante. El fuente usa `X-Forwarded-Host` como ejemplo y cita param-miner y `web-cache-vulnerability-scanner` como detectores de cabeceras sin clave. La petición y la invocación no se incluyen.

### 10.3 Inyección de cabecera Host

Si el backend arma enlaces absolutos (reset de contraseña, entre otros) a partir de `Host`, el enlace sale apuntando al host de la petición. Combinado con una caché que no incluye esa cabecera en la clave, el enlace llega a otros visitantes. La petición no se incluye.

### 10.4 Descubrimiento del origen

La CDN solo protege la app si la IP de origen es secreta. Dos técnicas la recuperan:

Certificate Transparency + Shodan:

```bash
# Saca todos los certificados emitidos alguna vez para el dominio:
curl -s 'https://crt.sh/?q=%25.example.com&output=json' | \
  jq -r '.[].name_value' | sort -u

# Toma cualquier subdominio que NO esté proxeado por la CDN y resuélvelo:
dig +short old-staging.example.com

# Cruza en Shodan:
shodan search 'ssl:"example.com" -org:"Cloudflare"'
```

Hash del favicon:

```bash
# Calcula el hash de favicon de Shodan del objetivo:
python3 -c "
import mmh3, requests, codecs
r = requests.get('https://target/favicon.ico')
b = codecs.encode(r.content, 'base64')
print(mmh3.hash(b))
"
# Luego busca ese hash en Shodan (cualquier host que sirva el mismo favicon):
shodan search 'http.favicon.hash:-1223143404'
```

## 11. Edge functions

Cloudflare Workers, Lambda@Edge, Vercel Edge Functions y Netlify Edge corren JavaScript/WASM en los nodos de la CDN.

### 11.1 Superficie de ataque

- Filtración en el cold start: la primera petición después de un deploy a menudo tarda segundos y puede devolver salida parcial.
- Mala configuración del almacén KV: Workers KV y DynamoDB Global Tables son eventualmente consistentes; las condiciones de carrera entre ubicaciones de edge son habituales.
- Bypass en el origen: como la edge function solo se hace cumplir en las peticiones que golpean la CDN, descubrir la IP de origen (sección 10.4) te permite saltarte el edge por completo.

### 11.2 Huella

```bash
curl -sI https://target/ | grep -i 'cf-worker\|x-vercel-id\|x-amz-cf-id.*Lambda'
# cf-worker: hello-world
# x-vercel-id: iad1::abcd
```

### 11.3 Sondear un Edge Worker

```bash
# Compara la respuesta procesada por el edge con el origen directo (si conoces la IP):
curl -sI https://target/ | md5sum
curl -sI --resolve target.example.com:443:1.2.3.4 https://target/ | md5sum
# Hashes distintos → el edge está reescribiendo respuestas. Hazles diff para ver qué elimina.
```

## 12. Arquitecturas de gestión de sesión

### 12.1 Sesión de servidor basada en cookie

La clásica. El servidor genera un ID de sesión aleatorio, guarda el estado en el lado del servidor (normalmente Redis) y envía el ID en una cookie. Busca `Set-Cookie: sessionid=...; HttpOnly; Secure; SameSite=Lax`. Sin `HttpOnly`, el XSS la roba. Sin `Secure`, se puede olfatear en la red sobre HTTP. Sin `SameSite`, hay CSRF.

### 12.2 JWT

Token autocontenido, firmado pero a menudo no cifrado. Decodifica en base64 el segmento del medio para leer los claims.

```bash
JWT='eyJhbGciOi...'
echo $JWT | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

El fuente nombra tres fallos clásicos: `alg=none` (la firma se ignora), confusión HS256/RS256 (la clave pública RSA se usa como secret HMAC) y un secret HMAC débil. El procedimiento de cada uno no se incluye.

### 12.3 Almacén de sesión del lado del servidor en Redis

Si el backend guarda los datos de sesión en Redis y Redis está expuesto (puerto 6379, a veces en redes internas alcanzables mediante SSRF), cada sesión activa se puede leer. Los comandos contra Redis no se incluyen.

## 13. CORS en profundidad

Cross-Origin Resource Sharing controla qué orígenes ajenos pueden leer una respuesta.

### 13.1 La combinación letal

```text
Access-Control-Allow-Origin: https://evil.com    <- reflected from Origin header
Access-Control-Allow-Credentials: true
```

Si el servidor refleja cualquier valor de `Origin` y pone `Access-Control-Allow-Credentials: true`, una página de otro origen puede leer las respuestas autenticadas. Dos variantes del mismo fallo: algunas librerías tratan el origen `null` como permitido, y un comodín `*.target.com` acepta un subdominio que no controla la organización. El fuente cita CORScanner para automatizar la comprobación. Las peticiones no se incluyen.

## 14. Content Security Policy

CSP es una cabecera (o una etiqueta `<meta>`) que pone en lista blanca los orígenes de scripts, estilos, conexiones y framing.

### 14.1 Leer una política

```bash
curl -sI https://target/ | grep -i content-security-policy
# content-security-policy: default-src 'self'; script-src 'self' 'nonce-abc123' https://cdn.example.com; object-src 'none'
```

### 14.2 Debilidades habituales

- `script-src 'unsafe-inline'`: se permiten scripts inline; la CSP no hace nada contra el XSS.
- `script-src *`: se salta de forma trivial.
- `script-src 'self'` con un endpoint JSONP en el mismo origen: el callback JSONP es tu gadget de script.
- `script-src https:`: vale cualquier host HTTPS.
- Falta `object-src 'none'`: el contenido Flash/plugin puede ejecutarse.
- Modo report-only: la política solo se registra, nunca se hace cumplir.

### 14.3 Análisis automatizado

```bash
# csp-evaluator (Google, wrapper de CLI):
npm i -g csp-evaluator
csp-evaluator "$(curl -sI https://target/ | grep -i content-security-policy | cut -d: -f2-)"

# Mozilla Observatory escanea toda la suite de cabeceras de seguridad:
curl -s "https://http-observatory.security.mozilla.org/api/v1/analyze?host=target.example.com" -X POST
```

## 15. Fingerprinting de una arquitectura de punta a punta

Junta todo en una sola pasada de reconocimiento.

### 15.1 El one-liner de reconocimiento

```bash
TARGET=target.example.com
whatweb -a 3 https://$TARGET
npx wappalyzer https://$TARGET --pretty
curl -sI https://$TARGET | tee headers.txt
grep -iE 'server|x-powered-by|cf-ray|x-vercel|x-amz-cf|cache-control|set-cookie' headers.txt

# Recolección de bundles JS:
curl -s https://$TARGET/ | grep -oE '(src|href)="[^"]+\.(js|css)"' > assets.txt
```

### 15.2 Caza de source maps

```bash
for js in $(cat assets.txt | grep -oE '[^"]+\.js'); do
  code=$(curl -sI "https://$TARGET$js" -o /dev/null -w "%{http_code}")
  map=$(curl -sI "https://$TARGET$js.map" -o /dev/null -w "%{http_code}")
  echo "$js js=$code map=$map"
done
```

### 15.3 Extracción de endpoints de API

```bash
mkdir -p bundles && cd bundles
curl -s https://$TARGET/ | grep -oE 'src="[^"]+\.js"' | \
  sed 's/src="//;s/"$//' | \
  xargs -I{} curl -s "https://$TARGET{}" -o "$(basename {})"

grep -hoE '"/api/[a-zA-Z0-9_/{}.-]+"' ./*.js | sort -u > endpoints.txt
wc -l endpoints.txt
```

### 15.4 Hash del favicon

```bash
python3 -c "
import mmh3, requests, codecs
r = requests.get('https://$TARGET/favicon.ico', timeout=10)
b = codecs.encode(r.content, 'base64')
print('Shodan favicon hash:', mmh3.hash(b))
"
```

## 16. Práctica: ataca una app Next.js SSR de laboratorio

Este ejercicio junta cada sección. El objetivo del laboratorio es una app Next.js 14 vulnerable a propósito, corriendo bajo nginx, desplegada en `http://lab-nextjs.pentest.local:3000` dentro del contenedor del curso.

### 16.1 Levanta el laboratorio

```bash
cd /opt/labs/module-04
docker compose up -d lab-nextjs nginx-proxy
docker compose ps
# lab-nextjs   Up   3000/tcp
# nginx-proxy  Up   0.0.0.0:8080->80/tcp
```

### 16.2 Paso 1: huella del stack

```bash
TARGET=http://lab-nextjs.pentest.local:8080
curl -sI $TARGET/ | tee step1.txt
# server: nginx/1.25.3
# x-powered-by: Next.js
# x-nextjs-cache: MISS

whatweb -a 3 $TARGET
# Detects: Next.js, React, nginx
```

Veredicto: Next.js SSR detrás de nginx. Espera SSRF/SSTI en `getServerSideProps`, IDOR en `/api/*` y bugs de override de cabeceras en nginx.

### 16.3 Qué busca el resto del laboratorio

El fuente sigue con tres pruebas. Los comandos no se incluyen. El impacto de cada una es este:

- Los chunks de `/_next/static/` citan rutas `/api/users/[id]`, `/api/orders/[id]` y `/api/admin/config`.
- `/api/orders/{id}` devuelve el pedido de otro usuario con el token de Alice. El fuente lo marca como IDOR.
- nginx reenvía `Host` sin cambios y `getServerSideProps` arma URLs absolutas con esa cabecera. Un reset de contraseña sale apuntando al host que puso quien hace la petición. Si delante hay caché ISR, el mismo enlace envenenado se sirve a más gente.

### 16.4 Entregable

Redacta los cuatro hallazgos (huella, IDOR, enumeración de endpoints, inyección de cabecera Host) en el formato de `Module-03/Lesson-05-Report-Writing.md`. Describe el impacto y el control que lo cierra.

## 17. Ejercicios

### 17.1 Ejercicios guiados

**Ejercicio 1 — Identificación del modelo de renderizado**

Dados tres objetivos:

```text
https://blog.example.com/
https://app.example.com/
https://shop.example.com/
```

Usa curl, whatweb y dev-tools para identificar si cada uno es MPA, SPA, SSR, SSG o ISR. Justifica cada respuesta con dos piezas de evidencia de la respuesta HTTP.

**Ejercicio 2 — Introspección de GraphQL y coste**

Apunta graphw00f a `http://lab-graphql.pentest.local:4000/graphql`. Confirma el servidor. Corre la consulta de introspección, guarda el SDL en `schema.graphql` e identifica los tres resolvers con pinta de más caros. Anota por qué cada uno es caro (listas anidadas, resolvers sin límite). No construyas la consulta de alias que el fuente usa para medir el tiempo.

**Ejercicio 3 — Huella de reverse proxy**

Sobre `http://lab-smuggle.pentest.local:8080/`, anota a cuál de las tres variantes (CL.TE, TE.CL, TE.TE) apunta el laboratorio y qué parser (proxy o backend) se queda con cada cabecera. No pegues el payload.

### 17.2 Ejercicios independientes

**Ejercicio 4 — Informe de huella del stack completo**

Elige cualquier programa de bug bounty en HackerOne que esté in-scope para ti. Produce un informe de reconocimiento de una página que cubra: framework, modo de render, estilo de API, CDN, reverse proxy, modelo de sesión, postura de CORS y nota de CSP. No envíes un solo payload: solo reconocimiento.

**Ejercicio 5 — Descubrimiento del origen**

Elige un objetivo con Cloudflare delante y con tu permiso por escrito. Usando crt.sh, el hash de favicon de Shodan y cualquier fuente de DNS histórico, intenta recuperar la IP de origen real. Documenta cada paso y el razonamiento que hay detrás.

**Ejercicio 6 — Envenenamiento de caché**

El laboratorio `lab-cache.pentest.local` usa una cabecera `X-Forwarded-Host` que no entra en la clave de caché. El impacto es que la respuesta guardada se sirve después a otro visitante. Anota qué cabecera queda fuera de la clave y qué control la mete dentro (o la elimina en el proxy). No incluyas el payload.

## 18. Resumen y puntero a la lección siguiente

Esta lección sustituyó el enfoque de caja negra de «corre el escáner y espera» por una lectura arquitectónica sistemática de una aplicación web moderna. Ya sabes mirar respuestas HTTP, bundles de JavaScript y cabeceras de caché y reconstruir el stack entero en la cabeza: desde el edge de Cloudflare, a través del reverse proxy nginx, hacia el nodo SSR de Next.js, hacia el endpoint GraphQL y hasta el almacén de sesión en Redis.

Los cinco modelos de render (MPA, SPA, SSR, SSG, ISR) invitan cada uno a una clase distinta de bug. REST y GraphQL exigen tooling distinto y primeros payloads distintos. Los WebSockets y gRPC-Web son mundos propios. Los reverse proxies habilitan el smuggling; las CDN habilitan el envenenamiento de caché; las edge functions se pueden saltar si se filtra el origen; y las malas configuraciones de CORS y de CSP se encuentran leyendo dos cabeceras, no haciendo fuzzing.

La lección siguiente —Lección 08 — Container Escape Techniques and Kubernetes Attack Surface— gira de la capa de aplicación al runtime que la hospeda. Vas a tomar el mismo objetivo de la sección 16, salir de su contenedor Docker, pivotar hacia el plano de control de Kubernetes y demostrar por qué la comprensión arquitectónica en la capa de contenedores completa el cuadro.
