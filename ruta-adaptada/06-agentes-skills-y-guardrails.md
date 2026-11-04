# 06 — Agentes, skills y guardrails

Un agente del equipo es un runtime con tools, no un chat. El repositorio ya fija las dos capas: el prompt dice que no, y el router lo impide igual. Eso es lo que puedes presentar como diseño de un asistente interno de seguridad.

## Lee esto

De `red-team-ia/`:

- `ai-agent-architecture-and-security.md`: percibir, planear, actuar, observar; tools; MCP; allowlist; sandbox; human-in-the-loop; el contexto recuperado es dato, no una orden.
- `building-custom-ai-security-assistants.md`: prompt versionado, RAG, tools con scope, LlamaGuard y Prompt Guard, ledger, eval antes de cambiar el prompt. La tabla de decisión (prompt, luego RAG, luego tool, y LoRA solo al final) es la que usas frente al equipo.
- `multimodal-ai-in-security.md`: screenshots y diagramas a JSON. La sección de CAPTCHA del archivo es solo defensiva y no trae bypass. Los documentos de identidad, solo con autorización y sin extraer PII. Úsalo para evidencias de un hallazgo de pipeline o de un manifiesto, no para pretextos.

De `building-skills-with-claude-and-codex/`:

| Archivo | Uso en el rol |
|---|---|
| `01-claude-code-architecture-and-capabilities.md` | Qué es el runtime: hooks, skills, MCP, sesión |
| `02-creating-custom-pentest-skills-for-claude-code.md` | El formato `SKILL.md`. El skill que armas aquí es de review o de informe, no el de generación de payloads |
| `05-building-report-generation-skills.md` | El informe que el cliente lee |
| `06-codex-cli-for-security-automation.md` | El otro runtime, misma idea de workflow repetible |
| `07-mcp-servers-for-pentest-integration.md` | Un server MCP con schema y con refusal si el target no está en scope. Las frases que el código compara (`target out of scope`, `url out of scope`, `module not allowlisted`) se quedan |
| `08-workflow-automation-with-ai-agents.md` | Orquestación. El schedule diario de recon es el caso raro; el schedule diario de drift y de scan de imagen es el caso tuyo |
| `09-ai-assisted-vulnerability-triage.md` | Ordenar hallazgos y bajar el false positive |
| `10-integrating-ai-with-existing-pentest-pipelines.md` | Encajar el modelo en un pipeline que ya existe |
| `11-ethical-considerations-and-limitations.md` | El límite que el asistente no negocia |
| `12-lab-building-and-testing-custom-skills.md` | El lab de cerrar el módulo. Toma el skill de reporte y el de validación; el de reconocimiento ofensivo queda en el archivo |

De `llm-agent-red-team/`, los objetivos que son diseño y control:

- 05 memoria y encadenamiento de tareas, leído como estado del agente, con la lección de agentes de `red-team-ia` al lado para no dejar que el estado se convierta en una orden.
- 09 cómo se evalúa un agente.
- 13 y 14, qué es un agente y la arquitectura.
- 15, LangChain y CrewAI, como opción de framework, no como obligación.
- 18, confianza en el output y validación.
- 19, guardrails.
- 27, evaluación.
- 29, benchmarks.
- 24, 31, 32 y 33: respuesta blue, preparación, simulación controlada y pasar el hallazgo a compliance. Esos cuatro son el caso de consultoría.

Los objetivos 08, 16, 17, 20, 21, 22, 23, 25, 26, 28 y 30 se quedan en su archivo. Son el flujo ofensivo. No se adaptan reescribiendo el payload.

## Ejercicios

1. **Prompt con versión.** De `building-custom-ai-security-assistants.md`, escribe el system prompt del asistente interno con las cláusulas que esa lección exige: identidad, tools declaradas, formato de finding, refusal fuera de scope, y "no lo sé" antes que inventar. El archivo es un `.prompt.md` con frontmatter semver, como la sección de versionado. El eval de esa lección busca `Out of scope` y las secciones del finding: no las traduzcas en el string que el test compara.

2. **Tool con allowlist.** Implementa el `Scope` de esa misma lección delante de cualquier tool que dispare un comando. Aunque el modelo pida otro host, el router responde antes de la red. Añade el chequeo de Prompt Guard sobre el chunk de RAG, como la sección de guardrails. El objetivo 19 de `llm-agent-red-team` es el mismo tema: léelo y no dupliques el mecanismo si ya está en el archivo de `red-team-ia`.

3. **Skill de informe.** Con el formato de la lección 02 de building-skills y el contenido de la lección 05, un `SKILL.md` que redacte el hallazgo a partir de evidencia ya capturada (el scanner de imagen, el informe de IaC, el JSON del review). Tiene que negarse si falta la evidencia. El `finding-writer` del lab de `red-team-ia` es el contrato: secciones obligatorias, evidencia textual, fallback local.

4. **MCP de una sola tool de lectura.** Del lab de la lección 12 de building-skills, el criterio de calificación de un server MCP: schema tipado, jobs si la llamada es larga, y rechazo si el target no está nombrado. Expón una tool de lectura que ya uses (el scanner de la lección 03 de contenedores, o el informe de IaC). No expongas una tool de exploit.
