# Objetivo 07 — Fine-Tuning vs Prompt Engineering

Entender los trade-offs estratégicos entre fine-tuning y prompt engineering para optimizar modelos de lenguaje en tareas de seguridad ofensiva como OSINT, descubrimiento de vulnerabilidades, ingeniería social y elaboración de exploits.

## 3.5.1 Definiciones

| Técnica | Descripción |
| --- | --- |
| Prompt engineering | El proceso de diseñar prompts de entrada para obtener la salida deseada de un LLM base, sin alterar sus pesos internos. |
| Fine-tuning | El proceso de reentrenar un LLM con datos específicos de una tarea, creando una variante especializada con mejor rendimiento en esa tarea. |

## 3.5.2 Prompt engineering para tareas de seguridad ofensiva

El prompt engineering es un enfoque ligero y barato cuando el modelo base ya es capaz.

**Casos de uso:**

- Crear flujos de OSINT: «Genera un plan de reconocimiento para el dominio target.com usando solo tools pasivas.»
- Investigación de exploits: «Lista las CVEs de Apache 2.4.52 que pueden llevar a RCE.»

El caso de uso del fuente sobre elaborar un payload de exfiltración no se reproduce.

**Técnicas:**

- **Zero-shot:** das la tarea directamente
- **Few-shot:** incluyes 1–3 ejemplos
- **Chain-of-thought:** guías al LLM a través del razonamiento
- **RAG (retrieval-augmented generation):** inyectas fragmentos de datos externos de forma dinámica

**Pros:**

- Iteración rápida
- No hace falta entrenar
- Compatible con modelos cerrados como GPT-4 y Claude

**Contras:**

- El comportamiento del modelo puede ser inconsistente
- Vulnerable a prompt injection o a deriva
- Ventana de memoria limitada (contexto de tokens)

## 3.5.3 Fine-tuning para especialización ofensiva

El fine-tuning encaja cuando quieres agents de IA ofensiva que rindan de forma consistente en entornos de red team.

**Casos de uso:**

- Encapsular conocimiento interno (PoCs, playbooks, tooling)
- Agents persistentes (diseño de C2, árboles de decisión de movimiento lateral)
- Simulación adversaria dirigida por IA

**Flujo:**

1. **Recogida del conjunto de datos.** Prepara ficheros JSONL con prompts y completaciones esperadas a partir de:
   - Informes de red team
   - Pasos de explotación de CVEs
   - Cadenas de ataque (TTPs de MITRE)
2. **Preprocesado.** Normaliza el formato, quita ruido y redacta identificadores sensibles.
3. **Selección del modelo.** Usa modelos que admitan fine-tuning:
   - Mistral-7B, LLaMA2, OpenLLaMA, Falcon, GPT-J
4. **Entrenamiento.** Usa LoRA, QLoRA o fine-tuning completo con tools como:
   - HuggingFace Trainer
   - Axolotl
   - OpenLLM
5. **Pruebas y validación.** Usa prompts de ataque reales para validar la precisión y la tasa de alucinación.

## 3.5.4 Comparación de trade-offs

| Factor | Prompt engineering | Fine-tuning |
| --- | --- | --- |
| Tiempo de puesta en marcha | Minutos | De horas a días |
| Coste | Muy bajo | Medio a alto (hace falta cómputo) |
| Adaptabilidad | Alta | Media |
| Control | Bajo | Alto |
| Riesgo de seguridad | Prompt injection | Envenenamiento del modelo |
| Uso con modelos cerrados | ✅ | ❌ (salvo que la API lo admita) |
| Mejor para | Prototipado, versatilidad | Especialización, agents de automatización |

## 3.5.5 Comparación de ejemplo: sugerencia de payload

El fuente contrasta un prompt que pide un comando concreto con la salida de un modelo que pasó por fine-tuning sobre scripts previos de la misma tarea. El comando de exfiltración no se reproduce.

La idea que sí queda es esta: el modelo con fine-tuning conoce los patrones a partir de datos previos de esa tarea y no necesita contexto adicional en el prompt. Con prompt engineering, en cambio, tienes que describir la tarea en el propio prompt.

## 3.5.6 Estrategia híbrida: prompt tuning + RAG

Combina lo mejor de los dos:

- Usa un modelo base con prompt engineering
- Inyecta, vía RAG, datos de playbooks recuperados por vectores
- Usa prompt tuning o adapters LoRA para fijar el estilo o el comportamiento

Esto encaja bien cuando tienes:

- Entornos con poco cómputo
- Desarrollo iterativo
- Agents con memoria y uso de tools

## 3.5.7 Cuándo hacer fine-tuning y cuándo prompt engineering

| Situación | Recomendación |
| --- | --- |
| Prototipado rápido de agents de OSINT | Prompt engineering |
| Integración de una base interna de exploits | Prompt + embedding |
| Agent de red team persistente | Fine-tuning |
| Personalización de un LLM offline/local | Fine-tuning o LoRA |
| Flujos con presupuesto o recursos limitados | Prompt engineering |

## 3.5.8 Tools para fine-tuning

| Tool | Caso de uso | Notas |
| --- | --- | --- |
| Axolotl | Fine-tuning con LoRA, modelos abiertos | Configuración sencilla, basado en HuggingFace |
| QLoRA | Entrenamiento cuantizado | Eficiente para modelos grandes |
| HuggingFace | Fine-tuning completo o basado en adapters | El ecosistema más grande |
| OpenLLM | Despliegue + serving | CLI + soporte de interfaz de agent |

## 3.5.9 Consideraciones de seguridad

- Los modelos con fine-tuning pueden memorizar datos sensibles.
- Prueba contra prompts adversarios (por ejemplo, bypasses de prompt injection).
- Prefiere fine-tuning on-premise cuando trabajes con TTPs o exploits de red team.

## 3.5.10 Resumen

| Idea |
| --- |
| El prompt engineering es ágil y barato, ideal para iterar |
| El fine-tuning aporta consistencia y especialización |
| El diseño híbrido de RAG + prompt es eficaz para flujos de reconocimiento |
| Elige el método según el tipo de tarea, el riesgo y el nivel de recursos |
