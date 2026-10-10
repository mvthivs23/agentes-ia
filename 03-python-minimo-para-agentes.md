# 03 · Python mínimo para agentes

> *Lo justo para seguir el libro*
>
> Capítulo 03 · *Agentes de IA con Python: guía práctica para diseñar, construir y desplegar sistemas agénticos*

---

## Introducción

Un agente de IA, antes de invocar cualquier SDK o framework, manipula estructuras simples:

- una lista de mensajes,
- un diccionario de configuración,
- un objeto JSON enviado a una API,
- una respuesta HTTP,
- y, tarde o temprano, un error que hay que saber leer.

Python es adecuado para este trabajo por su orientación a scripts, su sintaxis cercana al pseudocódigo y su biblioteca estándar, suficiente para lo que un agente básico necesita sin instalar nada extra.

No hace falta dominar clases, herencia, generadores ni la mayor parte del ecosistema del lenguaje para seguir los primeros scripts agénticos. Basta con entender el vocabulario mínimo para leerlos, modificarlos y escribir uno sencillo.

El capítulo 2 dejó el entorno listo: Python instalado, entorno virtual activo, dependencias disponibles. El capítulo 4 arranca la primera conversación por código. Este material funciona como puente entre ambos.

---

## Variables: etiquetas de trabajo

Una variable es un nombre que apunta a un valor. La utilidad es simple: en lugar de escribir el mismo valor en diez lugares distintos del script, se escribe una vez y se nombra.

```python
model = "claude-sonnet-5"
temperature = 0.2
user_prompt = "Resume este documento en tres puntos."
max_tokens = 1024
```

Cuando el modelo cambia, se edita una línea, no diez. Cuando el prompt necesita ajuste, está en un solo sitio. Esa es la razón por la que los scripts de agentes usan variables desde la primera línea.

### Tipos

Python deduce el tipo a partir del valor asignado:

- `model` es una cadena de texto,
- `temperature` es un número decimal,
- `max_tokens` es un entero.

En agentes, los tipos habituales son estables: cadenas para nombres de modelos y prompts, números para temperaturas y límites de tokens, y estructuras compuestas (listas y diccionarios) para mensajes y configuración.

### Convenciones de nombres

- Usa nombres descriptivos (`user_prompt`, no `u` ni `p`), en minúsculas y con guion bajo cuando tienen varias palabras.
- Las constantes de configuración que no cambian durante la ejecución van en mayúsculas: `MAX_RETRIES = 3`.

### Micro-ejercicio

Abre el editor con un archivo nuevo, `config.py`. Declara tres variables:

1. el nombre del modelo que vas a usar en el capítulo 4,
2. la temperatura que prefieras para ese modelo,
3. un límite de tokens razonable.

Guarda el archivo. No hace falta ejecutarlo todavía; basta con que el editor no muestre errores de sintaxis. Es el primer bloque de configuración real del proyecto.

---

## Funciones: pasos que se repiten

Una función nombra un bloque de código. Cuando la misma secuencia de pasos aparece más de una vez en un script, conviene aislarla y reutilizarla.

Sintaxis básica:

```python
def crear_mensaje_usuario(texto):
    return {"role": "user", "content": texto}
```

- `def` abre la definición.
- `crear_mensaje_usuario` es el nombre.
- `texto` es el parámetro: el valor que la función recibe al ser llamada.
- `return` devuelve el resultado.

Para usarla:

```python
mensaje = crear_mensaje_usuario("¿Qué es un agente de IA?")
```

`mensaje` ahora contiene el diccionario `{"role": "user", "content": "¿Qué es un agente de IA?"}`. La función se puede llamar muchas veces con textos distintos y mantener siempre la misma estructura.

En scripts agénticos, las funciones suelen construir datos con formato fijo, envolver operaciones con efectos secundarios (llamar a una API, escribir un archivo) o nombrar lógica de decisión: `elegir_herramienta(respuesta)`, `respuesta_es_final(texto)`.

Una función puede no tener parámetros:

```python
def timestamp_actual():
    import datetime
    return datetime.datetime.now().isoformat()
```

Y puede no devolver nada explícito, en cuyo caso solo ejecuta sus instrucciones:

```python
def mostrar_respuesta(respuesta):
    print(f"Asistente: {respuesta}")
```

### El nombre importa

