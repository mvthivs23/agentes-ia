# Lección 01 — Claude Code Architecture and Capabilities

Claude Code y OpenAI Codex no son chatbots. Son runtimes de agent extensibles, con hooks, skills, MCP servers, tool calling y memoria durable. Tratados como chatbots te ahorran minutos; tratados como motores programables te ahorran días por engagement y, con el tiempo, reemplazan categorías enteras de scripting manual.

Este módulo recorre toda la superficie de autoría de Claude Code y Codex — anatomía de un skill, diseño de un MCP server, orquestación de workflows, patrones de subagent, hooks, settings — y construye juntos tres skills de pentest completos: un scanner de reconnaissance, un validador de vulnerabilidades y un generador de reports. Cada artefacto que produzcas acá se vuelve un componente reutilizable en cada engagement posterior.

*Claude Code agent loop con skills, MCP servers, hooks y subagents — la maquinaria que este módulo cablea en tu workflow de pentest.*

Módulo 06: Building Pentest Skills with Claude and Codex · Penetration Testing with Kali Linux 2026 · Red Team Leaders

## Introducción

La mayoría de la gente conoce a Claude por una ventana de chat en el navegador y nunca mira más allá. Para un pentester, eso es un error. Claude Code no es una ventana de chat: es un runtime de agent que vive en tu caja Kali, tiene una shell, lee y escribe archivos, sigue políticas explícitas de uso de tools y se puede extender con tus propios scripts, playbooks, subagents y servers conectados a la red. Entender ese runtime como entiendes nmap, Burp Suite o Metasploit es lo que convierte a Claude de una novedad en un multiplicador de fuerza durante los engagements.

Esta lección es un recorrido desde cero del runtime tal como se entrega en 2026: la CLI, las tools integradas, el sistema de permisos y settings, el contexto de la conversación y la compaction, la composición de `CLAUDE.md`, los MCP servers, los hooks, los slash commands, el sistema de Skills, los subagents, los tiers de razonamiento y cómo se compara con Codex CLI. Al final vas a tener una instalación de Claude Code en Kali configurada para trabajo ofensivo: un `settings.json`, un hook que registra cada comando corrido durante un engagement, un `CLAUDE.md` que mantiene al modelo dentro del scope, un skill propio que codifica tu workflow de recon y un slash command que produce un esqueleto de report. Cada comando de esta lección es uno que deberías correr en tu lab.

El resto del Módulo 06 se apoya en esta base. La Lección 02 te enseña a escribir tu propio MCP server para una tool de recon a medida. La Lección 03 recorre cómo cablear Codex CLI al lado de Claude Code. La Lección 04 arma un workflow de engagement completo que encadena skills, hooks y subagents. Si pasas esta lección por arriba, las siguientes se van a sentir como magia — y la magia es exactamente lo que no te puedes permitir cuando un cliente te paga para probar o refutar la explotabilidad.

## 1. Claude Code es un agent runtime, no una ventana de chat

### 1.1 Qué significa «agent runtime»

Una ventana de chat toma un prompt y devuelve texto. Un runtime de agent toma un prompt, corre un loop de decisión y llama tools para cambiar el mundo hasta que la tarea está hecha. El loop de Claude Code se ve así:

```text
┌──────────────────────────────────────────────────────────────────┐
│  1. Lee el prompt del usuario (o una inyección de hook, o un     │
│     slash command)                                               │
│  2. Carga el contexto: CLAUDE.md, skills, turnos previos         │
│  3. El modelo produce texto y/o tool calls                       │
│  4. El runtime ejecuta las tool calls (con compuertas de permiso)│
│  5. Los resultados de las tools vuelven al modelo                │
│  6. El loop sigue hasta que el modelo deja de pedir tools        │
│  7. Emite el mensaje final del assistant y corre los hooks Stop  │
└──────────────────────────────────────────────────────────────────┘
```

Cada paso es un lugar donde puedes interceptar: hooks en los pasos 1, 4, 6 y 7; `CLAUDE.md` y skills en el paso 2; reglas de permiso en el paso 4; los MCP servers extienden el conjunto de tools disponible en el paso 3.

### 1.2 Por qué debería importarle a un pentester

- **Reproducibilidad.** Un runtime bien configurado registra cada prompt, cada tool call y cada cambio de archivo. Las notas del engagement se escriben solas.
- **Guardrails.** Puedes denegar en duro los comandos destructivos y exigir confirmación para cualquier cosa que salga del lab.
- **Velocidad.** Un skill bien afinado corre un workflow de recon de cinco pasos con un solo prompt.
- **Enforcement de scope.** Un hook puede negarse a correr nmap contra cualquier cosa que esté fuera de `scope.txt`. Eso te protege a ti y al cliente.
- **Composabilidad.** El mismo runtime maneja pentests web, trabajo de Active Directory, auditorías cloud y redacción de reports.

### 1.3 Qué está en scope para esta lección

Cubrimos el runtime: instalación, el REPL, tools, permisos, contexto, `CLAUDE.md`, MCP, hooks, slash commands, skills, subagents, modos de razonamiento y una mirada breve a Codex CLI. No cubrimos escribir un MCP server (Lección 02) ni armar un workflow de engagement completo (Lección 04). Todo lo de acá es la plomería previa.

## 2. Instalar Claude Code en Kali 2026

Kali 2026 trae Node.js 22 LTS en su repositorio por defecto, que es lo que apunta Claude Code. Instálalo de forma global con npm.

### 2.1 Prerrequisito de Node.js

```bash
# Confirma que Node.js está presente y es al menos la versión 20:
node --version
# v22.9.0

# Si falta:
sudo apt update
sudo apt install -y nodejs npm
```

Si prefieres un version manager, nvm y fnm funcionan. No uses el paquete snap: el sandbox va a interferir con la capacidad de Claude Code de leer los archivos de tu proyecto.

### 2.2 Instalar la CLI

```bash
# Instala de forma global para que el binario `claude` quede en tu PATH:
sudo npm install -g @anthropic-ai/claude-code

# Verifica la instalación:
claude --version
# 2.4.1 (Claude Code)

# Muestra dónde cayó el binario:
which claude
# /usr/bin/claude
```

Si npm se queja por permisos, usa sudo o configura un prefix global escribible por tu usuario:

```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @anthropic-ai/claude-code
```

### 2.3 Autenticación

Tienes dos opciones: login interactivo por la consola de Anthropic, o una API key en el entorno. En Kali, el camino de la API key es más confiable porque puede que estés operando sin navegador.

