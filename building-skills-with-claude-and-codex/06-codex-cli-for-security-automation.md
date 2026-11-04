# Lección 06 — Codex CLI for Security Automation

Módulo 06: Building Pentest Skills with Claude and Codex · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Claude Code no es el único agent de código autónomo que pertenece a la caja de herramientas de quien hace red team. Codex CLI de OpenAI — publicado en abril de 2025 y endurecido de forma continua a lo largo de 2026 — se ha convertido en la otra mitad del stack moderno de IA para pentest. Donde Claude Code destaca en la exploración de contexto largo, la planificación estructurada y el encadenamiento de tools impulsado por MCP, Codex CLI es el agent ligero y cómodo para trabajo por lotes al que recurres cuando necesitas aplicar el mismo razonamiento sobre doscientos targets sin pasarte medio día escribiendo una orquestación a medida.

Las dos herramientas se complementan. Claude Code es el cirujano: cuidadoso, verboso, deliberado. Codex CLI es el médico de campaña: rápido, no interactivo, con opiniones firmes sobre su sandbox, barato de invocar a escala. En un engagement completo vas a usar casi seguro ambos — a menudo contra los mismos datos, en fases distintas.

Esta lección recorre la instalación de Codex CLI en Kali 2026, cómo se diferencia de Claude Code a nivel mecánico, cómo dejarlo listo para trabajo de pentest y cómo construir un script real de triage por lotes que pasa findings de nuclei por el agent para una validación asistida por el LLM. Al terminar vas a tener un `codex-triage.sh` funcional en disco, vas a saber con exactitud cuándo recurrir a Codex y cuándo a Claude, y vas a saber cómo ejecutar un mismo MCP server detrás de ambos agents.

Todo en esta lección usa la sintaxis de Codex CLI vigente a septiembre de 2026. Codex se ha movido deprisa; si un flag de un comando se ve distinto de un post de blog de 2025, fíate primero de la salida de `--help` en tu máquina.

## 1. Qué es Codex CLI (y qué no es)

Codex CLI es el agent de código autónomo de código abierto de OpenAI que corre por completo en tu estación de trabajo. El binario es un proceso en Rust (`codex`) que incluye un bucle de agent, un sandbox, un cliente MCP pequeño y un conjunto de tools integradas para leer y escribir archivos y ejecutar comandos de shell. Habla con un modelo de OpenAI — por defecto GPT-5.1-Codex — a través de tu cuenta de ChatGPT o de una API key.

En concreto:

- Codex CLI es la herramienta en disco: `codex`, invocada desde la terminal.
- Codex (el modelo) es lo que lo impulsa: la familia GPT de OpenAI afinada para código (`gpt-5.1-codex`, `gpt-5.1-codex-mini`, y así sucesivamente).
- ChatGPT Codex es un producto distinto, alojado en la nube; no es lo que nos importa aquí.

Codex CLI es un par de Claude Code, no un reemplazo. Sus objetivos de diseño son:

- Correr en local, en tu terminal, con un sandbox real.
- Ser scriptable — `codex exec "<prompt>"` es un ciudadano de primera clase.
- Hablar MCP para que las tools externas se reutilicen entre agents.
- Mantenerse opinado: menos perillas, defaults más sensatos.

No es un chatbot de propósito general, y no es un agent remoto totalmente aislado en un sandbox: corre como tu usuario, con los privilegios que tengas. Trátalo en consecuencia durante el trabajo de red team.

## 2. Instalar Codex en Kali 2026

Codex CLI se distribuye como paquete npm. En Kali 2026 el camino recomendado es Node.js 20 LTS o 22 LTS desde el paquete `nodejs`.

### 2.1 Requisitos previos

```bash
# Comprueba que Node.js está presente y es lo bastante reciente:
node --version
# v22.9.0

# Si no está, instálalo desde el repo de Debian:
sudo apt update
sudo apt install -y nodejs npm

# Codex necesita Node 20+. Si el default de Kali es demasiado viejo, usa nvm:
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts
```

### 2.2 Instalar Codex CLI

```bash
# Instalación global:
sudo npm install -g @openai/codex

# Verifica la instalación:
codex --version
# codex-cli 0.42.0

which codex
# /usr/local/bin/codex
```

Si prefieres una instalación por usuario (recomendado, para que `sudo` no forme parte de tu flujo diario):

```bash
mkdir -p ~/.npm-global
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc

npm install -g @openai/codex
codex --version
```

### 2.3 Autenticación

Codex admite dos modos de autenticación:

- Cuenta de ChatGPT (login basado en el navegador; usa tu plan existente).
- API key (variable de entorno `OPENAI_API_KEY`; se cobra por token).

En una estación de trabajo de pentest, el modelo de API key suele ser más simple porque no ata la máquina a una sesión personal de ChatGPT.

```bash
# Login interactivo (abre una pestaña del navegador y pega el token de vuelta):
codex login

# O define una API key en el perfil de tu shell:
export OPENAI_API_KEY="sk-proj-..."
echo 'export OPENAI_API_KEY="sk-proj-..."' >> ~/.bashrc

# Comprobación rápida:
codex exec "imprime hello world en python" --output-last-message /tmp/hello.txt
cat /tmp/hello.txt
```

> **Pentester Insight:** En los portátiles de engagement, guarda tu API key en un archivo legible solo por tu usuario (`chmod 600 ~/.codex/auth.json` o un vault de systemd-creds). Si el SOC de un target llega a incorporar tu estación de trabajo como evidencia, una key sin cifrar en el historial de tu shell es un finding vergonzoso — y peor, puede seguir siendo válida.

### 2.4 Notas específicas de Kali

- El repo Rolling de Kali 2026 ya trae Node 22 LTS; rara vez necesitas nvm.
- El sandbox de Codex CLI usa namespaces de Linux (`unshare`, landlock). Los kernels de Kali los soportan bien, pero si ejecutas Codex dentro de un contenedor Docker tienes que pasar `--cap-add=SYS_ADMIN` o bajar a `--sandbox danger-full-access`.
- La caché por defecto vive en `~/.codex/cache`; mantenla fuera de montajes de red compartidos: hace muchas escrituras pequeñas.

