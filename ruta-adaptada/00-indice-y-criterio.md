# Ruta adaptada — ingeniero DevSecOps en un equipo de innovación

Esta carpeta no agrega técnicas que el repositorio no tenga. Reordena el material que ya está, con el corte que corresponde a tu trabajo: productos con IA, pipelines de CI/CD, contenedores, IaC, cloud, y automatizaciones de seguridad. Lo ofensivo queda en los archivos originales. Aquí se usa la mitad que el propio material ya escribe al lado: detección, mitigación, hardening, validación de output, guardrails y el informe.

## Qué entra y qué no

Entra, porque el repositorio lo desarrolla y tu rol lo usa todas las semanas:

- Entorno Linux y Python para arrancar un proyecto de automatización.
- Contenedores, Compose, Kubernetes, microservicios, CI/CD, cloud, IaC y API gateway.
- IA local, prompt engineering, RAG, agentes con scope, skills de Claude Code y Codex, y la evaluación de lo que el modelo devuelve.
- El Top 10 de OWASP para aplicaciones y el tratamiento de fallas de aplicaciones LLM que ya está en las lecciones de RAG, agentes y prompt injection.

No se reescribe aquí, porque es el caso raro y el procedimiento ya está en el fuente:

- Escapes de contenedor, abuso de RBAC, inyección en workflows y la cadena ofensiva de los objetivos 17, 20–23, 25, 26, 28 y 30 de `llm-agent-red-team/`.
- Generación de payloads, phishing y PoC de las lecciones de skills y de red team.

Cuando un lab del curso dice "haz el ataque y después el control", esta ruta te pide el control y la nota de detección. El ataque se lee en el archivo original, en el lab, contra el target del lab.

## Huecos del fuente

No inventé las lecciones que el `demo.txt` no trae:

- `containers-security`: no está la lección 08. El texto de la 07 la nombra ("Container Escape Techniques and Kubernetes Attack Surface") y el contenido equivalente está repartido en las lecciones 03 y 04.
- `python-for-security`: solo están la 01 y la 05. Faltan la 02, la 03 y la 04.
- `llm-agent-red-team`: no hay "Lesson NN". Hay 33 bloques `Objective`, numerados en el orden del archivo.
- `building-skills-with-claude-and-codex`: no está la lección 03. El cierre de la 02 la anuncia ("Creating Custom Pentest Prompts for Codex") y el archivo no la incluye.
- `linux-security`: el `demo.txt` tiene las lecciones 01 a 12, todavía sin partir en Markdown. Esta ruta cita ese archivo y el número de lección.

## Orden de lectura

1. `01-base-linux-y-python.md`
2. `02-contenedores-supply-chain-y-runtime.md`
3. `03-kubernetes-microservicios-y-apis.md`
4. `04-cicd-cloud-e-iac.md`
5. `05-ia-local-prompts-rag-y-codigo.md`
6. `06-agentes-skills-y-guardrails.md`
7. `07-automatizacion-defensiva.md`
8. `08-proyectos-para-la-consultora.md`

Cada archivo dice qué leer, qué sección es el trabajo de tu rol, y un ejercicio que ya existe en ese material, cambiado de target: el artefacto es un control del pipeline, de la imagen, de la policy o del asistente interno. No es un target nuevo.
