# 08 — Cuatro proyectos para presentar

Cada uno junta ejercicios de esta carpeta. No añade un scanner, un modelo ni un formato que los módulos no tengan. El informe de cierre usa las secciones que el objetivo 33 de `06-agentes-llm-flujos-y-guardrails/33-findings-to-compliance.md` ya pide: hallazgo, a qué control mapea, y la remediación. El insumo es tu JSON, no una simulación ofensiva.

## A — Gate de imagen e IaC

Piezas: archivos 02 y 04.

1. Construye `hardened-nginx:lab` y deja los `docker inspect` en el repo del lab.
2. `hadolint` y `trivy image`. El workflow falla en HIGH o CRITICAL.
3. `tfsec` o `checkov` sobre un Terraform de laboratorio. El state está en `.gitignore`.
4. El Dockerfile del servicio usa digest.
5. Una página: qué falló el gate, qué policy o qué línea del Dockerfile lo cierra.

Eso es un control de supply chain que la consultora puede poner delante de un producto con contenedores.

## B — Asistente interno, modelo local

Piezas: archivos 05 y 06.

1. `~/soc-ai/` con `phi3:mini` para clasificar y `qwen2.5-coder:7b` para el fix.
2. El corpus son tus notas de los archivos 02, 03 y 04.
3. Dos preguntas grabadas: una respondida con una cita, otra con `Not covered by the corpus`.
4. El ledger de la lección de API: modelo, tokens y la versión del prompt. Sin eso no explicas por qué el asistente dijo lo que dijo.

No ofrezcas este asistente como un agente que explota. La lección defensiva lo separa de `~/pentest-ai/` a propósito.

## C — Triage de CVE que no alucina

Piezas: archivos 06 y 07.

1. Trivy o tfsec producen un identificador.
2. `ai-cve-check.sh` lo contrasta con NVD.
3. La nota de cinco líneas solo se escribe si `totalResults` no es 0.
4. El skill tiene la description del ejercicio de la lección 01 de skills.

El valor para la consultora es el hook: un modelo no publica un CVE que la API no devolvió.

## D — Admission en el clúster de laboratorio

Piezas: archivo 03.

1. kind, el binding `oops`, la policy, el webhook que lo niega.
2. La policy de pods privilegiados, con su rechazo.
3. El informe en el formato del objetivo 33, con esas dos salidas como evidencia.

## Qué no es un proyecto de esta ruta

Un agente que genere un payload, un flujo de phishing, o un escape de contenedor. Esos laboratorios están en los Markdown originales y no se reempaquetan aquí.