```bash
# Opción A — login interactivo (abre una pestaña del navegador):
claude login
# Sigue la URL, aprueba el dispositivo y vuelve a la terminal.

# Opción B — API key en el entorno:
export ANTHROPIC_API_KEY="sk-ant-api03-..."
echo 'export ANTHROPIC_API_KEY="sk-ant-api03-..."' >> ~/.bashrc

# Confirma que la auth funciona:
claude --print "Say hello."
# Hello!
```

> **Pentester Insight:** Nunca dejes horneada una API key real en una VM de lab que planeas compartir o snapshotear. Usa una key con scope acotado que puedas rotar. Las keys de Anthropic no traen permisos por tool propios: cualquiera con la key puede gastar de la cuenta, así que trátala como una clave privada SSH.

### 2.4 Layout de directorios después de la instalación

```bash
ls -la ~/.claude
# CLAUDE.md         # tus instrucciones globales
# settings.json     # tus settings globales
# commands/         # slash commands de usuario
# skills/           # skills de usuario
# agents/           # definiciones de subagent de usuario
# projects/         # estado por proyecto y transcripts
# logs/             # logs del runtime (si están habilitados)
```

El directorio `~/.claude` es tu taller. Todo usuario serio de Claude Code le va dando forma a este directorio con el tiempo; trátalo como tu `~/.zshrc` o tu `~/.gnupg`.

## 3. Tu primera session interactiva — el REPL

Entra a un directorio de proyecto vacío y lanza Claude Code sin argumentos.

```bash
mkdir -p ~/engagements/lab-scratch
cd ~/engagements/lab-scratch
claude
```

Caés en el REPL. Escribe prosa para hablarle al modelo; escribe `/` para invocar un slash command.

### 3.1 Slash commands centrales

| Comando | Propósito |
|---|---|
| `/help` | Lista cada comando integrado con una descripción de una línea |
| `/config` | Abre el editor de settings (theme, modelo, telemetría, permisos) |
| `/model` | Cambia el tier de modelo activo (Opus, Sonnet, Haiku, etc.) |
| `/clear` | Borra la conversación y arranca de nuevo con los mismos archivos de contexto |
| `/compact` | Resume la conversación hasta ahora para recuperar contexto |
| `/status` | Muestra conteos de tokens, modelo, session id y estado de conexión MCP |
| `/permissions` | Editor interactivo de reglas allow/ask/deny |
| `/mcp` | Lista los MCP servers conectados y sus tools |
| `/agents` | Lista los tipos de subagent disponibles |
| `/skills` | Lista los skills cargados y sus descripciones de trigger |
| `/cost` | Imprime el costo acumulado en tokens y dólares de esta session |
| `/logout` | Limpia la auth cacheada |
| `/exit` | Termina la session |

### 3.2 Qué hacen de verdad `/clear` y `/compact`

- `/clear` borra los turnos de la conversación. El turno siguiente se comporta como si acabaras de lanzar la session. `CLAUDE.md`, los skills y los MCP servers se recargan; la discusión previa desaparece.
- `/compact` le pide al modelo que resuma la conversación en una forma compacta y reemplaza los turnos con ese resumen. Usa `/compact` cuando una session larga de investigación produjo estado útil que no quieres perder, pero te estás quedando sin contexto. Usa `/clear` entre tareas que no se relacionan.

### 3.3 Modo no interactivo

El REPL no es la única entrada. `--print` (o `-p`) corre un solo turno y sale, que es lo que usas dentro de scripts de shell y en CI:

```bash
# Haz una pregunta de un solo tiro:
claude --print "Resume el último scan de nmap en scans/target.xml"

# Mete stdin en el prompt:
cat scans/target.xml | claude --print "Extrae los puertos abiertos como JSON"

# Fuerza un modelo específico:
claude --print --model sonnet "Redacta un resumen ejecutivo breve"

# Continúa la session más reciente de forma no interactiva:
claude --continue --print "Ahora lista solo los servicios HTTP."
```

`--print` escribe en stdout solo el mensaje final del assistant. El output de las tools y el razonamiento intermedio no se imprimen, salvo que pases `--verbose`.

### 3.4 Estado de la session en disco

Cada session cae en `~/.claude/projects/<hash>/`. Cada turno es una entrada JSONL. Esta es la fuente de verdad si necesitas probar qué se dijo o qué ejecutó el modelo durante un engagement.

```bash
ls ~/.claude/projects/-home-kali-engagements-lab-scratch/
# 019abc.jsonl
# 019abd.jsonl

# Mira el primer mensaje de la session más reciente:
jq 'select(.type=="user") | .message.content' \
  ~/.claude/projects/-home-kali-engagements-lab-scratch/019abd.jsonl | head
```

Archiva los archivos JSONL con la evidencia del engagement. Son lo más cerca que tienes de un audit log completo del comportamiento del assistant.

## 4. El sistema de tools integradas

El runtime trae un conjunto fijo de tools. Cada tool call es un pedido estructurado que el modelo emite y el runtime ejecuta. Tienes que conocer cada una por nombre, porque las vas a permitir, denegar y loguear de forma individual.

### 4.1 Inventario de tools integradas

| Tool | Qué hace | Uso típico en un pentest |
|---|---|---|
| `Bash` | Ejecuta un comando de shell | Correr nmap, nikto, gobuster, scripts propios |
| `Read` | Lee un archivo (con números de línea, estilo `cat -n`) | Revisar output de scans, código fuente, archivos de config |
| `Write` | Crea o sobrescribe un archivo | Producir borradores de report, scripts de PoC, wordlists |
| `Edit` | Reemplazo de string por coincidencia exacta en un archivo | Iterar sobre exploits de PoC |
| `Grep` | Búsqueda de contenido respaldada por ripgrep | Encontrar credenciales, tokens y endpoints en dumps de código |
| `Glob` | Matching de patrones de nombre de archivo | Localizar archivos de config en un árbol de código grande |
| `Task` | Lanza un subagent (ver Sección 12) | Delegar un fan-out de recon o el borrador de un report |
| `WebFetch` | Trae una URL y le hace al modelo una pregunta sobre ella | Revisar páginas de CVE, advisories de vendors, docs |
| `TodoWrite` | Mantiene una lista de tareas estructurada, visible en la UI | Seguir findings durante un engagement en vivo |
| `NotebookEdit` | Edita notebooks de Jupyter | Limpiar notebooks de análisis |
| `ExitPlanMode` | Sale del modo solo-plan | Fin de una session de planificación, antes de ejecutar |

Las tools que provee MCP se nombran `mcp__<server>__<toolname>` y, desde el punto de vista del modelo, se comportan como las integradas. Los slash commands y los hooks se les aplican igual.

### 4.2 Qué requiere cada tool integrada

