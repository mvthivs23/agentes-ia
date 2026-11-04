# Notas de seguridad, contenedores e IA local

Material de estudio en español, sacado de los `demo.txt` de un curso de Kali Linux / Red Team Leaders y de la introducción *Agentes de IA con Python*. Cada lección o objetivo es un Markdown dentro de su carpeta. El número del archivo es el de la lección, para que el directorio quede en el orden del curso. El número de la carpeta es la prioridad de estudio: 01 es la que más conviene abrir primero.

La prosa está en español. Se dejan en inglés las palabras que el material usa como término técnico: prompt, system prompt, LLM, RAG, Ollama, MCP, skill, tool, scope, CI/CD, IaC, Kubernetes, SBOM, embeddings, guardrails, eval, CVE, CWE, CVSS. El código conserva la sintaxis en inglés (`string`, `int`, flags, claves JSON, tokens que un test compara).

Cada carpeta que viene de un `demo.txt` guarda ese archivo al lado de los Markdown. El `demo.txt` es el fuente en inglés. El Markdown es la lección. Donde el fuente no trae una lección, el Markdown no se inventó.

## Cómo está armado

| Prioridad | Carpeta | Qué es |
| --- | --- | --- |
| 01 | [contenedores, pipelines e IaC](#01-contenedores-pipelines-e-iac) | Docker, Compose, Kubernetes, microservicios, CI/CD, cloud, Terraform, API gateway |
| 02 | [Python, entorno y tools](#02-python-entorno-y-tools) | Entorno, tipos, y cómo pasar de un script a una tool |
| 03 | [IA local, prompts y RAG](#03-ia-local-prompts-y-rag) | Prompt engineering, Ollama, RAG, APIs, revisión de código, asistentes |
| 04 | [Skills de Claude y Codex](#04-skills-claude-y-codex) | Claude Code, skills, Codex, MCP, triage, pipelines, NVD |
| 05 | [Agentes LLM, flujos y guardrails](#05-agentes-llm-flujos-y-guardrails) | El agente que encadena tools, memoria, guardrails, eval y compliance |
| 06 | [Linux base](#06-linux-base) | Sistema, usuarios, procesos, red, paquetes y scripting. Solo en `demo.txt` |
| 07 | [Introducción a agentes](#07-introduccion-agentes) | Qué es un agente, el primer entorno de Python y dos resúmenes de LLM |

La carpeta 03 y la 05 no son el mismo tema. La 03 es el modelo: cómo se escribe un prompt, cómo se levanta Ollama, cómo se indexa un corpus y cómo se revisa código. La 05 es el flujo: plataformas, memoria, LangChain, CrewAI, guardrails, laboratorios y la evaluación del agente.

## Huecos del fuente

- Contenedores: no existe la lección 08. El cierre de la 07 la nombra. El contenido equivalente está en las lecciones 03 y 04.
- Python: solo están la 01 y la 05. El fuente anuncia la 02 (Python para red) y no la trae. Tampoco están la 03 ni la 04.
- Skills: no existe la lección 03. El cierre de la 02 la anuncia y el archivo no la incluye. El fuente de esa carpeta se llama `demo..txt`, con dos puntos.
- IA local: el primer Markdown es la lección 02. No hay archivo de lección 01.
- Linux: las 12 lecciones siguen juntas en `06-linux-base/demo.txt`. No hay un Markdown por lección.
- En varios Markdown de contenedores y de agentes, el procedimiento de un escape, de un payload o de un volcado no se copió. El impacto, la detección y el control sí. El texto inglés completo está en el `demo.txt` de esa carpeta.

## 01 — Contenedores, pipelines e IaC

Carpeta: `01-contenedores-pipelines-e-iac/`. Fuente: `demo.txt`.

Cubre el camino de un servicio desde la imagen hasta el clúster, el pipeline, la cuenta cloud y la API. Es el módulo que más se solapa con un trabajo de DevSecOps: runtime, supply chain, RBAC, workflows, Terraform y el gateway.

| Archivo | Qué desarrolla |
| --- | --- |
| `01-docker-fundamentals-and-internals.md` | Por qué importa un container, diferencia con una VM, instalación de Docker en Kali, namespaces, cgroups v2, capabilities, seccomp, overlay, imágenes, Dockerfile, multi-stage, redes, volumes, el socket del daemon, runtimes alternativos y el laboratorio de una imagen de Nginx endurecida (distroless, `USER nonroot`, `--read-only`, `--cap-drop ALL`). |
| `02-docker-compose-and-lab-orchestration.md` | Compose v2, services, networks, volumes, sustitución de entorno, profiles, overrides, límites, un lab de varios targets, health checks, inyección de secretos, limpieza, depuración y cuándo pasar de Compose a Kubernetes. |
| `03-container-security-and-escape-techniques.md` | Alcance del lab, modelo de aislamiento, enumeración desde dentro, ocho familias de escape (socket, privileged, `release_agent`, `core_pattern`, paths del host, `hostPID`, kernel, API cruda) con detección y mitigación, endurecimiento de la imagen, Trivy, Grype, hadolint, checkov, y el laboratorio que cierra en la configuración bloqueada. |
| `04-kubernetes-architecture-and-attack-surface.md` | Componentes del clúster, kind en Kali, API server, etcd, kubelet, kube-proxy, objetos, RBAC, ServiceAccount, el top 10 de malas configuraciones, herramientas de enumeración, tres caminos (pod a cluster-admin, kubeconfig a cloud, CI/CD), audit logs, Kyverno y el laboratorio del binding `cluster-admin` sobre la ServiceAccount `default`. |
| `05-microservices-architecture-patterns.md` | Monolito frente a microservicios, service discovery, API gateway, service mesh y mTLS, BFF, sync y async, gRPC, GraphQL, Kafka y RabbitMQ, eventos y CQRS, circuit breakers, trazas (Jaeger, Zipkin, OpenTelemetry), configuración y secrets, y el laboratorio de dos servicios con reflection de gRPC. |
| `06-cicd-pipeline-security.md` | Dónde viven los secrets en cada fase, GitHub Actions, `pull_request_target`, runners self-hosted, `GITHUB_TOKEN` con `contents: write`, GitLab CI, Jenkins, CircleCI, Buildkite y Drone, dependencias, builds envenenados, digests, cosign, SLSA, SBOM, in-toto, commits firmados y el workflow parcheado que pasa el título por una variable de entorno. |
| `07-web-application-architecture-deep-dive.md` | MPA, SPA, SSR, SSG e ISR, frameworks de frontend, REST, GraphQL, WebSocket, gRPC-Web, reverse proxy y request smuggling, balanceadores, CDN, edge functions, sesión, CORS, CSP, fingerprint del stack y el laboratorio de una app Next.js detrás de nginx. |
| `09-cloud-architecture-fundamentals.md` | AWS, GCP y Azure lado a lado, CLIs, IAM y los permisos que componen una escalada (`iam:PassRole`, `iam:CreateAccessKey`, `sts:AssumeRole`), enumerate-iam y Pacu a nivel de detección, VPC, IMDS v1 y v2, buckets públicos, serverless, bases managed, containers en cloud, secretos, CloudTrail y GuardDuty. |
| `10-infrastructure-as-code-and-configuration.md` | Terraform, el state como artefacto sensible, backends remotos cifrados, tfsec y checkov, Ansible, ansible-lint, Ansible Vault, cloud-init, Packer, Pulumi y CDK, drift, secretos en repos de IaC y policy-as-code. |
| `11-api-gateway-and-authentication-architecture.md` | El gateway como cuello de botella, bypass por backend directo o por header de confianza, OAuth 2.0 y PKCE, fallos de `redirect_uri`, OpenID Connect, estructura de un JWT, `alg: none`, confusión HS256/RS256, `jku` y `kid`, higiene del verifier, API keys, mTLS, rate limiting, WAF y el laboratorio que pasa de un secret HS256 débil a RS256 con `kid` en allow-list. |
| `12-lab-container-security-and-architecture-analysis.md` | Cuatro desafíos de laboratorio: socket de Docker, RBAC de Kubernetes, workflow de CI y reconocimiento de un stack con nginx, API y Elasticsearch. Cada uno trae la configuración vulnerable, la detección y el endurecimiento. |

## 02 — Python, entorno y tools

Carpeta: `02-python-entorno-y-tools/`. Fuente: `demo.txt`.

Sirve para arrancar un proyecto de automatización. No es un curso de explotación en Python: la 01 deja el entorno y el lenguaje, la 05 deja la forma de una tool que se puede entregar.

| Archivo | Qué desarrolla |
| --- | --- |
| `01-python-environment-and-language-fundamentals.md` | Python 3.12 en Kali, virtualenv, paquetes, tipos, control de flujo, funciones, errores, archivos, biblioteca estándar, PEP 8, un enumerador de headers HTTP y los ejercicios `merge_scans.py`, `@timed(threshold_ms=500)`, deduplicación de subdominios y un header de Basic auth. |
| `05-python-tool-development-building-real-pentest-tools.md` | Layout `src` y `pyproject.toml`, argparse, click y typer, TOML y variables de entorno, logging, salida con rich, JSON y CSV, concurrencia, reintentos, un enumerador de subdominios, un scanner HTTP de paths expuestos y headers de seguridad, empaquetado y tests. |

## 03 — IA local, prompts y RAG

Carpeta: `03-ia-local-prompts-y-rag/`.

Es el módulo del modelo. Empieza en el prompt, sigue por Ollama en loopback, el corpus recuperado y la API, y termina en un asistente con scope. La lección 05 tiene dos archivos: el del curso y un complemento defensivo escrito con la misma matriz de modelos, guardado en `~/soc-ai/` para no mezclarlo con `~/pentest-ai/`.

| Archivo | Qué desarrolla |
| --- | --- |
| `02-fundamentos-prompt-engineering.md` | Qué es un prompt, roles, few-shot, formato de salida y los límites de un modelo que predice el token siguiente. |
| `03-advanced-prompt-engineering-for-pentest.md` | Prompts más largos para tareas de seguridad, plantillas y el lugar donde se pega un advisory en vez de pedirle al modelo que lo invente. |
| `04-setting-up-local-ai-with-ollama.md` | Instalación de Ollama, bind a `127.0.0.1:11434`, GPU, modelos (Llama, Mistral, Qwen coder, Codestral, DeepSeek, Phi) y el uso de disco y VRAM. |
| `05-local-ai-for-offensive-security.md` | Los mismos modelos aplicados a notas de engagement, clustering, triage de salida de herramientas y un RAG de notas. El directorio de trabajo del curso es `~/pentest-ai/`. |
| `05b-local-ai-for-defensive-security.md` | El mismo stack, del lado SOC: clustering de assets, IOC, triage de exposición, resumen de logs, caza de sinks con un campo de fix, borrador de detección y RAG de notas de incidente. |
| `06-rag-systems-architecture-and-attack-surface.md` | Embeddings, chunking, vector database, filtros por tenant, rerank, hybrid search, HyDE, envenenamiento del corpus y un RAG mínimo en local. El laboratorio usa el OWASP Top 10 2021 como corpus. |
| `07-ai-agent-architecture-and-security.md` | Loop del agente, tools, memoria, permisos y los puntos donde un agente se sale del scope. |
| `08-llm-api-integration-for-pentest-automation.md` | Anthropic y OpenAI, tool use, reintentos, caché, presupuesto, ledger de gasto y un triage que clasifica hallazgos. |
| `09-ai-powered-code-analysis-and-vulnerability-discovery.md` | tree-sitter, chunking de código, índice, revisión con un modelo local y el objeto de finding con `evidence_snippet` y `suggested_fix`. Si el snippet no está literal en el archivo, el finding se descarta. |
| `10-multimodal-ai-in-security.md` | Modelos de visión locales, lectura de capturas y de páginas, y procedencia de una imagen. |
| `11-building-custom-ai-security-assistants.md` | PentestPal: Chroma, allowlist de scope, LlamaGuard y Prompt Guard, y el server que los junta. |
| `12-lab-ai-setup-and-prompt-engineering-exercises.md` | Laboratorio de cierre: Ollama, prompts, un RAG cuya frase de rechazo es `Not covered by the corpus`, y los desafíos de punta a punta. |

## 04 — Skills de Claude y Codex

Carpeta: `04-skills-claude-y-codex/`. Fuente: `demo..txt`.

Cómo se mete un modelo dentro de un flujo que ya existe: Claude Code, un `SKILL.md`, Codex CLI, un servidor MCP y un pipeline. La lección 11 es la que explica por qué un CVE escrito por un modelo se contrasta con NVD antes de entrar en un informe. El script de esa lección llama a `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=`.

| Archivo | Qué desarrolla |
| --- | --- |
| `01-claude-code-architecture-and-capabilities.md` | El loop de Claude Code, `CLAUDE.md`, permisos, hooks, slash commands, subagents y el ejercicio de un skill `cve-triage` que trae la entrada de NVD y escribe una nota de cinco líneas. |
| `02-creating-custom-pentest-skills-for-claude-code.md` | Formato de un skill, frontmatter, tools permitidas y cómo se dispara por la description. El fuente repite esta lección; el Markdown está una sola vez. |
| `04-building-exploitation-and-validation-skills.md` | El skill `cve-validate`: scope, log, el JSON de NVD, CVSS, CPE y referencias. El archivo del fuente se corta a mitad de una frase final, y el Markdown también. |
| `05-building-report-generation-skills.md` | Skills que arman un informe. Los datos de CVE salen de NVD, no del modelo. Si la CVE no está, se anota y no se inventa la descripción. |
| `06-codex-cli-for-security-automation.md` | Codex CLI como otra superficie para las mismas automatizaciones. |
| `07-mcp-servers-for-pentest-integration.md` | Servidores MCP, tools de solo lectura y rechazos cuando el target o el módulo quedan fuera de scope. |
| `08-workflow-automation-with-ai-agents.md` | Agentes de workflow: cuándo un paso corre solo y cuándo se detiene. |
| `09-ai-assisted-vulnerability-triage.md` | Triage asistido: qué se le pasa al modelo, qué tiene que citar y cómo se separa un hallazgo confirmado de uno posible. |
| `10-integrating-ai-with-existing-pentest-pipelines.md` | Dónde se cuelga el paso de IA en un pipeline que el equipo ya tiene, con fallback si la API no responde. |
| `11-ethical-considerations-and-limitations.md` | Alucinación de CVE, la asimetría de verificar un identificador falso, el script `ai-cve-check.sh` y los límites de usar la salida de un modelo como evidencia. |
| `12-lab-building-and-testing-custom-skills.md` | Laboratorio: construir un skill, probar que se dispara con el prompt correcto y que no se dispara con el equivocado. |

## 05 — Agentes LLM, flujos y guardrails

Carpeta: `05-agentes-llm-flujos-y-guardrails/`. Fuente: `demo.txt`.

No son lecciones numeradas. Son 33 objetivos, en el orden del archivo. Van de las plataformas y el fine-tuning al agente con tools, a los laboratorios y al informe de compliance.

Los objetivos 01 a 12 arman el modelo y la memoria. 13 a 17 definen qué es un agente y lo conectan a LangChain, CrewAI y tools. 18 y 19 son el control: no fiarse de la salida y poner guardrails. 20 a 26 son laboratorios. 27 a 33 evalúan, comparan contra un benchmark y cierran en preparación defensiva y compliance.

| Archivo | Qué desarrolla |
| --- | --- |
| `01-llm-platforms-and-frameworks.md` | Tabla de herramientas de IA ofensiva, comparación por escenario y aislamiento del laboratorio. |
| `02-tailored-offensive-llm-agents.md` | Arquitectura de un agente, SDKs, MCP, hosting y el ejemplo de phishing, que en el Markdown no incluye el flujo del descargador. |
| `03-why-customize-llms-and-embeddings.md` | Por qué un modelo genérico no alcanza, tipos de modelo y fuentes de contexto. |
| `04-adaptation-layers-for-red-teaming.md` | Capas: prompt engineering, instruction tuning, fine-tuning y RAG, con la tabla que las compara. |
| `05-operation-aware-agents-and-memory.md` | Agentes con contexto, MCP, memoria, perfiles por rol y coordinación. |
| `06-private-data-embeddings-and-rag.md` | Embeddings, vector databases, el camino del texto al índice y las prácticas para no tratar el contexto recuperado como una orden. |
| `07-fine-tuning-vs-prompt-engineering.md` | Cuándo ajustar el modelo y cuándo basta el prompt, más la estrategia híbrida con RAG y LoRA. |
| `08-red-team-agents-with-offensive-tools.md` | Arquitectura de un agente que llama tools, function calling y el bucle de feedback. |
| `09-assessing-offensive-agents.md` | Pruebas del agente, amenazas contra el propio agente, checklist de auditoría, logging e integridad de la memoria vectorial. |
| `10-osint-workflow-automation.md` | LangChain, AutoGen y CrewAI para un flujo de reconocimiento, y el objetivo de superficie que el fuente deja a continuación. |
| `11-attack-surface-weak-points.md` | Seis pasos de un flujo de reconocimiento y las notas de OPSEC. Es corto porque el fuente trae poco entre un objetivo y el siguiente. |
| `12-persisting-recon-in-vector-db.md` | Indexar reconocimiento en FAISS o en una vector database y consultarlo después. |
| `13-what-llm-agents-are.md` | Diferencia entre un LLM y un agente, frameworks y límites. |
| `14-autonomous-agent-architecture.md` | Flujo autónomo, ReAct, varios agentes, nivel de autonomía y las compuertas que lo frenan. |
| `15-langchain-and-crewai.md` | LangChain para cadenas de tools y CrewAI para roles. |
| `16-osint-agent-hands-on.md` | Un agente de OSINT con memoria, CrewAI y embeddings. |
| `17-exploitation-agent.md` | Rol de un agente que elige el siguiente paso a partir de un CVE y del contexto del objetivo. |
| `18-output-trust-and-validation.md` | No aceptar la salida del modelo sin contrastarla. Incluye la URL de NVD para comprobar un CVE. |
| `19-guardrails-for-llm-agents.md` | Guardrails de entrada y de salida, y qué se registra cuando el agente se niega. |
| `20-lab-ai-reconnaissance.md` | Laboratorio de reconocimiento asistido. |
| `21-lab-osint-and-fingerprinting.md` | Laboratorio de OSINT y fingerprint. |
| `22-lab-poc-crafting.md` | Laboratorio de armado de PoC. El Markdown no reproduce los payloads del fuente. |
| `23-lab-langchain-red-team-workflows.md` | Laboratorio de flujos con LangChain. |
| `24-lab-blue-team-response.md` | Laboratorio de respuesta: qué detecta el blue team de la actividad anterior. |
| `25-lab-full-adversarial-chain.md` | Laboratorio de una cadena completa. |
| `26-lab-kill-chain-and-alignment.md` | Laboratorio que pide un kill chain teórico y, a la vez, prueba el alignment y los guardrails del modelo. |
| `27-evaluating-llms-and-agents.md` | Cómo se evalúa un LLM o un agente en seguridad, distinto de accuracy o F1. |
| `28-full-spectrum-offensive-simulations.md` | Simulaciones de espectro completo. |
| `29-benchmarks-for-red-team-agents.md` | Benchmarks para comparar agentes. |
| `30-real-world-threat-simulations.md` | Simulaciones a partir de casos reales. |
| `31-blue-team-readiness.md` | Qué tiene que tener listo el equipo defensivo antes de una simulación. |
| `32-live-red-vs-blue-simulation.md` | Simulación en vivo de los dos equipos. |
| `33-findings-to-compliance.md` | Pasar los hallazgos a informes para técnico, SOC, GRC y dirección, mapeados a NIST, ISO 27001 y MITRE ATT&CK. |

## 06 — Linux base

Carpeta: `06-linux-base/`. Solo está `demo.txt`, en inglés, con las 12 lecciones juntas.

| Lección en el `demo.txt` | Qué desarrolla |
| --- | --- |
| 01 — Introduction to Linux and Kali 2026 Setup | Instalación y primer arranque de Kali 2026. |
| 02 — Linux Filesystem Hierarchy and Navigation | Jerarquía del sistema de archivos y cómo moverse. |
| 03 — Users, Groups, and the Permission Model | Usuarios, grupos y permisos. |
| 04 — Process Management and Job Control | Procesos y control de jobs. |
| 05 — Shell Mastery: Bash Internals and Power Usage | Bash por dentro. |
| 06 — Text Processing Pipeline | Tuberías de texto. |
| 07 — Networking from the Command Line | Red desde la línea de comandos. |
| 08 — Package Management and Tool Installation | Paquetes e instalación de herramientas. |
| 09 — Service Management and Persistence | Servicios y persistencia. |
| 10 — Linux Security Fundamentals | Fundamentos de seguridad en Linux. |
| 11 — Scripting Automation for Pentesters | Scripts de automatización. |
| 12 — Lab Setup: Building Your Attack Platform | Laboratorio de la plataforma. |

## 07 — Introducción a agentes

Carpeta: `07-introduccion-agentes/`.

Los capítulos 01 a 04 salen del libro *Agentes de IA con Python: guía práctica para diseñar, construir y desplegar sistemas agénticos*. El código de esos capítulos está en <https://github.com/studio35post/agentes-ia-python>. Antes de ejecutarlo: entorno virtual del capítulo 02, claves en `.env`, y `.env` fuera de Git.

Los otros dos Markdown no son capítulos del libro. `fundamentos-LLM.md` resume las lecciones de Red Team Leaders sobre IA, machine learning, LLM, RAG, copilotos y agentes, con la marca de la lección de origen. `introduccion-LLM-seguridad.md` es una guía de seguridad de LLM: arquitectura, OWASP Top 10 for LLM Applications 2026, prompt injection, gobernanza, privacidad y MCP. Esa guía vive solo en este archivo. El resto del repositorio no repite esa lista.

| Archivo | Qué desarrolla |
| --- | --- |
| `01-que-es-un-agente.md` | Del chat que responde al sistema que actúa. |
| `02-entorno-de-trabajo.md` | Entorno de Python, SDK de Anthropic, OpenAI y Google, y OpenRouter. |
| `02-entorno-paso-a-paso-rapido.md` | La versión corta del entorno. |
| `03-python-minimo-para-agentes.md` | El Python mínimo para seguir los capítulos. |
| `04-primera-conversacion.md` | La primera llamada a Claude, GPT y Gemini desde código. |
| `fundamentos-LLM.md` | Resumen transversal de las lecciones de LLM, con la referencia a la lección de origen. |
| `introduccion-LLM-seguridad.md` | Guía de seguridad de LLM y de la OWASP Top 10 for LLM Applications 2026. |
| `leccion-01-linux-kali-2026-github-dark.pdf` | PDF de la lección 01 de Linux, Kali 2026 y GitHub. |
| `requirements.txt` | Dependencias de los capítulos de agentes. |
