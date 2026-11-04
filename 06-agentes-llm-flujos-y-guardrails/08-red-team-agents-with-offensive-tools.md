# Objetivo 08 — Red Team Agents with Offensive Tools

Esta sección te muestra cómo diseñar, implementar y poner en operación agents de IA de red team que se integran con tools ofensivas habituales (por ejemplo, Nmap, Metasploit, Burp Suite, Recon-ng), combinando el razonamiento del LLM con la ejecución de tools.

## 3.6.1 Qué es un agent de IA de red team

Un agent de IA de red team es una entidad de software que:

- Usa un LLM para decidir y razonar
- Automatiza tareas de seguridad ofensiva
- Integra la ejecución de tools (por CLI, API o scripts)
- Mantiene estado y memoria (corto y largo plazo)
- Opcionalmente interactúa por GUI o por línea de comandos

Estos agents pueden:

- Hacer reconocimiento
- Sugerir y ejecutar exploits
- Encadenar TTPs en simulaciones de ataque
- Adaptar la estrategia según la respuesta de las tools

## 3.6.2 Panorama de la arquitectura

```text
[LLM Core] ←→ [Prompt/Plan Generator]
     ↓                 ↑
[Tool API Layer] ←→ [Execution Engine]
     ↓                 ↑
[Output Parser] ←→ [Memory / Vector DB]
```

**Componentes:**

- **LLM Core:** decisión y razonamiento (GPT-4, Claude, OpenLLM)
- **Tool API Layer:** wrappers alrededor de tools como Nmap, Metasploit, dirsearch
- **Execution Engine:** ejecutor de shell o de subprocess de Python
- **Memoria:** vector database como Chroma o FAISS para guardar resultados anteriores
- **Output Parser:** convierte la salida de la CLI en datos estructurados para las acciones siguientes

## 3.6.3 Tipos de tarea del agent y ejemplos

| Tarea | Descripción | Instrucción de ejemplo |
| --- | --- | --- |
| Reconocimiento | Reunir información de dominio o IP con Subfinder y Nmap | «Escanea los puertos de 192.168.1.1 y encuentra los servicios abiertos» |
| Análisis de vulnerabilidades | Ejecutar plantillas de Nuclei y parsear resultados de CVE | «Enumera las vulnerabilidades de esta aplicación web» |
| Sugerencia de exploit | Cruzar CVEs con módulos de Metasploit | «Busca el módulo de Metasploit para CVE-2021-41773» |
| Manejo de payloads | Generar y codificar shellcode | El ejemplo del fuente pide un reverse shell con msfvenom. No se reproduce. |
| Ataques web | Usar tools como Burp, ffuf y sqlmap | El ejemplo del fuente pide inyección SQL contra un login. No se reproduce. |

## 3.6.4 Integrar tools externas

### A. Integración directa con la shell

Con `subprocess` en Python:

```python
import subprocess
result = subprocess.run(['nmap', '-p-', 'target.com'], capture_output=True, text=True)
```

### B. APIs de tools

- **Metasploit:** interfaz RPC (`msfrpc`)
- **Nmap:** salida XML o grepable (`-oX`, `-oG`)
- **Nuclei:** salida JSONL (`-json`)
- **Shodan/Censys:** APIs REST
- **Burp Suite:** extensiones vía Burp Extender API o BApp Store

### C. Wrappers de agents (LangChain, CrewAI, AutoGPT)

Puedes construir agents con:

- **Agents de LangChain:** las tools son funciones que el LLM invoca según sus planes
- **CrewAI:** ejecución multi-agent con definición de roles
- **Agents a medida:** construidos desde cero con OpenLLM o HuggingFace

## 3.6.5 Ejemplo: agent de descubrimiento de exploits

1. El usuario introduce: «Busca exploits para target.com».
2. El agent usa httpx y nmap para enumerar tecnología y puertos.
3. Pasa la salida a Nuclei y parsea las CVEs.
4. Busca módulos de Metasploit que encajen, vía Searchsploit o API.
5. Devuelve un plan de explotación estructurado.

**Prompt de LLM para planificar:**

```text
Eres un agent de red team. A partir de la salida de abajo, genera una lista de exploits.
Entrada:
- Puerto 80 abierto (Apache 2.4.49)
- CVE-2021-41773 detectada
```

## 3.6.6 Interpretación de la salida y bucle de feedback

- Parsea las salidas de la CLI con regex o con NLP
- Devuelve los hallazgos a la memoria del prompt
- Usa vector databases (por ejemplo, FAISS, ChromaDB) para persistir el conocimiento
- Permite refinar de forma iterativa (por ejemplo, «prueba otro método»)

El ejemplo del fuente sobre evadir un WAF no se reproduce.

## 3.6.7 Uso de tools mediante function calling (OpenAI, Claude)

Los LLMs más recientes admiten function calling, lo que permite una interacción determinista con las tools:

```json
{
  "function": "run_nmap_scan",
  "arguments": {
    "target": "192.168.0.1",
    "ports": "22,80,443"
  }
}
```

Tu backend ejecuta esas funciones y devuelve los resultados al LLM.

## 3.6.8 Buenas prácticas

- Sanea siempre las entradas de las tools para evitar inyección de comandos.
- Usa salidas estructuradas (JSON, YAML) cuando encadenes tools.
- Registra todos los pasos para poder auditarlos.
- Usa OpenLLM o algo similar para un despliegue local seguro.
- Aplica prompts por rol (por ejemplo, «Eres un agent de reconocimiento» frente a «Eres un generador de exploits»).

## 3.6.9 Consideraciones de seguridad

| Riesgo | Estrategia de mitigación |
| --- | --- |
| Mal uso de tools | Define capas estrictas de permisos de tools |
| Envenenamiento de la salida | Sanea y parsea todos los datos devueltos |
| Prompt injection | Limita la personalización de prompts que ve el usuario |
| Sobrecarga de memoria | Usa resumen o fragmentación en los almacenes vectoriales |

## 3.6.10 Resumen

| Punto clave |
| --- |
| Los agents de IA pueden automatizar el red team integrando tools |
| Cada tarea (reconocimiento, exploit, payloads) se mapea a tools externas |
| Combina el razonamiento del LLM con interacciones fiables de CLI/API |
| Usa memoria vectorial y bucles de feedback para un comportamiento adaptativo |
