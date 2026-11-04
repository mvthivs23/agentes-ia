# Objetivo 24 — Lab: Blue Team Response

Este lab muestra cómo simular una respuesta de blue team con agents impulsados por LLM. Vas a desplegar un entorno honeypot sencillo en el que cualquier interacción dispara un bucle de feedback basado en LLM que analiza, clasifica y responde a la actividad sospechosa.

La meta es introducir el razonamiento defensivo con LLM, de modo que puedas generar alertas automáticas, recomendar playbooks y aplicar tácticas de decepción activa con agents de IA.

## 7.5.1 Alcance de la actividad

- Despliega un honeypot (p. ej., servidor web, SSH, SMB)
- Vigila las interacciones con logging simple o captura de paquetes
- Usa un agent LLM para analizar los logs en tiempo real y aportar:
  - Clasificación del incidente (p. ej., port scan, fuerza bruta de autenticación)
  - Respuesta o acción de contención sugerida
  - Generación simulada de alerta o informe
  - Opcional: disparar respuestas de decepción (p. ej., credenciales falsas, archivo señuelo)

## 7.5.2 Requisitos

| Tool | Propósito |
| --- | --- |
| Cowrie, Honeyd o un honeypot Flask propio | Capturar la entrada maliciosa |
| Python | Monitoreo de logs y scripting del pipeline |
| OpenAI, Claude, GPT4All o LLaMA.cpp | LLM para clasificación y razonamiento |
| LangChain o CrewAI (opcional) | Encadenar tareas del agent |
| Streamlit, Flask (opcional) | GUI para alertas en vivo |

## 7.5.3 Preparación del entorno

- Despliega un honeypot sencillo (p. ej., SSH o una interfaz web)
- Registra las peticiones entrantes o los comandos de shell
- Usa watchdog o un monitor de archivos para reenviar los logs al agent LLM
- Parsea las líneas de log hacia la estructura del prompt
- El LLM entrega la clasificación y la respuesta recomendada

## 7.5.4 Prompt y respuesta de ejemplo

**Plantilla de prompt (del log al razonamiento)**

```text
Eres un analista de blue team. Analiza esta actividad y clasifícala:

Entrada de log: [2025-06-29 14:23:12] Failed SSH login from 192.168.56.101, user=root, 5 attempts in 30 seconds.

Indica:
1. Clasificación
2. Nivel de severidad
3. Respuesta sugerida
```

**Salida esperada del LLM**

```json
{
  "classification": "Brute-force login attempt",
  "severity": "Medium",
  "suggested_response": "Block IP 192.168.56.101 at firewall, notify security team"
}
```

## 7.5.5 Diseño del bucle de feedback

Puedes usar un agent de LangChain con estos tools:

- **LogParserTool:** parsea los logs en entradas limpias
- **ThreatClassifierLLM:** determina el tipo de amenaza
- **ResponseRecommender:** sugiere contención
- **AlertFormatter:** genera el informe

Cadena del agent:

```text
Log → Parser → Classifier → Response → Alert Generator
```

## 7.5.6 Decepción y respuesta activa (opcional)

Si quieres, puedes reforzar el bucle con:

- Inyección de datos falsos (credenciales o documentos señuelo)
- Respuesta por túnel inverso (interacción simulada con una honeynet)
- Clasificación de trampas de comportamiento (p. ej., marcadores de exfiltración, disparadores de beacon)

**Prompt para el LLM:**

```text
Genera un banner SSH falso para confundir a los scanners automatizados:

System: Ubuntu 16.04 LTS
```

**Salida del LLM:**

```text
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)
Your activity is monitored. All connections are logged.
```

## 7.5.7 Criterios de evaluación

- Clasificación precisa de los tipos de ataque por parte del LLM
- Sugerencia de respuesta coherente (alineada con un humano)
- Generación correcta de alerta o informe a partir de logs en crudo
- Opcional: uso de una respuesta de decepción

## 7.5.8 Extensiones sugeridas

- Usa tshark o tcpdump para analizar tráfico a nivel de red y reenviar resúmenes al agent
- Añade memoria temporal para seguir patrones que se repiten
- Construye una UI web en tiempo real que muestre:
  - Ataques detectados
  - Camino de decisión del LLM
  - Playbook sugerido
  - Puntuación de confianza
