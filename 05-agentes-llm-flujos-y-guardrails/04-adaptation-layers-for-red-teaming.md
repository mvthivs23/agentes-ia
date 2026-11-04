# Objetivo 04 — Adaptation Layers for Red Teaming

Darte una comprensión estructurada de las capas en las que puedes adaptar modelos de IA para red team y tareas de seguridad ofensiva, desde una configuración mínima hasta un reentrenamiento avanzado. Cada capa se describe con ejemplos ofensivos prácticos.

## 3.2.1 Panorama de las capas de personalización

| Capa | Descripción | Nivel de esfuerzo | Riesgo de mal uso | Soporte offline |
| --- | --- | --- | --- | --- |
| Prompt engineering | Redactar entradas para guiar el comportamiento del modelo | Bajo | Medio | Sí |
| Instruction tuning | Entrenar con pares prompt/respuesta etiquetados para alinear el comportamiento del modelo | Medio | Alto | Sí |
| Fine-tuning | Reentrenar los pesos del modelo con conjuntos de datos ofensivos | Alto | Alto | Sí |
| Embedding + RAG | Acoplar conocimiento externo mediante embedding y mecanismos de recuperación | Medio | Medio | Sí |

## 3.2.2 Prompt engineering para operaciones de red team

El prompt engineering es el método de personalización más rápido y de menor coste. Se centra en modificar la formulación de la entrada para orientar la salida del modelo.

Los ejemplos de prompt del fuente piden un PoC funcional, rutas de explotación a partir de un resultado de Naabu y un reverse shell. No se reproducen.

**Tools que ayudan al prompt engineering:**

- LangChain
- PromptLayer
- FlowiseAI
- LM Studio (para experimentación local)

## 3.2.3 Instruction tuning

El instruction tuning consiste en entrenar modelos para seguir tareas ofensivas concretas mediante pares de prompt y respuesta. Así el modelo puede:

- Replicar flujos típicos de red team
- Encadenar lógica de varios pasos, como «escanear → analizar → explotar → informar»

El fuente incluye, además, alinear el modelo para saltarse guardrails que restringen contenido. Esa indicación no se reproduce.

### Ejemplos de conjunto de datos de caso de uso

| Tarea | Ejemplo de entrada | Salida esperada |
| --- | --- | --- |
| Análisis de escaneo de puertos | `naabu -p 80,443 192.168.1.5` | «El objetivo tiene HTTP/HTTPS. Prueba Gobuster y Nuclei» |
| Vinculación de CVEs | Apache 2.4.49 | «CVE-2021-41773 — RCE mediante path traversal» |

La fila del fuente sobre generación de un loader que inyecta shellcode no se reproduce.

**Sugerencias de modelo:**

- OpenLLM: admite modelos fine-tuned basados en instrucciones
- LLaMA.cpp + fine-tunes al estilo Alpaca/Orca

## 3.2.4 Fine-tuning para inteligencia ofensiva

El fine-tuning consiste en modificar los pesos internos de un modelo base reentrenándolo con datos específicos de la tarea, como:

- Entradas de Exploit-DB
- PoCs de GitHub
- Informes internos de red team
- Narrativas de técnicas de MITRE ATT&CK

**Ventajas:**

- Reduce la necesidad de prompts muy detallados
- Permite flujos multimodales (por ejemplo, imagen + texto)
- Evita inyectar el mismo contexto una y otra vez

**Riesgos:**

- Puede codificar sesgo o alucinaciones
- Puede retener contenido sensible o malicioso sin una trazabilidad clara
- Es caro en cómputo (exige GPUs o servicios como Colab Pro o RunPod)

## 3.2.5 Pipelines de embedding (retrieval-augmented generation, RAG)

Los pipelines de embedding permiten que los modelos busquen en bases de conocimiento vectorizadas durante la inferencia, sin cambiar el modelo en sí.

**Componentes:**

- Modelo de embedding (por ejemplo, e5-small, nomic-embed)
- Índice vectorial (por ejemplo, FAISS, ChromaDB, Weaviate)
- Buscar e inyectar los top-k resultados en el contexto del LLM

**Ejemplos en uso ofensivo:**

- **Consulta:** «¿Qué vulnerabilidades afectan a GoAnywhere MFT?»
- **Recuperación:** descripción del exploit CVE-2023-0669

La línea de salida del fuente (pasos de explotación y consejos para evitar la detección) no se reproduce.

**Tools recomendadas:**

- LlamaIndex o LangChain para la orquestación
- FAISS + SQLite para almacenamiento local
- ChromaDB para prototipado rápido

## 3.2.6 Tabla resumen comparativa

| Técnica | Cambios en el modelo | Datos propios necesarios | Montaje rápido | Alto rendimiento | Buena para IA de red team |
| --- | --- | --- | --- | --- | --- |
| Prompt engineering | No | No | ✅ | ⚠️ Depende del modelo base | ✅ Buen punto de partida |
| Instruction tuning | Parcial | ✅ Pares prompt/respuesta | ⚠️ Moderado | ✅ | ✅ Camino recomendado |
| Fine-tuning | Sí | ✅ Conjuntos de datos completos | ❌ | ✅✅✅ | ✅ Usuarios avanzados |
| RAG + embeddings | No | ✅ Corpus externo | ✅ | ✅✅ | ✅ Consultas de alto contexto |

## 3.2.7 Ejemplo de integración: reconocimiento + RAG + LLM

**Flujo:**

1. Escanea objetivos con amass, naabu y whatweb.
2. Indexa las salidas en FAISS.
3. Consulta al LLM con: «A partir del escaneo del host, enumera la superficie de ataque y dame las plantillas de Nuclei que debo usar.»

**Salida:**

- Inventario de activos
- CVEs que encajan
- Tools y scripts sugeridos

## 3.2.8 Resumen

Cada capa de personalización cumple un papel:

- Prompt engineering para experimentar rápido
- Instruction tuning para flujos de red team específicos de una tarea
- Fine-tuning para un control más profundo del comportamiento del modelo
- Embedding + RAG para inyectar conocimiento de forma dinámica y escalable

Elegir el método depende de la madurez operativa del red team, de la capacidad de cómputo disponible y de la calidad de los datos.