## 3. Comparar Codex CLI con Claude Code

Los dos agents se parecen vistos desde treinta mil pies: ambos son agents de código nativos de la terminal, impulsados por un modelo autorregresivo sobre un bucle de tools. De cerca, difieren en formas que importan para el trabajo de pentest.

### 3.1 Lado a lado

| Dimensión | Claude Code | Codex CLI |
| --- | --- | --- |
| Modelo | Claude Opus/Sonnet/Haiku | Familia GPT-5.1-Codex |
| Lenguaje del CLI | TypeScript (Node) | Binario en Rust |
| Bucle del agent | Gestionado por el SDK, oculto para el usuario | Visualización explícita de turnos y pasos |
| Sandbox | Allow-list del filesystem, bash opt-in | Filesystem con Landlock/seccomp + interruptor de red |
| Cliente MCP | De primera clase, transports ricos | De primera clase, stdio + HTTP |
| Conjunto de tools | Read, Write, Edit, Bash, Grep, Glob, Task, WebFetch | shell, apply_patch, read_file, write_file, mcp |
| Modelo de aprobación | Permisos por tool + overrides de skill | on-request / on-failure / never |
| Modo no interactivo | `claude -p "..."` (print) | `codex exec "..."` (exec) |
| Reanudar | Basado en sesión, acotado al proyecto | `codex --resume <session-id>` |
| Formato de config | JSON (`~/.claude/settings.json`) | TOML (`~/.codex/config.toml`) |
| Streaming por defecto | TUI rica con tarjetas de tools | Log de turnos compacto |

### 3.2 Qué significa esto en la práctica

- Tareas largas y exploratorias (mapear un codebase enorme, planificar un engagement completo): Claude Code, porque la UX es más rica y la gestión de contexto perdona más.
- Razonamiento por lotes (triage de 200 findings, la misma auditoría sobre 50 subdominios): Codex CLI, porque `codex exec` es rápido y retorna de forma limpia.
- Planes estructurados con muchos subtareas: la tool Task de Claude Code más las skills no tiene rival.
- Bucles cerrados con tools de shell: Codex, porque su sandbox por defecto es más fácil de razonar y su modo exec no deriva hacia lo interactivo.
- Donde quieras MCP: cualquiera de los dos, y puedes apuntar ambos al mismo MCP server.

### 3.3 La diferencia de sandbox

El sandbox de Claude Code es una capa de política alrededor de un sistema de permisos: pregunta (o rechaza) según tus listas de allow/deny. El sandbox de Codex CLI es una capa de mecanismo alrededor de los namespaces de Linux: los archivos fuera del workspace literalmente no son visibles para los procesos hijo que lanza, y la red está apagada por defecto salvo que el modo diga lo contrario.

Ambos enfoques son válidos. El de Claude Code es más flexible; el de Codex CLI es más difícil de saltarse, y por eso mucha gente de red team recurre a Codex cuando quiere ejecutar un script sospechoso contra un payload.

## 4. Configuración — `~/.codex/config.toml`

Codex CLI lee la configuración desde `~/.codex/config.toml`. Se crea en la primera ejecución; el schema es estable pero pequeño.

### 4.1 Una config que funciona de verdad

```toml
# ~/.codex/config.toml

# Modelo por defecto para todas las invocaciones de `codex`.
model = "gpt-5.1-codex"

# Esfuerzo de razonamiento. Uno de: minimal, low, medium, high.
model_reasoning_effort = "medium"

# Sandbox por defecto para el REPL interactivo.
sandbox_mode = "workspace-write"

# Política de aprobación por defecto para el REPL interactivo.
approval_policy = "on-request"

# Dónde viven los transcripts y los archivos de rollout.
history_path = "~/.codex/history"

# Compacta el contexto automáticamente cuando se acerca a la ventana del modelo.
auto_compact = true

# Opcional: fija un modelo distinto para `codex exec`.
[profiles.exec]
model = "gpt-5.1-codex-mini"
model_reasoning_effort = "low"
sandbox_mode = "read-only"
approval_policy = "never"

# Opcional: un perfil pesado para trabajo de revisión en profundidad.
[profiles.review]
model = "gpt-5.1-codex"
model_reasoning_effort = "high"
sandbox_mode = "workspace-write"
approval_policy = "on-request"

# MCP servers que Codex debe adjuntar al arrancar.
[mcp_servers.nuclei]
command = "/opt/mcp-servers/nuclei-mcp/target/release/nuclei-mcp"
args = []
env = { NUCLEI_TEMPLATES = "/opt/nuclei-templates" }

[mcp_servers.burp]
command = "node"
args = ["/opt/mcp-servers/burp-mcp/dist/index.js"]

# Personalización del shell.
[shell]
# El wrapper que Codex usa para lanzar comandos hijo dentro del sandbox.
# El default está bien; sobreescríbelo solo si tienes necesidades raras de PATH.
default_timeout_ms = 120000
```

### 4.2 Elegir un perfil

Los perfiles se eligen con `--profile`:

```bash
codex --profile review
codex exec --profile exec "resume el diff entre main y feature/x"
```

Los perfiles se componen encima de los defaults de nivel superior, así que un perfil solo necesita nombrar lo que cambia.

### 4.3 Overrides de entorno

Cualquier clave de config se puede sobreescribir por invocación:

```bash
codex -c model="gpt-5.1-codex-mini" -c sandbox_mode="read-only" exec "..."
```

La forma `-c key=value` toma valores TOML, así que entrecomilla los strings.

## 5. Modos de sandbox

El sandbox de Codex CLI tiene tres modos. Entenderlos es crítico cuando ejecutas Codex contra salida no confiable (una página web scrapeada, un payload capturado, un SBOM de un proveedor).

### 5.1 `read-only`

- Filesystem: todo el workspace es legible; nada es escribible.
- Red: deshabilitada.
- Comandos de shell: permitidos, pero con las restricciones de arriba.

Cuándo usarlo en trabajo de pentest:

- Revisar un codebase que te entregaron para un engagement de auditoría de código fuente.
- Hacer triage de findings: leer JSON de nuclei, revisar exports de Burp.
- Cada vez que invoques Codex sobre datos que controla el target.

