# Lección 12 — Lab: AI Setup and Prompt Engineering Exercises

Módulo 05: Arquitectura de IA, Prompt Engineering e IA Local · Penetration Testing con Kali Linux 1.0 · Red Team Leaders

Vas a terminar este lab con un stack de IA on-prem chico corriendo en tu propio hardware — Ollama, ChromaDB, Open WebUI — más cuatro tools de Python que leen XML de Nmap, respuestas HTTP y archivos YAML de findings, y producen resúmenes listos para un pentest.

Todo corre en local, así puedes usarlo en engagements confidenciales del cliente sin filtrar datos a un modelo hosteado. La próxima vez que un cliente objete mandar logs a OpenAI o a Anthropic, ya vas a tener la alternativa.

## Diagrama del módulo

Módulo 05: Arquitectura de IA, Prompt Engineering e IA Local · Penetration Testing con Kali Linux 1.0 · Red Team Leaders

## 1. Introducción

El módulo 05 te llevó por las capas de arquitectura, de operación y de prompt engineering del workflow moderno de seguridad ofensiva aumentado con LLM. Cada lección anterior enseñó una tajada aislada: una sobre la intuición de los transformers, otra sobre system prompts y diseño few-shot, otra sobre tokenization y context windows, otra sobre retrieval-augmented generation, otra sobre correr modelos locales con Ollama, otra sobre el protocolo MCP, otra sobre patrones chain-of-thought y ReAct, otra sobre evaluación y guardrails.

Esta lección consolida el módulo. Vas a armar de punta a punta cuatro tools de pentest aumentadas con IA, de grado de producción. Cuando el lab termine deberías poder hacer `chmod +x` a cada una, dejarla en tu repositorio git personal `~/pentest-tools/` y alcanzarla durante las fases de enumeración, explotación y reporting de un engagement real. No son demos descartables. Cada módulo siguiente del curso asume que las tienes y te pide extenderlas. El módulo 06 (AI-Augmented Reconnaissance) conecta de forma explícita la tool de triage del desafío 3 a un pipeline de recon automatizado, y el módulo 12 (AI-Augmented Reporting) reutiliza el escritor de findings del desafío 4.

Las cuatro tools son:

- **pentest-assistant** — un LLM local especializado, encima de `llama3.1:8b`, con un system prompt enfocado en seguridad, entregado como un Modelfile de Ollama.
- **ragq** — una CLI de Python que indexa el OWASP Top 10 y un corpus chico de CVE en ChromaDB, recupera chunks relevantes con `nomic-embed-text` y responde preguntas de pentest con un modelo local de Ollama.
- **recon-triage** — una tool de Python que lee XML de Nmap, llama a un LLM local con un prompt de triage estructurado y devuelve hosts ordenados por valor de ataque probable, en JSON.
- **finding-writer** — una tool de Python que lee un YAML crudo de finding y redacta una card completa en Markdown, con executive summary, detalles técnicos, impact y remediación. Llama a la API de Claude Sonnet 5, con un fallback ordenado a un modelo local de Ollama cuando la API no está.

Cada desafío está acotado para que la implementación de referencia corra en Kali 2026 de entrada, y cada uno cierra con stretch goals que convierten la tool en algo que de verdad querrías en una operación real. Lee los requisitos, intenta escribirlo tú, y después compara contra la referencia. No copies y listo: escríbelo, rómpelo, entiéndelo. La rúbrica al final de esta lección es cómo se califican las tools cuando las entregas al cierre del módulo.

Una advertencia breve antes de arrancar: los modelos locales no son deterministas. Dos corridas del mismo prompt contra los mismos weights no producen output idéntico byte a byte. La rúbrica evalúa comportamiento, estructura y seguridad de tus outputs, no matches de strings. Cuando armes tests, afirma sobre la forma y sobre invariantes de contenido, no sobre strings exactos.

El trabajo es de un engagement autorizado. Las tools rechazan targets que el operador no nombró y que no están en el RoE.

## 2. Setup del entorno de lab

### 2.1 Requisitos del host

- VM de Kali Linux 2026.1 (fresca del módulo 01; mejor un snapshot)
- 16 GB de RAM como mínimo, 24 GB recomendados (los LLM tienen hambre de memoria)
- 50 GB de disco libre para los modelos
- Una CPU con AVX2 (cualquier Intel/AMD de la última década), o una GPU NVIDIA con 8 GB o más de VRAM si quieres latencia tolerable en `llama3.1:8b`
- HTTPS de salida, para que `ollama pull` pueda bajar los weights y la llamada a la API de Anthropic del desafío 4 pueda llegar a `api.anthropic.com`

Haz snapshot de la VM antes de empezar a bajar modelos. Si arruinas un Modelfile o un virtualenv, quieres volver atrás rápido.

### 2.2 Toolchain del sistema

Corre esto una vez como tu usuario normal (no root) en la VM de Kali:

```bash
sudo apt update
sudo apt install -y \
    python3 python3-venv python3-pip \
    build-essential git jq curl wget \
    tmux nmap yq

python3 --version   # 3.11+
git --version
```

### 2.3 Instalación de Ollama y pull de modelos

Ollama es el runner local de modelos. Instálalo directo desde upstream: el repo de Kali va atrasado.

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Confirma que corre como servicio de systemd y responde en la API de loopback:
systemctl status ollama --no-pager
curl -s http://127.0.0.1:11434/api/tags | jq .
```

Baja los tres modelos que esta lección necesita. El primer pull es lento. Anda por un café.

```bash
# Modelo de chat que usa cada desafío para generar.
ollama pull llama3.1:8b

# Un modelo un poco más pesado, usado como fallback en el desafío 4.
ollama pull llama3.1:8b-instruct-q5_K_M

# Modelo de embeddings que usa el desafío de RAG (desafío 2).
ollama pull nomic-embed-text
```

Chequeo de cordura de que la generación funciona, antes de escribir Python. El stdout del modelo puede salir en el idioma del prompt:

```bash
ollama run llama3.1:8b "En una oración, ¿qué es una reverse shell?"
```

### 2.4 Workspace de Python y dependencias

Crea el workspace y el virtualenv de Python que van a compartir todas las tools de esta lección:

```bash
mkdir -p ~/pentest-tools
cd ~/pentest-tools
git init
git branch -M main

python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip

pip install \
    anthropic==0.40.0 \
    openai==1.55.3 \
    ollama==0.4.4 \
    chromadb==0.5.23 \
    pyyaml==6.0.2 \
    rich==13.9.4 \
    typer==0.15.1 \
    lxml==5.3.0 \
    tenacity==9.0.0 \
    tiktoken==0.8.0

pip freeze > requirements.txt
git add requirements.txt
git commit -m "chore: pin baseline AI toolchain for Module 05 lab"
```

### 2.5 Credenciales de API

El desafío 4 llama a la API de Claude Sonnet 5. Pon tu key en un archivo estilo `.env` que no se commitea:

```bash
mkdir -p ~/.config/pentest-tools
cat > ~/.config/pentest-tools/env <<'EOF'
ANTHROPIC_API_KEY=sk-ant-...reemplazame...
ANTHROPIC_MODEL=claude-sonnet-5-20260101
OLLAMA_HOST=http://127.0.0.1:11434
OLLAMA_CHAT_MODEL=llama3.1:8b
OLLAMA_EMBED_MODEL=nomic-embed-text
EOF
chmod 600 ~/.config/pentest-tools/env
```

Cárgalo en cada shell que corra las tools del lab:

```bash
set -a
. ~/.config/pentest-tools/env
set +a
```

Agrega un `.gitignore` al workspace antes de commitear cualquier otra cosa:

```bash
cat > ~/pentest-tools/.gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
*.log
*.env
.env
chroma-db/
outputs/
EOF
git add .gitignore
git commit -m "chore: gitignore venv, cache, chroma, outputs"
```

### 2.6 Layout de directorios

Cada desafío vive en su propio subdirectorio de `~/pentest-tools/`:

```text
~/pentest-tools/
├── .venv/
├── .gitignore
├── requirements.txt
├── pentest-assistant/           # Desafío 1
│   ├── Modelfile
│   └── test_assistant.py
├── ragq/                        # Desafío 2
│   ├── corpus/
│   ├── index.py
│   └── ragq.py
├── recon-triage/                # Desafío 3
│   ├── triage.py
│   └── samples/
└── finding-writer/              # Desafío 4
    ├── write_finding.py
    └── samples/