- `Bash` requiere un comando y una descripción opcional. El runtime lo ejecuta en una shell con cwd persistente, pero el estado de la shell (variables de entorno, funciones) no persiste entre llamadas, salvo que las definas en un archivo de arranque.
- `Read` requiere un `file_path` absoluto. Leer un archivo enorme devuelve las primeras 2000 líneas por defecto; pasa `offset` y `limit` para el resto.
- `Write` requiere un `file_path` absoluto y el contenido completo. Sobrescribir un archivo que no leíste se rechaza.
- `Edit` requiere `file_path`, `old_string` y `new_string`. El `old_string` tiene que aparecer exactamente una vez en el archivo, o pasas `replace_all: true`.
- `Grep` soporta regex completo de ripgrep, modos de output (`content`, `files_with_matches`, `count`) y filtros por tipo de archivo.
- `Glob` devuelve paths ordenados por fecha de modificación. Es más rápido que `find` y entiende la recursión `**`.
- `Task` requiere un `subagent_type` y un prompt. Corre una instancia fresca del modelo con su propio conjunto de tools y devuelve el texto final.
- `WebFetch` requiere una `url` y un prompt. Trae la página, la convierte a Markdown y le hace al modelo tu pregunta.
- `TodoWrite` toma la lista completa en cada llamada: no hay updates parciales. Los estados son `pending`, `in_progress`, `completed`.

### 4.3 Flujo de una tool call, ilustrado

```text
Prompt del usuario:  "Escanea 10.10.10.5 en busca de puertos abiertos y guarda los resultados."
Turno 1 del modelo:  Bash(command="nmap -sV 10.10.10.5 -oX scan.xml")
Runtime:             [chequeo de permiso] -> [ejecuta] -> [devuelve exit code + output]
Turno 2 del modelo:  Read(file_path="/home/kali/scan.xml")
Runtime:             [chequeo de permiso] -> [devuelve el contenido del archivo]
Turno 3 del modelo:  (texto) "Puertos 22, 80 y 443 abiertos. HTTP es nginx 1.24..."
Hook Stop:           [corre, loguea el transcript]
```

No corre nada que el modelo no haya pedido, y nada de lo que el modelo pidió corre si la compuerta de permisos no dice que sí.

## 5. El modelo de permisos

Cada tool call pasa por una compuerta de tres veredictos: `allow`, `ask` o `deny`. Los veredictos vienen de archivos de settings, de hooks o del prompt interactivo en el momento de la llamada.

### 5.1 Veredictos

- `allow` — corre de inmediato, sin prompt.
- `ask` — pausa y le pregunta al usuario; la respuesta del usuario vale solo para esta llamada, salvo que elija `"always"`.
- `deny` — rechaza. El modelo ve el rechazo y, por lo general, se adapta.

### 5.2 Schema de `settings.json`

Tanto `~/.claude/settings.json` (global) como `<project>/.claude/settings.json` (de proyecto) comparten el mismo schema. El de proyecto pisa al global key por key.

```json
{
  "permissions": {
    "allow": [
      "Bash(nmap:*)",
      "Bash(gobuster:*)",
      "Bash(nikto:*)",
      "Read(//home/kali/engagements/**)",
      "Write(//home/kali/engagements/**)",
      "Edit(//home/kali/engagements/**)",
      "Grep(*)",
      "Glob(*)",
      "WebFetch(domain:nvd.nist.gov)",
      "WebFetch(domain:cve.mitre.org)",
      "WebFetch(domain:exploit-db.com)"
    ],
    "ask": [
      "Bash(sqlmap:*)",
      "Bash(hydra:*)",
      "Bash(msfconsole:*)",
      "Bash(metasploit:*)",
      "WebFetch(*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(dd:*)",
      "Bash(:(){:|:&};:)",
      "Bash(shutdown:*)",
      "Bash(reboot:*)",
      "Write(//etc/**)",
      "Write(//boot/**)",
      "Edit(//etc/**)"
    ]
  },
  "model": "sonnet",
  "env": {
    "ENGAGEMENT_ID": "ACME-2026-Q3"
  },
  "hooks": {},
  "outputStyle": "concise"
}
```

### 5.3 Sintaxis de las reglas

- `Bash(<pattern>)` matchea contra el comando inicial. `Bash(nmap:*)` matchea cualquier cosa que empiece con `nmap`.
- `Read(<glob>)` usa paths absolutos con prefijo de doble barra, para que la expansión de globs en la shell no se los coma.
- `WebFetch(domain:<host>)` matchea contra el host de la URL.
- Los nombres de tool pelados matchean cada llamada, por ejemplo `Grep(*)`.

### 5.4 Precedencia

Orden de evaluación:

1. Decisiones del hook `PreToolUse` (ver Sección 9).
2. `settings.local.json` del proyecto (git-ignored, overrides por usuario).
3. `settings.json` del proyecto (commiteado al repo, para todo el equipo).
4. `~/.claude/settings.json` global.
5. Prompt interactivo (en modo interactivo) o deny duro (en modo `--print` sin `--dangerously-skip-permissions`).

Dentro de un archivo, las reglas más específicas ganan sobre las menos específicas, y `deny` siempre le gana a `allow`.

### 5.5 Un default sano para engagements

Arranca restrictivo. Es más fácil agregar una regla `allow` cuando te aparece un prompt que explicar por qué el modelo acaba de correr hydra contra producción.

```bash
# Edita rápido los settings del proyecto:
mkdir -p .claude
$EDITOR .claude/settings.json

# O usa el editor interactivo desde el REPL:
> /permissions
```

## 6. Contexto de la conversación y compaction

Un modelo tiene una context window finita. En 2026 el tier Sonnet por defecto trae 1M de input tokens y Opus 500k, pero ni eso es infinito, y los engagements largos los van a agotar.

### 6.1 Qué cuenta contra la window

- Cada mensaje del usuario y cada respuesta del assistant.
- Cada tool call y cada resultado de tool.
- El system prompt.
- Cada archivo `CLAUDE.md` cargado para esta session.
- El frontmatter del `SKILL.md` de cada skill (no el cuerpo, hasta que se dispara).
- El schema de cada tool MCP.
- Imágenes cacheadas y lecturas de archivos.

### 6.2 Cómo medirlo

```text
# Dentro del REPL:
> /status
# Session: 019abd
# Model: claude-sonnet-4.7
# Tokens: 148,203 / 1,000,000 (14.8%)
# MCP: 3 connected, 47 tools

> /cost
# Input tokens:  148,203  ($0.44)
# Output tokens:  18,904  ($0.28)
# Total:                  ($0.72)
```

### 6.3 Compaction

Cuando te acercas al límite, el runtime va a ofrecer compactar. También puedes dispararlo en cualquier momento con `/compact`. El resultado compactado reemplaza el transcript en el lugar:

```text
Transcript original (350k tokens)
   |
   v
Resumen compactado (12k tokens) + resultados de tools originales
conservados para los últimos 5 turnos
```

Pierdes fidelidad, pero conservas la trama. Cualquier cosa crítica (paths a archivos, credenciales descubiertas, direcciones IP) ya debería estar persistida en disco antes de la compaction: el modelo puede volver a hacer `Read` si la necesita.

### 6.4 Hook `PreCompact`

El hook `PreCompact` corre antes de la compaction, así puedes copiar el transcript completo a algún lugar durable (ver Sección 9).

## 7. Archivos `CLAUDE.md`

`CLAUDE.md` es cómo inyectas instrucciones permanentes en cada session. Tres ubicaciones se componen, de la más amplia a la más estrecha.

### 7.1 Los tres tiers

| Path | Scope | Cuándo usarlo |
|---|---|---|
| `~/.claude/CLAUDE.md` | Cada session en esta máquina | Preferencias personales, nombre, tooling estándar |
| `<project>/CLAUDE.md` | Cada session arrancada en este árbol de directorios | Scope del engagement, lista de targets, estilo de la casa |
| `<project>/<subdir>/CLAUDE.md` | Solo cuando el cwd está en el subdir o debajo | Reglas específicas del componente |

Se concatenan en orden y se inyectan como parte del system prompt.

### 7.2 Un `CLAUDE.md` global orientado a pentest

```markdown
# Banco de pentest en Kali

- Operator: J. Santonio
- El toolbox vive en /opt/tools
- Los engagements viven en /home/kali/engagements/<client>-<year>-<quarter>/
- Cita siempre el path del archivo cuando referencies output de un scan
- Nunca ejecutes comandos destructivos (rm -rf, dd, mkfs) sin aprobación explícita
- Prefiere outputs de scan en XML/JSON para que el parsing posterior sea determinista
- Cuando edites exploits de PoC, incluye siempre un comentario de encabezado corto que nombre el CVE
- Si una IP target no está en ./scope.txt, rechaza y pregúntale al usuario
```

### 7.3 Un `CLAUDE.md` por engagement

```markdown
# Acme Corp — pentest externo Q3 2026

## Scope
- 203.0.113.0/24 (prefijo público de Acme)
- *.acme.example (subdominios bajo el dominio primario)
- FUERA DE SCOPE: 203.0.113.42 (Salesforce del cliente), 203.0.113.99 (MTA externo)

## Rules of engagement (RoE)
- Ventana de testing: 2026-09-05 08:00 a 2026-09-19 20:00 UTC
- Sin DoS, sin ingeniería social, sin físico
- Avisa al lead del cliente antes de correr cualquier exploit que cambie el estado del server

## Estilo de la casa
- Los borradores de report caen en ./report/
- Findings numerados acme-<seq>, por ejemplo acme-014
- Cada finding tiene severity, descripción, evidencia y remediación
```

### 7.4 Reglas de composición

- Los archivos se concatenan, no se mergean. Si el global dice «sé breve» y el proyecto dice «sé verboso», gana el último.
- Los symlinks se siguen una vez. Un loop se rechaza.
- Los archivos de más de 128KB se truncan con un warning.
- La recarga ocurre en cada `/clear`, al inicio de la session, y cada vez que editas el archivo a mitad de session y escribes `/reload-claudemd`.

### 7.5 La directiva `@import`

Dentro de un `CLAUDE.md` puedes traer otros archivos de forma textual:

```text
@import ./scope.txt
@import ~/team/red-team-house-style.md
```

Los imports se resuelven en el momento de la carga. Los imports cíclicos se rechazan.

## 8. MCP servers — el punto de extensión

Model Context Protocol (MCP) es cómo le adjuntas tools nuevas a Claude Code sin recompilar nada. Un MCP server es un proceso separado que habla el protocolo de cable de MCP por stdio, SSE o HTTP; Claude Code lo lanza o se conecta a él al inicio de la session y mezcla sus tools en el conjunto de tools del modelo.

### 8.1 Por qué te importa como pentester

Obtienes una tool a medida por workflow: un puente a Burp Suite, un controlador de beacons de Cobalt Strike, un server de búsqueda de wordlists, una tool de queries de BloodHound, un resumidor de resultados de scan en XML. Cada una se le aparece al modelo como si fuera integrada.

### 8.2 Agregar un server

```bash
# Agrega un MCP server que corre en local por stdio:
claude mcp add my-recon \
  --command /opt/tools/recon-mcp/target/release/recon-mcp \
  --scope user

# Agrega un server que habla HTTP:
claude mcp add corp-kb \
  --url https://mcp.internal.example/kb \
  --transport http \
  --scope project

# Agrega con variables de entorno:
claude mcp add jira \
  --command node --args /opt/mcp-jira/dist/index.js \
  --env JIRA_API_TOKEN=xxxxx \
  --scope user
```

### 8.3 Listar y quitar

```bash
# Lista cada server configurado, mostrando scope y estado de conexión:
claude mcp list
# my-recon    (user)     connected    12 tools
# corp-kb     (project)  connected     4 tools
# jira        (user)     failed       0 tools    connection refused

# Quita uno:
claude mcp remove jira --scope user
```

### 8.4 Scopes

- `user` (default): se guarda en `~/.claude/mcp.json`. Disponible en cada proyecto.
- `project`: se guarda en `<project>/.mcp.json`. Se commitea al repo para que el equipo también lo tenga.
- `local`: se guarda en `<project>/.claude/mcp.local.json`. Por máquina, git-ignored.

### 8.5 Modos de fallo y diagnóstico

Si un server no arranca, la CLI reporta el error al inicio de la session y deshabilita sus tools para esa session. Causas comunes:

- `ENOENT`: el path del binario está mal o no es ejecutable.
- `ConnectionRefused`: el endpoint HTTP está caído o el puerto está bloqueado.
- Transport mismatch: apuntaste la CLI a HTTP cuando el server habla stdio.

Dentro del REPL, `/mcp` muestra el estado actual y te deja reintentar la conexión.

## 9. Hooks — automatizar el runtime

Los hooks son comandos de shell que el runtime ejecuta en puntos definidos del loop. Te dejan enforcing política, loguear actividad y reformar el comportamiento sin tocar el modelo.

### 9.1 Los eventos de hook