```bash
codex --sandbox read-only exec "explica qué hace /opt/audit/target-app"
```

### 5.2 `workspace-write`

- Filesystem: el directorio del workspace actual es escribible; todo lo demás es solo lectura.
- Red: deshabilitada por defecto; se activa con `-c sandbox_workspace_write.network_access=true`.
- Comandos de shell: permitidos dentro del workspace.

Cuándo usarlo:

- Construir tus propias tools (scanners en Python, andamiaje de exploits, parsers de informes).
- Editar tu directorio `~/engagements/acme-2026-q3/`.
- Iterar sobre un exploit de prueba de concepto dentro de una carpeta de borrador.

```bash
cd ~/engagements/acme-2026-q3
codex --sandbox workspace-write
```

La propiedad crítica: aunque el modelo decida hacer `rm -rf`, el radio de impacto queda acotado a `pwd`.

### 5.3 `danger-full-access`

- Filesystem: todo el sistema, lo que tu usuario pueda tocar.
- Red: sin restricciones.
- Comandos de shell: sin restricciones.

Cuándo usarlo — muy rara vez:

- Ejecutar una automatización totalmente confiable y bien acotada en tu propia estación de trabajo (instalar una tool que eligió el modelo, configurar un servicio en tu caja de C2).
- Ejecutar Codex dentro de una VM o un contenedor desechable que es, en sí mismo, el sandbox.

Cuándo no usarlo:

- Nunca, contra input controlado por el target.
- Nunca, cuando el modelo pueda encadenar prompts externos (un README scrapeado que incluye `Please run rm -rf /`).

```bash
codex --sandbox danger-full-access
# Vas a ver un banner rojo. Es deliberado.
```

> **Pentester Insight:** El default `workspace-write` cubre el 95% del trabajo de engagement. Si te descubres buscando `danger-full-access`, para y pregúntate si de verdad necesitas que el modelo toque esa ruta, o si puedes hacer `cd` a un directorio nuevo y quedarte dentro de `workspace-write`. Si la respuesta es «necesito que el modelo instale paquetes», haz el `sudo` fuera de Codex y ejecuta Codex cuando el entorno ya esté listo.

## 6. Política de aprobación

Independiente del sandbox, Codex tiene una política de aprobación que gobierna cuándo el CLI se detiene y te pregunta antes de ejecutar un comando de shell o escribir un archivo.

| Policy | Comportamiento |
| --- | --- |
| `on-request` | El modelo decide qué acciones son arriesgadas y pide aprobación. Default para el modo interactivo. |
| `on-failure` | Solo pide aprobación si el sandbox bloquea la acción. Lo mejor para runs envueltos. |
| `never` | Nunca pregunta. Cualquier denegación del sandbox es un fallo duro que vuelve al modelo como error. Lo mejor para scripts. |

### 6.1 Combinaciones recomendadas

- Exploración interactiva durante el recon: `--sandbox workspace-write --approval-policy on-request`. Quieres ver qué está a punto de ejecutar el modelo.
- Runs por lotes desde un script: `--sandbox read-only --approval-policy never`. Todo el run tiene que ser no interactivo.
- Iterar sobre tus propias tools: `--sandbox workspace-write --approval-policy on-failure`. El agent avanza rápido, pero una denegación del sandbox hace una pausa para que decidas.

### 6.2 Recomendado para trabajo de red team

En runs de red team donde el modelo va a hacer algo que el SOC podría ver (tocar un servicio real, enviar una sonda), usa siempre `on-request`. El coste de 30 segundos de confirmar una acción vale la pena para evitar una petición descontrolada que dispare un WAF.

```bash
codex \
  --sandbox workspace-write \
  --approval-policy on-request \
  --profile review
```

## 7. Estilos de invocación

Codex tiene tres modos primarios de invocación: interactivo, exec y resume.

### 7.1 Interactivo

```bash
codex
```

Abre la TUI. Úsalo para exploración, planificación iterativa, cualquier cosa que se beneficie de una conversación. Ctrl+C cancela el turno actual; `/quit` sale.

Comandos útiles dentro de la sesión:

- `/model gpt-5.1-codex` — cambia de modelo a mitad de sesión.
- `/sandbox read-only` — aprieta el sandbox a mitad de sesión (aflojarlo exige un reinicio).
- `/mcp` — lista los MCP servers adjuntos y sus tools.
- `/compact` — fuerza una compactación de resumen si el contexto se está poniendo pesado.

### 7.2 `codex exec`

```bash
codex exec "tu prompt aquí"
```

Ejecuta un turno (o un número acotado de turnos) de forma no interactiva, envía la actividad de las tools a stderr, imprime el mensaje final del asistente en stdout y sale. Este es el modo que scriptas.

Flags que importan para la automatización:

- `--output-last-message <path>` — escribe solo el mensaje final del asistente en un archivo.
- `--json` — emite eventos JSON estructurados para cada paso.
- `--max-turns <n>` — tope duro de turnos de uso de tools.
- `--profile <name>` — elige un perfil de config.
- `-C <dir>` — hace `cd` a un workspace antes de ejecutar.
- `--include-plan` — obliga al modelo a emitir un bloque de plan.

Ejemplo:

```bash
codex exec \
  --profile exec \
  --json \
  --max-turns 6 \
  --output-last-message /tmp/last.txt \
  "lee ./target.txt y resume los tres findings principales como viñetas"
```

### 7.3 `codex --resume`

Cada sesión de Codex se guarda como un archivo de rollout bajo `~/.codex/history/`. Puedes reanudar por id de sesión:

```bash
codex resume --last
codex resume 018f9c7c-2c3d-7f0a-8e2b-71e2be7b32aa
```

En trabajo por lotes rara vez reanudas: simplemente empiezas de cero. Pero cuando estabas en una sesión interactiva larga y te desconectaste, `codex resume --last` te devuelve exactamente donde estabas.

## 8. Leer y escribir archivos en Codex — su conjunto de tools

Codex CLI trae un conjunto de tools pequeño y opinado. Cada tool respeta el sandbox y la política de aprobación actuales.

### 8.1 Tools integradas

