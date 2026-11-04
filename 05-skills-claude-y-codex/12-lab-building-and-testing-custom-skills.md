# Lección 12 — Lab: Building and Testing Custom Skills

Vas a terminar este módulo con tres skills de Claude Code listas para entregar —un escáner de reconocimiento, un validador de vulnerabilidades y un generador de reportes— más tres servers MCP a medida que envuelven Nmap, Burp y las notas del engagement, más un script de la tool de workflow que orquesta un fan-out de recon.

No los trates como demos. Mételos en `~/.claude/skills/` y úsalos en tu próximo engagement de práctica. Cada artefacto de este lab se escribió para usarse en producción, y el único camino a reusar una skill es la repetición.

## Diagrama del módulo

Módulo 06: Construcción de skills de pentest con Claude y Codex · Penetration Testing con Kali Linux 1.0 · Red Team Leaders

## 1. Introducción

El módulo 06 te llevó por la teoría de extender Claude Code y Codex CLI con tus propias skills: cómo funciona el contrato de `SKILL.md`, cómo el loader descubre y despacha skills, cómo enlazarlas a servers MCP, cómo mantenerlas idempotentes, cómo hacerlas seguras de correr contra un engagement real. Cada lección previa de este módulo enseñó piezas aisladas: la gramática del frontmatter en una, la allowlist de tools en otra, el cableado de hooks en otra parte, el empaquetado en otra más.

Esta lección consolida el módulo. Vas a construir tres skills de grado de producción de punta a punta, cada una lista para soltar en `~/.claude/skills/`, y cada una invocable mañana a la mañana contra un engagement real sin editar una sola línea. Cuando el lab esté terminado, deberías poder hacer `git push` de tu repositorio `~/pentest-tools/skills/` y que un compañero haga `git clone` y sea productivo en noventa segundos.

Las tres skills son:

- **recon-scanner** — una skill de reconocimiento externo consciente del scope que encadena subfinder, httpx y `nuclei -tags basic` para un objetivo, respeta `engagement/scope.yml` y emite un resumen legible por máquina.
- **vuln-validator** — una skill que toma un finding crudo de nuclei, verifica la alcanzabilidad, reproduce el request, captura la response y produce un paquete de evidencia que puedes adjuntar directo a un reporte.
- **report-generator** — una skill que recorre cada `finding.yml` bajo `./findings/`, enriquece con puntajes CVSS 4.0, redacta un resumen ejecutivo, arma un reporte Markdown y, de forma opcional, invoca `typst compile` para renderizar un PDF.

Sales de esta lección con tres skills entregables en `~/.claude/skills/` y casos de prueba reproducibles bajo `~/pentest-tools/skills/tests/`. Cada desafío está acotado para que el `SKILL.md` de referencia cargue limpio en Claude Code 1.x y Codex CLI 0.x, y cada uno termina con pasos de verificación que tienes que poder reproducir antes de marcar la rúbrica. Lee los requisitos, intenta escribir el `SKILL.md` tú mismo y después compáralo con la referencia. No copies nada más: escríbelo, rómpelo, entiéndelo.

El módulo siguiente (Módulo 07 — Real Client Engagements) va a asumir que tienes estas tres skills y las va a invocar por nombre en sus recorridos. Si falta un desafío, los módulos posteriores no van a funcionar.

## 2. Setup del entorno de lab

### 2.1 Requisitos del host

- VM Kali Linux 2026.1 (snapshot fresco del módulo 01 o posterior)
- 8 GB de RAM como mínimo, 30 GB de disco (Claude Code más los templates de nuclei más las imágenes de Docker te empujan hacia el tope de las asignaciones más chicas)
- Docker Engine 26+ con el plugin de compose
- HTTPS saliente hacia github.com, projectdiscovery.io, anthropic.com y pypi.org
- Alcanzabilidad de red entre la VM Kali y la subred del bridge de Docker `172.26.0.0/24`

### 2.2 Instalación del toolchain

Corre esto una vez como tu usuario normal (no root) en la VM Kali:

```bash
sudo apt update
sudo apt install -y \
    python3 python3-venv python3-pip pipx \
    git tmux jq yq ripgrep \
    curl wget ca-certificates \
    docker.io docker-compose-plugin \
    typst chromium

# Docker sin sudo
sudo usermod -aG docker "$USER"
newgrp docker

# Toolchain de ProjectDiscovery: estos son los primitivos que envuelven las tres skills.
# Instala por el instalador oficial; deja las versiones pineadas en ~/pentest-tools/skills/tool-versions.txt.
curl -sSfL https://raw.githubusercontent.com/projectdiscovery/pdtm/main/cmd/pdtm/install.sh | bash
export PATH="$HOME/.pdtm/go/bin:$PATH"
pdtm -i subfinder,httpx,nuclei

# Claude Code y Codex CLI (el módulo 06 asume que los dos están en PATH)
npm i -g @anthropic-ai/claude-code@1
npm i -g @openai/codex-cli@0.9

# Confirma las versiones
claude --version
codex --version
subfinder -version
httpx -version
nuclei -version
typst --version
```

Registra las versiones para que tu lab sea reproducible:

```bash
mkdir -p ~/pentest-tools/skills
cd ~/pentest-tools/skills
git init
{
  echo "kali-release=$(cat /etc/os-release | grep VERSION= | head -1)"
  echo "claude-code=$(claude --version | head -1)"
  echo "codex-cli=$(codex --version | head -1)"
  echo "subfinder=$(subfinder -version 2>&1 | head -1)"
  echo "httpx=$(httpx -version 2>&1 | head -1)"
  echo "nuclei=$(nuclei -version 2>&1 | head -1)"
  echo "typst=$(typst --version)"
} > tool-versions.txt
git add tool-versions.txt
git commit -m "chore: pin toolchain for Module 06 lab"
```

### 2.3 Archivo de scope del engagement

Toda skill de esta lección se niega a correr sin un archivo de scope. Crea ahora el canónico:

```bash
mkdir -p ~/pentest-tools/engagement
cat > ~/pentest-tools/engagement/scope.yml <<'YAML'
engagement:
  code: RTL-LAB-06
  client: "Red Team Leaders Internal Lab"
  window_start: "2026-09-01T00:00:00Z"
  window_end:   "2026-12-31T23:59:59Z"

in_scope:
  domains:
    - lab.local
    - "*.lab.local"
  cidrs:
    - 172.26.0.0/24
  urls:
    - http://172.26.0.10/
    - http://172.26.0.11/
    - http://172.26.0.12/

out_of_scope:
  domains:
    - admin.lab.local
  cidrs:
    - 172.26.0.99/32

rules:
  - "No destructive payloads (no drop, no rm, no fork bombs)."
  - "No brute force above 5 rps per host."
  - "All findings must include reproducible request/response."
YAML
```

