# Objetivo 10 — OSINT Workflow Automation

Este módulo te muestra cómo usar agents LLM para automatizar el flujo de OSINT completo, desde la recogida de datos hasta el análisis, el enriquecimiento, el resumen e incluso la planificación táctica. Compara tres frameworks de agents destacados: LangChain, AutoGen y CrewAI, y ofrece una guía práctica para construir agents de red team autónomos o semi-autónomos.

## 4.6.1 Por qué usar agents LLM en OSINT

| Capacidad | Beneficio en OSINT ofensivo |
| --- | --- |
| Razonamiento encadenado | Lógica de varios pasos sobre datos estructurados y no estructurados |
| Delegación de tareas | Agents especializados se ocupan de distintas fases de reconocimiento |
| Memoria a largo plazo | Guardar y reutilizar conocimiento de reconocimientos anteriores |
| Colaboración multi-agent | Flujos distribuidos de escaneo, enriquecimiento y análisis |
| Acción autónoma | Escaneo, informes y alertas dirigidos por el propio agent |

## 4.6.2 Comparación de frameworks

| Característica | LangChain | AutoGen | CrewAI |
| --- | --- | --- | --- |
| Soporte de lenguaje | Python | Python | Python |
| Estilo de uso | Chains declarativas, basadas en componentes | Agents conversacionales con memoria | Enrutado de tareas por equipos |
| Soporte de memoria | Sí | Sí | Sí |
| Integración de tools | Fuerte (APIs, retrievers, bases de datos) | Moderada | Buena (vía tools y Abilities) |
| Complejidad | Moderada | Baja a moderada | Moderada |
| Caso de uso ideal | Orquestación compleja de tools | Lógica de agent al estilo chat | Flujos de reconocimiento al estilo equipo |

## 4.6.3 Arquitectura de agents (ejemplo)

### Montaje de un agent de reconocimiento con CrewAI

- **Agent 1:** enumerador de dominios (subfinder, amass)
- **Agent 2:** fingerprinter tecnológico (httpx, whatweb)
- **Agent 3:** analista de inteligencia (LLM para resumir)
- **Agent 4:** planificador de red team (genera sugerencias de ataque)

### Fragmento de código (CrewAI): colaboración entre agents

```python
from crewai import Agent, Task, Crew
from langchain.llms import OpenLLM

# Define agents
enum_agent = Agent(name="Domain Enumerator", role="OSINT Scanner", goal="Identificar todos los subdominios relacionados")
tech_agent = Agent(name="Tech Analyzer", role="Fingerprint websites", goal="Detectar tecnologías desactualizadas")
analyst_agent = Agent(name="Intel Analyst", role="Summarize threats", goal="Identificar superficies de ataque")

# Define tasks
task1 = Task(description="Ejecuta Subfinder y guarda los resultados", agent=enum_agent)
task2 = Task(description="Ejecuta HTTPX + WhatWeb", agent=tech_agent, depends_on=[task1])
task3 = Task(description="Analiza las salidas y genera objetivos", agent=analyst_agent, depends_on=[task2])

# Launch crew
crew = Crew(agents=[enum_agent, tech_agent, analyst_agent], tasks=[task1, task2, task3])
crew.kickoff()
```

## 4.6.4 Usar LangChain para tooling de LLM en reconocimiento

LangChain facilita construir pipelines como:

- Preguntar a los datos (por ejemplo, «¿Qué subdominios tienen portales de login?»)
- Generación de informes (resumir todos los objetivos WordPress desactualizados)
- Búsqueda interactiva + memoria + vector databases

### Ejemplo: consultar datos de reconocimiento embebidos con LangChain

```python
from langchain.vectorstores import FAISS
from langchain.chains import RetrievalQA
from langchain.embeddings import SentenceTransformerEmbeddings
from langchain.llms import HuggingFaceHub

embedding = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")
db = FAISS.load_local("faiss_index/recon", embedding)
qa = RetrievalQA.from_chain_type(llm=HuggingFaceHub(), retriever=db.as_retriever())

qa.run("¿Qué dominios tienen pinta de sitios de phishing?")
```

## 4.6.5 AutoGen para agents de OSINT conversacionales

AutoGen permite agents conversacionales que intercambian pasos de razonamiento y comandos.

### Flujo de ejemplo con AutoGen

- **User Proxy Agent:** recibe los comandos del usuario
- **OSINT Agent:** ejecuta consultas a APIs
- **LLM Analyst Agent:** clasifica y etiqueta el riesgo

```python
from autogen import AssistantAgent, UserProxyAgent

user = UserProxyAgent(name="RedTeam", code_execution_config=False)
osint_agent = AssistantAgent(name="OSINTBot", llm_config={"model": "gpt4all"})

user.initiate_chat(osint_agent, message="Enumera los subdominios de tesla.com y analiza las exposiciones.")
```

## 4.6.6 Integrar tools en los flujos del agent

Puedes habilitar el uso de tools mediante:

- Wrappers Tool de LangChain (por ejemplo, scripts Python `run_subfinder()`)
- Objetos Ability de CrewAI para acceder a tools externas
- La interfaz de ejecución de comandos de AutoGen

Las tools pueden incluir:

| Tool | Propósito |
| --- | --- |
| Subfinder | Enumeración de subdominios |
| Httpx | Sondear y recoger metadatos HTTP |
| WhatWeb | Fingerprinting tecnológico |
| Gitleaks | Detección de exposición de credenciales |
| Python a medida | Interactuar con DNS, Shodan, etc. |

## 4.6.7 Ejemplo de flujo en el mundo real

El cuerpo de este ejemplo (objetivo, pasos y cierre) continúa en el bloque siguiente del fuente, a partir de la línea 1293.