`f2(x)` y `procesar_respuesta_del_modelo(texto)` hacen lo mismo en el intérprete, pero el segundo hace el código legible semanas después. En los capítulos siguientes aparecen funciones como `ejecutar_herramienta`, `construir_contexto` o `iterar_hasta_respuesta_final`: el nombre actúa como documentación.

### Micro-ejercicio

En el mismo archivo `config.py` del ejercicio anterior, escribe una función `crear_sistema(instruccion)` que devuelva un diccionario con `"role": "system"` y `"content": instruccion`. Llámala desde el mismo archivo con una instrucción de tu elección e imprime el resultado con `print()`. Ejecuta el archivo con `python config.py`. Si ves el diccionario en la terminal, la función funciona.

---

## Condicionales: tomar decisiones

Un script agéntico decide constantemente: si la respuesta de la API llegó con código 200 o 429, si el campo `tool_use` está presente, si el texto supera el límite de tokens. Esas decisiones se expresan con `if`.

```python
if response.status_code == 200:
    datos = response.json()
elif response.status_code == 429:
    print("Límite de tasa alcanzado. Espera un momento.")
else:
    print(f"Error inesperado: {response.status_code}")
```

- **La indentación define el bloque.** Python no usa llaves ni palabras clave de cierre: el nivel de sangría determina qué líneas pertenecen a cada rama. Cuatro espacios por nivel es la convención universal.
- **`elif`** es la contracción de *"else if"* y permite encadenar condiciones sin anidar.
- **`else`** captura todo lo que no satisface ninguna condición anterior.
- Ambos son opcionales: un `if` sin `else` simplemente no hace nada cuando la condición es falsa.

---

## Bucles: recorrer colecciones y repetir acciones

### `for`: recorrer una colección

```python
for mensaje in messages:
    print(mensaje["role"], ":", mensaje["content"])
```

Esto recorre cada elemento de la lista `messages` y ejecuta el bloque indentado para cada uno. En scripts agénticos, `for` aparece con frecuencia al construir el historial de conversación, al listar herramientas disponibles o al procesar un lote de documentos antes de enviarlos al modelo.

### `while`: repetir mientras se cumpla una condición

`while` es la estructura del bucle de agente: el ciclo de observar, razonar y actuar hasta que la tarea concluye. El capítulo 11 desarrolla esa implementación. Por ahora basta con la mecánica: `while` ejecuta su bloque mientras la condición sea verdadera.

```python
intentos = 0
while intentos < 3:
    # aquí irá la llamada a la API
    intentos += 1
```

Este fragmento reintenta una operación hasta tres veces. En agentes reales, la condición suele ser más sofisticada (si el agente terminó, si llegó al límite de iteraciones), pero la estructura es la misma.

### Micro-ejercicio

Escribe una lista con tres diccionarios, cada uno con `"role"` y `"content"` distintos (puedes usar la función del ejercicio anterior para construirlos). Luego escribe un `for` que recorra la lista e imprima solo los mensajes cuyo `"role"` sea `"user"`. Ejecuta el script y verifica que solo aparecen los mensajes correctos.

---

## Listas y diccionarios: la estructura de los mensajes

Todo agente que conversa con un modelo de lenguaje construye, antes o después, una lista de mensajes. La lista es la unidad de contexto: el historial que el modelo recibe para generar su respuesta.

```python
messages = [
    {"role": "system", "content": "Eres un asistente conciso y preciso."},
    {"role": "user", "content": "Explica qué es RAG en una frase."},
    {"role": "assistant", "content": "RAG es una técnica que combina recuperación de documentos con generación de texto para que el modelo responda con información actualizada o privada."},
    {"role": "user", "content": "¿Y MCP?"}
]
```

### Listas

Una lista en Python se define con corchetes. Sus elementos están separados por comas y pueden ser de cualquier tipo: cadenas, números, otros diccionarios, otras listas. En el caso de mensajes, cada elemento es un diccionario con dos claves: `"role"` y `"content"`.

Para añadir un mensaje al final de la lista:

```python
messages.append({"role": "user", "content": "Dame un ejemplo concreto."})
```

`append` modifica la lista en su lugar. No devuelve nada: el efecto es la modificación de `messages`. Es la operación más frecuente en el bucle de conversación: cada turno añade el mensaje del usuario y luego la respuesta del modelo.

El acceso por índice se usa con moderación. `messages[0]` devuelve el primer elemento; `messages[-1]` devuelve el último. En scripts agénticos suele ser más común recorrer la lista con `for` que acceder a posiciones específicas.

### Diccionarios