| Evento | Se dispara cuando | ¿Puede bloquear? |
|---|---|---|
| `SessionStart` | Arranca una session nueva | No |
| `UserPromptSubmit` | El usuario escribió un prompt, pero antes de que el modelo lo vea | Sí (exit 2 bloquea) |
| `PreToolUse` | El modelo pidió una tool call | Sí (exit 2 deniega) |
| `PostToolUse` | Una tool call ya volvió | No |
| `Notification` | El runtime quiere notificar al usuario | No |
| `Stop` | El modelo está por terminar su turno | Sí (exit 2 le pide que continúe) |
| `PreCompact` | Está por compactar el transcript | No |
| `SessionEnd` | La session se está cerrando | No |

### 9.2 Contrato del hook

Cada hook recibe un blob JSON por stdin que describe el evento. Exit code 0 significa «allow, sin mensaje». Exit code 2 significa «bloquear», y stderr se le devuelve al modelo. Cualquier otra cosa se trata como error del runtime y se loguea.

### 9.3 Cableado de hooks

Los hooks se declaran en `settings.json` bajo un objeto `hooks`. Los matchers pueden acotar por nombre de tool o por patrón.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "date '+inicio de session %F %T' >> ~/.claude/logs/audit.log"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/kali/.claude/hooks/scope-guard.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -c . >> ~/.claude/logs/tool-calls.jsonl"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'turno terminado' >> ~/.claude/logs/audit.log"
          }
        ]
      }
    ]
  }
}
```

### 9.4 Un scope guard

Rechaza cualquier comando nmap cuyo target sea una IP que no está en `scope.txt`.

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/scope-guard.sh
set -euo pipefail

payload="$(cat)"
tool="$(jq -r '.tool_name' <<< "$payload")"
cmd="$(jq -r '.tool_input.command // ""' <<< "$payload")"

if [[ "$tool" != "Bash" ]]; then exit 0; fi
if ! grep -qE '^(nmap|masscan|nikto|gobuster) ' <<< "$cmd"; then exit 0; fi

for target in $(grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' <<< "$cmd"); do
  if ! grep -qxF "$target" ./scope.txt 2>/dev/null; then
    echo "target $target is not in ./scope.txt — refusing" >&2
    exit 2
  fi
done
exit 0
```

Déjalo en `~/.claude/hooks/scope-guard.sh`, hazle `chmod +x` y conéctalo como `PreToolUse` sobre `Bash`, como arriba. Ahora el modelo no puede escanear hosts fuera de scope si el operator no edita `scope.txt` de forma explícita.

### 9.5 Un rate limiter

Evita que el modelo dispare scanners en un loop cerrado:

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/rate-limit.sh
set -euo pipefail

state=~/.claude/state/last-scan
mkdir -p "$(dirname "$state")"
now=$(date +%s)
last=$(cat "$state" 2>/dev/null || echo 0)
if (( now - last < 15 )); then
  echo "rate limit: el último scan fue hace $((now-last))s, espera 15s" >&2
  exit 2
fi
echo "$now" > "$state"
exit 0
```

### 9.6 Audit logging

Cada tool call y su resultado se vuelven una línea JSONL. Combinado con el transcript de la session en `~/.claude/projects/`, esto es un registro del comportamiento del assistant que se puede presentar. Rota el log con logrotate si lo vas a guardar a largo plazo.

## 10. Slash commands

Los slash commands son atajos definidos por el usuario, guardados como archivos Markdown. Cuando escribes `/foo`, el runtime encuentra `foo.md` en `~/.claude/commands/` o en `<project>/.claude/commands/` y trata el cuerpo como el prompt que le manda al modelo. Los argumentos después del slash command se sustituyen en `$ARGUMENTS`.

### 10.1 Un slash command de recon

```bash
mkdir -p ~/.claude/commands
cat > ~/.claude/commands/recon.md <<'EOF'
---
description: Corre un pase de recon ligero contra un target y resume
argument-hint: <ip-or-domain>
---
Ejecuta lo siguiente, en secuencia:

1. `nmap -sV -Pn --top-ports 200 -oX recon-nmap.xml $ARGUMENTS`
2. Si HTTP o HTTPS está abierto, corre `whatweb -a3 $ARGUMENTS`
3. Si HTTPS está abierto, corre `sslscan $ARGUMENTS`
4. Lee los archivos resultantes y produce un resumen de dos párrafos:
   - Párrafo 1: puertos abiertos, servicios, versiones
   - Párrafo 2: hipótesis inicial de attack surface
NO ejecutes nada intrusivo.
EOF
```

Ahora `/recon 10.10.10.5` dispara el workflow.

### 10.2 Un slash command de esqueleto de report

```bash
cat > ~/.claude/commands/finding.md <<'EOF'
---
description: Agrega un finding al report del engagement actual
argument-hint: "<short-title>"
---
Agrega un finding nuevo a ./report/findings.md con esta plantilla:

## Finding: $ARGUMENTS
- ID: (el siguiente en la secuencia)
- Severity: (Critical|High|Medium|Low|Info)
- Affected: (host, URL, endpoint)
- Description:
- Evidence: (extracto de log, path de screenshot, request/response)
- Impact:
- Remediation:
- References:

Completa lo que puedas a partir del contexto. Deja `TODO` en todo lo que no sepas.
EOF
```

### 10.3 Comandos con namespace

Los subdirectorios se vuelven namespaces de comandos. `~/.claude/commands/web/xss.md` se invoca como `/web:xss`.

### 10.4 Flags

- `description`: (requerido en el frontmatter) lo muestra `/help`.
- `argument-hint`: (opcional) se muestra en el hint de input.
- `allowed-tools`: (opcional) restringe las tools que el modelo puede usar para este comando.
- `model`: (opcional) fuerza un tier de modelo específico para este comando.

### 10.5 Slash commands frente a skills

Un slash command lo invoca el usuario. Un skill lo carga el runtime cuando matchean las condiciones de trigger. Usa comandos para one-shots de alta fricción y alto valor. Usa skills para conocimiento de fondo que debería moldear al modelo cada vez que aparezca una tarea de esa área.

## 11. El sistema de Skills

Los skills son el mecanismo de playbook del runtime. Cada skill vive en su propio directorio, con un archivo `SKILL.md` cuyo frontmatter trae un `name` y una `description`. La description hace también de condición de trigger: en cada turno el modelo considera si aplica y, si aplica, carga el cuerpo.

### 11.1 Layout de directorios

```text
~/.claude/skills/
  web-recon/
    SKILL.md
    references/
      wordlists.md
      httpx-flags.md
    scripts/
      quick-httpx.sh
  ad-enum/
    SKILL.md
  report-writing/
    SKILL.md
```

### 11.2 Frontmatter de `SKILL.md`

```markdown
---
name: web-recon
description: |
  Workflow de reconnaissance de aplicaciones web para engagements autorizados
  de bug bounty y penetration testing. Úsalo cuando el operator pida enumerar
  un target web, listar subdominios, descubrir endpoints o tomar la huella
  de un stack de webserver. No es para explotación: solo para mapear la superficie.
