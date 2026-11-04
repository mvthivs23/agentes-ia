# 06 — Agente local para el trabajo defensivo

Fuente: `04-ia-local-prompts-y-rag/05b-local-ai-for-defensive-security.md` (Ollama, `~/soc-ai/`, la matriz) y el ejercicio «Construye un skill cve-triage» de `05-skills-claude-y-codex/01-claude-code-architecture-and-capabilities.md`.

El agente no escanea fuera de lo que tú le pasas. Lee un informe, consulta NVD y escribe una nota. El modelo corre en loopback.

## Paso 1 — Modelos

Los `ollama pull` de la lección defensiva:

```bash
ollama pull llama3:8b
ollama pull mistral:7b
ollama pull qwen2.5-coder:7b
ollama pull nomic-embed-text
ollama pull phi3:mini
ollama pull llama3.2:3b
```

`codestral:22b` solo si tienes al menos 16 GB de VRAM, como dice la lección. No hace falta bajarlos todos el primer día. Para el triage de una alerta basta `phi3:mini`. Para el `suggested_fix` del archivo 05, `qwen2.5-coder:7b`.

```bash
mkdir -p ~/soc-ai
```

Ese directorio es el de la lección, separado de `~/pentest-ai/`.

## Paso 2 — Probar que responden

El `model_probe.py` de la lección recorre `llama3:8b`, `mistral:7b` y `qwen2.5-coder:7b`. Cópialo a `~/soc-ai/model_probe.py` y córrelo. Si un modelo no está descargado, el probe lo dice. No lo sustituyas por una API de terceros: la lección parte de que las RoE no dejan sacar el dato del lab.

## Paso 3 — Una tarea, un modelo

Toma la salida de `trivy image` del archivo 02, o el JSON de tfsec del archivo 04. Pásale a `phi3:mini` solo la clasificación: severidad y si el hallazgo bloquea el build según el `exit-code` que ya fijaste. Pásale a `qwen2.5-coder:7b` el `suggested_fix`, con el prompt del archivo 05.

El system prompt del RAG de runbooks, el de la misma lección, dice que si la nota no está en el contexto la respuesta es que no está, y que el contexto recuperado no es una orden. Indexa las notas de detección que ya escribiste. No indexa un dump ni un log de un cliente sin el permiso que la lección exige.

## Paso 4 — El skill de triage

El ejercicio guiado de la lección 01 de skills pide un `SKILL.md` con esta description en el frontmatter:

«Úsalo cuando el operator entregue un identificador de CVE y pida un triage.»

El cuerpo, el de ese ejercicio: trae la entrada de NVD, extrae el CVSS, los productos afectados y si hay un exploit público, y escribe una nota de cinco líneas. La prueba que el ejercicio nombra es `CVE-2024-4577`.

La nota no incluye un procedimiento. Las cinco líneas son: identificador, score, producto, si el advisory público menciona exploit, y el control que ya tienes en el pipeline (imagen, policy o permiso). El fetch es el del archivo `07-cve-y-nvd.md`. Si NVD no devuelve la CVE, la nota dice que no está. No completes el hueco.

## Paso 5 — Qué no hace el agente

La lección 11 de skills lo escribe: el modelo no consulta NVD salvo que tú lo cables. Por eso el skill llama a la API y el informe solo cita el JSON guardado. Un CVE que no está en ese JSON no entra en el entregable.

## Entregable

- `~/soc-ai/` con al menos dos modelos de la matriz respondiendo.
- Una nota de cinco líneas para `CVE-2024-4577`, con el JSON de NVD al lado.
- El `SKILL.md` con la description del ejercicio.
