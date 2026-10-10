# 04 · Tu primera conversación por código

> *Claude, GPT y Gemini*
>
> Capítulo 04 · *Agentes de IA con Python: guía práctica para diseñar, construir y desplegar sistemas agénticos*

---

## Introducción

Enviar texto a un modelo de lenguaje desde código propio no requiere más de veinte líneas de Python. Lo importante es leer esas líneas correctamente:

- un **cliente** que gestiona la autenticación,
- una **cadena** que selecciona el modelo,
- un **cuerpo de mensaje**,
- una **respuesta** que contiene más que el texto visible,
- y un **conteo de tokens** del que se deriva el coste.

Los capítulos 2 y 3 establecieron el entorno y el vocabulario de Python necesarios. Aquí se usa ese vocabulario para escribir el mismo programa tres veces —una por proveedor— con el mismo prompt de prueba, y observar dónde convergen y dónde difieren las implementaciones.

El ejercicio es mínimo: un único mensaje, sin historial, sin herramientas activadas y sin *streaming*. El prompt es:

> *"Explica en tres frases qué hace un agente de IA cuando usa una herramienta."*

La elección acota el experimento: un prompt breve mantiene bajo el conteo de tokens de entrada, tres frases limitan la salida y el tema será útil en los capítulos siguientes.

---

## Las cinco piezas de cualquier conversación por código

Antes de ver código, conviene nombrar las cinco piezas que lo componen. Aparecen en los tres proveedores aunque con nombres distintos; entenderlas como concepto, no como sintaxis, permite leer código desconocido sin haberlo escrito antes.

### 1. El cliente

Es el objeto Python que representa la conexión al proveedor. Al construirlo, el SDK lee la clave API del entorno, configura la URL base y establece la política de reintentos ante errores de red transitorios.

No realiza ninguna llamada de red en ese momento: almacena la configuración que cada petición posterior usará. Un único cliente puede reutilizarse en todo el programa sin *overhead* adicional.

### 2. El modelo

Es la cadena de texto que selecciona qué red neuronal procesa la petición. Determina las capacidades disponibles (razonamiento, visión, herramientas nativas), el límite de contexto y el coste por token.

Los identificadores no son apodos: `gpt-5.6-terra` y `gpt-5.6-sol` son modelos distintos, con precios y comportamientos distintos. Un carácter incorrecto produce un error inmediato; un alias informal que funciona en la interfaz web puede no funcionar en la API.

**Conviene extraer el ID a una constante en la parte superior del archivo**, donde sea fácil actualizarlo cuando el proveedor publique versiones nuevas.

### 3. El mensaje de usuario

Es el texto que se envía al modelo. En el caso más simple es una sola cadena; en una conversación multi-turno es una lista de objetos con roles (`user`, `assistant`). Aquí se usa exclusivamente un único mensaje. La gestión de historial y la construcción del bucle de conversación se tratan en el capítulo 11.

### 4. La respuesta

Es el objeto Python que devuelve el SDK una vez completada la llamada. Contiene el texto generado, pero también un identificador útil para trazabilidad y depuración, el motivo de finalización (*finish reason*) y el uso de tokens.

Leer solo el campo de texto basta para un experimento puntual, pero no para un sistema que deba detectar respuestas truncadas o auditar costes por llamada.

### 5. El coste

Se calcula a partir del conteo de tokens de entrada y salida que devuelve la respuesta. La fórmula es la misma en los tres proveedores, aunque los precios y los nombres de los campos varían:

```text
coste = (tokens_entrada / 1_000_000 × precio_entrada)
      + (tokens_salida  / 1_000_000 × precio_salida)
```

Los precios se publican por millón de tokens (MTok). Con un prompt corto y un modelo económico, cada llamada individual cuesta fracciones de céntimo. El coste se vuelve una decisión de diseño cuando el agente ejecuta decenas o cientos de llamadas en un bucle, o cuando el razonamiento exige modelos de gama alta.

Con estas cinco piezas identificadas, el código de cada proveedor se convierte en una instancia concreta del mismo patrón.

---

## OpenAI: primer mensaje con la API Responses

El SDK oficial de Python de OpenAI se instala con un único comando:

```bash
pip install openai
```

La autenticación usa una clave API generada desde el panel de control de OpenAI. La variable de entorno que el SDK busca por defecto es `OPENAI_API_KEY`:

```bash
export OPENAI_API_KEY="sk-proj-..."
```

Escribir la clave directamente en el código fuente es un riesgo de seguridad inmediato: si ese archivo llega a un repositorio o se comparte, la clave queda expuesta. El SDK la leerá automáticamente del entorno sin que el código la mencione.

OpenAI recomienda la **API Responses** para solicitudes directas al modelo, entradas multimodales y herramientas integradas. El punto de entrada es `client.responses.create(...)`, y el texto generado se accede mediante `response.output_text`:

```python
from openai import OpenAI

cliente = OpenAI()          # lee OPENAI_API_KEY del entorno

respuesta = cliente.responses.create(
    model="gpt-5.6-terra",
    input=(
        "Explica en tres frases qué hace un agente de IA "
        "cuando usa una herramienta."
    ),
)

print(respuesta.output_text)

entrada = respuesta.usage.input_tokens
salida = respuesta.usage.output_tokens
PRECIO_ENTRADA = 2.00   # USD por millón de tokens; verifica el precio vigente
PRECIO_SALIDA = 12.00   # USD por millón de tokens; verifica el precio vigente

coste = (entrada / 1_000_000 * PRECIO_ENTRADA) + (
    salida / 1_000_000 * PRECIO_SALIDA
)

print(f"Tokens - entrada: {entrada} / salida: {salida}")
print(f"Coste estimado: ${coste:.6f}")
```

### Puntos clave de este ejemplo

- El objeto `respuesta.usage` expone tres campos: `input_tokens`, `output_tokens` y `total_tokens`.
- Para `gpt-5.6-terra`, los precios se publican por separado para tokens de entrada y de salida. Conviene mantenerlos como constantes revisables, no como valores dispersos por el código.
- Con el prompt de prueba, la llamada completa cuesta unas pocas milésimas de dólar.
- `output_text` es una **propiedad de conveniencia** que el SDK construye a partir de la lista completa de bloques de salida (`response.output`). Una respuesta puede incluir texto, herramientas usadas y otros tipos de contenido estructurado; `output_text` extrae y concatena solo los bloques de texto. Para este ejemplo es suficiente. En el capítulo 12, cuando se activen herramientas, habrá que trabajar directamente con `response.output` para distinguir qué bloques son texto y cuáles son llamadas a herramientas.
- El parámetro `input` acepta tanto una cadena simple como una lista de mensajes con roles. Una cadena directa se interpreta como un mensaje de usuario sin historial previo: la forma más corta posible de iniciar una conversación.

---

## Anthropic: primer mensaje con el SDK de Claude

El SDK oficial de Anthropic se instala de la misma forma:

```bash
pip install anthropic
```

La variable de entorno es `ANTHROPIC_API_KEY`:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

La llamada difiere de OpenAI en dos puntos importantes al migrar código entre proveedores.

### Diferencia 1: formato obligatorio de mensajes

El método principal es `client.messages.create(...)`, y el parámetro para el contenido de la conversación es `messages`, siempre una lista de objetos con `role` y `content`, incluso cuando el historial tiene un solo elemento. **No existe un atajo que acepte una cadena directamente como mensaje de usuario.**

### Diferencia 2: `max_tokens` es obligatorio

`max_tokens` es **obligatorio** en Anthropic. El SDK no asume un límite por defecto y la llamada falla con un error de validación si se omite. Esta decisión obliga a definir el tamaño de la respuesta antes de hacer la petición, en lugar de recibir una salida de longitud arbitraria y coste impredecible.

```python
from anthropic import Anthropic

cliente = Anthropic()       # lee ANTHROPIC_API_KEY del entorno

respuesta = cliente.messages.create(
    model="claude-sonnet-5",
    max_tokens=256,
    messages=[
        {
            "role": "user",
            "content": (
                "Explica en tres frases qué hace un agente de IA "
                "cuando usa una herramienta."
            ),
        }
    ],
)

texto = respuesta.content[0].text
print(texto)

entrada = respuesta.usage.input_tokens
salida = respuesta.usage.output_tokens
print(f"Tokens - entrada: {entrada} / salida: {salida}")
```

### Lectura de la respuesta