```

Crea el esqueleto ahora para poder dejar los archivos directo:

```bash
cd ~/pentest-tools
mkdir -p pentest-assistant ragq/corpus recon-triage/samples finding-writer/samples
```

## 3. Desafío 1 — Modelfile de pentest-assistant en Ollama

### 3.1 Requisitos

Arma un LLM local especializado que puedas invocar como `ollama run pentest-assistant "..."`. El Modelfile tiene que:

- Usar `llama3.1:8b` como base model.
- Poner la temperature de sampling en 0.3, para que el asistente sea lo bastante determinista para trabajo de seguridad ofensiva (la creatividad es un pasivo cuando estás enumerando hosts).
- Poner `num_ctx` en 8192, para que outputs largos de Nmap, respuestas HTTP y snippets de código quepan en un turno.
- Poner `top_p` en 0.9 y `repeat_penalty` en 1.15.
- Llevar un system prompt enfocado en seguridad que:
  - Establezca la persona como un operador senior de seguridad ofensiva que apoya un engagement autorizado.
  - Exija citar IDs de técnicas de MITRE ATT&CK cuando corresponda.
  - Exija que el modelo rechace ayudar con ataques contra sistemas que el usuario no confirmó como in-scope, y que pida las Rules of Engagement (RoE) del engagement si el scope es ambiguo.
  - Exija respuestas estructuradas con las secciones: `Summary`, `Steps`, `Detection`, `Cleanup`.
- Agregar al menos un ejemplo few-shot `MESSAGE user` / `MESSAGE assistant` para que el modelo se enganche al formato de output.
- Buildear limpio con `ollama create pentest-assistant -f Modelfile`.

Entregables: el Modelfile, la invocación de `ollama create` y un `test_assistant.py` de Python que corre tres prompts de sonda y afirma la forma del output.

Los encabezados `## Summary`, `## Steps`, `## Detection` y `## Cleanup` se quedan en inglés: `test_assistant.py` los busca como strings.

### 3.2 Implementación de referencia

`~/pentest-tools/pentest-assistant/Modelfile`:

```text
# pentest-assistant — un LLM local especializado para engagements autorizados
# de seguridad ofensiva. Construido sobre llama3.1:8b. No es para sistemas
# fuera de scope.

FROM llama3.1:8b

PARAMETER temperature 0.3
PARAMETER num_ctx 8192
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.15
PARAMETER stop "<|user|>"
PARAMETER stop "<|assistant|>"

SYSTEM """
Eres pentest-assistant, un operador senior de seguridad ofensiva embebido en
un red team de un engagement *autorizado*. El operador ("usuario") es un
penetration tester profesional que firmó un documento de Rules of Engagement
(RoE) con el cliente. Solo apoyas trabajo que cae dentro de ese RoE.

Reglas de base que debes seguir en cada turno:

1. Asume que la autorización está en pie salvo que el usuario te dé un target
   que la contradiga (por ejemplo .gov, .mil, ".edu de una escuela que no está
   nombrada en el RoE", sistemas de producción nombrados de empresas conocidas).
   Cuando la autorización sea ambigua, pídele al usuario la lista de targets
   in-scope antes de seguir.
2. Nunca des guía para ataques contra personas, sistemas electorales,
   infraestructura crítica que el usuario no confirmó como in-scope, ni
   ningún sistema de una parte que el usuario no confirmó como cliente.
3. Cita IDs de técnicas de MITRE ATT&CK (por ejemplo T1059.001, T1021.006)
   cada vez que un paso mapee a una. Eso le deja al usuario reconciliar los
   findings contra la postura de detección del cliente.
4. Prefiere técnicas que respeten el tier de Impact que declaró el cliente.
   Si el usuario no declaró un tier, asume "Impact bajo — sin disrupción de
   servicio".
5. No fabriques números de CVE, flags de tools ni paths de archivos. Si no
   estás seguro, dilo y sugiere un paso de verificación.

Estructura cada respuesta sustantiva con estas cuatro secciones, en orden.
Los encabezados van exactamente así:

    ## Summary
    Una o dos oraciones, en español claro, de lo que estás por hacer.

    ## Steps
    Una lista numerada de comandos o acciones concretas, copiables. Marca
    la técnica de ATT&CK en cada paso cuando aplique, por ejemplo "(T1046)".

    ## Detection
    Qué vería el blue team si esto se corriera contra un entorno monitoreado.
    Nombra la fuente de log (event ID de Sysmon, categoría de regla de
    Suricata, telemetría de EDR) cuando puedas.

    ## Cleanup
    Qué artefactos dejas atrás y cómo quitarlos (archivos, usuarios, claves
    de registry, scheduled tasks, sockets de red, historial de shell).

Para preguntas conceptuales y no de procedimiento, conserva las cuatro
secciones y deja que "Steps" sea la explicación. Nunca te saltees una sección.
"""

TEMPLATE """{{ if .System }}<|system|>
{{ .System }}<|end|>{{ end }}{{ if .Prompt }}<|user|>
{{ .Prompt }}<|end|>{{ end }}<|assistant|>
{{ .Response }}<|end|>"""

MESSAGE user "Nmap dice que un host tiene TCP 445 abierto en Windows Server 2019. ¿Cuál es mi primer movimiento?"
MESSAGE assistant """## Summary
Enumerar el servicio SMB para tomar la huella del build del OS, listar shares
accesibles por null session y sondear CVEs de SMB conocidos y relevantes para
Server 2019.

## Steps
1. Toma el dialecto SMB y el build del OS (T1046):
   `nmap -p 445 --script smb-protocols,smb-os-discovery <TARGET>`
2. Enumera shares con null session y después con cualquier credencial que ya tengas (T1135):
   `smbclient -N -L //<TARGET>/`
   `nxc smb <TARGET> -u '' -p '' --shares`
3. Chequea la postura de SMB signing — required frente a not required (T1557.001):
   `nxc smb <TARGET> --gen-relay-list relay.txt`
4. Busca residuo de MS17-010 (raro en 2019, y barato de chequear) (T1210):
   `nmap -p 445 --script smb-vuln-ms17-010 <TARGET>`

## Detection
- El evento 5145 de Windows (acceso a share) se dispara en el paso 2 con logon anónimo.
- El evento 3 de Sysmon (network connect) sobre la IP de origen del atacante.
- Variantes de Suricata "ET SCAN Behavioral Unusually fast Terminal Server Traffic"
  se disparan si el scan va demasiado rápido.

## Cleanup
- No se escriben archivos en el target si es solo enumeración.
- Borra el output local del scan cuando hagas snapshot de los findings:
  `rm -f nmap.out smb-relay.txt` y desactiva el historial de shell de la sesión
  con `unset HISTFILE` antes de empezar."""
```

El ejemplo es enumeración sobre un target que ya está en el RoE. `<TARGET>` no es un host real. El paso 3 arma una lista para relay: no lanza el relay. Si el tier de impact es bajo, el operador se queda en la enumeración.

Build y registro del modelo:

```bash
cd ~/pentest-tools/pentest-assistant
ollama create pentest-assistant -f Modelfile
ollama list | grep pentest-assistant
```

Output esperado. Es el stdout de `ollama list`, en inglés:

```text
NAME                       ID              SIZE      MODIFIED
pentest-assistant:latest   a1b2c3d4e5f6    4.7 GB    a few seconds ago
```

Smoke test rápido en la CLI:

```bash
ollama run pentest-assistant "Tengo un foothold en una caja Linux como www-data. ¿Dónde miro primero para un privesc local?"
```

Deberías ver de vuelta las cuatro secciones obligatorias. Si falta alguna, aprieta el prompt `SYSTEM`. El prompt del modelo es código, y falla como código.

### 3.3 test_assistant.py

`~/pentest-tools/pentest-assistant/test_assistant.py`:

```python
#!/usr/bin/env python3
"""Smoke test del modelo de Ollama pentest-assistant.