Los diccionarios son la otra estructura fundamental. Un diccionario asocia claves con valores:

```python
config = {
    "model": "claude-sonnet-5",
    "timeout": 30,
    "max_retries": 2
}
```

Se definen con llaves. Las claves son cadenas (en estos ejemplos, siempre lo son). Para acceder a un valor: `config["model"]`. Para modificarlo: `config["timeout"] = 60`. Para añadir una clave nueva: `config["stream"] = True`.

### Composición

La combinación de listas y diccionarios explica cómo los SDK agénticos estructuran sus objetos. Un payload de API es un diccionario que contiene una lista de mensajes, y cada mensaje es a su vez un diccionario:

```python
payload = {
    "model": config["model"],
    "max_tokens": 1024,
    "messages": messages
}
```

Esta composición —diccionarios dentro de diccionarios, listas dentro de diccionarios— es el patrón común. Las llamadas directas a la API del capítulo 4 y los agentes con herramientas del capítulo 11 siguen esta misma forma.

### Micro-ejercicio

Construye el `payload` del fragmento anterior usando la variable `config` del primer ejercicio y la lista `messages` del ejercicio anterior. Imprime `payload["messages"]` y verifica que contiene la lista completa. Luego imprime `payload["messages"][-1]` para ver el último mensaje.

---

## JSON: el idioma de las APIs

Cuando un script envía datos a la API de Claude, OpenAI o Gemini, los envía como texto con un formato específico: **JSON**. Cuando la API responde, también responde en JSON. Python y JSON se parecen mucho en la superficie, pero no son lo mismo.

- Un **objeto JSON** es texto plano.
- Un **diccionario Python** es un objeto en memoria.

La conversión entre ambos la maneja el módulo `json` de la biblioteca estándar:

```python
import json

payload = {
    "model": "claude-sonnet-5",
    "messages": [{"role": "user", "content": "Hola"}]
}

texto_json = json.dumps(payload)
print(type(texto_json))   # <class 'str'>
print(texto_json)
# {"model": "claude-sonnet-5", "messages": [{"role": "user", "content": "Hola"}]}
```

- `json.dumps()` convierte el diccionario en una cadena de texto con formato JSON.
- El parámetro `indent=2` produce una versión legible para depuración: `print(json.dumps(payload, indent=2))`.
- En la dirección contraria, `json.loads()` convierte texto JSON en un diccionario Python:

```python
respuesta_texto = '{"id": "msg_01", "content": "Hola, ¿en qué puedo ayudarte?"}'
respuesta = json.loads(respuesta_texto)
print(respuesta["content"])   # Hola, ¿en qué puedo ayudarte?
```

Para trabajar con archivos, el módulo ofrece `json.dump()` (escribe en un archivo) y `json.load()` (lee de un archivo):

```python
# Guardar configuración en disco
with open("config.json", "w") as f:
    json.dump(config, f, indent=2)

# Cargar configuración desde disco
with open("config.json", "r") as f:
    config_leido = json.load(f)
```

La diferencia entre `dumps`/`loads` (trabajan con cadenas) y `dump`/`load` (trabajan con archivos) se recuerda por la `s` al final: **s de *string***.

### Un detalle que evita confusiones

JSON solo admite cadenas como claves. Python acepta otros tipos básicos en un diccionario, pero al serializarlo con `json.dumps()` esas claves se convierten en cadenas. En estos ejemplos, las claves de los payloads siempre son cadenas.

La documentación oficial de Python advierte que objetos JSON muy grandes pueden consumir una cantidad considerable de CPU y memoria si se cargan completos con `json.loads()`. En agentes básicos, los payloads y respuestas suelen ser pequeños; la advertencia importa sobre todo en proyectos con procesamiento masivo de documentos.

### Micro-ejercicio

Toma el diccionario `payload` del ejercicio anterior y conviértelo a texto JSON con `json.dumps(indent=2)`. Luego convierte ese texto de vuelta a un diccionario con `json.loads()`. Compara el resultado con el original con `print(resultado == payload)`. El resultado debe ser `True`.

---

## HTTP con `requests` y lectura de errores

Las APIs de modelos de lenguaje son servicios web. Desde Python, el script habla con ellas mediante peticiones HTTP. La biblioteca `requests` —instalada en el entorno del capítulo 2— es la forma más directa de hacerlo.

Una llamada mínima a una API tiene esta forma:

