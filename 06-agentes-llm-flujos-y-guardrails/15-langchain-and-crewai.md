# Objetivo 15 — LangChain and CrewAI

Esta sección introduce LangChain y CrewAI, dos frameworks muy usados para gestionar agents autónomos en operaciones de seguridad ofensiva. Verás sus arquitecturas, patrones de uso e implementación práctica para casos como reconocimiento, escaneo, explotación y post-explotación.

## 6.3.1 Panorama: LangChain frente a CrewAI

| Característica | LangChain | CrewAI |
| --- | --- | --- |
| Tipo | Framework modular para LLM + tools | Orquestación multi-agent |
| Estilo de diseño | ReAct (Reasoning + Action) | Colaboración basada en roles |
| Soporte de tools | Amplio (bash, Python, APIs, etc.) | Moderado (extiende las tools de LangChain) |
| Soporte de memoria | Sí | Sí |
| Caso de uso ideal | Chains basadas en tools, agents autónomos | Equipos de agents con objetivos, al estilo humano |

## 6.3.2 LangChain para agents ofensivos

LangChain permite definir un conjunto de tools y encadenar su uso mediante el razonamiento del LLM.

**Instalación:**

```bash
pip install langchain openai
```

**Montaje básico de un agent:**

```python
from langchain.agents import initialize_agent, Tool
from langchain.llms import OpenAI

def run_nmap(ip):
    return os.popen(f"nmap -T4 -Pn {ip}").read()

tools = [Tool(name="nmap", func=run_nmap, description="Ejecuta un escaneo Nmap")]

agent = initialize_agent(
    tools=tools,
    llm=OpenAI(temperature=0),
    agent="zero-shot-react-description"
)

agent.run("Escanea 192.168.1.10 y dime qué puertos están abiertos.")
```

**Funciones:**

- Admite plantillas de prompt, selección de tools y razonamiento intermedio
- Se puede conectar a APIs como Shodan, VirusTotal y Censys
- Admite vector databases, chats y agents de documentos

## 6.3.3 CrewAI para coordinación multi-agent

CrewAI extiende LangChain y ofrece una arquitectura para asignar roles y flujos de tareas entre varios agents.

**Conceptos clave:**

- **Crew:** un grupo de agents con tareas asignadas
- **Agent:** instancia de LLM con memoria, rol y función de tarea
- **Toolbox:** conjunto de acciones que la crew puede usar

**Instalación:**

```bash
pip install crewai
```

**Ejemplo básico de crew:**

```python
from crewai import Crew, Agent, Task

recon_agent = Agent(name="Recon", role="Subdomain Enumerator")
scan_agent = Agent(name="Scanner", role="Port Scanner")

recon_task = Task(agent=recon_agent, description="Enumera los subdominios de acme.com")
scan_task = Task(agent=scan_agent, description="Escanea los puertos abiertos de cada subdominio")

crew = Crew(agents=[recon_agent, scan_agent], tasks=[recon_task, scan_task])
crew.run()
```

**Flujo de caso de uso:**

1. El agent 1 recoge subdominios (subfinder).
2. El agent 2 escanea cada dominio con nmap o httpx.
3. El agent 3 hace fingerprint de tecnologías (whatweb, wappalyzer).
4. El agent 4 evalúa exploits o genera payloads.

## 6.3.4 Integración con cadenas de tools ofensivas

| Tool | Soporte en LangChain | Soporte en CrewAI |
| --- | --- | --- |
| Nmap | Sí (wrapper de CLI) | Sí |
| Subfinder / Amass | Sí | Sí |
| Nuclei / ExploitDB | Sí | Sí |
| Funciones Python a medida | Sí | Sí |

Ambos frameworks admiten:

- Capas de abstracción de tools
- Paso de contexto seguro
- Logs de auditoría y observaciones

## 6.3.5 Ventajas en operaciones ofensivas

| Capacidad | LangChain | CrewAI |
| --- | --- | --- |
| Encadenamiento de comandos | ✅ | ✅ |
| Colaboración y división del trabajo | ❌ | ✅ |
| Simulaciones multi-agent | ⚠️ (montaje manual) | ✅ (interacciones de agents integradas) |
| Planificación adaptativa | ✅ | ✅ |
| Memoria persistente entre agents | ⚠️ (memoria externa) | ✅ (memoria del agent + historial) |

## 6.3.6 Ejemplo de flujo en el mundo real

**Escenario:** una cadena completa OSINT → escaneo → exploit.

| Agent | Descripción de la tarea |
| --- | --- |
| Agent de OSINT | Recoge subdominios e IPs |
| Escáner | Sondea IPs con Nmap y detecta servicios |
| Analizador | Comprueba CVEs de los servicios detectados |
| Explotador | Obtiene PoCs de ExploitDB y genera payloads |

Esto se puede automatizar por completo con CrewAI, o ejecutarse paso a paso con agents de LangChain.

## Resumen

LangChain ofrece una base excelente para construir agents autónomos individuales con capacidad de tool-calling, mientras que CrewAI extiende ese paradigma para orquestar agents de IA ofensiva que colaboran. Dominar ambos permite a los red teams automatizar cadenas enteras de operaciones ofensivas con planificación inteligente, separación de roles e integridad de la ejecución.