Corre tres prompts de sonda y afirma que cada respuesta contiene las cuatro
secciones obligatorias. Usa el cliente oficial de Python de Ollama para
ejercer el mismo camino de código que van a usar los callers de producción.
"""

from __future__ import annotations

import os
import sys
from dataclasses import dataclass

import ollama

MODEL = os.environ.get("PA_MODEL", "pentest-assistant")
HOST = os.environ.get("OLLAMA_HOST", "http://127.0.0.1:11434")

REQUIRED_SECTIONS = ("## Summary", "## Steps", "## Detection", "## Cleanup")

PROBES = [
    "Tengo un foothold en una caja Linux como www-data. ¿Dónde miro primero para un privesc local?",
    "Una app web devuelve 500 en ?id=1' y 200 en ?id=1. ¿Cómo confirmo SQLi?",
    "Nmap muestra TCP/5985 abierto en un host Windows. Guíame por el acceso inicial.",
]


@dataclass
class ProbeResult:
    prompt: str
    ok: bool
    missing: list[str]
    excerpt: str


def run_probe(prompt: str) -> ProbeResult:
    client = ollama.Client(host=HOST)
    resp = client.chat(
        model=MODEL,
        messages=[{"role": "user", "content": prompt}],
    )
    body = resp["message"]["content"]
    missing = [s for s in REQUIRED_SECTIONS if s not in body]
    return ProbeResult(
        prompt=prompt,
        ok=not missing,
        missing=missing,
        excerpt=body[:400] + ("..." if len(body) > 400 else ""),
    )


def main() -> int:
    results = [run_probe(p) for p in PROBES]
    failures = [r for r in results if not r.ok]

    for r in results:
        status = "PASS" if r.ok else "FAIL"
        print(f"[{status}] {r.prompt}")
        if r.missing:
            print(f"       faltan secciones: {', '.join(r.missing)}")
        print(f"       excerpt: {r.excerpt}\n")

    if failures:
        print(f"{len(failures)}/{len(results)} sondas fallaron el chequeo de secciones.")
        return 1
    print(f"Las {len(results)} sondas devolvieron las secciones obligatorias.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Correrlo:

```bash
cd ~/pentest-tools/pentest-assistant
python test_assistant.py
```

Camino feliz esperado. `PASS` se queda en inglés porque lo imprime el script. El excerpt es narrativo:

```text
[PASS] Tengo un foothold en una caja Linux como www-data...
       excerpt: ## Summary\nEn un foothold fresco de www-data, ...

[PASS] Una app web devuelve 500 en ?id=1' y 200 en ?id=1...
       excerpt: ## Summary\nLos status codes divergentes sugieren ...

[PASS] Nmap muestra TCP/5985 abierto en un host Windows...
       excerpt: ## Summary\nTCP/5985 es WinRM sobre HTTP. ...

Las 3 sondas devolvieron las secciones obligatorias.
```

### 3.4 Stretch goals

- Agrega un segundo Modelfile que herede de `pentest-assistant` y fuerce "solo enumeración de lectura" para operadores junior.
- Agrega un `PARAMETER seed` para que el modelo sea determinista en unit tests, y agrega tests que afirmen que aparecen IDs concretos de ATT&CK en el output.
- Envuelve `ollama run pentest-assistant` como un widget de zsh atado a una tecla, para consultarlo desde cualquier panel de tmux.

## 4. Desafío 2 — Base de conocimiento RAG de pentest

### 4.1 Requisitos

Arma `ragq`, una CLI de Retrieval-Augmented Generation. Tiene que:

- Ingerir un corpus chico que contenga:
  - El OWASP Top 10 (2021), con una descripción corta de cada ítem.
  - Al menos ocho descripciones de CVE (título, CVSS, descripción técnica breve).
- Hacer chunking del corpus y embeddear cada chunk con el modelo local `nomic-embed-text` de Ollama.
- Persistir los vectores y la metadata en ChromaDB bajo `~/pentest-tools/ragq/chroma-db/`.
- Exponer una CLI de Typer con dos comandos:
  - `index` — (re)construye el vector store desde `corpus/`.
  - `ask "pregunta"` — recupera los top-k chunks (default `k=4`), se los pasa junto con la pregunta al modelo local `llama3.1:8b` e imprime una respuesta citada.
- Cada respuesta tiene que imprimir los source ids de los chunks que usó, para que el operador pueda auditar la alucinación.
- Rechazar preguntas que están fuera del scope del corpus (por ejemplo, preguntas de impuestos al azar). El modelo tiene que decirlo, no adivinar.

Entregables: `corpus/` poblado, `index.py`, `ragq.py`. Objetivo: unas 200 líneas de Python en total.

La frase de refusal del prompt es `Not covered by the corpus`. El chequeo manual del lab la busca así. El resto de las instrucciones va en español.

### 4.2 Archivos del corpus

Crea los archivos del corpus bajo `~/pentest-tools/ragq/corpus/`.

`corpus/owasp-top10-2021.md`:

```markdown
# OWASP Top 10 — 2021

## A01:2021 - Broken Access Control
El access control impone política de forma que los usuarios no puedan actuar
fuera de los permisos previstos. Las fallas suelen llevar a divulgación,
modificación o destrucción no autorizada de datos. Vectores comunes: IDOR,
forced browsing a páginas autenticadas, manipulación de JWT y mala
configuración de CORS.

## A02:2021 - Cryptographic Failures
Antes "Sensitive Data Exposure". Las causas de raíz son transmitir o guardar
datos sensibles sin cifrado, usar algoritmos débiles o deprecados
(MD5, SHA-1, DES, RC4) y manejar mal las keys. Las consecuencias incluyen
robo de credenciales, secuestro de sesión y breach de datos regulados.

## A03:2021 - Injection
Incluye injection de SQL, NoSQL, comandos de OS, LDAP y expression language.
Ocurre cuando datos no confiables se mandan a un intérprete como parte de un
comando o una query. XSS se plegó acá en 2021. Mitigaciones: queries
parametrizadas, validación positiva de input del lado servidor, escape del
output.

## A04:2021 - Insecure Design
Una categoría nueva, enfocada en diseño de controles ausente o ineficaz.
Distinta de las fallas de implementación: incluso un diseño inseguro
implementado a la perfección es explotable. Threat modeling, patrones de
diseño seguro y arquitecturas de referencia son las contramedidas.

## A05:2021 - Security Misconfiguration
Cuentas default, mensajes de error verbosos, features innecesarias
habilitadas, security headers ausentes, software desactualizado. Los
servicios cloud y las plataformas de contenedores ampliaron la superficie.
XXE se unió a esta categoría en 2021.

## A06:2021 - Vulnerable and Outdated Components
Usar componentes con CVEs conocidos. Incluye dependencias directas y
transitivas. Contramedidas: SBOM, pinning de dependencias, escaneo continuo
de vulnerabilidades (OWASP Dependency-Check, Snyk, Trivy).

## A07:2021 - Identification and Authentication Failures
Credenciales débiles, credential stuffing, MFA ausente, manejo inseguro de
sesión, session IDs expuestos, endpoints de fuerza bruta sin lockout.
Contramedidas: MFA, política fuerte de contraseñas, detección de credential
stuffing, vida corta de sesión, flags seguros de cookie.

## A08:2021 - Software and Data Integrity Failures
Compromiso del pipeline de CI/CD, actualizaciones de software sin firma,
deserialización de datos no confiables. CVEs relevantes incluyen RCE al
estilo Log4Shell a través de strings de lookup armados, e implantes de
supply chain al estilo SolarWinds.

## A09:2021 - Security Logging and Monitoring Failures
Logging ausente o inadecuado, sin alertas, sin incident response. Aumenta el
dwell time. La ingeniería de detección tiene que cubrir, como mínimo, eventos
de autenticación, fallas de access control y fallas de validación de input
del lado servidor.

## A10:2021 - Server-Side Request Forgery (SSRF)
La aplicación trae un recurso remoto sin validar la URL que suministró el
usuario. Especialmente peligroso en entornos cloud, donde el metadata service
(169.254.169.254) emite credenciales de vida corta. Mitigaciones: allow-list
de URLs, deshabilitar redirects, segmentar el egress de red.
```

`corpus/cve-notes.md`. Son fichas de impacto para el KB, no procedimientos de explotación:

```markdown
# Notas de CVE — vulnerabilidades seleccionadas de alto impacto

## CVE-2021-44228 — Log4Shell
Categoría: Software and Data Integrity Failures (A08).
CVSSv3: 10.0.
Apache Log4j 2, versiones 2.0-beta9 hasta 2.14.1, evalúa strings de lookup
JNDI que aparecen en mensajes logueados. Un atacante que puede hacer que una
aplicación vulnerable loguee un string bajo su control (User-Agent,
X-Forwarded-For, input de un form) puede lograr ejecución remota de código
sin autenticación, vía un callback LDAP o RMI.

## CVE-2017-0144 — EternalBlue (MS17-010)
Categoría: Vulnerable and Outdated Components (A06).
CVSSv3: 8.1.
SMBv1 en Microsoft Windows permite a un atacante remoto ejecutar código vía
paquetes armados que manejan mal las estructuras de transacción. Base de
WannaCry y NotPetya. Parcheado en MS17-010. Un Windows Server 2003/2008
legacy sin parche queda trivialmente comprometido.

## CVE-2014-0160 — Heartbleed
Categoría: Cryptographic Failures (A02).
CVSSv3: 7.5.
OpenSSL 1.0.1 hasta 1.0.1f parsea mal el largo de la extensión heartbeat de
TLS, y permite a un atacante remoto leer hasta 64 KB de memoria del proceso
por request. Filtró private keys y cookies de sesión. Mitigación: actualizar
OpenSSL, rotar keys, revocar certificados.

## CVE-2019-0708 — BlueKeep
Categoría: Vulnerable and Outdated Components (A06).
CVSSv3: 9.8.
Use-after-free pre-auth de RDP de Windows en versiones viejas (7, Server 2008
R2, XP, Server 2003). Propagable como worm. Habilitar NLA y parchear, o
bloquear TCP/3389.

## CVE-2021-34527 — PrintNightmare
Categoría: Security Misconfiguration (A05).
CVSSv3: 8.8.
El Print Spooler de Windows permite a un atacante remoto autenticado cargar
una DLL arbitraria vía llamadas RPC armadas, y ganar SYSTEM. Mitigaciones:
deshabilitar el Print Spooler en servidores que no lo necesitan, restringir
Point and Print.

## CVE-2020-1472 — Zerologon
Categoría: Identification and Authentication Failures (A07).
CVSSv3: 10.0.
Netlogon MS-NRPC usa AES-CFB8 con un IV de todo ceros, y deja que un atacante
no autenticado en la red ponga vacía la contraseña de la cuenta de máquina
del domain controller, escalando a Domain Admin. Parchear y monitorear los
eventos 4742 de Netlogon.

## CVE-2022-30190 — Follina
Categoría: Injection (A03).
CVSSv3: 7.8.
El handler de URL de Microsoft Support Diagnostic Tool (MSDT) ejecuta
PowerShell suministrado por el atacante cuando un documento de Word referencia
un URI ms-msdt:. Se entrega vía phishing. Workaround: deshabilitar el handler
ms-msdt en el registry.

## CVE-2023-23397 — Outlook NTLM Leak
Categoría: Broken Access Control (A01).
CVSSv3: 9.8.
Microsoft Outlook procesa una invitación de reunión armada que referencia un
path UNC en PidLidReminderFileParameter, y hace que el cliente mande NTLMv2
a un SMB controlado por el atacante. Se puede reenviar a servicios internos.
Zero-click.
```

### 4.3 index.py — construir el vector store

`~/pentest-tools/ragq/index.py`:

```python
#!/usr/bin/env python3
"""Construye (o reconstruye) el vector store de ChromaDB de ragq desde ./corpus/*.md."""

from __future__ import annotations

import os
import re
import sys
from pathlib import Path

import chromadb
import ollama

CORPUS_DIR = Path(__file__).parent / "corpus"
DB_DIR = Path(__file__).parent / "chroma-db"
COLLECTION = "pentest-kb"

EMBED_MODEL = os.environ.get("OLLAMA_EMBED_MODEL", "nomic-embed-text")
OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://127.0.0.1:11434")


def chunk_markdown(text: str) -> list[tuple[str, str]]:
    """Corta en encabezados de nivel 2, así cada chunk es un ítem OWASP o un CVE."""
    parts = re.split(r"(?=^## )", text, flags=re.MULTILINE)
    chunks: list[tuple[str, str]] = []
    for part in parts:
        part = part.strip()
        if not part or not part.startswith("## "):
            continue
        first_line, _, _ = part.partition("\n")
        title = first_line.removeprefix("## ").strip()
        chunks.append((title, part))
    return chunks


def embed(client: ollama.Client, text: str) -> list[float]:
    resp = client.embeddings(model=EMBED_MODEL, prompt=text)
    return resp["embedding"]


def main() -> int:
    if not CORPUS_DIR.exists():
        print(f"falta el directorio de corpus: {CORPUS_DIR}", file=sys.stderr)
        return 2

    files = sorted(CORPUS_DIR.glob("*.md"))
    if not files:
        print(f"no hay archivos markdown en {CORPUS_DIR}", file=sys.stderr)
        return 2

    DB_DIR.mkdir(parents=True, exist_ok=True)
    chroma = chromadb.PersistentClient(path=str(DB_DIR))
    if COLLECTION in [c.name for c in chroma.list_collections()]:
        chroma.delete_collection(COLLECTION)
    coll = chroma.create_collection(
        name=COLLECTION,
        metadata={"hnsw:space": "cosine"},
    )

    oll = ollama.Client(host=OLLAMA_HOST)

    ids: list[str] = []
    docs: list[str] = []
    metas: list[dict] = []
    vecs: list[list[float]] = []

    for f in files:
        for i, (title, body) in enumerate(chunk_markdown(f.read_text())):
            chunk_id = f"{f.stem}::{i:03d}"
            ids.append(chunk_id)
            docs.append(body)
            metas.append({"source": f.name, "title": title})
            vecs.append(embed(oll, body))
            print(f"  embedded {chunk_id} — {title}")

    coll.add(ids=ids, documents=docs, metadatas=metas, embeddings=vecs)
    print(f"\nindexados {len(ids)} chunks en {DB_DIR}/{COLLECTION}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Correrlo:

```bash
cd ~/pentest-tools/ragq
python index.py
```

Cola esperada del output:

```text
  embedded cve-notes::007 — CVE-2023-23397 — Outlook NTLM Leak
indexados 18 chunks en /home/kali/pentest-tools/ragq/chroma-db/pentest-kb
```

### 4.4 ragq.py — la CLI

`~/pentest-tools/ragq/ragq.py`:

```python
#!/usr/bin/env python3
"""ragq — pregunta de pentest contra una base de conocimiento RAG local."""

from __future__ import annotations

import os
import subprocess
import sys
from pathlib import Path

import chromadb
import ollama
import typer
from rich.console import Console
from rich.markdown import Markdown

DB_DIR = Path(__file__).parent / "chroma-db"
COLLECTION = "pentest-kb"

CHAT_MODEL = os.environ.get("OLLAMA_CHAT_MODEL", "llama3.1:8b")
EMBED_MODEL = os.environ.get("OLLAMA_EMBED_MODEL", "nomic-embed-text")
OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://127.0.0.1:11434")

app = typer.Typer(add_completion=False, help="RAG local para preguntas de pentest.")
console = Console()

SYSTEM = """Eres un pentester senior que responde la pregunta de un colega usando
SOLO los extractos de la base de conocimiento que se te dan. Sigue estas
reglas de forma estricta:

- Si los extractos no contienen la respuesta, di exactamente
  "Not covered by the corpus" y sugiere qué debería buscar el operador a
  continuación. No adivines.
- Cita cada afirmación con el id del chunk entre corchetes, por ejemplo
  [cve-notes::003].
- Prefiere el texto del extracto por encima del conocimiento general: este
  corpus es la autoridad del engagement.
- Mantén las respuestas por debajo de 250 palabras salvo que el operador pida
  profundidad.
"""


@app.command()
def index() -> None:
    """Reconstruye el vector store desde ./corpus/*.md (delega en index.py)."""
    here = Path(__file__).parent
    rc = subprocess.call([sys.executable, str(here / "index.py")])
    raise typer.Exit(code=rc)


@app.command()
def ask(
    question: str = typer.Argument(..., help="Pregunta de pentest a responder."),
    k: int = typer.Option(4, "--k", help="Cantidad de chunks a recuperar."),
    show_context: bool = typer.Option(
        False, "--show-context", help="Imprime los chunks recuperados antes de la respuesta."
    ),
) -> None:
    """Recupera los top-k chunks relevantes y responde la pregunta."""
    if not DB_DIR.exists():
        console.print(f"[red]no hay vector store en {DB_DIR}[/red] — corre `ragq index` primero")
        raise typer.Exit(code=2)

    chroma = chromadb.PersistentClient(path=str(DB_DIR))
    coll = chroma.get_collection(COLLECTION)

    oll = ollama.Client(host=OLLAMA_HOST)
    q_vec = oll.embeddings(model=EMBED_MODEL, prompt=question)["embedding"]

    hits = coll.query(query_embeddings=[q_vec], n_results=k)
    ids = hits["ids"][0]
    docs = hits["documents"][0]
    metas = hits["metadatas"][0]

    context_blocks = []
    for cid, doc, meta in zip(ids, docs, metas):
        header = f"[{cid}] {meta.get('title', '')} (source: {meta.get('source', '')})"
        context_blocks.append(header + "\n" + doc)
    context = "\n\n---\n\n".join(context_blocks)

    if show_context:
        console.rule("contexto recuperado")
        console.print(context)
        console.rule("respuesta")

    user_msg = f"""Pregunta:
{question}

Extractos de la base de conocimiento (cita por id de chunk entre corchetes):

{context}
"""

    resp = oll.chat(
        model=CHAT_MODEL,
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": user_msg},
        ],
        options={"temperature": 0.2, "num_ctx": 8192},
    )
    answer = resp["message"]["content"]

    console.print(Markdown(answer))
    console.rule()
    console.print("[dim]fuentes citadas:[/dim] " + ", ".join(ids))


if __name__ == "__main__":
    app()
```

Pruébalo:

```bash
cd ~/pentest-tools/ragq
python ragq.py ask "¿En qué categoría OWASP cae Log4Shell y qué lo acerca a ser propagable?"
```

Forma esperada del output:

```text
CVE-2021-44228 (Log4Shell) está clasificado bajo OWASP A08:2021 — Software and
Data Integrity Failures [cve-notes::000]. No es propagable como worm en el
sentido estricto de NotPetya, pero cualquier aplicación que loguee input
controlado por el atacante hacia un Log4j 2 vulnerable (2.0-beta9 hasta
2.14.1) evalúa el lookup JNDI y permite RCE sin autenticación vía callback
LDAP/RMI [cve-notes::000] ...

────────────────────────────
fuentes citadas: cve-notes::000, owasp-top10-2021::007, ...
```

Confirma el comportamiento de refusal fuera del corpus. La frase exacta es la del prompt:

```bash
python ragq.py ask "¿Cuál es la capital de Perú?"
# → "Not covered by the corpus. Consulta Wikipedia o una referencia de geografía."
```

### 4.5 Stretch goals

- Extiende el corpus para ingerir el libro completo de HackTricks. Vas a querer un chunker más inteligente (ventana deslizante con overlap) y metadata que registre la URL de origen.
- Agrega un modo de output `--json` que emita la respuesta más las citas como JSON legible por máquina, para que tools aguas abajo puedan consumirlo.
- Lleva la frecuencia de retrieval por chunk y muestra un report de "chunks más citados". Sirve para el análisis de huecos del KB.

## 5. Desafío 3 — Triage de recon asistido por IA

### 5.1 Requisitos

Arma `recon-triage`, una tool que ingiere un scan XML de Nmap y devuelve una lista JSON de hosts ordenada por valor de ataque probable. Tiene que:

- Tomar un archivo XML de Nmap en la línea de comandos (`nmap -oX`).
- Parsear el XML con lxml y producir un registro normalizado por host: IP, hostname, puertos, versiones de servicio, snippets de output de scripts.
- Agrupar los registros de host en una sola llamada al LLM (o una llamada por host si el batch es demasiado grande para el contexto) con un prompt de triage estructurado. El prompt tiene que pedirle al modelo que puntúe cada host de 0 a 100 por valor de ataque, y que produzca una rationale corta y hasta tres sugerencias de vector de ataque con IDs de MITRE ATT&CK.
- Parsear el output del modelo de vuelta a JSON. Nunca confíes en el output crudo del modelo: valida contra un schema y descarta los hosts que fallen la validación, con un warning claro.
- Emitir un documento JSON final ordenado por score descendente, e imprimir un resumen en tabla de rich a stdout.

Entregables: `triage.py`, un XML de muestra bajo `samples/`. Objetivo: unas 150 líneas de Python.

Las keys del JSON se quedan en inglés (`ip`, `score`, `rationale`, `vectors`, `technique`, `attack_id`, `note`): `validate()` las lee tal cual.

### 5.2 XML de Nmap de muestra

Genera una muestra para poder testear la tool sin un target vivo. Deja un scan realista bajo `~/pentest-tools/recon-triage/samples/scan.xml`. La red `10.10.10.0/24` es del lab:

```bash
cat > ~/pentest-tools/recon-triage/samples/scan.xml <<'EOF'
<?xml version="1.0"?>
<nmaprun scanner="nmap" args="nmap -sV -sC -oX scan.xml 10.10.10.0/24" version="7.94">
  <host><status state="up"/><address addr="10.10.10.5" addrtype="ipv4"/>
    <hostnames><hostname name="dc01.corp.local" type="user"/></hostnames>
    <ports>
      <port protocol="tcp" portid="53"><state state="open"/>
        <service name="domain" product="Microsoft DNS" version="10.0.17763"/></port>
      <port protocol="tcp" portid="88"><state state="open"/>
        <service name="kerberos-sec" product="Microsoft Windows Kerberos"/></port>
      <port protocol="tcp" portid="445"><state state="open"/>
        <service name="microsoft-ds" product="Windows Server 2019" version="10.0.17763"/></port>
      <port protocol="tcp" portid="3389"><state state="open"/>
        <service name="ms-wbt-server" product="Microsoft Terminal Services"/></port>
    </ports>
  </host>
  <host><status state="up"/><address addr="10.10.10.10" addrtype="ipv4"/>
    <hostnames><hostname name="web01.corp.local" type="user"/></hostnames>
    <ports>
      <port protocol="tcp" portid="80"><state state="open"/>
        <service name="http" product="Apache httpd" version="2.4.49"/></port>
      <port protocol="tcp" portid="22"><state state="open"/>
        <service name="ssh" product="OpenSSH" version="7.4"/></port>
    </ports>
  </host>
  <host><status state="up"/><address addr="10.10.10.20" addrtype="ipv4"/>
    <hostnames><hostname name="printer01.corp.local" type="user"/></hostnames>
    <ports>
      <port protocol="tcp" portid="9100"><state state="open"/>
        <service name="jetdirect" product="HP JetDirect"/></port>
      <port protocol="tcp" portid="631"><state state="open"/>
        <service name="ipp" product="CUPS" version="1.7.2"/></port>
    </ports>
  </host>
</nmaprun>
EOF
```

### 5.3 triage.py

`~/pentest-tools/recon-triage/triage.py`:

```python
#!/usr/bin/env python3
"""recon-triage — ordena hosts descubiertos por Nmap según valor de ataque
probable, usando un LLM local. Lee XML de Nmap, llama a Ollama con un prompt
de triage estructurado, valida la respuesta JSON y emite el JSON ordenado
en stdout."""

from __future__ import annotations

import json
import os
import re
import sys
from dataclasses import asdict, dataclass
from pathlib import Path

import ollama
import typer
from lxml import etree
from rich.console import Console
from rich.table import Table

CHAT_MODEL = os.environ.get("OLLAMA_CHAT_MODEL", "llama3.1:8b")
OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://127.0.0.1:11434")

app = typer.Typer(add_completion=False, help="Triage de recon de Nmap asistido por IA.")
console = Console(stderr=True)


@dataclass
class Port:
    port: int
    proto: str
    service: str
    product: str
    version: str


@dataclass
class Host:
    ip: str
    hostname: str
    ports: list[Port]


SYSTEM = """Eres un operador senior de red team que hace triage de resultados de
Nmap para un engagement con tiempo acotado. Por cada host que recibes,
devuelve un objeto JSON con:

  - "ip": la IP del host (textual, como vino en el input)
  - "score": entero 0-100. 100 = mayor valor de ataque (domain controller,
    candidato a RCE sin autenticación, almacén de credenciales). 0 = sin
    valor de ataque.
  - "rationale": una oración que explique el score
  - "vectors": array de hasta tres objetos, cada uno con las keys:
      "technique": nombre corto, por ejemplo "SMB relay"
      "attack_id": ID de MITRE ATT&CK, por ejemplo "T1557.001"
      "note": una oración, el siguiente paso concreto

Devuelve SOLO un array JSON de nivel superior con estos objetos de host.
Sin prosa y sin fences de markdown. Si un host no tiene puertos o no es
interesante, igual devuelve una entrada con score 0 y una rationale que
explique por qué.

Los nombres de las keys van exactamente como están escritos arriba.
"""


def parse_nmap(path: Path) -> list[Host]:
    tree = etree.parse(str(path))
    hosts: list[Host] = []
    for hnode in tree.findall(".//host"):
        state = hnode.find("status")
        if state is None or state.get("state") != "up":
            continue
        addr_node = hnode.find("address[@addrtype='ipv4']")
        if addr_node is None:
            continue
        ip = addr_node.get("addr", "")
        hn_node = hnode.find(".//hostname")
        hostname = hn_node.get("name", "") if hn_node is not None else ""
        ports: list[Port] = []
        for p in hnode.findall(".//port"):
            st = p.find("state")
            if st is None or st.get("state") != "open":
                continue
            svc = p.find("service")
            ports.append(
                Port(
                    port=int(p.get("portid", "0")),
                    proto=p.get("protocol", ""),
                    service=svc.get("name", "") if svc is not None else "",
                    product=svc.get("product", "") if svc is not None else "",
                    version=svc.get("version", "") if svc is not None else "",
                )
            )
        hosts.append(Host(ip=ip, hostname=hostname, ports=ports))
    return hosts


def build_prompt(hosts: list[Host]) -> str:
    lines = ["Hosts para triage:\n"]
    for h in hosts:
        lines.append(f"- {h.ip} ({h.hostname or 'sin-hostname'})")
        for p in h.ports:
            lines.append(
                f"    {p.port}/{p.proto}  {p.service}  {p.product} {p.version}".rstrip()
            )
    lines.append("\nResponde con el array JSON como se te indicó.")
    return "\n".join(lines)


def extract_json_array(raw: str) -> list[dict]:
    """Recupera lo mejor posible un array JSON del output del modelo."""
    raw = raw.strip()
    raw = re.sub(r"^```(?:json)?", "", raw)
    raw = re.sub(r"```$", "", raw)
    start = raw.find("[")
    end = raw.rfind("]")
    if start == -1 or end == -1:
        raise ValueError("no se encontró un array JSON en el output del modelo")
    return json.loads(raw[start : end + 1])


def validate(items: list[dict], hosts: list[Host]) -> list[dict]:
    ips = {h.ip for h in hosts}
    clean: list[dict] = []
    for item in items:
        if not isinstance(item, dict):
            console.print("[yellow]se descarta un ítem que no es objeto[/yellow]")
            continue
        if item.get("ip") not in ips:
            console.print(f"[yellow]se descarta ip desconocida {item.get('ip')!r}[/yellow]")
            continue
        try:
            score = int(item.get("score", -1))
        except (TypeError, ValueError):
            score = -1
        if not 0 <= score <= 100:
            console.print(f"[yellow]se descarta {item['ip']} — score inválido[/yellow]")
            continue
        clean.append({
            "ip": item["ip"],
            "score": score,
            "rationale": str(item.get("rationale", ""))[:400],
            "vectors": [
                {
                    "technique": str(v.get("technique", ""))[:80],
                    "attack_id": str(v.get("attack_id", ""))[:20],
                    "note": str(v.get("note", ""))[:400],
                }
                for v in (item.get("vectors") or [])[:3]
                if isinstance(v, dict)
            ],
        })
    return clean


@app.command()
def main(
    xml: Path = typer.Argument(..., exists=True, readable=True, help="Archivo XML de Nmap."),
    output: Path = typer.Option(None, "--out", help="Escribe el JSON ordenado acá."),
) -> None:
    hosts = parse_nmap(xml)
    if not hosts:
        console.print("[red]no hay hosts vivos en el scan[/red]")
        raise typer.Exit(code=1)

    oll = ollama.Client(host=OLLAMA_HOST)
    resp = oll.chat(
        model=CHAT_MODEL,
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": build_prompt(hosts)},
        ],
        options={"temperature": 0.2, "num_ctx": 8192},
        format="json",
    )
    raw = resp["message"]["content"]

    try:
        items = extract_json_array(raw)
    except (ValueError, json.JSONDecodeError) as e:
        console.print(f"[red]el modelo devolvió output que no se puede parsear: {e}[/red]")
        console.print(raw)
        raise typer.Exit(code=1)

    ranked = sorted(validate(items, hosts), key=lambda r: r["score"], reverse=True)

    table = Table(title=f"Triage — {xml.name}")
    table.add_column("Rank", style="dim")
    table.add_column("IP")
    table.add_column("Score", justify="right")
    table.add_column("Vector principal")
    table.add_column("Rationale")
    for i, r in enumerate(ranked, 1):
        top = r["vectors"][0]["technique"] if r["vectors"] else "-"
        table.add_row(str(i), r["ip"], str(r["score"]), top, r["rationale"])
    console.print(table)

    doc = {"source": str(xml), "hosts": [asdict(h) for h in hosts], "ranked": ranked}
    text = json.dumps(doc, indent=2, default=lambda o: asdict(o) if hasattr(o, "__dict__") else str(o))
    if output:
        output.write_text(text)
        console.print(f"[green]se escribió {output}[/green]")
    else:
        print(text)


if __name__ == "__main__":
    app()
```

Correrlo:

```bash
cd ~/pentest-tools/recon-triage
python triage.py samples/scan.xml --out samples/scan.ranked.json
```

Tabla esperada (los scores van a variar). El modelo nombra el vector. No ejecuta el ataque:

```text
                 Triage — scan.xml
┏━━━━━━┳━━━━━━━━━━━━┳━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Rank ┃ IP         ┃ Score ┃ Vector         ┃ Rationale              ┃
┡━━━━━━╇━━━━━━━━━━━━╇━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 1    │ 10.10.10.5 │ 92    │ Kerberoasting  │ Domain controller probable │
│ 2    │ 10.10.10.10│ 78    │ Apache CVE     │ Apache 2.4.49 → CVE-...│
│ 3    │ 10.10.10.20│ 25    │ Printer NTLM   │ HP JetDirect + IPP     │
└──────┴────────────┴───────┴────────────────┴────────────────────────┘
```

### 5.4 Stretch goals

- Agrega un modo `--per-host` que dispare una llamada al LLM por host en un `ThreadPoolExecutor`. Ayuda en scans enormes donde el prompt de batch se pasa de `num_ctx`.
- Agrega un parser de XML `.nessus` de Nessus, para ordenar scans de Nessus con la misma tool.
- Alimenta el JSON ordenado a `pentest-assistant` del desafío 1 para generar un markdown de plan de ataque por cada host del top-N. El plan respeta el tier de impact del RoE.

## 6. Desafío 4 — Escritor de findings redactado por IA

### 6.1 Requisitos

Arma `finding-writer`, una tool que lee un YAML crudo de finding y redacta una card completa en Markdown. Tiene que:

- Aceptar un archivo YAML que describa un finding crudo: `title`, `target`, `severity`, `evidence` (multilínea), `cvss` opcional, `references` opcionales.
- Llamar a la API de Claude Sonnet 5 (`claude-sonnet-5-20260101`) con un system prompt estructurado que exija las secciones `Executive Summary`, `Technical Details`, `Impact`, `Remediation`, `References`.
- Ante cualquier falla de la API (key ausente, error de red, error HTTP, rate limit), caer al modelo local `llama3.1:8b` con el mismo prompt.
- Nunca crashear por el output del modelo: aceptar el markdown que vuelva, pero chequear que las secciones obligatorias estén. Si falta alguna, pedirle una vez más al modelo primario un parche de "agrega las secciones que faltan", y si eso falla, imprimir un warning y emitir lo que tengas.
- Escribir el output en `outputs/<slug>-<timestamp>.md` e imprimir el path.

Entregables: `write_finding.py`, un YAML de muestra bajo `samples/`. Objetivo: unas 180 líneas de Python.

Los encabezados se quedan en inglés porque `REQUIRED` los compara como strings: `## Executive Summary`, `## Technical Details`, `## Impact`, `## Remediation`, `## References`.

### 6.2 YAML de finding de muestra

`~/pentest-tools/finding-writer/samples/idor-order-history.yaml`. Las keys se quedan en inglés porque el script las lee. El token del ejemplo es un placeholder, no una credencial real:

```yaml
title: "IDOR en /api/v1/orders/{id} — divulgación del historial de órdenes"
target: "https://shop.example.com"
severity: "High"
cvss: "8.6 (AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:N/A:N)"
evidence: |
  Autenticado como el usuario acct-1001 (Bearer eyJhbGci...). Un GET directo de
  /api/v1/orders/1002 (que pertenece a acct-1002) devuelve el payload completo
  de la orden, incluido PII (nombre, dirección de envío, últimos 4 del PAN) y
  las líneas de la orden.
  El servidor no chequea ownership. Cualquier usuario autenticado puede enumerar
  IDs enteros de orden y traer el historial de otros clientes.
  Respuesta de muestra:
      HTTP/1.1 200 OK
      Content-Type: application/json
      {"id":1002,"user_id":1002,"name":"Grace Hopper","address":"...","total":149.99}
  Se enumeró el rango 1000..1050. 43 de 51 IDs devolvieron datos de órdenes de
  otras cuentas. No se observó rate limit.
references:
  - "OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization"
  - "https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/"
```

### 6.3 write_finding.py

`~/pentest-tools/finding-writer/write_finding.py`:

```python
#!/usr/bin/env python3
"""finding-writer — redacta una card completa de finding en Markdown desde un YAML crudo.

Primario: Claude Sonnet 5 vía la API de Anthropic.
Fallback: llama3.1:8b local vía Ollama.

La tool nunca crashea por una falla del modelo. Si la API primaria no se
alcanza, reintenta una vez por Ollama con el mismo prompt. Si al markdown
redactado le falta alguna sección obligatoria, pide una vez un parche de
seguimiento. Lo que finalmente salga se escribe en outputs/<slug>-<ts>.md
y se imprime su path.
"""

from __future__ import annotations

import os
import re
import sys
import time
from datetime import datetime
from pathlib import Path

import ollama
import typer
import yaml
from rich.console import Console

console = Console()
app = typer.Typer(add_completion=False, help="Escritor de findings redactado por IA.")

ANTHROPIC_MODEL = os.environ.get("ANTHROPIC_MODEL", "claude-sonnet-5-20260101")
OLLAMA_MODEL = os.environ.get("OLLAMA_CHAT_MODEL", "llama3.1:8b")
OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://127.0.0.1:11434")

REQUIRED = ("## Executive Summary", "## Technical Details", "## Impact", "## Remediation", "## References")

SYSTEM = """Eres un penetration tester senior que escribe una card formal de
finding para un report de cara al cliente. Dada una descripción cruda del
finding, produce un documento Markdown pulido con estas secciones de nivel
superior, exactamente en este orden. Los encabezados van en inglés:

    # <título del finding>

    **Target:** ...
    **Severity:** ...
    **CVSS:** ...

    ## Executive Summary
    Dos o tres oraciones sobre las que un stakeholder no técnico pueda actuar.
    Enuncia el riesgo de negocio en lenguaje claro, en español.

    ## Technical Details
    Una descripción precisa de la vulnerabilidad, los pasos de reproducción y
    la evidencia. Conserva textuales todos los IDs, URLs y payloads de la
    evidencia cruda. Usa code blocks con fence para requests y responses HTTP.

    ## Impact
    Qué puede hacer un atacante con esto. Átalo a la tríada CIA y a cualquier
    dato regulado (PII, PCI, PHI). Incluye un peor caso realista.

    ## Remediation
    Pasos ordenados y concretos que ingeniería pueda seguir. Prefiere nombres
    de control específicos ("enforce object-level authorization en el
    OrdersController") por encima de consejos genéricos ("valida el input").
    Incluye mitigaciones de corto plazo si el arreglo completo no es inmediato.

    ## References
    Una lista de viñetas en markdown. Incluye cada referencia del input más
    cualquier fuente estándar adicional (OWASP, CWE, NIST) que sea relevante
    de forma directa.

Nunca inventes evidencia, IDs, responses HTTP ni nombres de clientes que no
estén en el input. Nunca especules sobre la identidad del cliente. Mantén el
tono neutro, factual y listo para auditoría.
"""


def slugify(text: str) -> str:
    s = re.sub(r"[^a-z0-9]+", "-", text.lower()).strip("-")
    return s[:60] or "finding"


def build_user_prompt(raw: dict) -> str:
    parts = [
        f"Title: {raw.get('title', '(falta)')}",
        f"Target: {raw.get('target', '(falta)')}",
        f"Severity: {raw.get('severity', '(sin especificar)')}",
    ]
    if raw.get("cvss"):
        parts.append(f"CVSS: {raw['cvss']}")
    parts.append("\nEvidencia cruda:\n" + str(raw.get("evidence", "")).rstrip())
    if raw.get("references"):
        parts.append("\nReferencias provistas:")
        for r in raw["references"]:
            parts.append(f"- {r}")
    parts.append("\nRedacta la card del finding como se te indicó.")
    return "\n".join(parts)


def try_anthropic(user_prompt: str) -> str:
    try:
        import anthropic  # type: ignore
    except ImportError as e:
        raise RuntimeError(f"falta el sdk de anthropic: {e}") from e

    key = os.environ.get("ANTHROPIC_API_KEY", "").strip()
    if not key:
        raise RuntimeError("ANTHROPIC_API_KEY no está definida")

    client = anthropic.Anthropic(api_key=key)
    resp = client.messages.create(
        model=ANTHROPIC_MODEL,
        max_tokens=2048,
        system=SYSTEM,
        messages=[{"role": "user", "content": user_prompt}],
    )
    return "".join(block.text for block in resp.content if getattr(block, "type", "") == "text")


def try_ollama(user_prompt: str) -> str:
    oll = ollama.Client(host=OLLAMA_HOST)
    resp = oll.chat(
        model=OLLAMA_MODEL,
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": user_prompt},
        ],
        options={"temperature": 0.3, "num_ctx": 8192},
    )
    return resp["message"]["content"]


def draft(user_prompt: str) -> tuple[str, str]:
    """Devuelve (markdown, model_id usado)."""
    try:
        md = try_anthropic(user_prompt)
        return md, ANTHROPIC_MODEL
    except Exception as e:
        console.print(f"[yellow]el camino de anthropic falló: {e} — fallback a ollama[/yellow]")
        try:
            md = try_ollama(user_prompt)
            return md, OLLAMA_MODEL
        except Exception as ee:
            console.print(f"[red]el fallback de ollama también falló: {ee}[/red]")
            raise


def missing_sections(md: str) -> list[str]:
    return [s for s in REQUIRED if s not in md]


def patch_missing(md: str, missing: list[str], user_prompt: str) -> str:
    nudge = (
        "A tu borrador anterior le faltan estas secciones obligatorias: "
        + ", ".join(missing)
        + ". Reescribe el documento completo con cada sección obligatoria presente. "
        + "Conserva todo lo que ya escribiste bien."
    )
    full = user_prompt + "\n\nBorrador anterior:\n" + md + "\n\n" + nudge
    try:
        return try_anthropic(full)
    except Exception:
        return try_ollama(full)


@app.command()
def main(
    yaml_path: Path = typer.Argument(..., exists=True, readable=True),
    out_dir: Path = typer.Option(Path("outputs"), "--out", help="Directorio de output."),
) -> None:
    raw = yaml.safe_load(yaml_path.read_text())
    if not isinstance(raw, dict) or "title" not in raw:
        console.print("[red]el YAML tiene que ser un mapping con al menos la key 'title'[/red]")
        raise typer.Exit(code=2)

    user_prompt = build_user_prompt(raw)
    md, used = draft(user_prompt)

    missing = missing_sections(md)
    if missing:
        console.print(f"[yellow]faltan secciones: {missing} — se pide un parche[/yellow]")
        try:
            md = patch_missing(md, missing, user_prompt)
        except Exception as e:
            console.print(f"[red]el intento de parche falló: {e}[/red]")
        missing = missing_sections(md)
        if missing:
            console.print(f"[red]al borrador final todavía le falta: {missing}[/red]")

    out_dir.mkdir(parents=True, exist_ok=True)
    ts = datetime.now().strftime("%Y%m%d-%H%M%S")
    out_path = out_dir / f"{slugify(raw['title'])}-{ts}.md"
    out_path.write_text(md.rstrip() + "\n")

    console.print(f"[green]se escribió {out_path}[/green]  [dim](modelo: {used})[/dim]")


if __name__ == "__main__":
    app()
```

Pruébalo de punta a punta. Primero con la API disponible:

```bash
cd ~/pentest-tools/finding-writer
python write_finding.py samples/idor-order-history.yaml
# → se escribió outputs/idor-en-api-v1-orders-id-...md  (modelo: claude-sonnet-5-...)
```

Después simula que la API no está, para probar el fallback:

```bash
env -u ANTHROPIC_API_KEY python write_finding.py samples/idor-order-history.yaml
# → el camino de anthropic falló: ANTHROPIC_API_KEY no está definida — fallback a ollama
# → se escribió outputs/idor-en-api-v1-orders-id-...md  (modelo: llama3.1:8b)
```

Abre el markdown resultante y chequea a ojo que las cinco secciones obligatorias estén, y que el bloque de evidencia se haya conservado textual (sin responses HTTP inventadas).

### 6.4 Stretch goals

- Agrega un modo `--batch` que recorra un directorio de archivos YAML y produzca un `report.md` agregado, con tabla de contenidos.
- Agrega un output `--json` junto al markdown, con campos estructurados (CWE ID, IDs de ATT&CK, likelihood, exploitability) para que los consuma un importador de defect tracker.
- Agrega un evaluador chico que puntúe cada finding redactado contra una rúbrica (5 dimensiones de 1 a 5) usando una segunda llamada al LLM, y que rechace aceptar borradores por debajo de un umbral: un loop de "reviewer" self-hosted.

## 7. Desafíos independientes

Estos son abiertos. No hay implementación de referencia: ese es el punto. Elige uno como mínimo. El estudiante ambicioso elige los tres. Los entregables van a tu repo `~/pentest-tools/` bajo `stretch/` y se califican con la rúbrica de abajo.

### 7.1 Fine-tune de un adapter LoRA con Unsloth

Arma un dataset chico (100–500 ejemplos) de pares `(pentest_question, ideal_answer)`. Sácalos de tus propias notas, de HackTricks, de tus write-ups de CTF. Usa Unsloth para hacer LoRA fine-tune de `llama3.1:8b` sobre el dataset, en un Colab o en una caja con GPU local. Exporta el adapter resultante, mergealo en un GGUF y empaquétalo como un Modelfile de Ollama llamado `pentest-assistant-tuned`. Compara su output en las tres sondas del desafío 1 contra el asistente sin tunear. Documenta qué mejoró y qué regresó.

La calificación enfatiza: calidad del dataset, evaluación honesta (incluidas las regresiones) y reproducibilidad de la corrida de entrenamiento. El adapter hereda las mismas reglas de scope del system prompt.

### 7.2 Envolver un scanner como servidor MCP

Elige un scanner que conozcas: nmap, nikto, gobuster, nuclei, o tu propio `portscan.py` del módulo 03. Envuélvelo como un servidor de Model Context Protocol que exponga al menos tres tools (por ejemplo `start_scan`, `get_status`, `get_results`) y un resource (el output crudo del scan). Usa el SDK de referencia de MCP para Python. Prueba que funciona conectándolo a `claude` (o a otro cliente MCP) y conduciendo un scan de punta a punta.

La calificación enfatiza: calidad del schema (firmas de tools bien tipadas), manejo correcto de scans de larga duración (jobs, no llamadas bloqueantes) y seguridad (rechazar scans de localhost o de targets no especificados).

### 7.3 UI de Streamlit para el escritor de findings

Envuelve la tool del desafío 4 en una UI de Streamlit. El usuario debería poder: pegar un finding crudo, elegir una severity, hacer clic en "Redactar", ver el markdown redactado lado a lado con el input crudo, editarlo inline y descargar el `.md` final. La página tiene que mostrar qué modelo produjo el borrador (Claude Sonnet 5 frente al fallback local) y exponer un botón de "regenerar".

La calificación enfatiza: usabilidad en condiciones reales (sin caminos de solo teclado, sin output que no se pueda copiar), degradación ordenada cuando la API no se alcanza, e higiene de secretos (ninguna API key en la URL ni en la página).

## 8. Rúbrica de calificación

Cada desafío se califica sobre 20 puntos. Lab total: 100 puntos entre los cuatro desafíos obligatorios (desafíos 1 a 4), más hasta 20 puntos de bonus por un desafío independiente de la sección 7 completado.

| Dimensión | 5 pts (excelente) | 3 pts (adecuado) | 1 pt (débil) |
|---|---|---|---|
| Correctness | Corre contra los inputs de muestra en Kali 2026 sin ediciones | Corre con retoques menores (env var, path) | Falla sin que te lleven de la mano |
| Diseño del prompt | System prompt afilado, few-shot presente, restricciones explícitas | Persona y formato adecuados | System prompt vago o ausente |
| Robustez y fallback | Maneja errores del modelo, de red y output mal formado | Maneja los errores comunes | Crashea en la primera sorpresa |
| Disciplina de seguridad | Ninguna key en el código, rechaza lo fuera de scope, conserva la evidencia textual | Deslices menores | Keys commiteadas, inventa evidencia |

Por desafío:

- **Desafío 1 — pentest-assistant:** 20 puntos. Correctness del Modelfile (5), calidad del system prompt (5), pertinencia del few-shot (5), el harness de test pasa (5).
- **Desafío 2 — ragq:** 20 puntos. Completitud del corpus (4), pipeline de embeddings (4), calidad del retrieval (4), cita de fuentes (4), refusal en preguntas fuera del corpus (4).
- **Desafío 3 — recon-triage:** 20 puntos. Parseo del XML de Nmap (4), estructura del prompt (4), validación del schema JSON (4), orden y tabla de rich (4), robustez ante output mal formado del modelo (4).
- **Desafío 4 — finding-writer:** 20 puntos. Camino primario de Claude Sonnet 5 (4), camino de fallback de Ollama (4), validación de secciones y auto-parche (4), conservación de la evidencia (4), higiene del archivo de output (4).
- **Desafío independiente (bonus):** hasta 20 puntos. Se juzga por scope, calidad y documentación honesta de las limitaciones.

Hay descuentos por:

- Cualquier commit que contenga una API key (−20, inmediato).
- Cualquier tool que escanee, sondee o ataque un target que el operador no nombró de forma explícita (−10).
- Cualquier número de CVE, ID de ATT&CK o artefacto de evidencia inventado en el output del escritor de findings (−5 por instancia).

## 9. Entregar el trabajo

Cada tool va a tu repo git personal `~/pentest-tools/`. La estructura y la higiene importan: un corrector que no encuentra tus archivos te da el puntaje que puede probar.

### 9.1 Layout del repo

```text
~/pentest-tools/
├── .gitignore
├── requirements.txt
├── README.md                          # corto: qué hace cada tool y cómo correrla
├── pentest-assistant/
│   ├── Modelfile
│   └── test_assistant.py
├── ragq/
│   ├── corpus/
│   ├── index.py
│   └── ragq.py
├── recon-triage/
│   ├── triage.py
│   └── samples/scan.xml
├── finding-writer/
│   ├── write_finding.py
│   └── samples/idor-order-history.yaml
└── stretch/                           # opcional; desafíos independientes
    ├── lora-tune/
    ├── mcp-scanner/
    └── streamlit-finding-writer/
```

### 9.2 Higiene de commits

Commits chicos y atómicos, con mensajes que signifiquen algo. Un desafío por commit, como mínimo:

```bash
cd ~/pentest-tools

git add pentest-assistant/
git commit -m "feat(pentest-assistant): Modelfile + smoke test for llama3.1:8b build"

git add ragq/
git commit -m "feat(ragq): OWASP+CVE RAG CLI over ChromaDB + local Ollama"

git add recon-triage/
git commit -m "feat(recon-triage): Nmap XML → ranked JSON via local LLM"

git add finding-writer/
git commit -m "feat(finding-writer): Claude Sonnet 5 → Ollama fallback finding drafter"
```

### 9.3 Checklist previo a entregar

Corre cada uno de estos antes de pushear, o el corrector los va a correr por ti y te va a sacar puntos:

```bash
cd ~/pentest-tools
source .venv/bin/activate

# 1. Ningún secreto commiteado.
git ls-files | xargs grep -EIn 'sk-ant-|AKIA[0-9A-Z]{16}|BEGIN PRIVATE KEY' && echo "SECRETS FOUND" || echo "clean"

# 2. Cada entry point de Python corre --help sin traceback.
python ragq/ragq.py --help
python recon-triage/triage.py --help
python finding-writer/write_finding.py --help

# 3. El smoke test del desafío 1 pasa.
python pentest-assistant/test_assistant.py

# 4. El desafío 2 de punta a punta.
python ragq/index.py
python ragq/ragq.py ask "¿Qué categoría OWASP cubre Log4Shell?"

# 5. El desafío 3 de punta a punta sobre la muestra.
python recon-triage/triage.py recon-triage/samples/scan.xml --out /tmp/rt.json
jq '.ranked | length' /tmp/rt.json

# 6. El desafío 4 de punta a punta (los dos caminos).
python finding-writer/write_finding.py finding-writer/samples/idor-order-history.yaml
env -u ANTHROPIC_API_KEY python finding-writer/write_finding.py finding-writer/samples/idor-order-history.yaml
```

`SECRETS FOUND` y `clean` se quedan como los imprime el checklist.

### 9.4 Push y abrir el PR

```bash
git remote add origin git@gitlab.rtl.internal:students/<tu-handle>/pentest-tools.git
git push -u origin main

# O, para la instancia interna de Gitea que corre el curso:
git remote add course git@gitea.rtl.internal:module05/<tu-handle>.git
git push -u course main
```

Abre un merge request contra la rama de calificación que provee el curso y pega el output del checklist previo en la descripción del MR. El corrector va a hacer pull, reproducir y puntuar.

## 10. Resumen y puntero al módulo 06

Acabas de armar cuatro tools que juntas representan un stack completo de seguridad ofensiva aumentado con IA:

- Un modelo local especializado que puedes consultar offline, sin filtrar datos del cliente.
- Una capa de retrieval que ancla al modelo en un KB que tú controlas, con citas.
- Un agente que parsea input y convierte output de scan no estructurado en inteligencia estructurada y ordenada.
- Un agente de redacción que convierte findings crudos en entregables de cara al cliente, listos para auditoría, con la calidad de modelo cloud de Claude Sonnet 5 como camino primario y un fallback del todo local para cuando desaparecen la red o el presupuesto.

Cada una de estas tools vuelve a aparecer en el resto del curso. El módulo 06 (AI-Augmented Reconnaissance) retoma justo donde deja el desafío 3: vas a encadenar `recon-triage` detrás de un pipeline automatizado de Nmap, agregar enriquecimiento de Shodan y de CT logs como fuentes nuevas de tools para el mismo modelo, y empezar a conducir todo desde `pentest-assistant`. El módulo 09 (Exploitation with AI Assist) reutiliza `ragq` como la capa de memoria que le deja al modelo recordar primitivas de exploit que ya usaste. El módulo 12 (AI-Augmented Reporting) reutiliza `finding-writer` como el núcleo de tu pipeline de generación de reports.

Dos hábitos para llevar adelante:

- **Los prompts son código.** Versionálos, testéalos y trata el drift en ellos como el drift de cualquier otra dependencia de librería. El system prompt de `finding-writer` es la diferencia entre un report por el que tu cliente va a pagar y un report que tu engagement manager te va a devolver.
- **Local primero, cloud cuando ayuda.** Los datos del cliente pertenecen a tu máquina. Alcanza la API cuando la ganancia marginal de calidad justifica mandar los datos, y nunca antes. Cada una de las tools que acabas de armar hace default a local por una razón.

Haz snapshot de la VM. Pushea el repo. Tómate un descanso. El módulo 06 espera con tu primer target de recon vivo, dentro del scope.