| Tool | Propósito | Notas |
| --- | --- | --- |
| `shell` | Ejecutar un comando de shell | Acotado en tiempo; stdout/stderr capturados se devuelven al modelo |
| `read_file` | Leer un archivo por ruta | Respeta los límites de read-only y del workspace |
| `write_file` | Sobrescribir o crear un archivo | Se rechaza fuera del workspace salvo con `danger-full-access` |
| `apply_patch` | Aplicar un diff unificado | Preferible a `write_file` para ediciones; más seguro |
| `list_dir` | Listar un directorio | Barato; se usa para orientarse |
| `search` | Búsqueda tipo ripgrep por el workspace | Más rápido que lanzar `rg` vía shell |
| `mcp` | Llamar a una tool provista por MCP | Con namespace por MCP server |

### 8.2 Cómo decide Codex qué tool usar

El modelo está entrenado para preferir `apply_patch` en las ediciones frente a escrituras del archivo completo; para usar `search` antes de caer en `rg` por shell; y para usar `read_file` en lugar de `cat` por shell. Son preferencias fuertes, no reglas duras: puedes empujarlas con prompts:

```text
Usa siempre apply_patch para las ediciones. No uses shell para modificar archivos.
```

Incluye esa línea en el `CODEX.md` de un proyecto (el equivalente en Codex de `CLAUDE.md`) y se precargará en cada sesión iniciada en ese directorio.

### 8.3 La tool `shell` en detalle

`shell` es la tool en la que más tiempo piensas, porque es la que el sandbox restringe con más fuerza.

- Los comandos corren bajo `bash -lc`.
- El directorio de trabajo es la raíz del workspace por defecto; el modelo puede hacer `cd`.
- El comando se ejecuta dentro de un user namespace de Linux con reglas landlock que coinciden con el modo de sandbox.
- Las syscalls de red se descartan en `read-only` y en `workspace-write` (salvo override).
- La salida está limitada a unos 200 KB por invocación para proteger el contexto.

Si un comando supera el tope de salida, Codex trunca y se lo dice al modelo. Si un comando supera el timeout por defecto de 120 segundos, Codex lo mata. Ambos límites se configuran por invocación con `-c shell.default_timeout_ms=...`.

### 8.4 `CODEX.md`

Igual que Claude Code lee `CLAUDE.md`, Codex lee `CODEX.md` (a nivel de proyecto) y `~/.codex/CODEX.md` (a nivel de usuario). Un buen `CODEX.md` de proyecto de pentest se ve así:

```markdown
# Instrucciones de Codex para el engagement acme-2026-q3

## Scope
- In-scope: *.acme.corp, *.acme-labs.com
- Out-of-scope: TODO lo demás. No toques dominios de terceros.

## Convenciones
- Usa `apply_patch` para las ediciones.
- Guarda los findings bajo `./findings/<host>/<yyyy-mm-dd>.md`.
- Todos los comandos de shell tienen que ser idempotentes; no envíes sondas que alteren el estado.

## Tools
- Las templates de nuclei viven en /opt/nuclei-templates
- Archivo de estado de burp: ./burp/state.burp
- NO ejecutes tools activas sin el script de guarda `--target-in-scope`
```

## 9. Construir un bucle de automatización

Ahora viene la parte divertida. Aquí tienes un wrapper en bash que ejecuta `codex exec` sobre cada uno de 200 targets y recoge salida JSON.

### 9.1 Requisitos

- Entrada: `targets.txt`, un hostname por línea.
- Salida: `results.jsonl`, un objeto JSON por target con los campos `{target, verdict, evidence, model, ts}`.
- Restricciones: no tocar nunca un target fuera del archivo; timeout duro de 60 segundos por invocación; paralelismo configurable.

### 9.2 El wrapper: `codex-batch.sh`

```bash
#!/usr/bin/env bash
# codex-batch.sh — ejecuta codex exec contra una lista de targets en paralelo.
#
# Uso: ./codex-batch.sh targets.txt results.jsonl [PARALELISMO]

set -euo pipefail

TARGETS="${1:?falta el archivo de targets}"
OUT="${2:?falta el archivo de salida}"
PAR="${3:-4}"

: > "$OUT"

# El prompt que enviamos para cada target. Mantenlo estable para que el
# schema JSON del modelo no derive entre invocaciones.
read -r -d '' PROMPT <<'PROMPT_EOF' || true
Eres un asistente de triage. Dado un hostname, ejecuta recon no invasivo
(DNS, whois, HTTP HEAD a /) y decide si parece
in-scope para un engagement contra acme.corp.

Responde con EXACTAMENTE un objeto JSON en stdout, nada más:
{"target": "<hostname>", "verdict": "in-scope"|"out-of-scope"|"unknown",
 "evidence": "<justificación en una oración>"}
PROMPT_EOF

# Función invocada por target, en paralelo.
run_one() {
    local host="$1"
    local tmp
    tmp="$(mktemp)"

    # Acota todo el run a 60s.
    if timeout 60 codex exec \
            --profile exec \
            --json \
            --sandbox workspace-write \
            --approval-policy never \
            --max-turns 8 \
            --output-last-message "$tmp" \
            "$(printf '%s\n\nTarget: %s' "$PROMPT" "$host")" \
            >/dev/null 2>&1; then

        local last
        last="$(cat "$tmp")"

        # Exige el schema JSON. Si el modelo se desvió, marca unknown.
        if jq -e '.target and .verdict and .evidence' \
                <<<"$last" >/dev/null 2>&1; then
            jq -c \
                --arg model "gpt-5.1-codex-mini" \
                --arg ts "$(date -Is)" \
                '. + {model: $model, ts: $ts}' \
                <<<"$last" >> "$OUT"
        else
            jq -cn \
                --arg t "$host" \
                --arg ev "codex devolvió contenido que no es JSON: $last" \
                --arg ts "$(date -Is)" \
                '{target:$t,verdict:"unknown",evidence:$ev,ts:$ts}' >> "$OUT"
        fi
    else
        jq -cn \
            --arg t "$host" \
            --arg ts "$(date -Is)" \
            '{target:$t,verdict:"unknown",evidence:"timeout o error de codex",ts:$ts}' >> "$OUT"
    fi

    rm -f "$tmp"
}

export -f run_one
export PROMPT

# Abanico en paralelo vía xargs.
< "$TARGETS" xargs -I{} -P "$PAR" bash -c 'run_one "$@"' _ {}

echo "listo. se escribieron $(wc -l < "$OUT") líneas en $OUT"
```