```python
import requests

url = "https://api.anthropic.com/v1/messages"
headers = {
    "x-api-key": "sk-ant-...",
    "anthropic-version": "2023-06-01",
    "content-type": "application/json"
}
payload = {
    "model": "claude-sonnet-5",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "¿Qué es un agente de IA?"}]
}

response = requests.post(url, headers=headers, json=payload)
```

- `requests.post()` envía la petición.
- El parámetro `json=payload` convierte el diccionario a JSON y establece el encabezado `content-type` correcto.
- El objeto `response` contiene lo que devuelve el servidor.

Para comprobar si la petición tuvo éxito: `response.status_code`. Un 200 indica éxito. Para acceder al cuerpo de la respuesta como diccionario: `response.json()`. Para acceder al texto crudo: `response.text`.

```python
if response.status_code == 200:
    datos = response.json()
    contenido = datos["content"][0]["text"]
    print(contenido)
else:
    print(f"Error {response.status_code}: {response.text}")
```

> **Nota:** la clave de ejemplo `sk-ant-...` es un marcador. En tu código, la clave se obtiene de `.env` con `load_dotenv()` y `os.environ`, nunca se escribe en el script (ver capítulo 2).

### Leer los errores

Los errores son inevitables: claves de API inválidas, límites de tasa superados, modelos no disponibles, payloads malformados. Todos producen respuestas con códigos distintos de 200. Ante un error, lo primero es **leer el cuerpo de la respuesta**: casi todas las APIs incluyen un mensaje descriptivo.

```text
Error 401: {"type":"error","error":{"type":"authentication_error","message":"invalid x-api-key"}}
```

Este mensaje indica que la clave de API no es válida. Leer `response.text` antes de cualquier otra acción es uno de los hábitos más útiles al trabajar con APIs.

Dos errores frecuentes aparecen en los primeros scripts:

- **`requests.exceptions.ConnectionError`**: la petición no llegó al servidor. Las causas habituales son problemas de red, URL incorrecta o servicio caído. No es un error de la API, sino de transporte.
- **`json.JSONDecodeError`** al llamar a `response.json()`: el cuerpo de la respuesta no es JSON válido. Puede ocurrir cuando la API devuelve HTML de error en lugar del JSON esperado. En ese caso, `response.text` muestra el contenido real.

### Capturar excepciones

Para manejarlas sin que el script se detenga abruptamente:

```python
try:
    response = requests.post(url, headers=headers, json=payload, timeout=30)
    response.raise_for_status()   # lanza excepción si el código es 4xx o 5xx
    datos = response.json()
except requests.exceptions.Timeout:
    print("La petición tardó demasiado. Reintenta.")
except requests.exceptions.HTTPError as e:
    print(f"Error HTTP: {e}")
    print(f"Detalle: {response.text}")
except Exception as e:
    print(f"Error inesperado: {e}")
```

- `raise_for_status()` convierte cualquier código de error HTTP en una excepción, evitando escribir un `if` para cada código posible.
- El bloque `try/except` captura esa excepción y la maneja sin interrumpir el programa.
- El parámetro `timeout=30` establece un tiempo máximo de espera en segundos. Sin él, un servidor que no responde deja el script colgado indefinidamente. **En scripts agénticos, siempre se pasa un timeout.**

### Micro-ejercicio

Este ejercicio no requiere una clave de API real todavía. Usa la URL `https://httpbin.org/post`, un servicio público que devuelve los datos que recibe. Construye un payload mínimo con un campo cualquiera, envíalo con `requests.post(url, json=payload)` e imprime `response.json()`. Deberías ver tu payload reflejado en la respuesta bajo la clave `"json"`. Si ves un error de conexión, verifica que el entorno virtual está activo y que `requests` está instalado.

---

## Cierre

Con estas seis piezas —variables, funciones, condicionales y bucles, listas y diccionarios, JSON, HTTP con manejo de errores— el código agéntico básico resulta legible. No es todo Python, ni siquiera la mayor parte. Es lo que un agente sencillo necesita: guardar configuración, construir mensajes, hablar con una API, entender la respuesta y reaccionar si algo falla.

El capítulo 4 usa estas estructuras para hacer la primera llamada real a un modelo de lenguaje.

---

## Material del capítulo

El código canónico de este capítulo, completo y organizado por archivo, está en el repositorio del libro:

**https://github.com/studio35post/agentes-ia-python**

El Anexo A explica cómo descargarlo paso a paso y el Anexo B resuelve los errores comunes.
