# 08 — Proyectos para presentar

Cada proyecto de abajo es un ensamblado de piezas que ya están en el repositorio. No hay una arquitectura nueva. El valor para la consultora es que el dato del cliente se queda en el lab, el control corre en el pipeline, y el informe cita la evidencia sin inventarla.

Si una pieza todavía se está traduciendo, el contenido está en el `demo.txt` de esa carpeta, en las líneas que el índice de esta ruta ya asoció a la lección.

## Proyecto A — Gate de imagen y de IaC

Piezas:

- Imagen endurecida, lección 01 de `containers-security`, sección 15, y scanners, lección 03, sección 14.
- Secret injection, lección 02 de contenedores.
- Las seis fases y el cierre con SBOM, in-toto y cosign, lección 06.
- Informe de postura, lección 10 de IaC, sin tocar producción.
- El paso de CI colgado como en el archivo 07 de esta ruta.

Entregable: un repo de lab donde el pipeline firma la imagen, adjunta el SBOM y falla si el scanner o la policy de IaC fallan. La nota de detección sale de la lección 12 de contenedores, no de un playbook nuevo.

## Proyecto B — Asistente interno de seguridad, on-prem

Piezas:

- Ollama en loopback, `red-team-ia/setting-up-local-ai-with-ollama.md`.
- Prompt versionado, RAG con cita y refusal si no está en el corpus, `building-custom-ai-security-assistants.md` y el desafío `ragq` del lab.
- Router con `Scope` y guardrails de esa misma lección.
- Skill de informe, lecciones 02 y 05 de `building-skills-with-claude-and-codex`.
- MCP de una tool de lectura, lección 07 y el criterio del lab 12 de esa carpeta.
- Límite ético, lección 11 de esa carpeta.

Corpus del RAG: solo notas de mitigación y runbooks (lecciones 03, 04 y 06 de contenedores, más `local-ai-for-defensive-security.md`). El modelo dice "no está" cuando la pregunta se sale.

Entregable: una demo en la que un hallazgo real del proyecto A entra al asistente y sale una card con la evidencia textual, el arreglo y la cita del chunk. Si alguien pide un target fuera del scope del lab, la respuesta contiene `Out of scope` y la tool no corre.

## Proyecto C — Review de una aplicación con LLM

Piezas:

- Superficie de RAG, `red-team-ia/rag-systems-architecture-and-attack-surface.md`: provenance, delimitadores, filtro de salida, schema, filtro de metadata entre tenants.
- Arquitectura de agentes, `red-team-ia/ai-agent-architecture-and-security.md`: el contexto recuperado no es una orden; allowlist; human-in-the-loop.
- Review de código con `suggested_fix` y confirmación del snippet, `ai-powered-code-analysis-and-vulnerability-discovery.md`.
- Objetivos 18, 19 y 27 de `llm-agent-red-team`: validar el output, guardrails, evaluar.

Entregable: sobre un servicio de lab del equipo, un informe que nombre solo fallas que esos archivos ya describen (inyección vía documento indexado, tool sin allowlist, secreto en el prompt, dato de otro tenant en el retrieve). Cada ítem lleva el snippet confirmado y el arreglo en una frase. Lo que el modelo no pueda citar se va a la lista de no confirmados, como la lección de code analysis pide con el false positive.

## Proyecto D — Del hallazgo al paquete de compliance

Piezas:

- `finding-writer` del lab de `red-team-ia`: secciones Executive Summary, Technical Details, Impact, Remediation, References. Esas cadenas se quedan en inglés porque el script las busca.
- Objetivo 33 de `llm-agent-red-team`: el hallazgo técnico pasa al formato de compliance que ese objetivo define.
- Ledger de llamadas, para defender el costo y la reproducibilidad.

Entregable: un Markdown de un hallazgo del proyecto A o C, más el anexo en el formato del objetivo 33, más el JSONL del ledger. Sin CVE inventado: la rúbrica del lab de `red-team-ia` descuenta cada CVE, técnica o evidencia fabricados.

## Qué no presentar como proyecto

- Un agente que genere payloads, arme una cadena de exploit o redacte phishing. Esos objetivos existen en `llm-agent-red-team/` y en las skills ofensivas. No son el trabajo semanal del equipo y esta carpeta no los convierte en una oferta.
- Una lección que el `demo.txt` no contiene. Si el cliente pide la lección 08 de contenedores o la 03 de skills de Codex, la respuesta honesta es que el material no está y el tema vive en las lecciones vecinas ya citadas.