### 9.3 Ejecutarlo

```bash
chmod +x codex-batch.sh

# 200 targets, 8 en paralelo:
./codex-batch.sh targets.txt results.jsonl 8

# Inspeccionar:
jq -s 'group_by(.verdict) | map({verdict: .[0].verdict, n: length})' results.jsonl
# [
#   {"verdict":"in-scope","n":142},
#   {"verdict":"out-of-scope","n":51},
#   {"verdict":"unknown","n":7}
# ]
```

### 9.4 Notas sobre robustez

- Envuelve siempre `codex exec` en `timeout`. El modelo puede entrar en bucle, y un run atascado desperdicia dinero y tiempo de calendario.
- Valida siempre el JSON con `jq -e`. Codex es mucho más disciplinado que los modelos de la era 2023, pero todavía cuela de vez en cuando un prefijo parlanchín. El wrapper exige el schema.
- `--approval-policy never` solo es seguro porque `--sandbox workspace-write` acota las escrituras a `pwd`. No quites uno sin el otro.
- `xargs -P` basta hasta unos 16 workers por máquina. Más allá vas a chocar con los rate limits de OpenAI y vas a querer una cola de verdad.

## 10. MCP servers en Codex

La mayor ventaja de ejecutar a la vez Claude Code y Codex CLI es que pueden compartir MCP servers. Escribes un MCP de nuclei una vez y ambos agents pueden llamarlo.

### 10.1 Configuración

Los MCP servers van bajo `[mcp_servers.<name>]` en `~/.codex/config.toml`:

```toml
[mcp_servers.nuclei]
command = "/opt/mcp-servers/nuclei-mcp/target/release/nuclei-mcp"
args = ["--templates", "/opt/nuclei-templates"]
env = { RUST_LOG = "info" }

[mcp_servers.pentest-toolkit]
command = "node"
args = ["/opt/mcp-servers/pentest-toolkit/dist/index.js"]

# El transport HTTP también está soportado para servers remotos:
[mcp_servers.corp-scanner]
url = "https://mcp.internal.redteamleaders.example/scanner"
headers = { Authorization = "Bearer $MCP_CORP_TOKEN" }
```

### 10.2 Verificar que el MCP quedó adjunto

```text
codex
> /mcp
# nuclei          9 tools
# pentest-toolkit 14 tools
# corp-scanner    3 tools

> /mcp nuclei
# nuclei:run(target: string, tags?: string[])
# nuclei:validate_template(path: string)
# ...
```

### 10.3 Usar el mismo MCP desde Claude Code

En `~/.claude/settings.json`:

```json
{
  "mcp": {
    "servers": {
      "nuclei": {
        "command": "/opt/mcp-servers/nuclei-mcp/target/release/nuclei-mcp",
        "args": ["--templates", "/opt/nuclei-templates"]
      }
    }
  }
}
```

El mismo binario, las mismas tools, el mismo comportamiento. La ventaja de centralizar tus MCP de pentest (nuclei, ffuf, burp, katana, scripts de recon a medida) es enorme: construyes la tool una vez y eleva tu conjunto efectivo de tools en ambos agents.

> **Pentester Insight:** Durante engagements largos, el propio MCP server se convierte en un log de auditoría canónico. Cada llamada a una tool queda registrada con sus argumentos; puedes volver en el momento de escribir el informe y reconstruir exactamente qué sondas se dispararon contra qué targets. Esto vale más que el oro cuando el cliente pregunta si disparaste una alarma a las 03:47 del día tres.

## 11. Comparar estilos de prompt

Claude y Codex siguen bien las instrucciones, pero responden a formas de prompt distintas.

### 11.1 Lo que prefiere Claude

Los modelos Claude — en especial Sonnet y Opus — responden excepcionalmente bien a:

- Tags XML explícitos para input estructurado (`<scope>...</scope>`, `<findings>...</findings>`).
- Planes numerados que puedan ir tachando.
- Few-shot con varios ejemplos, incluidos ejemplos negativos.

Ejemplo:

```xml
<task>
Haz triage del finding de abajo. Decide si es una vulnerabilidad real
o un falso positivo.
</task>

<finding>
CVE-2024-12345 en https://acme.corp/api/v2/users
</finding>

<criteria>
1. ¿El endpoint es realmente alcanzable?
2. ¿La versión vulnerable corre de verdad ahí?
3. ¿Hace falta autenticación?
</criteria>

<output_schema>
{"real": bool, "confidence": "low"|"medium"|"high", "notes": "..."}
</output_schema>
```

### 11.2 Lo que prefiere Codex

Los modelos Codex GPT-5.1 responden mejor a:

- Instrucciones tersas e imperativas. «Haz X. Luego Y. Devuelve Z.»
- Schemas JSON expresados como JSON, no como prosa.
- Menos marcos de roleplay. «Eres un asistente de triage» está bien; un párrafo de persona es ruido.
- Preferencias explícitas de tools («usa apply_patch, no shell»).

Ejemplo:

```text
Haz triage de este finding.

Input:
{"cve":"CVE-2024-12345","url":"https://acme.corp/api/v2/users"}

Pasos:
1. Haz HEAD a la URL.
2. Lee la cabecera server.
3. Compara contra las versiones vulnerables conocidas.

Output (schema exacto, nada más):
{"real": true|false, "confidence": "low|medium|high", "notes": "..."}
```

### 11.3 Una regla práctica

- Copia un prompt de Claude en Codex y va a funcionar, pero estás quemando tokens.
- Copia un prompt de Codex en Claude y puede funcionar, pero Claude va a seguir haciendo preguntas de aclaración.

Mantén dos bibliotecas de prompts. No es mucho trabajo: la mayoría de los prompts solo difieren en el marco.

## 12. Usar Codex para sprints de construcción de tools

Donde Codex brilla durante un engagement es en los sprints de construcción de tools: los momentos en los que necesitas un scanner, un parser o un filtro para el entorno del cliente, y lo necesitas en la próxima hora, no al día siguiente.

