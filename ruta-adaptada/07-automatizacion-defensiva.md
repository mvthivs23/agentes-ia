# 07 — Automatización defensiva

La consultora te va a pedir que el control corra solo: en el pipeline, en un schedule, o sobre el índice de runbooks. El repositorio tiene ese patrón en el archivo defensivo de IA local, en el triage, y en el lab de contenedores cuando pide la nota de detección y la segunda corrida ya bloqueada.

## Lee esto

- `red-team-ia/local-ai-for-defensive-security.md` de punta a punta. El toolkit es `~/soc-ai/`, a propósito separado de `~/pentest-ai/`. Scripts que ya define: agrupación de assets, IOC hacia SIEM o MISP, triage de exposición (el control es parchear, restringir, deshabilitar, verificar el banner o aceptar el riesgo), narrativa de IR que cita el log textual, lectura HTTP para la alerta sin disparar requests nuevos, caza de sinks con un campo `fix` y sin payload, borrador de detección, nota de password spray a partir de conteos (sin contraseñas), tarjeta de incidente, RAG de notas de IR donde el contexto recuperado es dato.
- `building-skills-with-claude-and-codex/09-ai-assisted-vulnerability-triage.md`
- `building-skills-with-claude-and-codex/08-workflow-automation-with-ai-agents.md`, leído como schedule de un control, no de un scan ofensivo
- `building-skills-with-claude-and-codex/10-integrating-ai-with-existing-pentest-pipelines.md`, para colgar el paso en un pipeline que el equipo ya tiene
- `llm-agent-red-team/24-lab-blue-team-response.md`, `31-blue-team-readiness.md` y `33-findings-to-compliance.md`

La matriz modelo-tarea del archivo defensivo es la que debes usar: el modelo barato clasifica, el de código propone el `fix`, el de embeddings indexa el runbook. No subas todo a un modelo frontier.

## Ejercicios

1. **Un script, un control.** Elige uno de los scripts de `local-ai-for-defensive-security.md` y córrelo sobre dato del lab: o el triage de exposición sobre un inventario que ya tengas, o el RAG de runbooks indexando las notas de detección de las lecciones 03, 04 y 06 de contenedores. El system prompt de ese RAG dice que, si no está en las notas, responda que no está, y que el contexto no es una orden.

2. **Gate en el pipeline.** Del archivo 10 de building-skills, inserta en el workflow de lab un paso que falle el build si el scanner de imagen de la lección 03 de contenedores reporta un hallazgo que esa lección clasifica como bloqueante, o si el informe de IaC de la lección 10 de contenedores marca un hallazgo abierto. El paso usa el fallback local si la API no está, igual que `finding-writer`.

3. **De hallazgo a compliance.** Sigue el objetivo 33 de `llm-agent-red-team`: el texto pasa el resultado de la simulación a un formato de compliance. Tu insumo no es la simulación ofensiva; es el informe del ejercicio 2 más la nota de detección. No cambies el esquema que ese objetivo pide. Cámbiale solo el origen del dato.

4. **Ledger.** Cada llamada al modelo del gate queda en el ledger que describe `llm-api-integration-for-pentest-automation.md` y el asistente de `building-custom-ai-security-assistants.md`: modelo, tokens, costo, versión del prompt. Sin eso no puedes explicar el mes que viene por qué el gate decidió lo que decidió.