### 2.4 Objetivos vulnerables (Docker Compose)

Los objetivos del módulo 04 se reutilizan aquí. Crea un stack de compose dedicado para el módulo 06 para poder levantarlo y bajarlo sin molestar los otros labs:

```bash
mkdir -p ~/labs/module06
cd ~/labs/module06
```

`~/labs/module06/docker-compose.yml`:

```yaml
version: '3.8'

networks:
  m06-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.26.0.0/24

services:
  juice-shop:
    image: bkimminich/juice-shop:v17
    container_name: m06-juice
    networks:
      m06-net:
        ipv4_address: 172.26.0.10
    ports:
      - "8110:3000"
    restart: unless-stopped

  dvwa:
    image: vulnerables/web-dvwa
    container_name: m06-dvwa
    networks:
      m06-net:
        ipv4_address: 172.26.0.11
    ports:
      - "8111:80"
    restart: unless-stopped

  vulnnode:
    image: appsecco/dsvw
    container_name: m06-vulnnode
    networks:
      m06-net:
        ipv4_address: 172.26.0.12
    ports:
      - "8112:8000"
    restart: unless-stopped
```

Levanta el stack y haz un chequeo de cordura de la alcanzabilidad:

```bash
cd ~/labs/module06
docker compose up -d
docker compose ps
curl -s -o /dev/null -w "%{http_code}\n" http://172.26.0.10/   # 200
curl -s -o /dev/null -w "%{http_code}\n" http://172.26.0.11/   # 302
curl -s -o /dev/null -w "%{http_code}\n" http://172.26.0.12/   # 200
```

### 2.5 Layout del directorio de skills

Cada skill de esta lección vive bajo `~/.claude/skills/<skill-name>/` mientras la desarrollas, y bajo `~/pentest-tools/skills/<skill-name>/` en el repo de fuente. Haz symlink de este último hacia el primero para que tus ediciones aparezcan de inmediato tanto en Claude Code como en Codex CLI:

```bash
mkdir -p ~/.claude/skills
mkdir -p ~/pentest-tools/skills/{recon-scanner,vuln-validator,report-generator}
ln -sf ~/pentest-tools/skills/recon-scanner    ~/.claude/skills/recon-scanner
ln -sf ~/pentest-tools/skills/vuln-validator   ~/.claude/skills/vuln-validator
ln -sf ~/pentest-tools/skills/report-generator ~/.claude/skills/report-generator
```

Confirma que Claude Code ve las skills (vacías en este punto, pero registradas):

```bash
claude skills list | grep -E "recon-scanner|vuln-validator|report-generator"
```

## 3. Desafío 1 — Skill de escáner de recon

### 3.1 Requisitos

Construye `recon-scanner/SKILL.md` que satisfaga cada viñeta de abajo. Una implementación parcial pierde puntos aunque la prosa sea hermosa.

- La skill debe aceptar un solo parámetro de entrada `target` (un dominio pelado, por ejemplo `lab.local`).
- Antes de hacer cualquier cosa, debe cargar `engagement/scope.yml` (ruta o desde `$ENGAGEMENT_SCOPE` o desde el directorio de trabajo actual) y fallar duro si `target` no está en `in_scope.domains`.
- Debe encadenar tres tools de ProjectDiscovery en orden:
  - `subfinder -silent -d <target>` para enumerar subdominios.
  - `httpx -silent -json` sobre la lista de subdominios para sondear servicios HTTP(S) vivos.
  - `nuclei -silent -tags basic -jsonl` sobre las URLs vivas.
- La tasa nunca debe superar 5 rps por host (hazlo cumplir vía `-rate-limit 5`).
- La salida debe caer bajo `./recon/<target>/` y debe incluir:
  - `subdomains.txt` — salida cruda de subfinder
  - `httpx.jsonl` — líneas JSON crudas de httpx
  - `nuclei.jsonl` — líneas JSON crudas de nuclei
  - `summary.md` — digest legible por humanos, claveado por activo
  - `summary.json` — digest legible por máquina para las skills posteriores
- La skill debe ser idempotente: correrla dos veces contra el mismo objetivo produce el mismo árbol de salida y nunca agrega duplicados.
- Sin secretos ni API keys embebidos en el `SKILL.md`.

### 3.2 SKILL.md completo

Guarda lo siguiente como `~/pentest-tools/skills/recon-scanner/SKILL.md`:

````markdown
---
name: recon-scanner
description: >-
  Reconocimiento externo consciente del scope. Dado un dominio objetivo,
  enumera subdominios con subfinder, sondea servicios HTTP vivos con httpx
  y corre un barrido de nuclei de bajo ruido con el tag "basic". Se niega
  a correr contra cualquier cosa que no esté listada en engagement/scope.yml.
  Escribe ./recon/<target>/ con la salida cruda de las tools más summary.md
  y summary.json.
version: 1.0.0
author: Red Team Leaders
license: MIT
inputs:
  - name: target
    type: string
    required: true
    description: Dominio pelado a enumerar. Debe aparecer en el scope del engagement.
tools_required:
  - subfinder
  - httpx
  - nuclei
  - jq
  - yq
allowed_bash:
  - "subfinder *"
  - "httpx *"
  - "nuclei *"
  - "jq *"
  - "yq *"
  - "mkdir -p *"
  - "sort -u *"
  - "wc -l *"
  - "test *"
---

# recon-scanner

## Cuándo invocarla

Usa esta skill al comienzo de un engagement externo, o cada vez que el
usuario pida "recon", "enumerar", "mapear la superficie de ataque de" o
"escanear" un dominio que aparece en el scope del engagement.

No invoques esta skill para rangos de IP solo internos: usa `internal-recon`
para esos. No la invoques contra un objetivo que el usuario no haya nombrado
de forma explícita en esta sesión.

## Precondiciones

1. `engagement/scope.yml` existe en `${ENGAGEMENT_SCOPE:-./engagement/scope.yml}`.
2. El parámetro de entrada `target` está definido.
3. `subfinder`, `httpx`, `nuclei`, `jq` y `yq` están en PATH.

## Procedimiento

Ejecuta los pasos siguientes en orden. Si cualquier paso falla, detente y
repórtale el fallo al usuario con el comando exacto que falló y su código
de salida. No intentes "recuperarte" saltándote pasos.

### Paso 1 — Cargar y validar el scope

```bash
SCOPE="${ENGAGEMENT_SCOPE:-./engagement/scope.yml}"
test -f "$SCOPE" || { echo "scope file not found: $SCOPE"; exit 2; }

TARGET="{{ target }}"
if ! yq -r '.in_scope.domains[]' "$SCOPE" | grep -qxE "${TARGET}|\*\.${TARGET#*.}"; then
  echo "target ${TARGET} is NOT in engagement scope"; exit 3
fi
```

