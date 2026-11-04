# Objetivo 01 — LLM Platforms and Frameworks

Ofrecerte una visión comparativa de las plataformas de LLM y los frameworks de IA más usados en seguridad ofensiva. Incluye diferencias de arquitectura del modelo, capacidad offline, usabilidad en laboratorios de red team y compatibilidad real para crear agents de IA adversarios.

## 2.3.1 Tabla general: herramientas clave de IA ofensiva

| Herramienta / plataforma | Tipo de modelo | Soporte offline | Acceso a plugin / API | Alojamiento de modelo propio | Casos de uso ideales |
| --- | --- | --- | --- | --- | --- |
| ChatGPT (OpenAI) | LLM (GPT-4-turbo) | ❌ Solo cloud | ✅ Sí | ❌ Modelo cerrado | Encadenamiento de prompts, planificación de exploits, automatización de reconocimiento |
| Claude (Anthropic) | LLM (Claude 3) | ❌ Solo cloud | ✅ Sí (vía API) | ❌ Modelo cerrado | Ingeniería social, redacción de phishing, lógica ética |
| GPT4All | LLM | ✅ Totalmente offline | ❌ Limitado | ✅ Sí (carga cualquier `.bin`) | Red team en air-gap, pruebas de laboratorio seguras |
| OpenLLM (BentoML) | LLM + orquestación de agents | ✅ Completo | ✅ Sí | ✅ Sí | Tools a medida, encadenamiento de modelos, simulaciones con API |
| LM Studio | Interfaz de usuario para LLM | ✅ Sí | ❌ UI limitada | ✅ (GGUF/GGML) | Pruebas locales ligeras, pruebas de jailbreak |
| LLaMA.cpp | LLM cuantizado | ✅ Sí | ❌ No | ✅ Muy portable | Tools embebidas, integración con agents en vivo, poco consumo |
| LocalAI | Orquestación + LLM | ✅ Sí | ✅ Sí | ✅ Sí | Encadenamiento multi-modelo, alojamiento de agents ofensivos |

## 2.3.2 Desglose de funciones por herramienta

### ChatGPT (GPT-4-turbo)

- **Fortalezas:** razonamiento avanzado, generación de código sólida, manejo de contexto.
- **Limitaciones:** solo cloud; no hay soporte de plugins propios fuera del ecosistema de ChatGPT.
- **Mejor para:** generación de exploits complejos y planificación de ataques en varios pasos.
- **Ejemplo de uso:** «Dado este output de Metasploit, ¿qué acciones de post-explotación puedo automatizar a continuación?»

### Claude 3

- **Fortalezas:** ventana de contexto larga (hasta 200k+), alineado con seguridad, muy bueno parseando estructura.
- **Limitaciones:** no hay versión self-hosted; las salidas son más verbosas.
- **Mejor para:** generación de correos de phishing y redacción estructurada de payloads.
- **Ventaja propia:** alta coherencia lingüística para simulaciones de suplantación.

### GPT4All

- **Fortalezas:** totalmente offline, biblioteca grande de modelos, soporte de pesos cuantizados.
- **Limitaciones:** modelos más pequeños (GPT-J, Mistral, etc.); inferencia más lenta que los modelos cloud.
- **Mejor para:** laboratorios de red team, simulaciones encubiertas, pruebas de jailbreak.
- **Puesta en marcha:** necesitas modelos `.bin` e inferencia por GPU o CPU con el backend llama.cpp.

### OpenLLM (BentoML)

- **Fortalezas:** despliegue de modelos full-stack, orquestación de agents, endpoints de API.
- **Limitaciones:** exige montaje y despliegues con Docker o Bento.
- **Mejor para:** construir agents ofensivos y encadenar modelos y tools.
- **Ejemplo de caso de uso:** servir un LLM local que, mediante un prompt, dispare nmap, whois o searchsploit.

### LM Studio

- **Fortalezas:** interfaz de usuario, pruebas de prompts sencillas, carga de modelos cuantizados.
- **Limitaciones:** sin API, solo GUI, ecosistema de plugins limitado.
- **Mejor para:** probar jailbreaks y simular prompts de un atacante.
- **Ejemplo de uso:** cargar un GPT-J de 4 bits y probar cadenas de prompts de escalada de privilegios.

### LLaMA.cpp

- **Fortalezas:** portabilidad extrema (puede correr en una Raspberry Pi), modelos cuantizados a 4 bits.
- **Limitaciones:** sin API nativa; la carga del modelo es manual.
- **Mejor para:** agents de IA en el propio dispositivo y lógica ofensiva portable.
- **Contexto de uso:** embebido en payloads o en agents de post-explotación para apoyar el C2 mediante inferencia local.

### LocalAI

- **Fortalezas:** API self-hosted compatible con OpenAI, soporte multi-modelo, proxy de API de LLM rápido.
- **Limitaciones:** configuración compleja; menos soporte para los modelos más recientes.
- **Mejor para:** alojar agents a medida en plataformas de red team (como C2).
- **Función destacada:** reemplazo directo de la API de OpenAI para tools que esperan la interfaz de GPT-3.5/4.

## 2.3.3 Comparación de tools por escenario ofensivo

| Escenario | Mejor tool o tools |
| --- | --- |
| Laboratorio offline de desarrollo de malware | GPT4All + LM Studio |
| Simulación de prompt injection (jailbreaks) | LM Studio + LLaMA.cpp |
| Phishing en vivo y generación de payloads | Claude o ChatGPT vía API |
| Agent asistente de OSINT offline | LocalAI + modelo de embeddings (por ejemplo, E5-base) |
| Integración de IA a medida en un framework de red team | OpenLLM + SDK de BentoML |
| C2 portable con un agent de IA embebido | LLaMA.cpp + ejecutor de shell |

## 2.3.4 Resumen de criterios de evaluación

| Criterio | GPT4All | Claude | ChatGPT | LLaMA.cpp | OpenLLM | LM Studio | LocalAI |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Capaz de operar offline | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Código abierto | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Soporte multi-modelo | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Integración por API | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ |
| Facilidad de despliegue | Media | Fácil | Fácil | Fácil | Media | Fácil | Media |
| Sistema de plugins / agents | Limitado | Sí | Sí | No | Sí | No | Sí |

## 2.3.5 Consideraciones éticas y aislamiento del laboratorio

- Usa siempre modelos locales cuando pruebes payloads ofensivos, para evitar que una API registre o filtre esos datos.
- Prefiere runtimes de LLM en contenedores (Docker/OpenLLM) cuando simules ataques reales.
- No dependas de LLMs cloud (por ejemplo, ChatGPT) para ejecutar payloads o hacer reconocimiento dinámico.
- Valida la tasa de alucinación del modelo antes de usar sus salidas en la generación de exploits.

## 2.3.6 Resumen

Cada tool ofrece un equilibrio distinto entre rendimiento, privacidad, flexibilidad y riesgo. En seguridad ofensiva, GPT4All, OpenLLM y LocalAI destacan como opciones que priorizan la privacidad y el uso local, mientras que Claude y ChatGPT aportan un razonamiento fuerte, pero debes usarlos con cuidado por su naturaleza SaaS/cloud.