La respuesta de Anthropic no expone una propiedad `output_text` directa. El contenido generado vive en `respuesta.content`, una lista de bloques tipados. En una respuesta de texto simple, `content[0]` es un `TextBlock` y su campo `.text` contiene la cadena generada.

Cuando el modelo invoca herramientas, la lista incluye bloques `ToolUseBlock`; por eso no conviene asumir que `content[0].text` siempre existe sin verificar antes el tipo del bloque.

### El límite de tokens

El valor `max_tokens=256` limita la respuesta a 256 tokens. Para un prompt que solicita tres frases es suficiente y evita respuestas largas accidentales. En producción, el límite debe dimensionarse según la tarea: demasiado bajo y el modelo trunca respuestas legítimas; demasiado alto y abre la puerta a costes descontrolados en llamadas masivas.

### El cálculo de coste

Para el cálculo de coste, los campos tienen los mismos nombres que en OpenAI (`usage.input_tokens` y `usage.output_tokens`) y la fórmula inicial aplica directamente. Los precios varían por modelo y se publican en la página de precios de Anthropic; con un prompt de este tamaño, el coste por llamada es igualmente marginal.

### Errores específicos del SDK

El SDK de Anthropic expone clases de error específicas que facilitan el diagnóstico:

- **`RateLimitError`**: la cuenta alcanzó el límite de peticiones por minuto o por día. La respuesta correcta es esperar y reintentar con *backoff* exponencial, no relanzar de inmediato.
- **`APIStatusError`**: el servidor devolvió un código HTTP de error no esperado (5xx, o un 400 con mensaje de API). El atributo `.status_code` contiene el código y `.message` el texto explicativo.
- **`APIConnectionError`**: fallo de red antes de recibir respuesta, ya sea por conectividad, por *timeout* o por problemas transitorios del servidor.

La gestión sistemática de errores, reintentos, trazas y límites aparece más adelante, cuando el agente deja de hacer una llamada aislada y empieza a encadenar decisiones. En este ejercicio basta con dejar que fallen de forma visible: si la clave no está configurada, el error es inmediato y descriptivo.

---

## Google Gemini: primer mensaje con el SDK

El paquete oficial para la API de Gemini es `google-genai`:

```bash
pip install google-genai
```

La clave API se obtiene desde Google AI Studio y se carga mediante la variable `GOOGLE_API_KEY`:

```bash
export GOOGLE_API_KEY="AIza..."
```

La misma clave funciona tanto en Google AI Studio (el entorno de prototipado en el navegador) como desde el SDK de Python. Esto facilita pasar de experimentos interactivos a código reproducible sin reconfiguración.

La documentación vigente de Google recomienda `models.generate_content` para solicitudes directas al modelo. El cliente se construye con `genai.Client()`, la llamada principal es `client.models.generate_content(...)`, y el texto generado se accede en `respuesta.text`:

```python
from google import genai

cliente = genai.Client()    # lee GOOGLE_API_KEY del entorno

respuesta = cliente.models.generate_content(
    model="gemini-3.5-flash",
    contents=(
        "Explica en tres frases qué hace un agente de IA "
        "cuando usa una herramienta."
    ),
)

print(respuesta.text)

entrada = respuesta.usage_metadata.prompt_token_count
salida = respuesta.usage_metadata.candidates_token_count
print(f"Tokens - entrada: {entrada} / salida: {salida}")
```

### Convención de conteo de tokens

Google sigue una convención diferente para el conteo de tokens: los campos se ubican dentro de `usage_metadata`, y se denominan `prompt_token_count` y `candidates_token_count` en lugar de `input_tokens` / `output_tokens` que usan OpenAI y Anthropic. El significado es equivalente para el cálculo básico de coste; cambia la ruta dentro del objeto y el nombre de los campos. Para estimar el importe se aplica la misma fórmula, con los precios publicados por Google para el modelo elegido.

### Entrada y modelo

El parámetro `contents` acepta una cadena simple cuando el historial tiene un único mensaje, de forma análoga al `input` de OpenAI. Para conversaciones multi-turno, `contents` se convierte en una lista de objetos con roles, como se verá en el capítulo 11 al construir el bucle de conversación del agente.

`gemini-3.5-flash` es el modelo económico de la familia Gemini 3.5, optimizado para velocidad y coste en tareas que no requieren razonamiento extendido. El esquema de nombrado de Google usa el número para indicar la generación y el sufijo —`flash`, `pro`— para situar el modelo en la gama de capacidad y precio.

