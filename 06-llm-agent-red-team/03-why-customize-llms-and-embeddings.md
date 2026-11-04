# Objetivo 03 — Why Customize LLMs and Embeddings

Darte una base sobre por qué y cómo conviene personalizar los large language models (LLMs), los sistemas de embedding y otros componentes de IA para ciberseguridad ofensiva, con énfasis en automatización del reconocimiento, análisis de vulnerabilidades y apoyo a la explotación.

## 3.1.1 Motivo para una IA a medida en seguridad ofensiva

El pentesting moderno y las operaciones de red team manejan grandes volúmenes de datos no estructurados, análisis repetitivo y un panorama de amenazas que cambia rápido. Los LLMs de propósito general no están optimizados para:

- Parsear e interpretar salidas de tools de reconocimiento (por ejemplo, nmap, subfinder, naabu)
- Sugerir vectores de explotación válidos a partir de datos en vivo
- Mapear TTPs (tácticas, técnicas y procedimientos) a vulnerabilidades en tiempo real
- Generar payloads o scripts ajustados a los entornos descubiertos
- Hacer análisis OSINT contextual y persistente entre activos

Personalizar los modelos para estos flujos ofensivos mejora la velocidad operativa, la precisión y la capacidad de adaptación.

## 3.1.2 Casos de uso ofensivos que habilitan los modelos de IA a medida

| Categoría | Ejemplo de tarea |
| --- | --- |
| Automatización del reconocimiento | Extraer metadatos del objetivo desde DNS pasivo, Shodan, WHOIS o GitHub |
| Correlación OSINT | Relacionar dominios con credenciales filtradas e infraestructura mediante embeddings |
| Recomendación de CVEs | Sugerir vulnerabilidades de alto riesgo según las versiones de software detectadas |
| Generación de payloads | Crear o modificar shellcodes y scripts de loader mediante cadenas de prompts ofensivos |
| Agents LLM | Construir asistentes de red team que se adapten a salidas de escaneo o respondan por CLI |
| Ajuste de malware | Ajustar stagers y loaders según telemetría del endpoint o perfiles de EDR |

## 3.1.3 Tipos de modelos para personalizar

| Tipo de modelo | Descripción |
| --- | --- |
| LLMs | Generan texto, scripts, instrucciones y cadenas de ataque |
| Generadores de código | Modelos especializados en automatización de exploits (por ejemplo, StarCoder, CodeLLaMA) |
| Modelos de embedding | Búsqueda semántica vectorial sobre datos de ataque (por ejemplo, nomic-embed, e5) |
| Clasificadores | Detectores binarios o multiclase de viabilidad de un exploit, de intención o de uso de una tool |
| Modelos multimodales | Manejan entradas de imagen y texto (por ejemplo, análisis de capturas en plantillas de phishing) |

## 3.1.4 Objetivos de la personalización

- Alinear el comportamiento del modelo con flujos reales de un atacante (por ejemplo, MITRE ATT&CK)
- Integrar memoria contextual para enlazar el reconocimiento con rutas de explotación
- Reducir la dependencia de APIs externas que bloquean consultas ofensivas
- Permitir inferencia offline, reproducibilidad y pipelines encadenables

## 3.1.5 Panorama de técnicas de personalización

| Técnica | Resumen |
| --- | --- |
| Prompt engineering | Diseño manual de la formulación de la entrada para guiar el comportamiento del modelo |
| Instruction tuning | Entrenamiento con prompts de tarea estructurados para acciones ofensivas repetibles |
| Fine-tuning | Modificar los pesos del modelo sobre corpus concretos (por ejemplo, exploits, CVEs) |
| Entrenamiento de adapters | Insertar módulos ligeros LoRA/QLoRA en LLMs base |
| Pipelines de embedding | Indexar resultados de reconocimiento y CVEs para consultas vectoriales (FAISS, Weaviate) |

## 3.1.6 Fuentes de entrada para contexto ofensivo

| Fuente de entrada | Uso en la personalización |
| --- | --- |
| Bases de CVE y de exploits | Alimentar la memoria del modelo con primitivas de ataque (Exploit-DB, NVD, PacketStorm) |
| Informes de red team | Modelar cadenas de ataque que funcionaron (cadenas de TTPs) |
| Salidas de tools de reconocimiento | Servir como entrada contextual (nmap, httpx, amass) |
| Repositorios OSINT | Entrenar sistemas de correlación de entidades (correos, dominios, volcados públicos) |
| Matriz MITRE ATT&CK | Anotar y entrenar agents conscientes de TTPs |

## 3.1.7 Beneficios y limitaciones

**Beneficios:**

- Mayor precisión en las sugerencias de exploits
- Automatización del flujo desde el reconocimiento hasta la post-explotación
- Capacidad de simular de forma dinámica la toma de decisiones de un atacante
- El uso de un modelo local permite operaciones en air-gap

**Limitaciones:**

- Sesgo de los datos o alucinación del modelo en tareas críticas de seguridad
- Los LLMs pueden generar código inseguro o inválido
- Coste de mantenimiento para actualizar y reentrenar el modelo
- Hace falta conocimiento de dominio para hacer fine-tuning y validar las salidas

## 3.1.8 Escenario de ejemplo

- **Objetivo:** servidor web con los puertos 80 y 8080 abiertos
- **Entrada:** un escaneo de WhatWeb revela Apache 2.4.49
- **Prompt al modelo a medida:** «Enumera las CVEs probables de Apache 2.4.49 y los métodos de explotación asociados.»
- **Salida esperada en el fuente:** identifica CVE-2021-41773 (path traversal, RCE). El fuente también espera un PoC en Python y payloads sugeridos; esa parte no se reproduce aquí.

## 3.1.9 Resumen

Personalizar modelos de IA permite a quien opera un red team superar los límites de las tools de propósito general, automatizar el bucle descubrimiento-exploit y conservar el control de los flujos ofensivos sin las restricciones éticas o funcionales que imponen los sistemas de API cerrados. Esto sienta la base para construir sistemas de IA ofensiva ajustados a tus modelos de amenaza, tus conjuntos de datos y tus tools.
