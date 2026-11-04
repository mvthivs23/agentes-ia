# Ruta adaptada — ejercicios DevSecOps

Cada archivo de esta carpeta es un ejercicio con pasos. Los comandos, los schemas y las policies salen de los módulos del repositorio. El artefacto cambia: en vez de un target de pentest, el resultado es un control que puedes enseñar en la consultora.

No hay técnicas nuevas. Si un paso no está en el módulo citado, no está aquí.

## Carpetas del repositorio, por prioridad

El 1 es lo que conviene abrir primero.

1. `01-ejercicios-devsecops` — estos ejercicios.
2. `02-contenedores-pipelines-e-iac` — imagen, Kubernetes, CI/CD, cloud e IaC.
3. `03-python-entorno-y-tools` — el entorno y el esqueleto de una tool.
4. `04-ia-local-prompts-y-rag` — cómo hablarle a un modelo local: prompts, Ollama, RAG y revisión de código. No arma un agente.
5. `05-skills-claude-y-codex` — skills, triage y la API de NVD dentro de Claude Code y Codex.
6. `06-agentes-llm-flujos-y-guardrails` — el agente que encadena tools, los guardrails y cómo se evalúa. La carpeta 04 es el modelo; esta es el flujo.
7. `07-linux-base` — la base del sistema. El `demo.txt` sigue sin partir.
8. `08-introduccion-agentes` — qué es un agente y el primer entorno. Ya está cubierta por las carpetas de arriba.

## Orden de esta carpeta

1. `01-entorno-python.md` — el entorno con el que arranca cualquier automatización.
2. `02-imagen-y-runtime.md` — imagen endurecida, scanners y el gate que falla el build.
3. `03-kubernetes-rbac-y-admission.md` — el binding malo y la policy que lo rechaza.
4. `04-cicd-e-iac.md` — permisos del workflow, digest, firma, tfsec y checkov.
5. `05-prompts-devsecops.md` — los prompts que el material ya fija: evidencia, fix, y «no está en el corpus».
6. `06-agente-local.md` — Ollama en loopback, la matriz de modelos de `~/soc-ai/` y un skill de triage.
7. `07-cve-y-nvd.md` — la API de NVD que el material usa para no publicar un CVE inventado.
8. `08-proyectos.md` — cuatro ensamblados con esas piezas, listos para presentar.

## Qué no está en los pasos

Los escapes, el volcado de Secrets, el payload de `pull_request_target`, el crack de JWT y la cadena de IMDS se quedan en el Markdown original. Aquí se corre la mitad que el propio lab llama detección, endurecimiento o parche.

No hay en el repositorio un documento titulado OWASP Top 10 for LLM Applications. El control de aplicaciones de IA que sí está escrito es el de las lecciones de RAG, de agentes y de prompt injection: allowlist, contexto recuperado que no es una orden, y la frase `Not covered by the corpus`.