Como ocurre con los otros proveedores, estos identificadores deben verificarse contra la documentación oficial antes de cada proyecto: cualquier lista impresa tiene una vida útil limitada.

---

## Tres proveedores: diferencias que importan y errores que hay que conocer

Las tres implementaciones muestran un patrón de convergencia estructural y tres áreas de divergencia real.

### La convergencia: el ciclo de vida

Los tres SDK siguen la misma secuencia: construir cliente, llamar al modelo, leer texto, leer uso de tokens. Los proveedores diseñan sus SDK para resultar familiares a quien ya haya usado otro. Una vez dominado uno, los otros dos se leen en minutos.

### Primera divergencia: la estructura de la entrada

- **OpenAI** acepta una cadena directa en `input`.
- **Anthropic** requiere una lista explícita de objetos en `messages`.
- **Google** acepta una cadena en `contents` para el caso de un solo mensaje, pero espera una lista para conversaciones multi-turno.

La consecuencia práctica es concreta: pasar una cadena donde Anthropic espera la lista de mensajes produce un error de validación que puede parecer críptico si no se conoce la causa.

### Segunda divergencia: el límite de salida

Anthropic lo hace obligatorio: `max_tokens` sin valor por defecto. OpenAI y Google tienen límites implícitos por modelo. La ausencia de un `max_tokens` explícito en un sistema de producción que maneja prompts variables es un riesgo de coste: el modelo puede generar respuestas mucho más largas de lo esperado si el propio prompt no las acota.

### Tercera divergencia: la ruta al texto generado

- OpenAI: `response.output_text`
- Anthropic: `response.content[0].text`
- Google: `response.text`

Si se escribe código que mezcla SDK sin adaptadores, este es exactamente el punto donde el programa fallará con un `AttributeError` que no indica el problema real.

### Errores frecuentes antes de la primera sesión de pruebas

**Clave ausente o mal configurada.** Si la variable de entorno no existe o contiene un valor incorrecto, el SDK devuelve un error de autenticación. La causa más común no es la clave en sí, sino el entorno desde el que se ejecuta el script: si la variable se exportó en un terminal diferente al que ejecuta el código, el proceso Python no la hereda.

**Modelo mal escrito.** Un ID de modelo con un carácter incorrecto o un alias informal que no reconoce la API devuelve un error 404 o un mensaje de "modelo no encontrado". Los nombres que funcionan en una interfaz web no siempre son válidos en la API; el SDK espera el identificador exacto publicado por el proveedor.

**Límite de cuota alcanzado.** En cuentas nuevas, la cuota de peticiones por minuto es baja. Ejecutar el mismo script varias veces en sucesión rápida puede agotar ese límite y producir un `RateLimitError` o su equivalente. La solución inmediata es esperar unos segundos antes de reintentar; el diseño para sistemas que encadenan muchas llamadas se trata en el capítulo 22.

**Respuesta truncada.** Si `max_tokens` es demasiado bajo, la respuesta puede cortarse en mitad de una frase. El campo `finish_reason` —o su equivalente en cada SDK— indicará `"max_tokens"` o `"length"` en lugar de `"end_turn"` o `"stop"`. Un agente robusto verifica este campo antes de procesar la respuesta como completa; de lo contrario, puede operar sobre contenido incompleto.

---

## Cierre

La comparación entre los tres proveedores no busca señalar uno como preferible. Cada SDK refleja decisiones de diseño distintas: OpenAI prioriza la ergonomía en el caso simple; Anthropic prefiere la explicitación de parámetros que afectan el coste; Google mantiene consistencia con su ecosistema más amplio.

El capítulo 5 entra en los detalles de cada plataforma agéntica. Los capítulos posteriores muestran cómo estas diferencias de superficie se amplifican cuando los agentes ejecutan herramientas reales, mantienen estado entre turnos y coordinan múltiples llamadas al modelo.

---

## Material del capítulo

El código canónico de este capítulo, completo y organizado por archivo, está en el repositorio del libro:

**https://github.com/studio35post/agentes-ia-python**

El Anexo A explica cómo descargarlo paso a paso y el Anexo B resuelve los errores comunes.
