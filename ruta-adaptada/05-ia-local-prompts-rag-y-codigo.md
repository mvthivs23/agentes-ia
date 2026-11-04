# 05 — IA local, prompts, RAG y revisión de código

El producto de IA del equipo y el asistente interno salen del mismo sitio: un modelo que no se lleva el dato del cliente, un prompt versionado, un índice que se puede auditar y una revisión que devuelve el arreglo. Eso ya está escrito en `red-team-ia/`. La mitad ofensiva de esos archivos (wordlists, variantes, el embudo de ataque) no se rehace aquí.

## Lee esto, en este orden

| Archivo | Qué te llevas |
|---|---|
| `red-team-ia/setting-up-local-ai-with-ollama.md` | Ollama en la máquina, GPU, Modelfile, API en `127.0.0.1:11434`, cuantización |
| `red-team-ia/fundamentos-prompt-engineering.md` | Roles, zero-shot, few-shot, structured output, delimitadores |
| `red-team-ia/advanced-prompt-engineering-for-pentest.md` | El método de prompt. El uso aquí es el informe y la revisión, no el sketch de exploit |
| `red-team-ia/rag-systems-architecture-and-attack-surface.md` | Ingesta, chunk, embed, índice, retrieve. Y la superficie: documento envenenado, filtro de metadata entre tenants, DoS por chunks, y las defensas en capas (provenance, delimitadores, filtro de salida, schema, tools en sandbox) |
| `red-team-ia/ai-powered-code-analysis-and-vulnerability-discovery.md` | El pipeline de review: pasada determinista, triage con LLM, campo de arreglo, confirmación por el snippet, FPR |
| `red-team-ia/local-ai-for-defensive-security.md` | El espejo defensivo: IOC hacia el SIEM, triage de exposición, narrativa de IR, RAG de runbooks |
| `fundamentos-LLM.md` e `introduccion-LLM-seguridad.md` | El vocabulario común del equipo antes de los labs |

De `llm-agent-red-team/`, cuando estén los archivos:

- `01-llm-platforms-and-frameworks.md`: la tabla offline frente a cloud. Para un dato de cliente gana la fila offline.
- `06-private-data-embeddings-and-rag.md`: índice sobre material propio.
- `07-fine-tuning-vs-prompt-engineering.md`: el mismo criterio que `red-team-ia/building-custom-ai-security-assistants.md`: el prompt enseña el rol, el RAG enseña el hecho, el fine-tuning entra tarde.

El lab `red-team-ia/lab-ai-setup-and-prompt-engineering-exercises.md` arma Ollama, Chroma y cuatro tools. De esas cuatro, en esta ruta entran `ragq` (el índice tiene que citar la fuente y negarse si la pregunta no está en el corpus) y `finding-writer` (la evidencia no se inventa; hay fallback local). `pentest-assistant` y `recon-triage` quedan para el caso ofensivo raro.

## OWASP y el Top 10 de aplicaciones LLM

No hay en el repo un documento aparte titulado "OWASP Top 10 for LLM Applications". Lo que sí está, y es el estándar que el material usa:

- En el corpus de `ragq` (lab de `red-team-ia`, desafío 2): el OWASP Top 10 2021, ítem por ítem, más fichas de CVE atadas a esa categoría. La A10 del texto es SSRF, incluido el metadata service.
- En `rag-systems-architecture-and-attack-surface.md`: inyección indirecta por documento, robo de contexto entre tenants, y las defensas.
- En `ai-agent-architecture-and-security.md`: confused deputy, allowlist, human-in-the-loop, y el prompt que trata el contexto recuperado como dato.
- En `building-custom-ai-security-assistants.md` y en la lección 11 de `building-skills-with-claude-and-codex`: el modelo puede ser inyectado; el runtime también enforce el scope.

Eso es el cuerpo de seguridad de una aplicación LLM que este repositorio alcanza. No le sumes una lista de riesgos que ningún archivo numere.

## Ejercicios

1. **Modelo local.** Sigue `setting-up-local-ai-with-ollama.md` hasta tener el servicio solo en loopback. Anota el Modelfile y la cuantización. Ese es el runtime de los productos que no pueden salir a una API pública. La lección de multimodal y la de asistentes repiten la misma regla: dato regulado, modelo local, salvo permiso escrito.

2. **Índice con cita.** Del lab de prompt engineering, el desafío 2 (`ragq`): indexa en el corpus las secciones de mitigación de la lección 03 de contenedores y el stack defensivo de la lección 06 de CI/CD, no un dump de exploits. Pregunta algo que esté en ese corpus y exige el id del chunk. Pregunta algo que no esté y confirma la frase de refusal que el desafío fija (`Not covered by the corpus`).

3. **Review con arreglo.** Del archivo de code analysis, corre el esquema de la lección: una función por chunk, JSON con `severity`, `evidence_snippet` y `suggested_fix`, y el chequeo de que el snippet existe en el archivo. El repo objetivo es un servicio del equipo, en el lab. Descarta lo que no confirma el regex, como dice la sección de verificación.

4. **Presupuesto y redacción.** Si el review o el informe salen por API, usa de `llm-api-integration-for-pentest-automation.md` solo el patrón de presupuesto, redactado de PII antes del request, ledger y el modelo barato en el embudo ancho. La key no se commitea. El desafío 4 del lab (`finding-writer`) ya exige conservar la evidencia textual y caer a Ollama si la API no está.