### Paso 2 — Preparar el árbol de salida

```bash
OUT="./recon/${TARGET}"
mkdir -p "$OUT"
```

### Paso 3 — Enumeración de subdominios

```bash
subfinder -silent -d "$TARGET" -o "$OUT/subdomains.txt.new"
sort -u "$OUT/subdomains.txt.new" > "$OUT/subdomains.txt"
rm -f "$OUT/subdomains.txt.new"
```

### Paso 4 — Sondeo de hosts vivos

```bash
httpx -silent -json -rate-limit 5 \
      -l "$OUT/subdomains.txt" \
      -o "$OUT/httpx.jsonl.new"
sort -u "$OUT/httpx.jsonl.new" > "$OUT/httpx.jsonl"
rm -f "$OUT/httpx.jsonl.new"
```

### Paso 5 — Barrido básico de vulns

```bash
jq -r '.url' "$OUT/httpx.jsonl" > "$OUT/urls.txt"
nuclei -silent -tags basic -jsonl \
       -rate-limit 5 \
       -l "$OUT/urls.txt" \
       -o "$OUT/nuclei.jsonl.new"
sort -u "$OUT/nuclei.jsonl.new" > "$OUT/nuclei.jsonl"
rm -f "$OUT/nuclei.jsonl.new"
```

### Paso 6 — Emitir summary.json

```bash
jq -s --arg target "$TARGET" '{
  target: $target,
  subdomains: [inputs | select(.url) | .input] | unique,
  live_hosts: [inputs | select(.url) | {url: .url, status: .status_code, title: .title, tech: .tech}],
  findings:   [inputs | select(.template_id) | {id: .template_id, severity: .info.severity, url: .matched_at}]
}' "$OUT/httpx.jsonl" "$OUT/nuclei.jsonl" > "$OUT/summary.json"
```

### Paso 7 — Emitir summary.md

Compón `summary.md` con las secciones siguientes en este orden exacto:

- `# Recon summary — {{ target }}`
- `## Subdomains` — lista con viñetas a partir de `subdomains.txt`.
- `## Live services` — tabla con columnas URL | Status | Title | Tech.
- `## Findings` — tabla con columnas Template ID | Severity | URL.
- `## Next steps` — una recomendación de una oración por cada severidad presente.

## Postcondiciones

- `./recon/{{ target }}/summary.md` existe y no está vacío.
- `./recon/{{ target }}/summary.json` parsea como JSON válido.
- Devuélvele al usuario la ruta de `summary.md`.

## Modos de fallo

- Violación de scope: exit 3, no toques el filesystem más allá de `mkdir`.
- Tool ausente: exit 4, nombra la tool.
- Cero subdominios: emite un `summary.md` vacío pero bien formado, con una nota "no subdomains discovered"; no te saltes los pasos siguientes.
````


### 3.3 Verificación

Levanta el stack del lab, llena un override de DNS local para que `lab.local` resuelva sobre el bridge de Docker e invoca la skill:

```bash
echo "172.26.0.10 lab.local juice.lab.local"  | sudo tee -a /etc/hosts
echo "172.26.0.11 dvwa.lab.local"             | sudo tee -a /etc/hosts
echo "172.26.0.12 vulnnode.lab.local"         | sudo tee -a /etc/hosts

cd ~/pentest-tools
ENGAGEMENT_SCOPE=./engagement/scope.yml \
  claude skills run recon-scanner --input target=lab.local
```

Confirma el árbol de salida:

```bash
tree recon/lab.local/
# recon/lab.local/
# ├── httpx.jsonl
# ├── nuclei.jsonl
# ├── subdomains.txt
# ├── summary.json
# ├── summary.md
# └── urls.txt

jq -e '.target == "lab.local"' recon/lab.local/summary.json
test -s recon/lab.local/summary.md
```

Prueba negativa: la skill debe rechazar un objetivo fuera de scope.

```bash
claude skills run recon-scanner --input target=example.com
# Esperado: exit 3, mensaje "target example.com is NOT in engagement scope"
```

Prueba de idempotencia: una segunda invocación no debe duplicar la salida.

```bash
BEFORE=$(wc -l < recon/lab.local/subdomains.txt)
claude skills run recon-scanner --input target=lab.local
AFTER=$(wc -l < recon/lab.local/subdomains.txt)
test "$BEFORE" -eq "$AFTER"
```

## 4. Desafío 2 — Skill validadora de vulns

### 4.1 Requisitos

Construye `vuln-validator/SKILL.md` que satisfaga cada viñeta de abajo.

- La skill acepta una entrada `finding`: o una ruta a un archivo de una sola línea JSON de nuclei, o un string JSON crudo.
- Antes de ejecutar nada, debe confirmar que la URL objetivo es alcanzable desde la red actual (un HEAD de un solo tiro).
- Debe reproducir el request exacto que emitió el template de nuclei, usando `curl -v` para que tanto el request como la response se puedan volcar.
- Debe guardar el paquete de evidencia bajo `./findings/<slug>/`, donde `<slug>` = `<template_id>-<host>-<sha1[:8]>`.
- El paquete debe contener:
  - `finding.yml` — metadatos canónicos (template ID, severidad, host, URL, matched_at, nombre del matcher, tags, timestamp).
  - `request.http` — el request HTTP crudo que reprodujo el finding.
  - `response.http` — la response cruda, headers más body (truncado a 64 KiB).
  - `screenshot.png` — si chromium está en PATH, un screenshot headless de la URL.
- La skill debe degradar con gracia: si falta chromium se salta el screenshot pero no falla la corrida.
- Si falla el chequeo de alcanzabilidad, la skill sale con un estado legible por máquina y no crea el paquete.

### 4.2 SKILL.md completo

Guarda lo siguiente como `~/pentest-tools/skills/vuln-validator/SKILL.md`:

````markdown
---
name: vuln-validator
description: >-
  Toma un finding crudo de nuclei y produce un paquete de evidencia
  reproducible. Verifica que el objetivo sea alcanzable, rejuega el request
  ofensor con curl, captura la response y (cuando chromium está disponible)
  saca un screenshot. Emite ./findings/<slug>/ con finding.yml, request.http,
  response.http y screenshot.png. Seguro de invocar sin supervisión.
version: 1.0.0
author: Red Team Leaders
license: MIT
inputs:
  - name: finding
    type: string
    required: true
    description: >-
      Ruta a un archivo JSONL de nuclei (una sola línea) o un string JSON
      crudo que contiene un finding de nuclei.
