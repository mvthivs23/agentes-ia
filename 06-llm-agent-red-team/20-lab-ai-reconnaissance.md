# Objetivo 20 — Lab: AI Reconnaissance

Este lab presenta técnicas de reconocimiento impulsadas por IA usando agents LLM y modelos propios. Vas a simular la fase inicial de un engagement ofensivo, apoyándote en large language models para automatizar y ampliar la recolección de inteligencia sobre objetivos, dominios e infraestructura de superficie.

## 7.1.1 Visión general del lab

**Meta:** usar IA para reunir datos de reconocimiento estructurados de una organización objetivo, entre ellos:

- Dominios y subdominios
- IP públicas y ASN
- Registros WHOIS y DNS
- Redes sociales y datos de empleados (OSINT)
- Detección del tech stack

## 7.1.2 Preparación del entorno

Tools necesarios:

- Python 3.10+
- LangChain o CrewAI
- Serper.dev o Google Search API para scraping
- API de Shodan, Hunter.io y theHarvester
- LLM local o en la nube (GPT-4, OpenLLM, LLaMA.cpp, etc.)

Instalación:

```bash
pip install langchain openai shodan requests
```

Configura tu `.env`:

```text
OPENAI_API_KEY=sk-...
SHODAN_API_KEY=...
SERPER_API_KEY=...
```

## 7.1.3 Diseño del agent (estilo LangChain)

Define un agent que reciba un dominio objetivo y haga lo siguiente:

- Consulta Google/Bing en busca de referencias recientes
- Resuelve subdominios con fuentes de certificate transparency
- Usa API de WHOIS para extraer registrador, fecha de creación y contactos
- Llama a Shodan para traer puertos y servicios expuestos
- Correlaciona los hallazgos en un resultado JSON estructurado

**Ejemplo de prompt:**

```python
agent.run(f"""
Eres un agent de recon OSINT.
Tu objetivo es: acme-corp.com
1. Encuentra subdominios.
2. Obtén información WHOIS.
3. Trae páginas indexadas recientes.
4. Consulta Shodan en busca de servicios abiertos.
Responde con los resultados en formato JSON.
""")
```

## 7.1.4 Mejora del recon basada en embeddings

Para enriquecer el reconocimiento, los embeddings vectoriales ayudan a encontrar similitudes semánticas en:

- Repos de GitHub
- Pastebins
- Datos de brechas
- Notas de prensa

Usa OpenAIEmbeddings o LlamaIndex con FAISS.

Caso de uso de ejemplo. Inserta palabras clave como:

```text
"acme internal repo", "vpn access", "admin panel", "dev credentials"
```

Compáralas con el corpus raspado de GitHub y de resultados de búsqueda para extraer pistas de alta similitud.

## 7.1.5 Flujo de trabajo de ejemplo realista

Objetivo: `acme-corp.com`

| Tarea | Salida |
| --- | --- |
| Scraping de búsqueda en Google | 10+ enlaces recientes que mencionan acme-corp.com |
| Descubrimiento de subdominios | vpn.acme-corp.com, dev.acme-corp.com |
| WHOIS | Creado en 2017, alojado en AWS, contacto oculto vía PrivacyGuard |
| Shodan | Puerto 80/443, puerto 8080 expuesto en vpn.acme-corp.com |
| OSINT semántico en GitHub | Repo: acme-devops-scripts con credenciales hardcodeadas |

## 7.1.6 Criterios de éxito

- El agent recoge al menos 3 tipos de datos de recon de forma automática
- La salida queda en JSON limpio, con contexto y referencias
- Verificación manual de al menos 1 vector descubierto (p. ej., un puerto abierto)

## 7.1.7 Formato de salida (ejemplo)

```json
{
  "target": "acme-corp.com",
  "subdomains": ["dev.acme-corp.com", "vpn.acme-corp.com"],
  "whois": {
    "created": "2017-08-14",
    "registrar": "NameCheap",
    "privacy": "enabled"
  },
  "shodan": {
    "vpn.acme-corp.com": {
      "open_ports": [80, 443, 8080],
      "technologies": ["nginx", "OpenVPN"]
    }
  },
  "recent_links": [
    "https://news.com/article?id=acme-corp-breach",
    "https://pastebin.com/leak_acme"
  ]
}
```

## 7.1.8 Notas sobre ética y aspectos legales

- Define siempre el alcance del objetivo y obtén permiso por escrito (simulado o real)
- No uses los datos descubiertos fuera del contexto del lab
- Evita raspar o sondear infraestructura del mundo real salvo que esté autorizado de forma explícita

## 7.1.9 Extensiones (opcionales)

- Integra subfinder y amass como tools invocables por el agent
- Visualiza los datos con networkx o Graphviz
- Guarda los datos de recon en una base de conocimiento (p. ej., ChromaDB)
