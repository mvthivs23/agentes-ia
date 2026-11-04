# Objetivo 21 — Lab: OSINT and Fingerprinting

En este lab vas a reforzar las operaciones de reconocimiento usando agents de IA para automatizar la inteligencia de fuentes abiertas (OSINT) y el fingerprinting de la infraestructura objetivo. Estas tareas aumentadas con IA incluyen perfilado de empleados, huella social, extracción de metadatos de dominio, parseo de documentos expuestos y fingerprinting de servicios.

Esta simulación refleja lo que los red teams ofensivos modernos y los actores APT pueden aprovechar durante la fase temprana de reconocimiento y preexplotación.

## 7.2.1 Alcance de la actividad

Dado un dominio o el nombre de una empresa objetivo (p. ej., acme-corp.com), vas a:

- Extraer datos OSINT relevantes (nombres de empleados, correos, actividad social)
- Descubrir tecnologías y servicios en uso
- Identificar buckets, documentos o metadatos expuestos
- Apoyarte en LLM para correlacionar, resumir y contextualizar los hallazgos

## 7.2.2 Tools necesarios

| Tool | Propósito |
| --- | --- |
| LangChain o CrewAI | Orquestación de agents LLM |
| Hunter.io o EmailHunter | Recolección de correos |
| GitHub Search API | Minería de repositorios |
| Google Dorking (vía Serper/Google API) | Consultas de búsqueda dirigidas |
| WhatWeb, Wappalyzer, BuiltWith | Fingerprinting del tech stack |
| DocMetadata Extractor (propio, o pdf-metadata, exiftool) | Información de documentos |
| BeautifulSoup + requests | Parseo de HTML y scraping |

## 7.2.3 Prompt de ejemplo de un agent de IA para enumeración OSINT

```python
agent.run("""
Eres un recolector de inteligencia OSINT.
Objetivo: acme-corp.com
1. Identifica empleados clave y roles en LinkedIn, Twitter y GitHub.
2. Busca correos públicos con Hunter.io o Google dorks.
3. Encuentra repos de GitHub que mencionen 'acme' o tools internos.
4. Identifica el tech stack y la infraestructura expuesta.
Responde en formato estructurado, con fuentes.
""")
```

## 7.2.4 Ejemplos de Google Dorks (se pueden automatizar)

```text
site:github.com "acme-corp" language:Python
site:pastebin.com "acme-corp.com" password
site:docs.google.com "confidential" AND "acme"
site:linkedin.com/in "acme corp" security engineer
```

Alimenta los resultados a un reranker basado en embeddings o a un LLM de resumen para priorizar la señal sobre el ruido.

## 7.2.5 Fingerprinting de servicios mediante tools del agent

Usa escaneo pasivo y fingerprinting basado en web:

```bash
whatweb -v https://vpn.acme-corp.com
```

Pasa la salida HTML o las cabeceras a un modelo de clasificación o a un agent reforzado con prompt:

```python
agent.run("""
Clasifica esta respuesta HTTP e identifica las tecnologías probables y el software stack:
<Inserta aquí las cabeceras HTTP o el HTML>
""")
```

## 7.2.6 Esquema de salida (ejemplo)

```json
{
  "company": "acme-corp.com",
  "employees": [
    {"name": "Alice Smith", "role": "CTO", "source": "LinkedIn"},
    {"name": "Bob Chen", "role": "DevOps", "source": "GitHub"}
  ],
  "emails": [
    "alice.smith@acme-corp.com",
    "recruiting@acme-corp.com"
  ],
  "technologies": {
    "vpn.acme-corp.com": ["OpenVPN", "nginx", "Ubuntu 20.04"],
    "www.acme-corp.com": ["ReactJS", "Cloudflare CDN"]
  },
  "documents_found": [
    {
      "title": "Internal roadmap.pdf",
      "author": "Alice Smith",
      "created": "2023-04-20",
      "source": "Google Drive"
    }
  ]
}
```

## 7.2.7 Extensión de la cadena de agents (flujo LangChain)

Cadena de agents de ejemplo:

- **SearchAgent:** Google dorks + perfilado social
- **ExtractorAgent:** parseo por regex de correos, nombres y documentos
- **ClassifierAgent:** clasifica objetivos según la superficie de ataque
- **SummarizerAgent:** entrega JSON limpio y etiquetas de riesgo

Cada agent pasa el resultado al siguiente usando la memoria de LangChain o callbacks encadenados.

## 7.2.8 Criterios de evaluación

- Al menos 3 categorías de OSINT recogidas (p. ej., empleados, tecnología, documentos)
- Uso de LLM para resumir o clasificar la salida
- Resultados mostrados en un formato estructurado y explicable
- Verificación manual de al menos una pista de alta confianza

## 7.2.9 Precaución y ética

- Usa siempre un objetivo de prueba o una organización aprobada para este lab
- Nunca accedas a infraestructura privada ni intentes un login no autorizado
- Valida tus fuentes de datos para evitar desinformación o alucinaciones