### 12.1 El patrón

1. `cd ~/engagements/acme/tools/`
2. `codex --sandbox workspace-write --approval-policy on-request`
3. Pide un esqueleto de módulo Python con el CLI exacto que quieres.
4. Itera sobre él, ejecutando la tool contra una muestra después de cada cambio.
5. Cuando funcione, pide tests y después pide un README.
6. Haz commit.

### 12.2 Un ejemplo real

Prompt:

```text
Constrúyeme un CLI de Python 3.11 llamado `acme-cookie-audit` que:

- Reciba una URL y un nombre de cookie.
- Obtenga la URL con httpx y siga los redirects.
- Extraiga la cookie nombrada de la cabecera Set-Cookie.
- La decodifique si parece base64 o hex; la parsee como JWT si parece uno.
- Imprima un informe estructurado en stdout como JSON.

Usa `click`. Añade tests unitarios con pytest y un test de integración que pegue a httpbin.org.
No uses requests. No añadas async salvo que haga falta.
```

Codex va a producir un paquete `acme_cookie_audit/` funcional en unos 30 segundos, va a ejecutar los tests y va a iterar hasta que pasen. Cuando se detenga tienes un CLI real y testeado para dejar en la caja de herramientas del engagement.

### 12.3 Cuándo no usar Codex para esto

- Cualquier cosa que tenga que ser memory-safe y auditada a mano (loaders de shellcode a medida, primitivas de exploit). Codex escribe el andamiaje; tú verificas las tripas a mano.
- Cualquier cosa con requisitos duros de rendimiento. El modelo tiende a elegir lo legible, no lo más rápido.

## 13. Gestión de coste

Codex CLI no es gratis. Cada turno es una llamada al modelo, y los bucles largos de agent pueden masticar tokens deprisa. Algunos hábitos que mantienen la factura en su sitio durante un engagement:

### 13.1 Elige el modelo correcto

- `gpt-5.1-codex` — caballo de batalla de razonamiento; úsalo para revisión, planificación y construcción compleja de tools.
- `gpt-5.1-codex-mini` — rápido y barato; úsalo para triage por lotes, clasificación y recon no invasivo.
- `gpt-5.1-codex-nano` (donde esté disponible en tu región) — el más barato; úsalo para tareas solo de forma (reformatear JSON, un sí/no de un solo disparo).

Fija el default en `~/.codex/config.toml` y sobreescríbelo por perfil.

### 13.2 Acota los turnos

`--max-turns 6` cuesta una fracción de `--max-turns 30`. Para jobs por lotes, capa los turnos al valor más pequeño que la tarea tolere.

### 13.3 Usa prompt caching

Codex CLI soporta el prompt caching de OpenAI para las partes estáticas de un system prompt. Pon tu preámbulo largo y estable al principio de cada prompt y el modelo lo va a facturar a la tarifa de caché en las llamadas repetidas dentro de la ventana de caché.

Para el wrapper por lotes de la sección 9, esto significa que la variable compartida `PROMPT` está (en su mayor parte) cacheada a lo largo de las 200 invocaciones. Sigues pagando la cola específica del target, pero el grueso del prompt es esencialmente gratis después de la primera llamada.

### 13.4 Vigila el medidor

```bash
codex usage --today
codex usage --this-month --profile exec
```

El subcomando `usage` informa conteos de tokens agregados, sacados de tus logs locales de rollout. Práctico para la contabilidad del engagement.

### 13.5 Una disciplina de presupuesto

Dale a cada engagement un presupuesto de tokens por adelantado. Si gastas 50 USD de crédito de API en triage, es dinero bien gastado. Si gastas 500 USD, algo ha ido mal: probablemente un bucle sin cota o un prompt parlanchín. Pon una alerta en tu dashboard de OpenAI y trata cruzarla como un evento de parar la línea.

## 14. Ejemplo completo que funciona — `codex-triage.sh`

Este es el pago. Un wrapper que toma una salida JSONL de nuclei, pasa cada finding por Codex CLI para un triage con el LLM y escribe de vuelta un JSONL decorado.

### 14.1 Qué hace

- Lee `nuclei.jsonl` (un finding por línea, el schema por defecto de `nuclei -jsonl`).
- Para cada finding, le pide a Codex que decida: `real`, `likely-fp` o `needs-manual`.
- Escribe `nuclei-triaged.jsonl` con el objeto original más un campo `triage`.
- Se salta los findings que el modelo ya ha visto (dedup por el `template-id` de nuclei más `matched-at`).

### 14.2 El script

