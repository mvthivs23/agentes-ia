# Objetivo 22 — Lab: PoC Crafting

Este lab explora el uso de LLM para elaborar, refinar y simular exploits de prueba de concepto (PoC). Hace hincapié en la experimentación controlada, el manejo ético y los mecanismos de validación de las salidas generadas con IA. Vas a experimentar con la generación de payloads de explotación para vulnerabilidades conocidas (p. ej., CVE) y a observar cómo los LLM interpretan patrones de exploit, shellcode y cadenas de explotación.

Este lab no promueve ni simula ataques del mundo real: muestra capacidades y salvaguardas al usar IA para el desarrollo de exploits en investigación defensiva y ofensiva.

## 7.3.1 Alcance del lab

Con un LLM consciente del prompt (local o alojado), vas a:

- Dar prompts estructurados que describan una vulnerabilidad conocida
- Pedirle al modelo que genere o explique código PoC
- Validar y probar el comportamiento del PoC en un entorno controlado (p. ej., Docker o una VM)
- Implementar comprobaciones de seguridad para código alucinado o mal formado

## 7.3.2 Tools y entornos

| Tool | Uso |
| --- | --- |
| GPT-4, Claude, GPT4All, OpenLLM | LLM para generación de código |
| VS Code + intérpretes de Python/PoC | Ejecución de código |
| Docker o Firejail | Sandbox seguro |
| Metasploit, ExploitDB | PoC de referencia |
| CodeLLama, StarCoder, GPT-NeoX | LLM locales centrados en código |
| gpt-engineer, smol-ai, promptfoo | Plantillas de prompt y evaluación |

## 7.3.3 Ejemplo de plantilla de prompt: exploit basado en CVE

```text
Eres un asistente de explotación con IA.

Genera un PoC en Python para CVE-2021-41773 (Apache Path Traversal RCE).
El servidor es vulnerable a path traversal y a command injection.

Restricciones:
- Usa Python 3.
- Envía una petición HTTP GET manipulada al endpoint vulnerable.
- Imprime el resultado de la ejecución remota de código.

NO incluyas payloads peligrosos del mundo real.
```

## 7.3.4 Ejemplo de salida del modelo

```python
import requests

target = "http://192.168.56.101:8080"
payload = "../../../../bin/bash"
url = f"{target}/cgi-bin/.%2e/%2e%2e/%2e%2e/{payload}"

r = requests.get(url)
print(r.text)
```

Después validarías la corrección de:

- La codificación de la URL
- La lógica de traversal
- La ejecución real del comando, o su simulación

## 7.3.5 Modelado de cadenas de exploit mediante prompts

Los LLM pueden modelar cadenas de exploit completas si les das pasos estructurados:

```text
Modela una cadena de exploit para CVE-2022-1388 (F5 BIG-IP auth bypass):

1. Explica el mecanismo de inyección de cabeceras HTTP.
2. Genera el payload de curl para saltar la autenticación.
3. Simula la ejecución remota de un comando whoami.
```

## 7.3.6 Alucinación y validación

Antes de cualquier uso:

- Valida sintaxis y lógica a mano o con linters
- Compara con un corpus de PoC (p. ej., de ExploitDB o NVD)
- Usa agents GPT para que critiquen sus propias salidas:

```text
¿Este payload es válido en cuanto a sintaxis?
¿Coincide con PoC conocidos para este CVE?
¿Provocaría efectos secundarios no previstos?
```

## 7.3.7 Emparejamiento de PoC basado en embeddings (avanzado)

Usa bases vectoriales (p. ej., ChromaDB, Weaviate) para comparar el PoC generado por el LLM con PoC conocidos:

```python
# Embed generated PoC and compare with ExploitDB embeddings
similarity_score = vector_compare(generated, known_pocs)
if similarity_score > 0.85:
    approve()
```

## 7.3.8 Criterios de éxito

- Generar al menos una simulación de exploit válida y no maliciosa
- Usar el LLM en formato basado en prompt o en tool-calling
- Validar el exploit con tools en sandbox (p. ej., Docker)
- Registrar todos los pasos, prompts y resultados en formato estructurado

## 7.3.9 Aviso de seguridad y ética

- Nunca despliegues PoC generados en servidores públicos ni en infraestructura de producción.
- Este lab está pensado para investigación y entrenamiento en entornos seguros y legales.
- Simula siempre en contenedores air-gapped o en VM de laboratorio.