tools_required:
  - curl
  - jq
  - yq
  - sha1sum
optional_tools:
  - chromium
allowed_bash:
  - "curl *"
  - "jq *"
  - "yq *"
  - "sha1sum *"
  - "chromium *"
  - "mkdir -p *"
  - "head -c *"
  - "test *"
  - "printf *"
---

# vuln-validator

## Cuándo invocarla

Usa esta skill después de que `recon-scanner` (o cualquier otra corrida de
nuclei) haya producido findings JSONL que quieras promover de "señal cruda"
a "evidencia reportable". Úsala también cada vez que el usuario diga
"validar", "reproducir", "confirmar" o "probar" un finding concreto.

No uses esta skill para correr requests arbitrarios: el request sale del
finding de nuclei, no del usuario.

## Precondiciones

1. La entrada `finding` está definida (ruta o JSON inline).
2. `curl`, `jq`, `yq` y `sha1sum` están en PATH.
3. El directorio de trabajo es escribible.

## Procedimiento

### Paso 1 — Parsear el finding

```bash
IN="{{ finding }}"
if test -f "$IN"; then
  RAW=$(cat "$IN")
else
  RAW="$IN"
fi

TEMPLATE_ID=$(echo "$RAW" | jq -r '.template_id')
SEVERITY=$(echo "$RAW"    | jq -r '.info.severity')
MATCHED_AT=$(echo "$RAW"  | jq -r '.matched_at')
HOST=$(echo "$MATCHED_AT" | awk -F/ '{print $3}')
METHOD=$(echo "$RAW"      | jq -r '.request // "GET" | tostring | split("\n")[0] | split(" ")[0] // "GET"')
REQBODY=$(echo "$RAW"     | jq -r '.request // ""')
SLUG_HASH=$(echo -n "$RAW" | sha1sum | cut -c1-8)
SLUG="${TEMPLATE_ID}-${HOST}-${SLUG_HASH}"
OUT="./findings/${SLUG}"
mkdir -p "$OUT"
```

### Paso 2 — Chequeo de alcanzabilidad

```bash
if ! curl -s -o /dev/null -m 5 -w "%{http_code}\n" "$MATCHED_AT" | grep -qE "^[23][0-9]{2}$|^401$|^403$"; then
  printf "unreachable: %s\n" "$MATCHED_AT" > "$OUT/.unreachable"
  echo "UNREACHABLE"; exit 5
fi
```

### Paso 3 — Reproducir el request

```bash
curl -sk -o "$OUT/response.body" \
     -D "$OUT/response.headers" \
     --trace-ascii "$OUT/curl.trace" \
     -m 15 -X "$METHOD" "$MATCHED_AT"

# Arma request.http a partir del trace (o del request que aportó nuclei)
if test -n "$REQBODY"; then
  printf "%s\n" "$REQBODY" > "$OUT/request.http"
else
  awk '/^=> Send header/{flag=1;next} /^<=|^== Info/{flag=0} flag {print}' \
      "$OUT/curl.trace" > "$OUT/request.http"
fi

# Arma response.http = headers + línea en blanco + primeros 64 KiB del body
{ cat "$OUT/response.headers"; echo; head -c 65536 "$OUT/response.body"; } > "$OUT/response.http"
rm -f "$OUT/curl.trace" "$OUT/response.body" "$OUT/response.headers"
```

### Paso 4 — Screenshot (best effort)

```bash
if command -v chromium >/dev/null 2>&1; then
  chromium --headless --disable-gpu --no-sandbox \
           --window-size=1280,900 \
           --screenshot="$OUT/screenshot.png" \
           "$MATCHED_AT" >/dev/null 2>&1 || true
fi
```

### Paso 5 — Emitir finding.yml

```bash
cat > "$OUT/finding.yml" <<YAML
template_id: "${TEMPLATE_ID}"
severity:    "${SEVERITY}"
host:        "${HOST}"
url:         "${MATCHED_AT}"
matched_at:  "${MATCHED_AT}"
slug:        "${SLUG}"
captured_at: "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
evidence:
  request:    "request.http"
  response:   "response.http"
  screenshot: "$([ -f "$OUT/screenshot.png" ] && echo screenshot.png || echo null)"
YAML
```

## Postcondiciones

- `./findings/<slug>/finding.yml` parsea como YAML válido.
- `./findings/<slug>/request.http` y `response.http` existen y no están vacíos.
- Si chromium estaba disponible, `./findings/<slug>/screenshot.png` existe y es un PNG válido.

## Modos de fallo

- JSON de finding malformado: exit 2, nombra el campo que falta.
- Objetivo inalcanzable: exit 5, escribe el centinela `.unreachable`, no crees los archivos de request/response.
- Timeout de curl: exit 6, no crees `finding.yml`.
````


### 4.3 Verificación

Saca una línea de la salida de nuclei del desafío 1 y aliméntala:

```bash
cd ~/pentest-tools
head -1 recon/lab.local/nuclei.jsonl > /tmp/one-finding.jsonl

claude skills run vuln-validator --input finding=/tmp/one-finding.jsonl

# Confirma el paquete
SLUG=$(ls findings/ | head -1)
test -f "findings/$SLUG/finding.yml"
test -s "findings/$SLUG/request.http"
test -s "findings/$SLUG/response.http"
yq -r '.template_id' "findings/$SLUG/finding.yml"
```

Prueba negativa: un finding que apunta a un host inalcanzable debe salir con 5 y no dejar otro paquete que el centinela.

```bash
echo '{"template_id":"tech-detect","info":{"severity":"info"},"matched_at":"http://172.26.0.99/"}' \
  > /tmp/dead.jsonl
claude skills run vuln-validator --input finding=/tmp/dead.jsonl
# Exit esperado 5, archivo .unreachable presente, sin request.http/response.http.
```

Prueba de degradación del screenshot: renombra chromium de forma temporal y vuelve a correr; la skill debe tener éxito sin producir `screenshot.png`.

```bash
sudo mv /usr/bin/chromium /usr/bin/chromium.bak
claude skills run vuln-validator --input finding=/tmp/one-finding.jsonl
sudo mv /usr/bin/chromium.bak /usr/bin/chromium
```

## 5. Desafío 3 — Skill generadora de reportes

### 5.1 Requisitos

Construye `report-generator/SKILL.md` que satisfaga cada viñeta de abajo.

