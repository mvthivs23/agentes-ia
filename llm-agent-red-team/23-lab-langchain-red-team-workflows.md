# Objetivo 23 — Lab: LangChain Red Team Workflows

Este lab te guía en la creación de flujos de red team automatizados con agents de LangChain. Estos flujos simulan tareas adversariales como reconocimiento, análisis de vulnerabilidades, recolección de credenciales y generación de payloads, usando agents modulares basados en LLM que razonan, ejecutan tools y deciden a partir de salidas intermedias.

Este módulo muestra cómo LangChain permite encadenar con flexibilidad tools, prompts y acciones en un pipeline cohesivo de automatización de red team.

## 7.4.1 Alcance de la actividad

Vas a:

- Construir agents de LangChain a medida que representen roles de red team (p. ej., agent de recon, agent de exploit)
- Integrar tools como whois, nmap, subfinder, whatweb y gitleaks
- Usar LLM para razonamiento en lenguaje natural, planificación y resumen
- Encadenar salidas entre agents para simular una operación de red team de varias fases

## 7.4.2 Requisitos

| Tool | Propósito |
| --- | --- |
| LangChain | Framework de agents LLM |
| OpenAI/GPT4All/Claude/modelo local | Backend de LLM |
| Tools de subprocess: nmap, subfinder, curl | Reconocimiento |
| Librerías de Python: requests, json, os, subprocess | Automatización |
| VectorDB (opcional): Chroma, FAISS | Base de conocimiento o memoria |
| Streamlit o Flask (opcional) | GUI para la interfaz de mando y control |

## 7.4.3 Arquitectura de agents

**Agent 1: ReconAgent**

- Entrada: dominio o IP objetivo
- Tools: subfinder, whois, nmap, whatweb
- Salida: IP, puertos abiertos, tech stack, servicios

**Agent 2: VulnAgent**

- Entrada: datos de recon
- Tools: Nuclei, consultas a bases de CVE
- Salida: vulnerabilidades conocidas con referencias a PoC

**Agent 3: CredentialHunter**

- Entrada: dominio o nombre de la empresa
- Tools: búsqueda en GitHub, gitleaks, Google Dorks
- Salida: credenciales, claves de API, secretos expuestos

**Agent 4: ExploitGenerator**

- Entrada: CVE o resumen de la vulnerabilidad
- Tarea: generar un payload PoC simulado y no malicioso
- Salida: código validado o descripción del vector de ataque

Los agents se comunican mediante cadenas de memoria, contexto compartido o un SequentialAgentExecutor de LangChain.

## 7.4.4 Ejemplo de implementación (tool + agent de LangChain)

```python
from langchain.agents import Tool, initialize_agent
from langchain.llms import OpenAI
from langchain.tools import PythonREPLTool

tools = [
    Tool(name="Recon Tool", func=lambda x: os.popen(f"subfinder -d {x}").read(), description="Enumera subdominios"),
    Tool(name="Scan Tool", func=lambda x: os.popen(f"nmap -T4 {x}").read(), description="Escanea puertos y servicios"),
    PythonREPLTool()
]

llm = OpenAI()
agent = initialize_agent(tools, llm, agent_type="zero-shot-react-description")

agent.run("Haz recon sobre acme-corp.com y encuentra servicios abiertos.")
```

## 7.4.5 Planificación basada en prompts por parte de los agents

```text
Eres un agent de recon adversarial. Objetivo: acme-corp.com.

1. Usa subfinder para encontrar subdominios.
2. Escanea los subdominios en busca de puertos abiertos (nmap).
3. Identifica tecnologías con whatweb.
4. Resume la superficie de ataque y sugiere vulnerabilidades.
```

## 7.4.6 Esquema de salida (informe estructurado)

```json
{
  "target": "acme-corp.com",
  "subdomains": ["vpn.acme-corp.com", "mail.acme-corp.com"],
  "open_ports": {
    "vpn.acme-corp.com": [443, 80],
    "mail.acme-corp.com": [25, 587]
  },
  "tech_stack": {
    "vpn.acme-corp.com": ["nginx", "OpenVPN"],
    "mail.acme-corp.com": ["Postfix", "Dovecot"]
  },
  "vulnerabilities": [
    {
      "cve": "CVE-2022-12345",
      "affected_service": "nginx",
      "description": "Path traversal vulnerability",
      "risk": "High"
    }
  ]
}
```

## 7.4.7 Extensiones avanzadas

- Usa LangGraph para construir simulaciones de red team multiagente asíncronas
- Incorpora memoria en los agents para seguimiento a largo plazo
- Construye un dashboard GUI con logs de tareas y decisiones ramificadas
- Guarda todas las salidas en una base vectorial para recuperación semántica

## 7.4.8 Criterios de éxito

- Automatización funcional basada en LangChain con al menos 2 agents distintos
- Integración de LLM para tareas de planificación o análisis
- Salida de recon y de descubrimiento de vulnerabilidades registrada en formato estructurado
- Documentación y justificación del rol de cada agent

## 7.4.9 Precaución

- Prueba siempre en entornos de lab aislados o en sistemas emulados
- No apuntes a infraestructura del mundo real sin permiso explícito
- Define los prompts con claridad para evitar que se genere comportamiento fuera de control