allowed-tools: Bash, Read, Write, Grep, Glob
---

# Web recon

## Precondiciones
- Existe `./scope.txt` y contiene los hosts que están en scope
- El directorio de trabajo contiene un subdirectorio `recon/` (créalo si falta)

## Workflow

1. Enumeración de subdominios
   - `subfinder -d $TARGET -o recon/subs.txt`
   - `assetfinder --subs-only $TARGET | tee -a recon/subs.txt`
   - Dedup: `sort -u -o recon/subs.txt recon/subs.txt`

2. Liveness
   - `httpx -l recon/subs.txt -status-code -title -tech-detect -o recon/http.txt`

3. Descubrimiento de endpoints en hosts vivos
   - `gau $TARGET | tee recon/urls.txt`
   - `waybackurls $TARGET | tee -a recon/urls.txt`

4. Brute de directorios
   - `ffuf -u https://<host>/FUZZ -w /usr/share/wordlists/dirb/common.txt -o recon/ffuf-<host>.json -of json`

5. Resumen
   - Produce `recon/summary.md` con:
     - Cantidad de subdominios vivos
     - Tech interesante (WordPress, Jenkins, GitLab, etc.)
     - Hits del brute de directorios que vale la pena investigar

## Referencias
Mira `references/httpx-flags.md` para el recetario de flags de httpx y
`references/wordlists.md` para la guía de selección de wordlists.
```

### 11.3 Reglas de carga

- El `name` tiene que ser único entre todas las fuentes; los conflictos se rechazan con un warning en la carga.
- El frontmatter (`name` + `description`) siempre se carga y cuenta contra el contexto.
- El cuerpo se carga solo cuando el modelo decide que el skill aplica — en general porque la description matchea la tarea que tiene entre manos.
- Las referencias y los scripts dentro del directorio del skill los lee el modelo de forma perezosa, vía `Read`.

### 11.4 De dónde vienen los skills

- Global: `~/.claude/skills/<name>/SKILL.md`
- Proyecto: `<project>/.claude/skills/<name>/SKILL.md`
- Skills de plugin (instalados vía `claude plugin`): directorios que son del plugin, listados con un prefijo `plugin:`.

Los skills de proyecto pisan a los skills globales del mismo nombre.

### 11.5 Cuándo construir un skill

Cada vez que te descubras tipeando el mismo preámbulo de cinco líneas al modelo al empezar una tarea, ese preámbulo debería ser un skill. Skills que se pagan en un contexto de pentesting:

- Recetarios por tool (referencia de flags de nmap, recetas de ffuf, flags de sqlmap)
- Playbooks por clase de target (web, AD, cloud, mobile)
- Estilo de la casa para redactar reports
- Investigación de CVE (dado un CVE, produce una nota de triage)
- Andamiaje de desarrollo de exploits (plantilla de PoC, esqueleto de módulo de Metasploit)

### 11.6 Probar un skill

```text
# Desde el REPL, lista los skills cargados ahora:
> /skills

# Fuerza la carga de uno para revisar su contenido:
> /skill web-recon show
```

Si un skill que escribiste no se dispara, la description es la culpable el 90 % de las veces. Reescríbela en la forma «Úsalo cuando …» y por lo general se enciende.

## 12. Subagents — fan-out y especialización

La tool `Task` lanza un subagent. Un subagent es un loop fresco de Claude Code con su propia conversación, su propio conjunto de tools y su propio tier de modelo. El turno padre sigue una vez que el subagent devuelve su mensaje final de assistant.

### 12.1 Por qué subagents

- **Aislamiento de contexto.** El output de tools de un subagent no llena la window del padre.
- **Especialización.** Un subagent se puede configurar con un conjunto de tools más estrecho (solo lectura para una tarea de investigación, shell completa para un builder).
- **Paralelismo.** Varios subagents pueden correr a la vez cuando el padre emite varias llamadas `Task` en un turno.

### 12.2 Definir un tipo de subagent

Los tipos de subagent son archivos en `~/.claude/agents/` o en `<project>/.claude/agents/`.

```markdown
---
name: recon-scout
description: |
  Agent de reconnaissance de solo lectura. Corre tools de recon contra hosts
  en scope y devuelve un resumen. Nunca edita archivos.
tools: Bash, Read, Grep, Glob
model: sonnet
---

# Recon scout