```bash
#!/usr/bin/env bash
# codex-triage.sh — triage con LLM de un archivo JSONL de nuclei.
#
# Uso: ./codex-triage.sh nuclei.jsonl nuclei-triaged.jsonl

set -euo pipefail

IN="${1:?jsonl de entrada}"
OUT="${2:?jsonl de salida}"
PAR="${PAR:-6}"
MODEL="${MODEL:-gpt-5.1-codex-mini}"

: > "$OUT"

# Directorio de caché para el dedup. Memoización barata en disco, con clave
# un hash estable del finding.
CACHE="${TRIAGE_CACHE:-$HOME/.cache/codex-triage}"
mkdir -p "$CACHE"

read -r -d '' SYSTEM_PROMPT <<'SYS_EOF' || true
Eres un asistente de triage de findings de nuclei.

Vas a recibir un finding de nuclei como JSON. Decide si es:
- "real": una vulnerabilidad real que un humano debería confirmar e informar
- "likely-fp": casi con seguridad un falso positivo
- "needs-manual": ambiguo; requiere verificación manual

Reglas:
- NO sondees el target. Razona solo a partir del JSON.
- Ten en cuenta la severidad de la template, el tipo de matcher, los valores extraídos y las templates conocidas por ruidosas.
- Las templates que terminan en "-detect" o "-fingerprint" sin un CVE obvio suelen ser informativas.
- Si curl-response está presente y muestra un 200 con contenido que coincide, trátalo como evidencia.

Responde con EXACTAMENTE un objeto JSON, nada más:
{
  "triage": "real"|"likely-fp"|"needs-manual",
  "confidence": "low"|"medium"|"high",
  "rationale": "<una o dos oraciones>"
}
SYS_EOF

triage_one() {
    local line="$1"
    local key hash cached

    # Clave estable del finding.
    key="$(jq -rc '[.["template-id"], .["matched-at"]] | @tsv' <<<"$line")"
    hash="$(printf '%s' "$key" | sha256sum | cut -c1-32)"
    cached="$CACHE/$hash.json"

    local triage
    if [[ -f "$cached" ]]; then
        triage="$(cat "$cached")"
    else
        local tmp
        tmp="$(mktemp)"

        if timeout 45 codex exec \
                -c model="\"$MODEL\"" \
                --json \
                --sandbox read-only \
                --approval-policy never \
                --max-turns 3 \
                --output-last-message "$tmp" \
                "$(printf '%s\n\nFinding:\n%s' "$SYSTEM_PROMPT" "$line")" \
                >/dev/null 2>&1; then
            triage="$(cat "$tmp")"
        else
            triage='{"triage":"needs-manual","confidence":"low","rationale":"error de codex o timeout"}'
        fi

        rm -f "$tmp"

        # Exige el schema; si falla, degrada.
        if ! jq -e '.triage and .confidence' <<<"$triage" >/dev/null 2>&1; then
            triage='{"triage":"needs-manual","confidence":"low","rationale":"respuesta que no es JSON"}'
        fi

        echo "$triage" > "$cached"
    fi

    # Fusiona el objeto de triage bajo el finding original.
    jq -c \
        --argjson t "$triage" \
        --arg model "$MODEL" \
        --arg ts "$(date -Is)" \
        '. + {triage: ($t + {model:$model, ts:$ts})}' \
        <<<"$line"
}

export -f triage_one
export SYSTEM_PROMPT MODEL CACHE

# Abanico. La salida se añade de forma atómica porque jq -c imprime una línea.
< "$IN" xargs -d '\n' -I{} -P "$PAR" bash -c 'triage_one "$@"' _ {} >> "$OUT"

# Resumen.
echo "se escribieron $(wc -l < "$OUT") findings con triage."
jq -c 'group_by(.triage.triage) | map({verdict:.[0].triage.triage, n:length})' \
    <(jq -s '.' "$OUT")
```

### 14.3 Ejecutarlo contra una salida real de nuclei

```bash
# Ejecuta nuclei contra un archivo de scope:
nuclei -l scope.txt -jsonl -o nuclei.jsonl

# Haz triage de todo:
PAR=8 MODEL=gpt-5.1-codex-mini ./codex-triage.sh nuclei.jsonl triaged.jsonl

# Saca los findings "real" para trabajo manual:
jq -c 'select(.triage.triage == "real")' triaged.jsonl > real.jsonl

# Saca los "likely-fp" de confianza medium/high para una segunda mirada:
jq -c 'select(.triage.triage=="likely-fp" and .triage.confidence!="high")' \
    triaged.jsonl > review-fp.jsonl
```

En un run típico de nuclei de 800 findings, esto reduce la carga de triage manual a unos 60 findings: el resto son o falsos positivos de alta confianza que puedes descartar, o fingerprints informativos que el modelo ignoró correctamente.

### 14.4 Qué te compra esto

- Tiempo. 800 findings a 30 segundos de atención humana cada uno son casi siete horas. Este script lo hace en unos seis minutos.
- Consistencia. Los mismos criterios aplicados a cada finding. El triage humano deriva a medida que avanza el día.
- Reproducibilidad. La caché hace que ejecutar el script dos veces dé las mismas respuestas. A los auditores les encanta esto.

### 14.5 Modos de fallo conocidos

- Las templates que el modelo no ha visto (templates de nuclei internas, hechas en casa) se triagan como `"needs-manual"` más a menudo. Es el comportamiento correcto, pero sesga la carga de casos.
- Los campos de valor extraído muy largos pueden empujar un finding por encima del tamaño máximo del prompt; el wrapper hoy no trunca. Añade un paso de `jq` para capar `extracted-results` si esto te muerde.
- No ejecutes esto contra findings de una fuente no confiable sin inspeccionarlos antes. Un «finding» malicioso podría contener un payload de inyección de prompt dirigido a tu agent de Codex.

## 15. Combinar Claude Code y Codex en el mismo engagement

Los mejores red teams usan ambos. Aquí hay un reparto del trabajo que funciona.

### 15.1 Claude Code se queda con

- El arranque y la planificación. Leer el SOW, mapear el scope, generar el `CLAUDE.md` del engagement, redactar el modelo de amenazas inicial.
- Las revisiones de codebase. Las auditorías de código fuente se benefician del contexto largo de Claude y de su escritura cuidadosa.
- El desarrollo de exploits. El razonamiento paso a paso y el flujo impulsado por skills no tienen rival cuando iteras sobre un payload.
- La redacción de informes. La prosa de Claude encaja mejor con los entregables al cliente.

### 15.2 Codex se queda con

- El triage por lotes. Salidas de nuclei, dalfox, ffuf y scanners a medida.
- Los sprints de construcción de tools. CLIs de Python o Rust de menos de una hora que solo van a vivir durante este engagement.
- Los chequeos estilo CI. Hooks de pre-commit en el repo del engagement, comprobaciones de cordura antes de publicar un finding.
- Cualquier sitio donde el bucle sea `for x in xs; do llm(x); done`.

### 15.3 Un día en la vida

```text
09:00 — Claude Code, planifica el día, revisa los findings de ayer.
10:00 — arranca un run de nuclei en segundo plano.
10:30 — Codex hace triage de los resultados a medida que llegan (`codex-triage.sh --watch`).
12:00 — Claude Code, inmersión profunda en los dos findings "real" que marcó Codex.
14:00 — Codex construye un fuzzer por endpoint para uno de ellos.
15:00 — Claude Code, redacta el finding confirmado.
16:30 — Codex audita por lotes el scope restante en busca de la misma clase de bug.
17:00 — Claude Code, enrolla el día en la actualización de estado de cara al cliente.
```

Ningún agent es un reemplazo estricto del otro. Trátalos como dos colegas con especialidades distintas.