- La skill acepta entradas opcionales `findings_dir` (por defecto `./findings`) y `out` (por defecto `./report`).
- Recorre cada `finding.yml` bajo `findings_dir`, a un nivel de directorio.
- Para cada finding calcula un base score CVSS 4.0 usando el vector guardado en el finding, o cae a una tabla de severidad a puntaje cuando el vector está ausente.
- Redacta un resumen ejecutivo de dos párrafos que nombra al cliente (desde `engagement/scope.yml`), el conteo de findings por severidad y el tema de mayor riesgo.
- Arma un reporte Markdown `report.md` con las secciones: Cover, Executive Summary, Scope, Methodology, Findings (una subsección por finding), Appendix.
- Cada subsección de finding embebe los recortes de request/response (las primeras 40 líneas de cada uno) y, si existe un screenshot, lo referencia con un enlace de imagen relativo.
- Si typst está en PATH y existe un template `report.typ` junto a la skill, la skill invoca `typst compile` y produce `report.pdf`.
- La skill debe ser segura de volver a correr: sobrescribe `report.md` y `report.pdf` pero nunca toca los `findings/` de origen.

### 5.2 SKILL.md completo

Guarda lo siguiente como `~/pentest-tools/skills/report-generator/SKILL.md`:

````markdown
---
name: report-generator
description: >-
  Lee cada findings/*/finding.yml, calcula base scores CVSS 4.0, redacta
  un resumen ejecutivo, arma un reporte Markdown de pentest y (si typst
  está disponible) renderiza un PDF. Lee engagement/scope.yml para los
  metadatos del cliente. Nunca modifica el árbol de findings de origen.
version: 1.0.0
author: Red Team Leaders
license: MIT
inputs:
  - name: findings_dir
    type: string
    required: false
    default: "./findings"
    description: Directorio a recorrer en busca de archivos finding.yml.
  - name: out
    type: string
    required: false
    default: "./report"
    description: Directorio donde escribir report.md y report.pdf.
tools_required:
  - yq
  - jq
  - python3
optional_tools:
  - typst
allowed_bash:
  - "yq *"
  - "jq *"
  - "python3 *"
  - "typst *"
  - "mkdir -p *"
  - "ls *"
  - "find *"
  - "test *"
  - "wc *"
  - "head *"
  - "cp *"
---

# report-generator

## Cuándo invocarla

Usa esta skill al final de un engagement, una vez que `vuln-validator` haya
producido uno o más paquetes `findings/<slug>/finding.yml`. Úsala también
cada vez que el usuario diga "redactar el reporte", "compilar el reporte",
"resumen ejecutivo" o "renderizar el reporte de pentest".

No uses esta skill para inventar findings: cada subsección tiene que
rastrearse hasta un `finding.yml` en disco.

## Precondiciones

1. `findings_dir` existe y contiene al menos un `finding.yml`.
2. `engagement/scope.yml` existe (para el nombre del cliente, la ventana y el código).
3. `yq`, `jq` y `python3` están en PATH.

## Procedimiento

### Paso 1 — Preparar el árbol de salida

```bash
FDIR="{{ findings_dir | default('./findings') }}"
OUT="{{ out | default('./report') }}"
mkdir -p "$OUT/evidence"
```

### Paso 2 — Cargar el scope

```bash
SCOPE="${ENGAGEMENT_SCOPE:-./engagement/scope.yml}"
CLIENT=$(yq -r '.engagement.client'       "$SCOPE")
CODE=$(yq -r   '.engagement.code'         "$SCOPE")
WINDOW_START=$(yq -r '.engagement.window_start' "$SCOPE")
WINDOW_END=$(yq -r   '.engagement.window_end'   "$SCOPE")
```

### Paso 3 — Enumerar findings y puntuarlos

```bash
python3 - <<'PY' "$FDIR" "$OUT/findings.json"
import sys, json, glob, os
try:
    import yaml
except ImportError:
    o​s.system("pip install --quiet pyyaml")
    import yaml

SEV_FALLBACK = {"critical": 9.5, "high": 8.0, "medium": 5.5, "low": 3.0, "info": 0.0}

def cvss4_from_vector(v):
    # Puntaje CVSS 4.0 mínimo: esto es una implementación de lab; la
    # calculadora real vive en una skill helper aparte. Aproximamos
    # mapeando las métricas base a la rúbrica estándar.
    if not v or "CVSS:4.0" not in v:
        return None
    parts = dict(x.split(":") for x in v.split("/")[1:] if ":" in x)
    weights = {"AV":{"N":0.85,"A":0.62,"L":0.55,"P":0.2},
               "AC":{"L":0.77,"H":0.44},
               "PR":{"N":0.85,"L":0.62,"H":0.27},
               "UI":{"N":0.85,"P":0.62,"A":0.27}}
    impact = {"H":0.56,"L":0.22,"N":0.0}
    exploit = 8.22 * weights["AV"].get(parts.get("AV","N"),0.85) \
                   * weights["AC"].get(parts.get("AC","L"),0.77) \
                   * weights["PR"].get(parts.get("PR","N"),0.85) \
                   * weights["UI"].get(parts.get("UI","N"),0.85)
    isc = 1 - ((1-impact.get(parts.get("VC","N"),0.0))
              *(1-impact.get(parts.get("VI","N"),0.0))
              *(1-impact.get(parts.get("VA","N"),0.0)))
    isc = 7.52*(isc - 0.029) - 3.25*((isc - 0.02)**15)
    if isc <= 0: return 0.0
    return round(min(10.0, exploit + isc), 1)

out = []
for f in sorted(glob.glob(os.path.join(sys.argv[1], "*", "finding.yml"))):
    with open(f) as fh:
        y = yaml.safe_load(fh)
    vec = y.get("cvss_vector")
    score = cvss4_from_vector(vec) if vec else SEV_FALLBACK.get(y.get("severity","info").lower(), 0.0)
    y["cvss4_score"] = score
    y["source_dir"]  = os.path.dirname(f)
    out.append(y)

with open(sys.argv[2], "w") as fh:
    json.dump(out, fh, indent=2)
print(f"se cargaron {len(out)} findings", file=sys.stderr)
PY
```

### Paso 4 — Redactar el resumen ejecutivo

```bash
python3 - <<'PY' "$OUT/findings.json" "$OUT/summary.md" "$CLIENT" "$CODE"
import sys, json, collections
findings = json.load(open(sys.argv[1]))
client, code = sys.argv[3], sys.argv[4]
bysev = collections.Counter(f["severity"].lower() for f in findings)
top   = sorted(findings, key=lambda f: -f.get("cvss4_score", 0.0))[:3]
top_names = ", ".join(f["template_id"] for f in top) or "no reportable findings"

with open(sys.argv[2], "w") as fh:
    fh.write(f"## Executive summary\n\n")
    fh.write(f"Red Team Leaders realizó un pentest externo para "
             f"{client} bajo el engagement {code}. A lo largo de la ventana de prueba "
             f"identificamos {len(findings)} findings: "
             f"{bysev.get('critical',0)} critical, {bysev.get('high',0)} high, "
             f"{bysev.get('medium',0)} medium, {bysev.get('low',0)} low y "
             f"{bysev.get('info',0)} informational.\n\n")
    fh.write(f"Los temas de mayor riesgo fueron: {top_names}. Estos son los "
             f"issues que recomendamos priorizar primero; cada uno es reproducible "
             f"a partir de la evidencia empaquetada con este reporte.\n")
PY
```


### Paso 5 — Armar report.md

```bash
python3 - <<'PY' "$OUT/findings.json" "$OUT/summary.md" "$OUT/report.md" "$SCOPE"
import sys, json, os, yaml, textwrap
findings = json.load(open(sys.argv[1]))
summary  = open(sys.argv[2]).read()
scope    = yaml.safe_load(open(sys.argv[3]))
client   = scope["engagement"]["client"]
code     = scope["engagement"]["code"]

def snippet(path, n=40):
    if not os.path.exists(path): return "(missing)"
    with open(path, errors="replace") as fh:
        return "".join(fh.readlines()[:n])

with open(sys.argv[2].replace("summary.md","report.md"), "w") as fh:
    fh.write(f"# Penetration Test Report — {client}\n\n")
    fh.write(f"Engagement code: **{code}**\n\n")
    fh.write(f"Preparado por Red Team Leaders.\n\n---\n\n")
    fh.write(summary + "\n\n---\n\n")
    fh.write("## Scope\n\n")
    for d in scope["in_scope"]["domains"]:
        fh.write(f"- {d}\n")
    fh.write("\n## Methodology\n\n")
    fh.write("El reconocimiento se hizo con la skill `recon-scanner` "
             "(subfinder + httpx + nuclei -tags basic). Cada finding candidato "
             "se promovió a un finding validado con la skill `vuln-validator`, "
             "que reproduce el request con curl y captura la "
             "response y (cuando está disponible) un screenshot de chromium headless.\n\n")
    fh.write("## Findings\n\n")
    for f in sorted(findings, key=lambda x: -x.get("cvss4_score",0.0)):
        fh.write(f"### {f['template_id']} — {f['severity'].upper()} (CVSS4 {f['cvss4_score']})\n\n")
        fh.write(f"- Host: `{f['host']}`\n")
        fh.write(f"- URL: `{f['url']}`\n")
        fh.write(f"- Captured: {f['captured_at']}\n\n")
        fh.write("#### Request\n\n```http\n")
        fh.write(snippet(os.path.join(f["source_dir"], "request.http")))
        fh.write("```\n\n#### Response\n\n```http\n")
        fh.write(snippet(os.path.join(f["source_dir"], "response.http")))
        fh.write("```\n\n")
        shot = os.path.join(f["source_dir"], "screenshot.png")
        if os.path.exists(shot):
            fh.write(f"![screenshot]({shot})\n\n")
    fh.write("## Appendix\n\nToda la salida cruda de las tools se conserva bajo `./recon/` y `./findings/`.\n")
PY
```


### Paso 6 — Opcional: renderizar PDF vía typst

```bash
if command -v typst >/dev/null 2>&1 && test -f "$(dirname "$0")/report.typ"; then
  cp "$(dirname "$0")/report.typ" "$OUT/report.typ"
  typst compile "$OUT/report.typ" "$OUT/report.pdf" || echo "typst compile failed (non-fatal)"
fi
```

## Postcondiciones

- `${out}/report.md` existe y empieza con `# Penetration Test Report`.
- `${out}/findings.json` existe y es un array JSON.
- Si typst tuvo éxito, `${out}/report.pdf` existe.

## Modos de fallo

- `findings_dir` vacío: emite `report.md` con una nota "empty engagement"; no falles.
- `finding.yml` malformado: salta ese finding, registra su ruta en stderr y sigue.
- typst ausente: salta el PDF, no falles.
````

### 5.3 Verificación

Fija dos findings de lab en su lugar antes de correr la skill, para que el test sea determinista:

```bash
mkdir -p ~/pentest-tools/findings/tech-detect-172.26.0.10-aaaaaaaa
cat > ~/pentest-tools/findings/tech-detect-172.26.0.10-aaaaaaaa/finding.yml <<YAML
template_id: tech-detect
severity: info
host: 172.26.0.10
url: http://172.26.0.10/
matched_at: http://172.26.0.10/
slug: tech-detect-172.26.0.10-aaaaaaaa
captured_at: "2026-09-05T09:00:00Z"
cvss_vector: "CVSS:4.0/AV:N/AC:L/PR:N/UI:N/VC:N/VI:N/VA:N"
YAML
printf 'GET / HTTP/1.1\nHost: 172.26.0.10\n\n' > ~/pentest-tools/findings/tech-detect-172.26.0.10-aaaaaaaa/request.http
printf 'HTTP/1.1 200 OK\nServer: nginx\n\n<html>Juice Shop</html>\n' > ~/pentest-tools/findings/tech-detect-172.26.0.10-aaaaaaaa/response.http

mkdir -p ~/pentest-tools/findings/xss-dvwa-172.26.0.11-bbbbbbbb
cat > ~/pentest-tools/findings/xss-dvwa-172.26.0.11-bbbbbbbb/finding.yml <<YAML
template_id: dvwa-reflected-xss
severity: high
host: 172.26.0.11
url: http://172.26.0.11/vulnerabilities/xss_r/?name=%3Cscript%3Ealert(1)%3C%2Fscript%3E
matched_at: http://172.26.0.11/vulnerabilities/xss_r/
slug: xss-dvwa-172.26.0.11-bbbbbbbb
captured_at: "2026-09-05T09:05:00Z"
cvss_vector: "CVSS:4.0/AV:N/AC:L/PR:N/UI:P/VC:L/VI:L/VA:N"
YAML
printf 'GET /vulnerabilities/xss_r/?name=<script>alert(1)</script> HTTP/1.1\nHost: 172.26.0.11\n\n' > ~/pentest-tools/findings/xss-dvwa-172.26.0.11-bbbbbbbb/request.http
printf 'HTTP/1.1 200 OK\nContent-Type: text/html\n\n<pre>Hello <script>alert(1)</script></pre>\n' > ~/pentest-tools/findings/xss-dvwa-172.26.0.11-bbbbbbbb/response.http
```

Corre la skill y afirma la estructura de la salida:

```bash
cd ~/pentest-tools
claude skills run report-generator --input findings_dir=./findings --input out=./report

test -f report/report.md
grep -q "^# Penetration Test Report" report/report.md
grep -q "Executive summary" report/report.md
grep -q "dvwa-reflected-xss" report/report.md
grep -q "tech-detect"        report/report.md
jq -e 'length == 2' report/findings.json
```

Si existe un `report.typ` junto a la skill, confirma que el PDF se renderiza:

```bash
test -f report/report.pdf && echo "PDF rendered" || echo "PDF skipped (expected if no report.typ)"
```

## 6. Metodología de testing

Las skills fallan de tres formas características: el frontmatter está mal y el loader las rechaza, el cuerpo está bien pero hace lo incorrecto con entradas raras, o el cuerpo está bien para un objetivo pero no es idempotente a través de dos. Prueba en tres niveles.

### 6.1 Tests de schema (nivel loader)

Toda skill que escribas tiene que parsear limpio por el loader. Agrega un harness chico en `~/pentest-tools/skills/tests/schema_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
for s in recon-scanner vuln-validator report-generator; do
  claude skills validate "$s" >/dev/null
  echo "OK: $s"
done
```

Corre `bash tests/schema_test.sh`: espera tres líneas `OK`.

### 6.2 Tests unitarios (guiados por fixtures)

Toda skill tiene uno o dos puntos de decisión fáciles de aislar y baratos de probar. Alimenta entradas de fixture y afirma los efectos secundarios.

Para recon-scanner, el punto de decisión es la validación de scope. Escribe un archivo de scope de fixture, invoca la skill contra un objetivo dentro de scope y uno fuera, y chequea los códigos de salida:

```bash
#!/usr/bin/env bash
# tests/recon_scope_test.sh
set -euo pipefail
export ENGAGEMENT_SCOPE=./tests/fixtures/scope.yml
mkdir -p tests/fixtures
cat > tests/fixtures/scope.yml <<'YAML'
engagement: {code: TEST, client: T, window_start: "", window_end: ""}
in_scope:  {domains: [example.test], cidrs: [], urls: []}
out_of_scope: {domains: [], cidrs: []}
rules: []
YAML
if claude skills run recon-scanner --input target=example.test >/dev/null 2>&1; then
  echo "in-scope PASS"
else
  echo "in-scope FAIL"; exit 1
fi
if claude skills run recon-scanner --input target=evil.test >/dev/null 2>&1; then
  echo "out-of-scope FAIL"; exit 1
else
  echo "out-of-scope PASS"
fi
```

Para vuln-validator, el punto de decisión es la alcanzabilidad. Apúntala a una URL alcanzable y a una inalcanzable y afirma sobre el centinela `.unreachable`.

Para report-generator, el punto de decisión es la enumeración de findings. Apúntala a un `findings_dir` vacío y afirma que el reporte igual se arma con una nota "empty engagement".

### 6.3 Tests de integración (lab en vivo)

Cuando pasen los tests de schema y los unitarios, corre las skills contra el lab de Docker de punta a punta. Cablea las tres skills juntas en un solo script: este es exactamente el patrón que usa el módulo 07.

```bash
#!/usr/bin/env bash
# tests/integration_full_stack.sh
set -euo pipefail
cd ~/pentest-tools
claude skills run recon-scanner --input target=lab.local
for line in $(cat recon/lab.local/nuclei.jsonl); do
  echo "$line" > /tmp/f.jsonl
  claude skills run vuln-validator --input finding=/tmp/f.jsonl || true
done
claude skills run report-generator --input findings_dir=./findings --input out=./report
test -f report/report.md
echo "integration OK"
```

Todo lo que pondrías detrás de CI en un engagement real de cliente vive en este script.

### 6.4 Testing de regresión

Cada vez que cambies un `SKILL.md`, vuelve a correr los tres niveles de test. Las skills derivan en silencio: un campo de frontmatter renombrado por el loader en un release nuevo de Claude Code no va a dar error, simplemente se va a ignorar, y tu skill va a empezar a hacer algo sutilmente mal. Pinea las versiones del CLI en `tool-versions.txt` y pinea los schemas de las skills manteniendo los tests en verde en cada commit.

## 7. Empaquetado de skills (distribución como plugin)

Claude Code acepta un plugin como un directorio que contiene un `plugin.json` y una o más `skills/`. Cuando las tres skills pasen los tests, empaquétalas para que un compañero pueda instalar el conjunto entero con un comando.

### 7.1 Layout del directorio

```text
~/pentest-tools/plugins/rtl-pentest/
├── plugin.json
├── README.md
└── skills/
    ├── recon-scanner/
    │   └── SKILL.md
    ├── vuln-validator/
    │   └── SKILL.md
    └── report-generator/
        ├── SKILL.md
        └── report.typ
```

### 7.2 plugin.json

```json
{
  "name": "rtl-pentest",
  "version": "1.0.0",
  "description": "Paquete de skills de pentest de Red Team Leaders: recon-scanner, vuln-validator, report-generator.",
  "author": "Red Team Leaders",
  "license": "MIT",
  "homepage": "https://github.com/red-team-leaders/rtl-pentest",
  "requires": {
    "claude-code": ">=1.0.0",
    "codex-cli":  ">=0.9.0"
  },
  "external_tools": [
    "subfinder", "httpx", "nuclei", "curl", "jq", "yq", "python3", "chromium", "typst"
  ],
  "skills": [
    "skills/recon-scanner",
    "skills/vuln-validator",
    "skills/report-generator"
  ]
}
```

### 7.3 Instalación local

```bash
claude plugins install ~/pentest-tools/plugins/rtl-pentest
claude plugins list | grep rtl-pentest
claude skills list | grep -E "recon-scanner|vuln-validator|report-generator"
```

### 7.4 Publicar a un remote de git

```bash
cd ~/pentest-tools
git add plugins/rtl-pentest
git commit -m "feat(plugin): package rtl-pentest v1.0.0"
git remote add origin git@github.com:<you>/pentest-tools.git
git push -u origin main
```

Un compañero lo instala con:

```bash
claude plugins install git+https://github.com/<you>/pentest-tools#subdirectory=plugins/rtl-pentest
```

### 7.5 Disciplina de versionado

Sube la versión de `plugin.json` en cada cambio de `SKILL.md`. Etiqueta el commit de git con la misma versión:

```bash
git tag -a rtl-pentest-v1.0.0 -m "rtl-pentest 1.0.0"
git push origin rtl-pentest-v1.0.0
```

Semver aplica también a nivel de skill: un cambio al schema de entrada es un bump major; una entrada opcional nueva es un minor; un arreglo al cuerpo del `SKILL.md` es un patch.

## 8. Desafíos independientes

Estos son abiertos. Cada uno debería tomar entre dos y seis horas. No abras un PR hasta que los tests pasen en local.

### 8.1 Skill de OSINT pasivo

Construye `passive-osint/SKILL.md` que nunca toque el objetivo. Encadena tres cualesquiera de: transparencia de certificados de crt.sh, resolución pasiva de dnsx, API de chaos, enumeración de URLs de Wayback Machine, dorks de GitHub vía la API. Emite `./osint/<target>/summary.md` con la misma forma que recon-scanner. La rúbrica es: nada de lo que hagas debe producir un paquete que el WAF del objetivo pueda registrar.

### 8.2 Skill de diff de reportes

Construye `report-diff/SKILL.md` que consuma dos archivos `report.md` (un engagement inicial y un retest) y produzca `diff.md` mostrando qué findings están arreglados, siguen abiertos, cambiaron de severidad y son nuevos. La rúbrica es: el diff tiene que ser reproducible a partir de las dos entradas solas, sin estado oculto.

### 8.3 Skill envuelta en MCP

Envuelve el server MCP `nuclei-templates-server` como punto de entrada de una skill llamada `templates-lookup`. La skill acepta una entrada `cve`, llama al server MCP para traer el template que coincide e imprime el cuerpo del template. La rúbrica es: el `SKILL.md` debe declarar el server MCP en su frontmatter `mcp_servers` y debe fallar limpio cuando el server MCP no está conectado (recuerda, del módulo 06 lección 09, que un fallo de conexión es una condición de tiempo de ejecución, no un error de configuración).

## 9. Rúbrica de calificación

| Ítem | Máx | Criterio de aprobado |
|---|---|---|
| Entorno configurado correctamente | 5 | `docker compose ps` muestra tres servicios sanos; `claude skills list` muestra las tres skills |
| `recon-scanner` SKILL.md válido | 10 | `claude skills validate recon-scanner` sale con 0 |
| Cumplimiento de scope de recon-scanner | 10 | Un objetivo fuera de scope sale con 3 sin escribir archivos |
| Estructura de salida de recon-scanner | 10 | `summary.md` y `summary.json` presentes, bien formados, idempotentes |
| `vuln-validator` SKILL.md válido | 10 | `claude skills validate vuln-validator` sale con 0 |
| Paquete de evidencia de vuln-validator | 10 | `finding.yml`, `request.http` y `response.http` presentes y no vacíos |
| Degradación con gracia de vuln-validator | 5 | La ausencia de chromium no falla la corrida |
| `report-generator` SKILL.md válido | 10 | `claude skills validate report-generator` sale con 0 |
| Estructura del reporte de report-generator | 10 | `report.md` tiene portada, resumen ejecutivo, scope, metodología, findings y apéndice |
| Enriquecimiento CVSS de report-generator | 5 | Findings ordenados por puntaje CVSS 4.0 en el reporte |
| Metodología de testing aplicada | 5 | `tests/*.sh` todos en verde |
| Empaquetado del plugin | 5 | `claude plugins install .../rtl-pentest` tiene éxito; las skills son descubribles |
| Desafíos independientes (1 de 3) | 5 | Solución presente, probada y documentada |
| **Total** | **100** | Nota de aprobado: 75 |

## 10. Entregar tu trabajo

El objetivo de la entrega es tu repositorio git personal `~/pentest-tools/skills/`. Todo módulo posterior asume que este repo existe y está al día.

```bash
cd ~/pentest-tools
git add skills/ plugins/ tests/ tool-versions.txt engagement/scope.yml
git status
```

Lista esperada de no trackeados o modificados:

```text
skills/recon-scanner/SKILL.md
skills/vuln-validator/SKILL.md
skills/report-generator/SKILL.md
skills/report-generator/report.typ         (opcional)
plugins/rtl-pentest/plugin.json
plugins/rtl-pentest/README.md
tests/schema_test.sh
tests/recon_scope_test.sh
tests/integration_full_stack.sh
tool-versions.txt
engagement/scope.yml
```

Haz commit con un mensaje semántico:

```bash
git commit -m "feat(module-06): recon-scanner, vuln-validator, report-generator v1.0.0"
```

Haz push a tu remote:

```bash
git push origin main
git tag -a module-06-complete -m "Module 06 lab complete"
git push origin module-06-complete
```

Verifica una vez más que un checkout fresco en un host limpio puede correr los tres tests:

```bash
cd /tmp
git clone git@github.com:<you>/pentest-tools.git
cd pentest-tools
bash tests/schema_test.sh
bash tests/recon_scope_test.sh
bash tests/integration_full_stack.sh
```

Si esos tres comandos salen todos con 0, terminaste el lab.

Entrega los artefactos siguientes al portal del curso:

- El SHA del commit del tag `module-06-complete`.
- Un screenshot de `claude skills list` que muestre las tres skills registradas.
- El `report/report.md` generado por el test de integración.
- Un párrafo corto que describa cuál de los tres desafíos independientes intentaste y cómo lo probaste.

## 11. Resumen y puntero al módulo 07

Ahora tienes tres skills en `~/.claude/skills/` que entre las tres cubren el loop completo de un engagement externo: enumerar, validar, reportar. Tienes un harness de test que atrapa roturas a nivel de loader, una suite guiada por fixtures que atrapa regresiones de puntos de decisión, y un script de integración que ejercita toda la cadena contra un lab dockerizado. Empaquetaste las tres skills como un plugin distribuible de Claude Code, etiquetaste un v1.0.0 y lo pusheaste a un remote de git desde el que tus compañeros pueden instalarlo.

Las skills son deliberadamente simples. Envuelven primitivos bien conocidos (subfinder, httpx, nuclei, curl, typst) y se apoyan en el contrato `SKILL.md` de Claude Code para la orquestación. Esa simplicidad es el punto: el valor no está en los wrappers de tools, está en la disciplina de cumplimiento del scope, el layout de salida idempotente, los paquetes de evidencia reproducibles y un reporte que rastrea cada afirmación hasta un archivo en disco.

El módulo 07 (Real Client Engagements) retoma aquí. Cada recorrido del módulo 07 empieza invocando `recon-scanner` contra el dominio dentro de scope del cliente, promueve findings seleccionados vía `vuln-validator` y cierra con `report-generator`. Si no construiste y probaste las tres skills, los recorridos del módulo 07 van a fallar en el primer paso.

Antes de dejar esta lección, recorre la lista de chequeo una última vez:

- `docker compose ps` en `~/labs/module06` muestra tres servicios sanos.
- `claude skills list` muestra recon-scanner, vuln-validator y report-generator.
- `bash tests/schema_test.sh` imprime tres líneas OK.
- `bash tests/integration_full_stack.sh` sale con 0 y produce `report/report.md`.
- `git log --oneline` muestra un commit `feat(module-06)` etiquetado `module-06-complete`.
- Tu remote git de `~/pentest-tools/skills/` está al día.

¿Todo en verde? Sigue al módulo 07 — Real Client Engagements.