Eres un agent de solo recon. No escribes ni editas archivos.
Devuelves un resumen breve de lo que encontraste, con paths de archivos.
```

Ahora el padre puede lanzarlo:

```text
Task(subagent_type="recon-scout", prompt="Haz recon de 10.10.10.5, 10.10.10.6 y 10.10.10.7 en paralelo y resume.")
```

### 12.3 Tipos de agent integrados

- `general-purpose`: sin restricciones.
- `Explore`: agent de búsqueda amplia, de solo lectura.
- `Plan`: arquitectura y planificación, sin escrituras.
- `statusline-setup`: configura la status line.
- Los tipos de la comunidad y los que proveen los plugins aparecen al lado de estos.

### 12.4 Concurrencia

Cuando el turno padre emite varias llamadas `Task` a la vez, corren en paralelo. El runtime espera a que todas vuelvan antes de avanzar el turno padre. Úsalo para abrir en abanico el recon por target o el análisis por host.

### 12.5 Costo y quotas

Los subagents queman tokens por su cuenta. Un fan-out de cinco vías en Sonnet puede ser barato en wall-clock y caro en tokens. Síguelo con `/cost` y ponle un tope con un hook `PreToolUse` sobre `Task` si lo necesitas.

## 13. Fast mode y tiers de razonamiento

El runtime te deja elegir un modelo por session, por comando o por subagent.

### 13.1 Tiers de modelo en 2026

| Tier | Latencia | Profundidad de razonamiento | Costo típico | Cuándo usarlo |
|---|---|---|---|---|
| `haiku` | Por debajo del segundo | Liviana | El más barato | Escaneo masivo de archivos, transforms simples de JSON |
| `sonnet` | 1–3 s | Fuerte | Del medio | Default para la mayor parte del trabajo de engagement |
| `opus` | 3–8 s | La más profunda | El más caro | Desarrollo de exploits difíciles, triage ambiguo |
| `fast` (alias) | El más rápido disponible | Lo que sea que el alias fast apunte ahora | El más barato | Modo `/fast`: saltear razonamiento para ediciones rápidas |

### 13.2 Cambiar a mitad de session

```text
> /model sonnet
> /model opus
> /fast on
> /fast off
```

`/fast on` pone al runtime en un modo que suprime los tokens de razonamiento extendido y prefiere el modelo más chico que igual pueda llamar tools. Préndelo cuando sabes que la tarea es mecánica (renombrar una variable en un archivo). Apágalo antes del trabajo difícil.

### 13.3 Modelo por comando y modelo por agent

- El frontmatter de un slash command puede setear `model: opus`.
- El frontmatter de un tipo de agent puede setear `model: haiku`.
- La variable de entorno `ANTHROPIC_MODEL=sonnet` setea el default para todas las sessions que arranquen desde esa shell.

### 13.4 Política práctica

- Deja la session en Sonnet por defecto.
- Pasa a Opus para desarrollo de exploits, análisis de criptografía y cualquier triage que tenga una pregunta de plausibilidad.
- Manda el resumen masivo de logs y los subagents de pegamento con grep a Haiku.

## 14. Codex CLI — una comparación breve

OpenAI entrega un runtime de agent competidor llamado Codex CLI (`codex`). Tiene una forma parecida en grande y distinta en el detalle. Un pentester debería conocer los dos, porque el modelo detrás de cada uno tiene modos de fallo distintos, y de vez en cuando vas a querer una segunda opinión.

### 14.1 Paridad de capacidades

| Capacidad | Claude Code | Codex CLI |
|---|---|---|
| Bash/Read/Write/Edit integrados | Sí | Sí |
| Búsqueda web / fetch | Sí (`WebFetch`) | Sí |
| Modelo de permisos con allow/ask/deny | Sí | allow/deny más simple |
| Hooks (7 eventos) | Sí | Solo hook de session-start |
| Cliente MCP | Sí | Sí |
| MCP server (exponer tus propias tools) | Vía SDK | Vía SDK |
| Slash commands | Archivos Markdown | Archivos Markdown |
| Skills | Sí | Sin equivalente nativo (workaround: SDK más system prompt) |
| Subagents | Sí | Sí (`--agent`) |
| Modo no interactivo | `claude --print` | `codex exec` |
| Transcripts de session | JSONL bajo `~/.claude/projects/` | JSONL bajo `~/.codex/history/` |

### 14.2 Cuándo ir a Codex

- El tier actual de Codex tiene fortalezas particulares en una clase de problema en la que estás trabado.
- Quieres una segunda opinión de triage con otro sesgo.
- Las rules of engagement de un cliente prohíben un vendor específico.

### 14.3 Cuándo quedarte en Claude Code

- Dependés de hooks, skills o tools MCP que ya construiste.
- Necesitas el conjunto más amplio de verbos de permiso.
- Quieres el sistema de Skills, que es el mayor multiplicador de productividad que ofrece este runtime.

Vamos a correr los dos lado a lado en la Lección 03.

## 15. Ejemplo completo — una instalación de Claude Code lista para pentest

Junta todo. Arranca con un `~/.claude` vacío (o haz un backup) y después arma la configuración pieza por pieza y verifica cada paso.

### 15.1 `CLAUDE.md` global

```bash
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md <<'EOF'
# Banco de pentest en Kali — global

- Operator: J. Santonio
- Los engagements viven bajo /home/kali/engagements/
- Las wordlists viven bajo /usr/share/wordlists/
- Cita siempre los paths de archivo cuando referencies output de un scan
- Nunca ejecutes rm -rf, dd, mkfs, shutdown o reboot sin aprobación explícita
- Prefiere outputs XML o JSON de los scanners para el parsing posterior
- No corras tests intrusivos contra un host que no esté en ./scope.txt
EOF
```

### 15.2 `settings.json` global

```bash
cat > ~/.claude/settings.json <<'EOF'
{
  "permissions": {
    "allow": [
      "Bash(nmap:*)",
      "Bash(gobuster:*)",
      "Bash(ffuf:*)",
      "Bash(nikto:*)",
      "Bash(whatweb:*)",
      "Bash(sslscan:*)",
      "Bash(httpx:*)",
      "Bash(subfinder:*)",
      "Bash(assetfinder:*)",
      "Bash(gau:*)",
      "Bash(waybackurls:*)",
      "Bash(jq:*)",
      "Bash(grep:*)",
      "Bash(sort:*)",
      "Bash(tee:*)",
      "Read(*)",
      "Grep(*)",
      "Glob(*)",
      "Write(//home/kali/engagements/**)",
      "Edit(//home/kali/engagements/**)",
      "WebFetch(domain:nvd.nist.gov)",
      "WebFetch(domain:cve.mitre.org)",
      "WebFetch(domain:exploit-db.com)"
    ],
    "ask": [
      "Bash(sqlmap:*)",
      "Bash(hydra:*)",
      "Bash(medusa:*)",
      "Bash(msfconsole:*)",
      "Bash(metasploit:*)",
      "Bash(msfvenom:*)",
      "WebFetch(*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(dd:*)",
      "Bash(mkfs:*)",
      "Bash(shutdown:*)",
      "Bash(reboot:*)",
      "Write(//etc/**)",
      "Write(//boot/**)",
      "Write(//root/**)",
      "Edit(//etc/**)"
    ]
  },
  "model": "sonnet",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          { "type": "command", "command": "mkdir -p ~/.claude/logs && date '+inicio de session %F %T' >> ~/.claude/logs/audit.log" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "/home/kali/.claude/hooks/scope-guard.sh" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "mkdir -p ~/.claude/logs && jq -c '{ts: now, tool: .tool_name, input: .tool_input, result: (.tool_response | tostring | .[0:2000])}' >> ~/.claude/logs/tool-calls.jsonl" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "date '+fin de turno %F %T' >> ~/.claude/logs/audit.log" }
        ]
      }
    ]
  }
}
EOF
```

### 15.3 El hook scope-guard

```bash
mkdir -p ~/.claude/hooks
cat > ~/.claude/hooks/scope-guard.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

payload="$(cat)"
tool="$(jq -r '.tool_name' <<< "$payload")"
cmd="$(jq -r '.tool_input.command // ""' <<< "$payload")"

if [[ "$tool" != "Bash" ]]; then exit 0; fi
case "$cmd" in
  nmap*|masscan*|nikto*|gobuster*|ffuf*|whatweb*|sslscan*|httpx*)
    ;;
  *)
    exit 0
    ;;
esac

targets="$(grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}|[A-Za-z0-9.-]+\.[A-Za-z]{2,}' <<< "$cmd" || true)"

if [[ ! -f ./scope.txt ]]; then
  echo "no scope.txt in cwd — refusing scanner invocation" >&2
  exit 2
fi

for t in $targets; do
  if ! grep -qxF "$t" ./scope.txt; then
    echo "target $t not in ./scope.txt — refusing" >&2
    exit 2
  fi
