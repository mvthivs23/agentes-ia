# Objetivo 13 — What LLM Agents Are

Esta sección introduce el concepto de agents LLM y su papel al automatizar tareas de seguridad ofensiva con IA. Verás qué son los agents, en qué se diferencian de un LLM aislado, cómo interactúan con tools, memoria y funciones externas, y el valor concreto que aportan a los flujos de red team.

## 6.1.1 Qué es un agent LLM

Un agent LLM es una arquitectura dirigida por lógica que usa un modelo de lenguaje para decidir qué acciones tomar, en qué orden, con un conjunto de tools o APIs, y en respuesta a objetivos concretos.

**Componentes clave:**

- **Planner / Controller:** el LLM toma decisiones («debería llamar a una tool», «necesito hacer otra pregunta»).
- **Tools:** funciones, scripts o APIs externas que el agent puede invocar (por ejemplo, `shodan_search`, `nmap_scan`, `exploit_runner`).
- **Memoria (opcional):** permite al agent guardar contexto o pasos pasados (se usa en investigaciones de varios pasos).
- **Entorno:** el conjunto de restricciones y datos en el que opera el agent (puede ser local o cloud).

## 6.1.2 Diferencia entre LLM y agent

| Aspecto | Solo LLM | Agent LLM |
| --- | --- | --- |
| Propósito | Genera una salida para un único prompt | Interactúa con tools y con tareas de varios pasos |
| Uso de tools | Sin integración nativa | Tool-calling explícito mediante planificación |
| Consciencia del contexto | Limitada a la entrada del prompt | Usa memoria y pasos anteriores |
| Autonomía | Reactivo | Proactivo, orientado a un objetivo |
| Ejemplo | «Explica CVE-2023-36884» | «Escanea un objetivo, detecta CVEs y recomienda un exploit» |

## 6.1.3 Frameworks de agents LLM

### Agents de LangChain

- Ofrece agents modulares con integración de tools
- Admite tipos de agent ReAct y Zero-Shot
- Tools integradas: Python, requests, bash, shell, E/S de ficheros

```python
from langchain.agents import initialize_agent, Tool
from langchain.llms import OpenAI

tools = [Tool(name="shodan_lookup", func=shodan_lookup_function)]
agent = initialize_agent(tools, OpenAI(), agent="zero-shot-react-description")
```

### CrewAI

- Centrado en la colaboración multi-agent
- Los agents pueden tener roles, memoria y colaboración dinámica

### OpenAgents

- Hub de agents basado en plugins, con GUI
- Pensado para desplegar rápido agents específicos de una tarea

## 6.1.4 Casos de uso en seguridad ofensiva

| Nombre del agent | Ejemplo de funcionalidad |
| --- | --- |
| OSINT-Agent | Usa las tools subfinder, whois y github_dork |
| Exploit-Agent | Toma CVEs, las contrasta con ExploitDB y construye payloads |
| Infrastructure-Agent | Monta infraestructura de phishing, configura DNS/SSL y monitoriza el C2 |
| Simulation-Agent | Automatiza cadenas de ataque para formación o demo |

## 6.1.5 Ejemplo de integración de una tool

Declaración de tool de ejemplo:

```python
def shodan_lookup(ip: str):
    # Shodan API integration logic
    return shodan_client.host(ip)

tools = [Tool(name="shodan_lookup", func=shodan_lookup, description="Busca IPs en Shodan.")]
```

El agent decide cuándo llamar a `shodan_lookup("1.1.1.1")` según el contexto del prompt.

## 6.1.6 Beneficios para el red team

- Automatización de flujos repetitivos
- Comportamiento orientado a un objetivo (por ejemplo, encontrar una vía de entrada al objetivo)
- Cadenas que se documentan solas (prompt + log de tools = informe de red team)
- Interoperabilidad con tools de escaneo (Nmap, Nuclei, Amass)

## 6.1.7 Retos y limitaciones

- Riesgo de alucinación cuando usas cadenas de tools complejas
- Prompt injection o abuso de la memoria del agent (hace falta saneado)
- Llamar de más a las tools aumenta la latencia y el coste
- Preocupaciones de seguridad si el agent queda expuesto por API o por navegador

## Resumen

Los agents LLM representan un paradigma nuevo para automatizar operaciones de seguridad ofensiva. Al combinar razonamiento, memoria y tools externas, los agents permiten que el red team se centre en la estrategia, mientras el agent se ocupa del pipeline de ejecución. Entender cómo deciden los agents e integrar las tools con seguridad es la base de una ciberseguridad avanzada dirigida por IA.
