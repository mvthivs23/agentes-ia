# Objetivo 16 — OSINT Agent Hands-On

Esta sección es una guía práctica para construir un agent de OSINT funcional, impulsado por IA. El objetivo es automatizar el reconocimiento permitiendo que un agent respaldado por un large language model (LLM) recoja inteligencia —subdominios, direcciones IP, credenciales filtradas o artefactos de redes sociales— con encadenamiento de tools, lógica de decisión y memoria.

## 6.4.1 Panorama

Un agent de OSINT para seguridad ofensiva tiene que poder:

- Aceptar identificadores de objetivo (por ejemplo, dominio, correo, nombre de empresa)
- Usar tools o APIs para recoger inteligencia de fuentes abiertas
- Interpretar las salidas y refinar el flujo de reconocimiento de forma dinámica
- Registrar los resultados en un formato estructurado (JSON, Markdown o base de datos)
- Mantener memoria y contexto para no repetir acciones

## 6.4.2 Flujo de OSINT de ejemplo

**Entrada:** `acme-corp.com`

**Flujo:**

1. Ejecuta subfinder o amass para descubrir subdominios.
2. Para cada subdominio:
   - Resuelve la IP (dnsx)
   - Comprueba el estado (httpx)
   - Captura pantallas (aquatone)
3. Busca filtraciones en Pastebin, GitHub, Reddit y foros de la dark web.
4. Mézclalo todo en un informe y, si quieres, genera Markdown o HTML.

## 6.4.3 Construir el agent con LangChain

**Tools necesarias:**

```python
from langchain.agents import Tool, initialize_agent
from langchain.llms import OpenAI
import os

def subfinder(domain):
    return os.popen(f"subfinder -d {domain} -silent").read()

def httpx(url):
    return os.popen(f"httpx -silent -u {url}").read()

tools = [
    Tool(name="subfinder", func=subfinder, description="Encuentra subdominios de un dominio"),
    Tool(name="httpx", func=httpx, description="Comprueba si una URL está viva")
]

llm = OpenAI(temperature=0)

agent = initialize_agent(tools=tools, llm=llm, agent="zero-shot-react-description")
agent.run("Encuentra subdominios vivos de acme-corp.com")
```

## 6.4.4 Incorporar memoria

LangChain admite sistemas de memoria que permiten a los agents «recordar» conversaciones y acciones pasadas.

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(memory_key="chat_history")
agent_with_memory = initialize_agent(tools, llm, agent="zero-shot-react-description", memory=memory)
```

Esto permite:

- Evitar escaneos duplicados
- Seguir el alcance del reconocimiento a lo largo del tiempo
- Escribir planes de varios pasos a partir del contexto anterior

## 6.4.5 Usar CrewAI para OSINT en varias etapas

**Ejemplo de roles:**

| Nombre del agent | Descripción del rol |
| --- | --- |
| DomainHunter | Recoge subdominios |
| Responder | Resuelve IPs y ejecuta sondas |
| LeakAnalyst | Revisa la dark web y datos de brechas |
| Reporter | Agrega y da formato a los resultados |

```python
from crewai import Crew, Agent, Task

hunter = Agent(name="DomainHunter", role="Subdomain Enumerator")
leak = Agent(name="LeakAnalyst", role="Search for leaked data")

task1 = Task(agent=hunter, description="Encuentra todos los subdominios de acme-corp.com")
task2 = Task(agent=leak, description="Busca en GitHub y Pastebin secretos expuestos")

crew = Crew(agents=[hunter, leak], tasks=[task1, task2])
crew.run()
```

## 6.4.6 Mejoras de un caso real

| Tipo de mejora | Ejemplos |
| --- | --- |
| Formateadores de entrada | Limpiador de regex de dominios, normalizador de WHOIS |
| Parsers de salida | Informes en Markdown, CSV o JSON |
| Inteligencia de amenazas | Integrar APIs: VirusTotal, Shodan, Censys, AbuseIPDB |
| Visuales | Aquatone, Eyewitness, capturas, mapas de Graphviz |
| Constructor de línea de tiempo | Pon marca de tiempo a cada resultado y genera una línea de tiempo |

## 6.4.7 Contexto con embeddings

Usar embeddings (por ejemplo, `OpenAIEmbeddings` o FAISS) permite guardar hallazgos de OSINT para búsqueda por similitud y correlación de patrones.

```python
from langchain.vectorstores import FAISS
from langchain.embeddings.openai import OpenAIEmbeddings

db = FAISS.load_local("osint_vector_db", OpenAIEmbeddings())
similar_docs = db.similarity_search("subdominios relacionados con buckets S3")
```

## 6.4.8 Ejemplo de salida

```json
{
  "target": "acme-corp.com",
  "subdomains": [
    {"name": "dev.acme-corp.com", "ip": "192.168.1.100", "status": "200 OK"},
    {"name": "mail.acme-corp.com", "ip": "192.168.1.101", "status": "302 Redirect"}
  ],
  "leaks": [
    {"source": "Pastebin", "type": "API Key", "value": "AKIA123..."},
    {"source": "GitHub", "repo": "employee/acme-utils", "content": "Hardcoded secret"}
  ]
}
```

Los valores de ejemplo (`200 OK`, `302 Redirect`, `API Key`, `AKIA123...`, `Hardcoded secret`) se dejan como en el fuente: son datos de muestra, no texto de prompt.

## Resumen

Un agent de OSINT impulsado por un LLM puede transformar el reconocimiento clásico al meter toma de decisiones consciente del contexto, orquestación de varias tools y razonamiento dinámico en la fase de reconocimiento. Con LangChain o CrewAI, un red team puede automatizar la recogida, enriquecer la inteligencia y crear bots de reconocimiento adaptativos con memoria persistente y capas de análisis.