### 15.4 Compartir estado

El estado compartido más simple es el filesystem. Ambos agents pueden leer el mismo directorio de engagement, ambos escriben findings en la misma carpeta `findings/`, ambos consultan el mismo `CLAUDE.md` / `CODEX.md`. Más allá de eso:

- MCP servers compartidos (`nuclei-mcp`, `ffuf-mcp`, `recon-mcp` a medida): descritos en la sección 10.
- Bibliotecas de prompts compartidas en una carpeta `prompts/` con variantes `.claude.md` y `.codex.md`.
- Vault compartido para secretos (systemd-creds, Bitwarden CLI, o un `.env` cifrado con gpg, simple).

No intentes que los dos agents se hablen directamente. Cada intento de construir un chat «agent a agent» a este nivel de madurez termina en bucles confusos. Mantén a los humanos en el medio.

## 16. Ejercicios

### 16.1 Ejercicios guiados

**Ejercicio 1 — Instalar y establecer la línea base**

- Instala Codex CLI en tu VM de Kali usando el enfoque global con sudo o el enfoque por usuario.
- Autentícate con una API key (crea una key nueva solo para este curso; la vas a borrar después).
- Crea `~/.codex/config.toml` con un perfil por defecto que use `gpt-5.1-codex-mini` y `sandbox_mode = "workspace-write"`.
- Ejecuta `codex exec "escribe un hello world en rust; luego compílalo con rustc; luego ejecútalo"` y confirma que el binario produce la salida esperada.

**Ejercicio 2 — Comparación de sandbox**

- Arranca Codex con `--sandbox read-only`.
- Pídele que haga `echo hello > /tmp/hi`. Observa el rechazo.
- Reinicia con `--sandbox workspace-write` dentro de `/tmp/codex-exercise/`.
- Repite. Observa el éxito — pero solo dentro del workspace.
- Desde dentro de esa sesión, pídele que escriba `/etc/hosts`. Observa el rechazo.
- Escribe una nota de un párrafo en `~/notes/codex-sandbox.md` que describa qué modo de sandbox usarías para cada uno de estos casos: revisar un codebase de un cliente, ejecutar un POC, editar tus notas de engagement.

**Ejercicio 3 — El wrapper por lotes**

- Copia el script `codex-batch.sh` de la sección 9 a `~/tools/codex-batch.sh`.
- Rellena `targets.txt` con 20 líneas de tu entorno de laboratorio (nunca hostnames reales de internet sin autorización).
- Ejecuta el lote con `PAR=4`.
- Inspecciona `results.jsonl` y confirma que cada línea valida contra el schema `{target,verdict,evidence,model,ts}`.
- Modifica el prompt para que también devuelva un campo `port_hint` (por ejemplo, una conjetura del puerto del servicio principal). Vuelve a ejecutarlo y confirma que el cambio de schema se propaga.

### 16.2 Ejercicios independientes

**Ejercicio 4 — Tu propio prompt de triage**

Adapta `codex-triage.sh` para hacer triage de la salida de otra tool a tu elección: dalfox, ffuf o tu propio scanner. Diseña un schema, escribe un prompt y ejecútalo contra una muestra. Mide la reducción de falsos positivos frente al triage manual. Escribe una nota interna corta (unas 200 palabras) que describa qué funcionó y qué no.

**Ejercicio 5 — MCP compartido**

Construye o instala un MCP server (uno simple: un server «http-head» que exponga una sola tool). Conéctalo tanto en `~/.codex/config.toml` como en `~/.claude/settings.json`. Confirma que ambos agents pueden llamar a la tool y que ambos ven el mismo comportamiento. Escribe un `README.md` que describa el contrato de la tool.

**Ejercicio 6 — Estudio del compromiso entre modelos**

Toma la misma tarea no trivial (por ejemplo, «escribe un parser en Python para el siguiente formato de log a medida») y ejecútala tres veces con perfiles de Codex distintos: uno con `gpt-5.1-codex` a razonamiento high, uno con `gpt-5.1-codex` a razonamiento low y uno con `gpt-5.1-codex-mini`. Registra el tiempo de reloj, el conteo de tokens (desde `codex usage`) y si el código producido pasa tus tests. Presenta los resultados como una tabla pequeña con un párrafo de recomendación para la política de tu engagement.

## 17. Resumen y lección siguiente

Codex CLI es la otra mitad de la caja de herramientas de IA del pentester moderno. Complementa a Claude Code por ser rápido, no interactivo, cómodo para lotes y lo bastante barato como para correrlo sobre cientos de ítems. Su sandbox de namespaces de Linux es más fácil de confiar cuando el input lo controla el target, y su modo exec lo hace un encaje natural para los flujos shell-first en los que ya vive la gente de red team.

Las ideas clave:

- Instala con `npm i -g @openai/codex`; verifica con `codex --version`.
- Configura mediante `~/.codex/config.toml`; los perfiles te dejan tener varios setups listos para usar.
- Tres modos de sandbox (`read-only`, `workspace-write`, `danger-full-access`) y tres políticas de aprobación (`on-request`, `on-failure`, `never`). Hazlos coincidir con la tarea.
- `codex exec` es el caballo de batalla para scripting; envuélvelo en `timeout` y `jq -e` y se convierte en una primitiva de lote confiable.
- Comparte MCP servers entre Claude Code y Codex. Construye la tool una vez y úsala en ambos agents.
- Estilos de prompt distintos para modelos distintos; mantén dos bibliotecas.
- Usa Claude Code para explorar y Codex para lotes. No los mezcles dentro de un mismo subtarea.

En la lección siguiente — Lección 07: Multi-Agent Orchestration for Engagements — pasamos de ejecutar cada agent en solitario a coordinarlos mediante una cola compartida, usando un orquestador ligero que hace el handoff entre Claude Code y Codex CLI según la fase del engagement. Vas a construir un controlador pequeño que vigila findings nuevos de nuclei, despacha el triage a Codex y escala las vulnerabilidades confirmadas a Claude Code para el análisis en profundidad y el desarrollo del exploit.

Trae el `codex-triage.sh` que construiste en esta lección. Lo vas a cablear dentro del orquestador.