done
exit 0
EOF
chmod +x ~/.claude/hooks/scope-guard.sh
```

### 15.4 Un skill de recon

```bash
mkdir -p ~/.claude/skills/web-recon
cat > ~/.claude/skills/web-recon/SKILL.md <<'EOF'
---
name: web-recon
description: |
  Úsalo cuando el operator pida hacer recon de un target web — enumerar
  subdominios, tomar la huella del stack, descubrir endpoints o producir
  un mapa de superficie. No es para explotación.
allowed-tools: Bash, Read, Write, Grep, Glob
---

# Workflow de web recon

## Prechecks
- Confirma que existe ./scope.txt y que el target pedido aparece ahí.
- Asegúrate de que exista un directorio ./recon; créalo si falta.

## Pasos

1. Subdominios

subfinder -d $TARGET -silent | tee recon/subs.txt assetfinder --subs-only $TARGET | tee -a recon/subs.txt sort -u -o recon/subs.txt recon/subs.txt
2. Liveness

httpx -l recon/subs.txt -status-code -title -tech-detect -o recon/http.txt
3. URLs

gau $TARGET | tee recon/urls.txt waybackurls $TARGET | tee -a recon/urls.txt sort -u -o recon/urls.txt recon/urls.txt
4. Resumen
Produce recon/summary.md con:
- cantidad de subdominios vivos
- stacks de tech notables
- los tres leads más prometedores

## Reglas
- Nunca corras nada intrusivo desde este skill.
- Nunca toques un host que no esté en scope.txt.
EOF
```

### 15.5 Un slash command para cargar un finding

```bash
mkdir -p ~/.claude/commands
cat > ~/.claude/commands/finding.md <<'EOF'
---
description: Agrega un finding nuevo a ./report/findings.md
argument-hint: "<short-title>"
---
Asegúrate de que exista ./report/findings.md. Agrega este bloque, con el ID siguiente:

## Finding: $ARGUMENTS
- ID: acme-NN
- Severity: TODO
- Affected: TODO
- Description: TODO
- Evidence: TODO
- Impact: TODO
- Remediation: TODO
- References: TODO

Completa cualquiera de los TODO que puedas inferir de la conversación actual.
EOF
```

### 15.6 Verificar cada pieza

```bash
# 1. La CLI está en el PATH:
claude --version
# 2.4.1

# 2. La auth funciona:
claude --print "reply with the word ok"
# ok

# 3. El CLAUDE.md global está cargado:
claude --print "¿en qué directorio viven los engagements?"
# /home/kali/engagements/ ...

# 4. El deny de settings funciona:
claude --print "ejecuta: rm -rf /tmp/testfile"
# El runtime rechaza la tool call.

# 5. El skill se activa:
cd ~/engagements/lab-scratch
echo "example.com" > scope.txt
mkdir -p recon
claude --print "haz recon de example.com según nuestro workflow web-recon. No corras scanners de verdad; describe lo que correrías."
# La respuesta debería mencionar subfinder, httpx y gau en el orden que plantea el skill.

# 6. El hook deniega lo que está fuera de scope:
claude --print "ejecuta: nmap -sV 8.8.8.8"
# El hook se dispara; el comando se rechaza con "target 8.8.8.8 not in ./scope.txt".

# 7. El slash command existe:
claude
> /help
# `finding` aparece en la lista.
> /finding sql-injection-on-login
# El bloque del finding se agrega a ./report/findings.md.
```

### 15.7 Confirmar el audit log

```bash
tail -n 5 ~/.claude/logs/audit.log
tail -n 5 ~/.claude/logs/tool-calls.jsonl | jq
```

Ahora tienes un runtime que loguea todo, rechaza comandos destructivos, rechaza scans fuera de scope y codifica un workflow de recon como skill y una acción de report como slash command. Esa es la línea de base sobre la que construye cada lección siguiente.

## 16. Ejercicios

### Guiados

- **Agrega una regla allow de masscan.** Extiende `~/.claude/settings.json` para permitir `Bash(masscan:*)`. Confirma que el scope guard sigue rechazando cuando el target no está en `scope.txt`. Confirma que permite cuando el target sí está en `scope.txt`.

- **Escribe un hook PreCompact.** Copia el transcript completo a `~/.claude/logs/transcripts/<session-id>.jsonl` antes de la compaction, para no perder nunca la fidelidad. Verifica produciendo una session larga y disparando `/compact`.

- **Construye un skill cve-triage.** Description del frontmatter: «Úsalo cuando el operator entregue un identificador de CVE y pida un triage.» Cuerpo: trae la entrada de NVD vía `WebFetch`, extrae el CVSS, los productos afectados, si hay un exploit público, y produce una nota de triage de cinco líneas. Prueba con `CVE-2024-4577`.

### Independientes

- **Diseña un hook Notification.** Mándate una notificación de escritorio (`notify-send`) cada vez que el modelo esté esperando tu input. Justifica el diseño: cuándo quieres que te avise y cuándo no.

- **Política de ruteo de modelo.** Escribe un hook `PreToolUse` que inspeccione el nombre de la tool y, si es `Task`, fuerce el modelo del subagent a `haiku`. Justifica por qué sí o por qué no, y mide el ahorro de tokens en una session de recon en fan-out.

- **Triage de skills.** Toma tres prompts que suelas reescribir al inicio de una session y convierte cada uno en un skill. Prueba que los tres se disparen con los prompts que corresponden y que ninguno se dispare con el prompt equivocado. Si un skill se dispara de más, reescribe su description hasta que deje de hacerlo.

## 17. Resumen y siguiente lección

El runtime de Claude Code es un loop de decisión envuelto alrededor de un modelo, con puntos de inserción limpios en cada paso: `CLAUDE.md` y los skills le dan forma al conocimiento del modelo; los permisos y los hooks le dan forma a lo que tiene permitido hacer; los slash commands empaquetan prompts repetidos; los MCP servers extienden su conjunto de tools; los subagents abren en abanico o especializan; el tier de razonamiento elige el modelo que encaja con la tarea. En Kali 2026 lo instalas con un comando de npm y lo configuras con un `settings.json`, un `CLAUDE.md`, un par de hooks, un skill y un slash command — que es exactamente lo que te recorre el ejemplo completo de la Sección 15.

En la Lección 02 bajamos una capa: vas a escribir tu propio MCP server en Rust, exponerle a Claude Code una tool de recon a medida como capacidad de primera clase, y cablearla en el runtime que configuraste en esta lección. Si todavía no corriste cada comando de la Sección 15, hazlo antes de empezar la Lección 02: vas a necesitar la línea de base funcionando.
