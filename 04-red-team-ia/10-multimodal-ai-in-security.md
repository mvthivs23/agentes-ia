# Lección 10 — Multimodal AI in Security

Módulo 05: Arquitectura de IA, Prompt Engineering e IA Local · Penetration Testing con Kali Linux 1.0 · Red Team Leaders

## Introducción

El penetration testing no es una disciplina de solo texto. Un solo engagement genera cientos de screenshots: ventanas de Burp Suite, paneles de DevTools del navegador, output de terminal, diagramas de red que entrega el cliente, documentos PDF de políticas, documentos de identidad escaneados para escenarios adyacentes a phishing, dashboards de vulnerability scanners y fotografías de infraestructura física. Históricamente, las notas del tester estaban salpicadas de frases como "ver screenshot 47.png" y cada finding exigía que una persona releyera cada screenshot para escribir la narrativa del report.

Los large language models modernos cambiaron esto. Los vision-language models (VLMs) aceptan imágenes de forma nativa junto con prompts de texto. Puedes pasarle un screenshot de Burp Repeater a Claude Sonnet 5 y preguntar "qué detalles relevantes para la seguridad hay en esta imagen", alimentar un diagrama de red a Llama 3.2 Vision corriendo en local en tu laptop y preguntar "lista cada dirección IP, hostname y rol que puedas extraer", o apuntar un modelo Whisper a una grabación de voicemail de un engagement de social engineering autorizado y recibir una transcripción precisa en segundos.

Esta lección cubre el pipeline completo: cómo funcionan por dentro los vision-language models, qué modelos hosteados y locales usar, cómo armar pipelines de ingesta de imágenes y documentos, cuándo preferir OCR clásico por encima de un LLM, y cómo cablear input multimodal en el workflow diario de pentest a través de un skill de Claude Code. También traza líneas éticas duras: resolver CAPTCHAs se trata exclusivamente como tema de investigación defensiva sobre tus propias implementaciones, y el análisis de documentos queda acotado a artefactos autorizados por el cliente.

Al final de la lección vas a tener un script `image-triage.py` que procesa screenshots en batch, por un modelo de visión hosteado o local, y produce un resumen JSON por archivo, listo para alimentar tu pipeline de reporting.

## 1. Por qué lo multimodal importa para pentesters

Un engagement de pentest produce más o menos tres clases de imágenes:

| Categoría | Ejemplos | Uso típico en el report |
|---|---|---|
| Screenshots de tools | Burp Suite, capturas de output de Nmap, Wireshark, progreso de ffuf, alertas de ZAP | Evidencia del request/response que disparó un finding |
| Artefactos que entrega el cliente | Diagramas de topología de red, PDFs de arquitectura, SOWs escaneados, fotos on-site | Scoping, enumeración de targets, notas de seguridad física |
| Capturas de UI | Screenshots de la aplicación web target, pantallas de app mobile, diálogos nativos de Windows | Reproducir de forma visual el estado vulnerable |

Antes de los LLM multimodales, extraer texto de estas imágenes significaba transcribir a mano o pasar el archivo por Tesseract y esperar que la accuracy del OCR fuera tolerable. Los dos enfoques perdían la capa semántica. El pipeline clásico de OCR produce glifos crudos y no tiene noción de "esto es un formulario de login, y el banner amarillo dice incorrect password, lo que sugiere que la enumeración de usernames es posible".

Los vision-language models reemplazan el dos pasos de transcribir y después razonar por un modelo de una sola pasada que ve los píxeles y razona sobre ellos en conjunto. Eso es un cambio de escala para los pentesters.

> **Pentester Insight:** Cualquier cosa que antes te llevaba veinte minutos de reescritura manual — leer una pantalla de login, sacar cada IP de un diagrama de topología, volcar la URL de la barra de direcciones de un screenshot — ahora puede ser una llamada a la API de cinco segundos. La ganancia de productividad se compone porque después puedes devolver la estructura extraída a un prompt de seguimiento: "dado este diagrama de red, propone un camino de lateral movement desde el web server de la DMZ hasta el forest de AD."

Ese seguimiento es una hipótesis de scoping sobre el diagrama del cliente. No es un procedimiento. El tester confirma el camino contra el scope del engagement antes de probar nada.

## 2. Arquitectura de un vision-language model

Un vision-language model moderno no es una sola red monolítica. Son tres componentes pegados.

### 2.1 Los tres bloques

```text
┌────────────────┐     ┌────────────┐     ┌──────────────────┐
│  Image encoder │ ──▶ │  Projector │ ──▶ │  Text LLM (decoder) │
│  (ViT / CLIP)  │     │  (MLP o    │     │  (LLaMA / Qwen /  │
│                │     │  Q-Former) │     │  backbone Claude) │
└────────────────┘     └────────────┘     └──────────────────┘
       ▲                                             ▲
       │                                             │
     píxeles                                    texto tokenizado
```

- **Image encoder.** En general un Vision Transformer (ViT) entrenado con un objetivo contrastivo al estilo CLIP. Divide la imagen en patches de 14×14 o 16×16 píxeles, proyecta cada patch a un embedding y produce una secuencia de image tokens (típico: 256, 576 o 729 tokens para una sola imagen de 336×336).
- **Projector.** Un adapter liviano (a menudo un MLP de dos capas; a veces un Q-Former como en BLIP-2) que mapea el espacio de embeddings del image encoder al espacio de embeddings del text LLM. Sin este projector, los image tokens no significarían nada para el language model.
- **Text LLM.** Un transformer decoder-only estándar (LLaMA, Qwen, Mistral, o un backbone propietario en el caso de Claude y GPT). Los image tokens se anteponen o se intercalan en el stream de text tokens, y el modelo les presta atención como si fueran palabras.

### 2.2 Embeddings al estilo CLIP

Contrastive Language-Image Pre-training (CLIP), publicado por OpenAI en 2021, es la raíz intelectual de la mayoría de los image encoders modernos. CLIP entrena un image encoder y un text encoder lado a lado sobre 400 millones de pares (imagen, caption). El objetivo de entrenamiento es una contrastive loss: dentro de un batch de N pares, maximizar la cosine similarity de los embeddings (imagen, texto) que matchean y minimizar la similarity de los N−1 pares que no matchean.

El resultado es un espacio de embedding compartido de 512 dimensiones (o 768, o 1024) donde una fotografía de un golden retriever y el string "a photograph of a golden retriever" caen cerca. Esa alineación semántica es lo que hace que los encoders derivados de CLIP sean excelentes puntos de partida para vision-language models.

### 2.3 Cuántos image tokens se mandan al LLM

La contabilidad de tokens importa para el costo y para el manejo del context window:

| Modelo | Conteo de image tokens (típico) | Notas |
|---|---|---|
| Llama 3.2 11B Vision | ~1601 tokens por imagen | Tiles de 560×560 |
| Qwen2-VL 7B | Dinámico, ~256 a ~4096 | Se adapta a la resolución de la imagen |
| MiniCPM-V 2.6 | ~640 para una imagen de 1344×1344 | Encoder basado en slices |
| Claude Sonnet 5 | ~1600 para una imagen de 1600×1200 | Reportado en el bloque de usage de la API |
| GPT-4o | 85 de base + 170 por tile de 512×512 | Documentado en los pricing docs de OpenAI |

Un solo screenshot de Burp en alta resolución puede consumir de 2000 a 4000 tokens de contexto. Meter cien en un solo request y quemaste 200k tokens antes de que el modelo haya leído una sola palabra de tu prompt.

### 2.4 Late fusion frente a early fusion

Dos estrategias de arquitectura para combinar modalidades:

- **Late fusion (estilo adapter).** Los image tokens se inyectan en el stream de input del LLM y el LLM hace todo el razonamiento cross-modal a través de attention. LLaVA, Llama 3.2 Vision y MiniCPM-V usan variantes de esto.
- **Early fusion (multimodal nativo).** El modelo se entrena desde cero sobre secuencias intercaladas de imagen y texto. Se cree que Gemini y GPT-4o usan este enfoque. La arquitectura exacta de Claude es propietaria, pero el comportamiento sugiere un régimen fuerte de entrenamiento cross-modal.

Early fusion tiende a producir mejor razonamiento multimodal, al costo de un entrenamiento mucho más caro y menos flexibilidad para cambiar backbones.

## 3. APIs multimodales hosteadas

Para la mayoría de los pentesters, las APIs hosteadas son el punto de partida pragmático: no hay GPU que provisionar, el razonamiento de visión es de alta calidad y la facturación es directa.

### 3.1 Input de imagen en Claude Sonnet 5

Claude Sonnet 5 (publicado en 2026 por Anthropic) acepta hasta 100 imágenes por mensaje, con un borde máximo de 8000 píxeles y un tamaño máximo de archivo de 5 MB por imagen. Las imágenes se pueden pasar como bloques codificados en base64 o como URLs públicas.

```python
import anthropic
import base64
from pathlib import Path

client = anthropic.Anthropic()

def encode_image(path: str) -> tuple[str, str]:
    """Devuelve (media_type, base64_data) de un archivo de imagen."""
    suffix = Path(path).suffix.lower()
    media_type = {
        ".jpg": "image/jpeg",
        ".jpeg": "image/jpeg",
        ".png": "image/png",
        ".gif": "image/gif",
        ".webp": "image/webp",
    }[suffix]
    with open(path, "rb") as f:
        return media_type, base64.standard_b64encode(f.read()).decode("utf-8")

media_type, b64 = encode_image("burp-repeater-401.png")

resp = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": b64,
                    },
                },
                {
                    "type": "text",
                    "text": (
                        "Esto es un screenshot de Burp Suite Repeater. "
                        "Extrae: método HTTP, URL completa, todos los request headers, "
                        "el status code de la respuesta y cualquier response header "
                        "relevante para la seguridad (CSP, HSTS, cookies). "
                        "Devuelve JSON estricto."
                    ),
                },
            ],
        }
    ],
)

print(resp.content[0].text)
```

Límites clave:

- Los payloads en base64 tienen que ir en el campo `source.data`. No pegues el data URI completo (`data:image/png;base64,...`). Claude espera el string base64 crudo y un `media_type` aparte.
- Las imágenes muy chicas (menos de 200 píxeles en el borde corto) se pueden leer mal. Haz upscale antes de mandarlas.
- Las imágenes muy grandes las baja de escala la API de forma automática. No hay beneficio en mandar screenshots 4K.

### 3.2 Visión de GPT-4o

GPT-4o de OpenAI acepta imágenes por el mismo endpoint de Chat Completions que los mensajes de texto. Las imágenes se pasan como URL o como data URI:

```python
from openai import OpenAI
import base64

client = OpenAI()

with open("nmap-output.png", "rb") as f:
    b64 = base64.b64encode(f.read()).decode("utf-8")

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Extrae cada puerto abierto y cada service banner de este screenshot de nmap. Devuelve JSON."},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/png;base64,{b64}",
                        "detail": "high",
                    },
                },
            ],
        }
    ],
)

print(resp.choices[0].message.content)
```

Nota el campo `detail`: `low` usa 85 tokens fijos por imagen (más barato, bueno para clasificación gruesa); `high` usa tiling (preciso para screenshots cargados de texto). Pon `high` en cualquier screenshot de pentest con texto legible.

### 3.3 Capacidades y límites de un vistazo

| Feature | Claude Sonnet 5 | GPT-4o |
|---|---|---|
| Máximo de imágenes por request | 100 | 10 (por request; más por conversación) |
| Borde máximo de imagen | 8000 px | 2048 px (se baja de escala solo) |
| Soporte de video | Extracción de frames | Extracción de frames |
| Structured output | JSON vía prompt o tool use | Modo JSON schema |
| Privacidad de datos | No se usa para entrenamiento por defecto (tier de API) | No se usa para entrenamiento por defecto (tier de API) |

### 3.4 En qué son malas las APIs hosteadas

- Leer imágenes con fuentes chicas y densas (screenshots de terminal apretados): la accuracy cae por debajo de unos 10 px de altura de carácter.
- Contar grandes cantidades de objetos idénticos (una lista de 47 filas en una tabla).
- Preservar el whitespace exacto en snippets de código extraídos: la indentación a menudo se colapsa.
- CAPTCHAs: los modelos frontier rechazan de forma explícita resolver CAPTCHAs de sitios de terceros, y el cumplimiento es fuerte. La sección 10 deja este tema en investigación defensiva sobre tu propia implementación.

## 4. Modelos multimodales locales

En engagements donde los datos del cliente no pueden salir de la laptop del tester — evaluaciones internas de red team bajo NDA estricto, trabajo de contratistas de defensa, datos de salud — los modelos de visión locales son obligatorios.

### 4.1 Llama 3.2 Vision vía Ollama

Meta publicó Llama 3.2 Vision (11B y 90B) a fines de 2024. La variante 11B corre en una sola GPU de 24 GB o en Apple Silicon con 32 GB de memoria unificada.

```bash
# Instala Ollama (Kali o Debian):
curl -fsSL https://ollama.com/install.sh | sh

# Baja el modelo de visión 11B (unos 7.9 GB en disco):
ollama pull llama3.2-vision:11b

# Smoke test interactivo:
ollama run llama3.2-vision:11b "describe esta imagen" ./burp-401.png
```

Ollama expone un endpoint compatible con OpenAI en `http://localhost:11434/v1`, así que puedes usar el SDK de OpenAI sin cambiarlo:

```python
from openai import OpenAI
import base64

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

with open("network-diagram.png", "rb") as f:
    b64 = base64.b64encode(f.read()).decode("utf-8")

resp = client.chat.completions.create(
    model="llama3.2-vision:11b",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Lista cada IP, hostname y rol de nodo de este diagrama. Solo JSON."},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{b64}"}},
            ],
        }
    ],
)

print(resp.choices[0].message.content)
```

### 4.2 Qwen2-VL

Qwen2-VL de Alibaba (2B, 7B, 72B) tiene OCR excelente y soporte nativo de varias imágenes. Maneja resoluciones arbitrarias sin un resize previo.

```bash
ollama pull qwen2-vl:7b
```

Qwen2-VL es particularmente fuerte en texto chino, japonés y coreano. Sirve si estás testeando aplicaciones web localizadas cuya UI no está en inglés.

### 4.3 MiniCPM-V

MiniCPM-V 2.6 de Tsinghua es un modelo de 8B que supera a GPT-4V en muchos benchmarks de OCR y corre en una sola GPU de 12 GB. Soporta input de una imagen, de varias imágenes y de video corto.

```bash
ollama pull minicpm-v:8b
```

### 4.4 Uso directo de llama.cpp

Si necesitas control de más bajo nivel (quantisation custom, sampling determinista para un triage reproducible), corre llama.cpp directo:

```bash
# Compila llama.cpp con CUDA:
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j

# Baja un GGUF de Llama 3.2 Vision:
huggingface-cli download bartowski/Llama-3.2-11B-Vision-Instruct-GGUF \
    Llama-3.2-11B-Vision-Instruct-Q4_K_M.gguf \
    --local-dir ./models

# Baja también el mmproj (projector multimodal):
huggingface-cli download bartowski/Llama-3.2-11B-Vision-Instruct-GGUF \
    mmproj-Llama-3.2-11B-Vision-Instruct-f16.gguf \
    --local-dir ./models

# Corre:
./build/bin/llama-mtmd-cli \
    -m ./models/Llama-3.2-11B-Vision-Instruct-Q4_K_M.gguf \
    --mmproj ./models/mmproj-Llama-3.2-11B-Vision-Instruct-f16.gguf \
    --image ./burp-401.png \
    -p "Extrae la URL completa del request, el método y el status code."
```

### 4.5 Cuándo elegir local y cuándo hosteado

| Situación | Elección |
|---|---|
| Documentos del cliente con PII | Local |
| Target web público, un screenshot suelto | Hosteado |
| Batch de 10.000 screenshots de UI para un report | Local (costo) |
| Máxima accuracy en diagramas complejos | Hosteado (Claude/GPT-4o) |
| Entorno air-gapped | Local |
| Reproducibilidad a lo largo de meses | Local (fija el archivo del modelo) |

## 5. Análisis de screenshots

El workflow de screenshots es la tarea multimodal más común para un pentester. Acá hay un ejemplo endurecido, con forma de producción.

```python
"""
screenshot_extract.py
Extrae cada URL, label de botón, mensaje de error, campo de formulario
y cookie de un screenshot usando Claude Sonnet 5.
"""
import base64
import json
import sys
from pathlib import Path

import anthropic

PROMPT = """Estás analizando un screenshot de un engagement de penetration testing.
Extrae ESTRICTAMENTE como JSON con estas keys de nivel superior:

{
  "urls":            [strings — cada URL, path o hostname visible],
  "buttons":         [strings — cada label de botón],
  "form_fields":     [ {"label": str, "type": str} ],
  "error_messages":  [strings — texto de error visible en pantalla, textual],
  "cookies":         [ {"name": str, "value": str, "flags": [str]} ],
  "headers_visible": [ {"name": str, "value": str} ],
  "notes":           [strings — cualquier otra cosa relevante para la seguridad]
}

Devuelve solo JSON. No lo envuelvas en fences de markdown. Si una categoría
está vacía, devuelve una lista vacía."""


def encode(path: Path) -> tuple[str, str]:
    ext = path.suffix.lower().lstrip(".")
    media = "image/jpeg" if ext in ("jpg", "jpeg") else f"image/{ext}"
    return media, base64.standard_b64encode(path.read_bytes()).decode()


def extract(path: Path) -> dict:
    client = anthropic.Anthropic()
    media, b64 = encode(path)
    resp = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=2048,
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "image",
                     "source": {"type": "base64", "media_type": media, "data": b64}},
                    {"type": "text", "text": PROMPT},
                ],
            }
        ],
    )
    return json.loads(resp.content[0].text)


if __name__ == "__main__":
    for p in map(Path, sys.argv[1:]):
        print(json.dumps({"file": str(p), **extract(p)}, indent=2))
```

Correrlo contra un screenshot de una pantalla de login:

```bash
python screenshot_extract.py ./target-login.png
```

Output típico. Los strings que se ven en la UI se quedan como aparecen en pantalla. La nota es razonamiento del modelo, en español:

```json
{
  "file": "target-login.png",
  "urls": ["https://portal.example.com/login", "https://portal.example.com/reset"],
  "buttons": ["Sign in", "Forgot password?"],
  "form_fields": [
    {"label": "Username", "type": "text"},
    {"label": "Password", "type": "password"}
  ],
  "error_messages": ["User does not exist"],
  "cookies": [],
  "headers_visible": [],
  "notes": [
    "El mensaje de error 'User does not exist' distingue usernames inválidos de contraseñas inválidas. Confirma enumeración de usernames."
  ]
}
```

El campo `notes` es donde la capa semántica se gana el lugar: un pipeline de OCR de solo texto te habría dado el string "User does not exist", pero no la lectura de que habilita enumeración de usernames.

## 6. Análisis de documentos

Los documentos — contratos escaneados, políticas en PDF, facturas y, con autorización escrita explícita, documentos de identidad usados para simular pretextos de phishing — son la segunda gran categoría multimodal.

> **Límite ético:** Analiza solo documentos que caigan dentro del scope del contrato del engagement. Los documentos de identidad escaneados, en particular, tienen peso legal pesado: GDPR (UE), CCPA (California), LGPD (Brasil), PIPEDA (Canadá) y leyes equivalentes en la mayoría de las jurisdicciones exigen una base legal para el procesamiento. Si tu Statement of Work no autoriza de forma explícita el manejo de documentos de identidad personal, no los toques. Cuando estén autorizados, mantenlos en infraestructura controlada por el cliente y usa solo modelos locales.

### 6.1 PDFs de varias páginas

Claude Sonnet 5 acepta PDFs directo:

```python
import anthropic, base64, pathlib

client = anthropic.Anthropic()
pdf_b64 = base64.standard_b64encode(pathlib.Path("policy.pdf").read_bytes()).decode()

resp = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=4096,
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "document",
                 "source": {"type": "base64", "media_type": "application/pdf", "data": pdf_b64}},
                {"type": "text", "text":
                 "Resume la política de contraseñas de este documento. Extrae la longitud "
                 "mínima, los requisitos de complejidad, el período de rotación y las "
                 "restricciones de reutilización. Solo JSON."},
            ],
        }
    ],
)
```

Para procesamiento local, parte el PDF en imágenes de página con `pdftoppm`:

```bash
pdftoppm -r 200 -png policy.pdf policy-page
# Produce policy-page-1.png, policy-page-2.png, ...
```

Después alimenta cada imagen de página a tu modelo de visión local, en secuencia.

### 6.2 Extracción estructurada de facturas

Las facturas son un buen stress test porque tienen tablas semiestructuradas y texto legal de forma libre:

```python
INVOICE_PROMPT = """Extrae esta factura como JSON:
{
  "vendor":       {"name": str, "address": str, "tax_id": str|null},
  "customer":     {"name": str, "address": str},
  "invoice_no":   str,
  "invoice_date": str (ISO 8601),
  "due_date":     str|null,
  "line_items":   [ {"description": str, "qty": number, "unit_price": number, "total": number} ],
  "subtotal":     number,
  "tax":          number,
  "total":        number,
  "currency":     str (ISO 4217)
}
Devuelve solo JSON. Rechaza la extracción si la imagen no es una factura."""
```

La cláusula "Rechaza la extracción si la imagen no es una factura" es una válvula de seguridad contra basura que entra y basura que sale.

### 6.3 Documentos de identidad escaneados (solo autorizados)

Cuando un engagement de red team autoriza armar contenido de señuelo — por ejemplo, un memo interno de RR. HH. con forma plausible que imita los templates reales del cliente — un modelo de visión puede extraer la estructura de layout (fuente, bloque de encabezado, ubicación de la firma) sin reproducir jamás el PII real en tu pipeline.

La disciplina del prompt importa. En lugar de "extrae el nombre y la fecha de nacimiento de este documento", escribe:

```text
Describe el layout de este documento. Devuelve solo:
- Texto del encabezado (nombre de la organización, si es visible)
- Familia tipográfica (mejor estimación)
- Ubicación del placeholder de la fotografía (arriba a la izquierda, arriba a la derecha, etc.)
- Ubicación de la línea de firma
NO devuelvas nombres de personas, fechas de nacimiento, números de documento ni direcciones.
```

Esto te deja usar el modelo para inferir el template sin que datos personales salgan de las instalaciones del cliente.

## 7. Comparación de pipelines de OCR

No toda imagen necesita un LLM. El OCR clásico es más rápido, más barato y a veces más preciso, sobre todo en jobs de batch de alto volumen.

### 7.1 Tesseract

El motor de OCR open source de referencia, mantenido de forma activa por Google.

```bash
sudo apt install tesseract-ocr
tesseract screenshot.png stdout -l eng
```

Integración en Python:

```python
import pytesseract
from PIL import Image

text = pytesseract.image_to_string(Image.open("screenshot.png"))
```

Fortalezas: gratis, rápido (200–500 ms por imagen en la CPU de una laptop), del todo offline. Debilidades: le cuestan las fuentes de UI con anti-alias, los idiomas mezclados y el texto rotado. No hay capa semántica.

### 7.2 Google Cloud Vision API

Un servicio de OCR en la nube con accuracy excelente en texto impreso y accuracy razonable en escritura a mano.

```python
from google.cloud import vision

client = vision.ImageAnnotatorClient()
with open("scan.jpg", "rb") as f:
    image = vision.Image(content=f.read())
resp = client.text_detection(image=image)
print(resp.full_text_annotation.text)
```

Fortalezas: accuracy casi humana, bounding boxes estructurados, varios idiomas. Debilidades: costo (unos $1,50 por 1000 imágenes en el tier estándar), los datos salen de tu infraestructura, no hay razonamiento semántico.

### 7.3 OCR basado en LLM

Alimenta la imagen a un vision-language model con un prompt como "Transcribe cada carácter de esta imagen de forma textual."

Fortalezas: capa semántica (el modelo sabe qué está leyendo, no solo los glifos), tolera fuentes raras. Debilidades: más lento (2–5 s por imagen vía API, 5–30 s en local), costo más alto por imagen a alto volumen, texto alucinado de vez en cuando en inputs ambiguos.

### 7.4 Matriz de comparación

| Dimensión | Tesseract | GCP Vision | Basado en LLM |
|---|---|---|---|
| Accuracy (texto impreso) | Media | Muy alta | Alta |
| Accuracy (screenshots de UI) | Baja-media | Alta | Muy alta |
| Escritura a mano | Baja | Media | Media |
| Latencia | 200–500 ms | 500–1500 ms | 2–30 s |
| Costo por 1000 imágenes | Gratis | ~$1,50 | ~$3 hosteado, gratis en local |
| Privacidad | Local | Nube | Local o nube |
| Capa semántica | Ninguna | Ninguna | Sí |
| Structured output | Bounding boxes | Bounding boxes | Forma libre o JSON |
| Escalabilidad de batch | Excelente | Excelente | Pobre (rate limits) |

### 7.5 Regla práctica

- Extracción masiva de texto crudo de miles de documentos limpios: Tesseract.
- Extracción masiva de texto con alta accuracy exigida, y la nube está permitida: GCP Vision.
- Extraer significado, no solo texto — "¿qué está mostrando esta pantalla?": LLM.
- Híbrido: corre Tesseract primero y después alimenta al LLM tanto el texto crudo del OCR como la imagen, para que razone.

El enfoque híbrido suele ser el que gana:

```python
import pytesseract
from PIL import Image
import anthropic, base64, pathlib

path = pathlib.Path("dashboard.png")
raw = pytesseract.image_to_string(Image.open(path))

client = anthropic.Anthropic()
b64 = base64.standard_b64encode(path.read_bytes()).decode()

resp = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image",
             "source": {"type": "base64", "media_type": "image/png", "data": b64}},
            {"type": "text",
             "text": f"Pre-extracción OCR:\n{raw}\n\nUsando el texto crudo Y la imagen, resume el estado del dashboard en JSON."},
        ],
    }],
)
```

El OCR crudo le da al LLM un ancla fuerte para la accuracy a nivel de carácter. La imagen le da el contexto semántico.

## 8. Ejemplo completo — screenshot de Burp a narrativa de finding

El cuello de botella clásico de escribir reports: tienes una carpeta de screenshots de Burp, cada uno muestra un request/response vulnerable, y necesitas redactar la narrativa del finding de cada uno.

```python
"""
burp_to_finding.py
Convierte un screenshot de Burp Suite en un borrador de finding en Markdown.
"""
import base64
import pathlib
import sys

import anthropic

TEMPLATE = """Eres un penetration tester senior que escribe findings para un report de cliente.

La imagen adjunta es un screenshot de Burp Suite Repeater que muestra un solo
request y response HTTP. Produce un finding en Markdown con:

## Título
Título de la vulnerabilidad, de una línea.

## Severity
Base score de CVSS 4.0 y rating de severity (Critical/High/Medium/Low/Informational).

## Descripción
Explicación de dos párrafos de la vulnerabilidad en el request/response.

## Evidencia
Extracto textual de los headers o del body relevantes del request y del response.

## Reproducción
Pasos numerados para reproducir, solo con lo que se ve en el screenshot.

## Impact
Impact concreto en el contexto de un despliegue enterprise típico.

## Remediación
Pasos de remediación específicos que referencien el stack tecnológico real visible en la respuesta.

Usa solo lo que es visible en el screenshot. Si el screenshot no muestra una
vulnerabilidad, escribe `NO_FINDING` en una sola línea."""


def main(image_path: str) -> None:
    p = pathlib.Path(image_path)
    b64 = base64.standard_b64encode(p.read_bytes()).decode()
    media = "image/png" if p.suffix.lower() == ".png" else "image/jpeg"

    client = anthropic.Anthropic()
    resp = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=3000,
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "image",
                     "source": {"type": "base64", "media_type": media, "data": b64}},
                    {"type": "text", "text": TEMPLATE},
                ],
            }
        ],
    )
    print(resp.content[0].text)


if __name__ == "__main__":
    main(sys.argv[1])
```

Corrida de ejemplo contra un screenshot que ya muestra un request con `X-Original-URL: /admin` saltándose un chequeo de auth. El script no construye el request: lee la evidencia que ya capturaste.

```bash
python burp_to_finding.py ./evidence/screenshot-042.png > draft-finding-042.md
```

Produce un borrador que editas, en lugar de escribirlo desde cero. Ahorro de tiempo realista: 15 minutos de redacción por finding bajan a 2–3 minutos de edición.

> **Pentester Insight:** Nunca publiques un finding redactado por un LLM sin review humana. El modelo puede, y va a, confabular números de CVE, enunciar mal vectores de CVSS e inventar referencias a RFC con confianza. El valor está en el primer borrador, no en la versión final.

## 9. Ejemplo completo — diagrama de red a inventario estructurado

El otro caso de alto valor: el cliente entrega un diagrama de topología de red (a menudo un export de Visio o un PDF de su equipo de arquitectura) y necesitas extraer cada nodo a una lista estructurada de targets.

```python
"""
diagram_to_inventory.py
Convierte un diagrama de topología de red en un inventario JSON.
"""
import base64
import json
import pathlib
import sys

import anthropic

PROMPT = """Estás analizando un diagrama de topología de red que entregó el cliente.
Extrae cada host, cada segmento de red y cada arista en JSON estricto:

{
  "segments": [
    {
      "name":   str,          # p. ej. "DMZ", "Corp LAN", "OT VLAN"
      "cidr":   str|null,     # si es visible en el diagrama
      "hosts":  [
        {
          "hostname": str|null,
          "ip":       str|null,
          "role":     str,     # p. ej. "web server", "domain controller"
          "os":       str|null # si es visible
        }
      ]
    }
  ],
  "edges": [
    {"from": str, "to": str, "protocol": str|null, "port": int|null}
  ],
  "notes": [str]
}

Incluye solo información visible en el diagrama. No adivines.
Si un hostname o una IP no se puede leer, deja el campo en null."""


def main(path: str) -> None:
    p = pathlib.Path(path)
    media = "image/png" if p.suffix.lower() == ".png" else "image/jpeg"
    b64 = base64.standard_b64encode(p.read_bytes()).decode()

    client = anthropic.Anthropic()
    resp = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=4096,
        messages=[{
            "role": "user",
            "content": [
                {"type": "image",
                 "source": {"type": "base64", "media_type": media, "data": b64}},
                {"type": "text", "text": PROMPT},
            ],
        }],
    )
    inventory = json.loads(resp.content[0].text)
    print(json.dumps(inventory, indent=2))


if __name__ == "__main__":
    main(sys.argv[1])
```

Fragmento de output de un diagrama real de entrenamiento:

```json
{
  "segments": [
    {"name": "DMZ", "cidr": "10.0.10.0/24", "hosts": [
      {"hostname": "web01", "ip": "10.0.10.10", "role": "Reverse proxy (nginx)", "os": null},
      {"hostname": "web02", "ip": "10.0.10.11", "role": "Web application server", "os": "Ubuntu 22.04"}
    ]},
    {"name": "Internal", "cidr": "10.0.20.0/24", "hosts": [
      {"hostname": "dc01", "ip": "10.0.20.10", "role": "Domain controller", "os": "Windows Server 2022"},
      {"hostname": "fs01", "ip": "10.0.20.20", "role": "File server", "os": null}
    ]}
  ],
  "edges": [
    {"from": "web02", "to": "dc01", "protocol": "LDAP", "port": 389}
  ],
  "notes": ["No se ve una arista de entrada desde Internet hacia el segmento Internal"]
}
```

Este JSON se importa con facilidad en tu harness de enumeración: puedes recorrerlo y lanzar scans de Nmap acotados a cada host, por rol, dentro del scope autorizado. El diagrama no autoriza hosts que no estén en el engagement.

## 10. CAPTCHA — un tema solo defensivo

Los CAPTCHA (Completely Automated Public Turing test to tell Computers and Humans Apart) existen para distinguir humanos de agentes automatizados. Saltear CAPTCHAs de terceros — reCAPTCHA, hCaptcha, Cloudflare Turnstile, Arkose Labs — en sistemas que no son tuyos o que no te autorizaron de forma explícita a testear es:

- Una violación de los términos de servicio del operador.
- Con frecuencia una violación de las leyes de uso indebido de computadoras (CFAA en Estados Unidos, Computer Misuse Act en el Reino Unido, estatutos equivalentes en la mayoría de las jurisdicciones).
- Fuera del scope de cualquier penetration test legítimo, salvo que el CAPTCHA en cuestión sea el activo testeado.

Esta lección no trae un walkthrough para saltear CAPTCHAs de terceros. Lo que sigue se limita a investigación defensiva: ayudar a un cliente a evaluar la robustez de su propia implementación de CAPTCHA.

### 10.1 Escenarios defensivos legítimos

- Un cliente opera un CAPTCHA custom in-house (clasificación de imágenes, distorsión de texto o challenge-response). Quiere saber si un pipeline de ML modesto puede vencerlo. Esto es investigación de seguridad in-scope.
- Un vendor te pide evaluar su producto nuevo de CAPTCHA antes del lanzamiento al mercado.
- Investigación académica con aprobación de un IRB.

### 10.2 Cómo modelan el problema los atacantes (nivel alto)

Los CAPTCHA de clasificación de imágenes se atacan en general con un clasificador supervisado: históricamente un ResNet o un MobileNet afinado sobre un dataset etiquetado, armado resolviendo el CAPTCHA target una y otra vez a mano o vía un servicio pago de resolución. El clasificador aprende la fuente, el patrón de distorsión o la familia de objetos de ese CAPTCHA y alcanza alta accuracy sobre la misma distribución.

Conclusión defensiva: cualquier CAPTCHA cuya distribución de challenges sea lo bastante chica como para enumerarse cae ante un clasificador afinado en cuestión de días. La frecuencia de rotación, la diversidad de la distribución y la dificultad adaptativa son las palancas de diseño.

### 10.3 Un bosquejo de evaluación defensiva (nivel alto)

Sin código que saltee un CAPTCHA real:

1. Junta un dataset etiquetado de tu propio CAPTCHA bajo un harness de test (nunca de usuarios de producción).
2. Parte 80/10/10 en train/val/test.
3. Afina un clasificador chico (ResNet-18, unos 11M de parámetros) sobre el set de train.
4. Mide la accuracy de test. Cualquier cosa por encima de ~30 % en un challenge de 5 clases indica que el CAPTCHA no sirve contra un atacante motivado.
5. Itera el diseño del CAPTCHA: sube la entropía, agrega ruido adversarial, integra señales de comportamiento.

Esa es toda la discusión de CAPTCHAs en este curso. El resto del módulo se concentra en la mejora defensiva, no en el bypass ofensivo.

## 11. Inputs de audio — Whisper para transcripción de voicemail

Los engagements autorizados de red team a veces incluyen vishing (voice phishing) como vector de ataque aprobado. El vishing produce grabaciones de voicemail que hay que transcribir para evidencia y para refinar el script de forma iterativa.

### 11.1 OpenAI Whisper

Whisper es un modelo open source de automatic speech recognition (ASR) publicado por OpenAI en 2022. El checkpoint `large-v3` es el líder actual de accuracy. Las variantes más chicas (`base`, `small`, `medium`) cambian accuracy por velocidad.

```bash
pip install openai-whisper
# En Kali:
sudo apt install ffmpeg
```

```python
import whisper

model = whisper.load_model("large-v3")
result = model.transcribe("voicemail-2026-04-14.wav", language="en")
print(result["text"])
```

Corre entero en local. En una CPU moderna, `large-v3` transcribe a unas 3–5× el tiempo real. En una GPU, a 30–50× el tiempo real.

### 11.2 Faster-Whisper

Una reimplementación basada en CTranslate2, con 4× de throughput a la misma accuracy:

```bash
pip install faster-whisper
```

```python
from faster_whisper import WhisperModel

model = WhisperModel("large-v3", device="cuda", compute_type="int8_float16")
segments, info = model.transcribe("voicemail.wav", beam_size=5)
for seg in segments:
    print(f"[{seg.start:.1f}s -> {seg.end:.1f}s] {seg.text}")
```

### 11.3 Manejo de evidencia

Bajo un engagement de vishing autorizado, cada grabación puede quedar sujeta a leyes de consentimiento de dos partes (California, Florida, Illinois, la mayoría de los estados miembro de la UE) incluso cuando la campaña misma está autorizada por contrato. Trata las grabaciones como evidencia sensible:

- Guárdalas en volúmenes cifrados controlados por el cliente.
- No mandes grabaciones a APIs de transcripción hosteadas salvo que el SOW lo permita de forma explícita.
- Consérvalas solo por el tiempo que exige el report del engagement.
- Bórralas según el calendario de las Rules of Engagement.

## 12. Inputs de video

El manejo de video es la frontera multimodal más nueva. Dos líderes actuales del mercado:

- **Gemini 1.5 Pro / 2.5.** Acepta videos de hasta 1 hora directo vía la Files API, muestreando un frame por segundo por defecto. Excelente para revisar grabaciones de pantalla de una aplicación target.
- **Claude Sonnet 5.** Acepta videos como secuencias de frames extraídos. Alimenta frames pre-muestreados (típico: 1 frame por segundo, bajados a 768 px en el borde largo) y el modelo razona a través de ellos.

Ejemplo: extrae frames de una grabación de pantalla autorizada de la app target y aliméntalos a Claude. En el material original el comando de `ffmpeg` y el import de Python venían pegados en la misma línea. Van separados.

```bash
ffmpeg -i target-app-demo.mp4 -vf fps=1 -q:v 2 frames/frame-%04d.jpg
```

```python
import base64, pathlib, anthropic

client = anthropic.Anthropic()
frames = sorted(pathlib.Path("frames").glob("frame-*.jpg"))[:20]  # tope de 20 frames

content = []
for f in frames:
    b64 = base64.standard_b64encode(f.read_bytes()).decode()
    content.append({"type": "image",
                    "source": {"type": "base64", "media_type": "image/jpeg", "data": b64}})
content.append({"type": "text",
                "text": "Describe la secuencia de acciones del usuario en estos frames. Enfócate en cualquier flujo de autenticación."})

resp = client.messages.create(model="claude-sonnet-5", max_tokens=1500, messages=[{"role": "user", "content": content}])
print(resp.content[0].text)
```

Advertencia práctica: 20 frames a ~1600 tokens cada uno son 32.000 tokens antes de tu prompt. Razonar sobre video es caro. Muestrea de forma escasa.

## 13. Un skill de Claude Code — "Describe esta UI vulnerable"

Los skills de Claude Code son instrucciones empaquetadas que una sesión de Claude Code puede invocar. Un skill multimodal mínimo para pentesters:

`~/.claude/skills/vuln-ui-triage/SKILL.md`

```markdown
---
name: vuln-ui-triage
description: Analiza un screenshot de la UI de una aplicación target y describe detalles relevantes para la seguridad (superficie de auth, campos de formulario, mensajes de error, artefactos de sesión). Úsalo cuando el usuario diga "mira este screenshot" o adjunte una imagen de UI.
---

# Skill de triage de UI vulnerable

Cuando se invoque, haz lo siguiente:

1. Confirma que la imagen adjunta es un screenshot de UI (no un diagrama de red ni un documento).
2. Extrae y devuelve, en Markdown:
   - **URL** visible en la barra de direcciones
   - **Superficie de autenticación** (formulario de login, redirect de SSO, prompt de MFA)
   - **Campos de formulario** con sus labels y tipos de input
   - **Mensajes de error** textuales
   - **Artefactos de sesión** visibles en paneles de DevTools (cookies, localStorage, sessionStorage)
   - **Tecnologías del lado cliente** inferidas de marcadores de framework visibles
   - **Hipótesis de vulnerabilidad** — tres hipótesis ordenadas de qué testear después

No fabriques. Si una categoría no es visible, márcala "No visible".
```

Invócalo desde una sesión de Claude Code:

```text
> /vuln-ui-triage
[adjunta ./target-login.png]
```

Las instrucciones del skill pasan a ser parte del contexto de la sesión, y el LLM multimodal produce el output de triage directo. Los skills se componen: encadena `vuln-ui-triage` con un skill de redacción de findings para ir de screenshot a borrador de finding en dos prompts.

## 14. image-triage.py completo

Un script de triage en batch con forma de producción. Lee un directorio de screenshots, manda cada uno por Claude Sonnet 5 o por un modelo local Llama 3.2 Vision, y escribe un resumen JSON por archivo de input.

```python
"""
image-triage.py — Triage multimodal en batch de screenshots de pentest.

Uso:
    python image-triage.py --input ./screenshots --output ./triage
    python image-triage.py --input ./screenshots --output ./triage --backend local
    python image-triage.py --input ./screenshots --output ./triage --model claude-sonnet-5

Entorno:
    ANTHROPIC_API_KEY   Obligatorio para --backend hosted (el default).
    OLLAMA_HOST         Opcional, default http://localhost:11434 para --backend local.
"""
from __future__ import annotations

import argparse
import base64
import json
import os
import sys
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path
from typing import Any

SUPPORTED = {".png", ".jpg", ".jpeg", ".webp", ".gif"}
PROMPT = """Estás haciendo triage de un screenshot de un engagement de penetration testing.

Devuelve JSON ESTRICTO con esta forma (sin fences de markdown):

{
  "category":     "ui" | "tool_output" | "diagram" | "document" | "other",
  "summary":      "una oración, en español",
  "urls":         [ "..." ],
  "hostnames":    [ "..." ],
  "ips":          [ "..." ],
  "tech_stack":   [ "..." ],
  "credentials_visible":  bool,
  "pii_visible":  bool,
  "vulnerability_hints":  [ "..." ],
  "confidence":   0.0 a 1.0
}

Reglas:
- Incluye solo valores que puedas ver. No inventes hostnames, IPs ni URLs.
- credentials_visible es true solo si una contraseña en texto plano, una API key o un token es visible de forma legible.
- pii_visible es true solo si un nombre de persona aparece junto con una fecha de nacimiento, un número de documento o una dirección.
- Prefiere valores de confidence conservadores (0.4-0.7) salvo que la imagen sea inequívoca."""


def encode_image(path: Path) -> tuple[str, str]:
    ext = path.suffix.lower()
    media = "image/jpeg" if ext in (".jpg", ".jpeg") else f"image/{ext.lstrip('.')}"
    b64 = base64.standard_b64encode(path.read_bytes()).decode("utf-8")
    return media, b64


def triage_hosted(path: Path, model: str) -> dict[str, Any]:
    import anthropic
    client = anthropic.Anthropic()
    media, b64 = encode_image(path)
    resp = client.messages.create(
        model=model,
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": [
                {"type": "image",
                 "source": {"type": "base64", "media_type": media, "data": b64}},
                {"type": "text", "text": PROMPT},
            ],
        }],
    )
    return json.loads(resp.content[0].text)


def triage_local(path: Path, model: str) -> dict[str, Any]:
    from openai import OpenAI
    base = os.environ.get("OLLAMA_HOST", "http://localhost:11434") + "/v1"
    client = OpenAI(base_url=base, api_key="ollama")
    media, b64 = encode_image(path)
    resp = client.chat.completions.create(
        model=model,
        temperature=0.0,
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": PROMPT},
                {"type": "image_url",
                 "image_url": {"url": f"data:{media};base64,{b64}"}},
            ],
        }],
    )
    return json.loads(resp.choices[0].message.content)


def triage_one(path: Path, backend: str, model: str) -> dict[str, Any]:
    started = time.time()
    try:
        payload = triage_hosted(path, model) if backend == "hosted" else triage_local(path, model)
        status = "ok"
        error = None
    except Exception as exc:
        payload = {}
        status = "error"
        error = f"{type(exc).__name__}: {exc}"
    return {
        "file": str(path),
        "backend": backend,
        "model": model,
        "elapsed_s": round(time.time() - started, 2),
        "status": status,
        "error": error,
        "triage": payload,
    }


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--input", required=True, type=Path)
    ap.add_argument("--output", required=True, type=Path)
    ap.add_argument("--backend", choices=["hosted", "local"], default="hosted")
    ap.add_argument("--model", default=None)
    ap.add_argument("--workers", type=int, default=4)
    args = ap.parse_args()

    if args.model is None:
        args.model = "claude-sonnet-5" if args.backend == "hosted" else "llama3.2-vision:11b"

    args.output.mkdir(parents=True, exist_ok=True)
    files = [p for p in args.input.rglob("*") if p.is_file() and p.suffix.lower() in SUPPORTED]
    if not files:
        print(f"[!] no hay imágenes bajo {args.input}", file=sys.stderr)
        return 1

    print(f"[*] triage de {len(files)} imágenes vía {args.backend} ({args.model})")
    with ThreadPoolExecutor(max_workers=args.workers) as ex:
        futures = {ex.submit(triage_one, p, args.backend, args.model): p for p in files}
        done = 0
        for fut in as_completed(futures):
            done += 1
            result = fut.result()
            out_path = args.output / (Path(result["file"]).stem + ".json")
            out_path.write_text(json.dumps(result, indent=2))
            print(f"[{done}/{len(files)}] {result['status']:5} {result['file']} -> {out_path}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Correrlo:

```bash
# Hosteado (default):
export ANTHROPIC_API_KEY=sk-ant-...
python image-triage.py --input ./evidence/screenshots --output ./evidence/triage

# Local (offline):
python image-triage.py --input ./evidence/screenshots --output ./evidence/triage \
    --backend local --model llama3.2-vision:11b --workers 1
```

El modo local usa `workers=1` porque el modelo corre en tu única GPU y los requests en paralelo igual se encolan.

### 14.1 Cómo extender el script

Extensiones que los estudiantes suelen construir encima de esta base:

- Agrega un modo `--dedupe` que hashee cada imagen y saltee los casi-duplicados.
- Emite un resumen CSV junto al JSON por archivo, para importarlo fácil a un template de report.
- Agrega una pre-pasada de Tesseract para que el modelo reciba el texto crudo del OCR como contexto extra.
- Conéctalo a un hook de pre-commit que redacte credenciales de forma automática antes de que una imagen entre a un repositorio de reports.

## 15. Detectar imágenes generadas por IA

El otro lado del pentesting multimodal: un atacante (o un redactor de reports comprometido) puede meter una imagen fabricada en tu pipeline de evidencia. La procedencia y la detección importan.

### 15.1 C2PA — Content Credentials

La Coalition for Content Provenance and Authenticity (C2PA) define un estándar abierto de metadata de procedencia firmada de forma criptográfica, embebida en archivos de imagen y de video. C2PA tiene el respaldo de Adobe, Microsoft, Intel, ARM, la BBC y (desde 2024) OpenAI y Google.

Un manifest C2PA dentro de un PNG o un JPEG contiene:

- El software productor (`urn:uuid:...`) y su certificado de firma.
- Una lista de ediciones (`c2pa.actions`): creado, recortado, ajustado de color, generado por IA.
- Un hash de los datos de píxeles en el momento de la firma.
- Identidad opcional del actor, vía una cadena X.509 enlazada.

Inspecciona un archivo:

```bash
# Instala c2patool de la Content Authenticity Initiative:
cargo install c2patool

c2patool ./screenshot.png
# {
#   "manifest_store": { ... procedencia firmada ... }
# }
```

Si una entrada de `c2pa.actions` contiene `"action": "c2pa.ai_generated"` o `"c2pa.ai_derived"`, la imagen la produjo un modelo que participa del estándar.

### 15.2 Watermarks de modelo

Varios generadores comerciales de imágenes embeben watermarks invisibles:

- **SynthID (Google DeepMind).** Watermark en el dominio de la frecuencia para Imagen 3 y Veo. La detección exige la API de inferencia de Google.
- **Stable Signature (Meta).** Variante de Stable Diffusion afinada que embebe una firma de 48 bits por usuario en cada imagen generada.
- **DALL-E 3 de OpenAI.** Metadata C2PA por defecto. Se puede quitar re-codificando.

Cualquier watermark que sobrevive a la re-codificación sobrevive al ataque trivial de "hacer screenshot de la imagen". La metadata C2PA no: un screenshot de una imagen firmada con C2PA son píxeles sin firma.

### 15.3 Detectores estadísticos

Los detectores basados en clasificadores (por ejemplo AI or Not, Hive AI, y detectores académicos como DIRE) miran los patrones residuales que dejan los modelos de difusión. Sirven como una señal, pero la tasa de false positives sobre fotografías reales comprimidas en JPEG no es trivial (5–15 % según el dataset). Nunca trates el veredicto de un solo clasificador como verdad de base.

### 15.4 Qué significa esto para un pentester

- Si un cliente afirma que una imagen es auténtica, pide el original firmado con C2PA desde el dispositivo de captura, no una copia re-codificada.
- Si recibes evidencia presunta durante un engagement de red team, trata las imágenes sin firma con el mismo escepticismo que un header de email sin firma.
- Si produces imágenes tú (por ejemplo, mock-ups para pretextos de social engineering), considera firmarlas como `c2pa.ai_generated` para quedarte dentro de los límites acordados con el cliente.

## 16. Ejercicios

### Guiados

El walk-through está en el Apéndice A del curso.

1. **Enumeración de página de login.** Con el script `screenshot_extract.py` de la sección 5, córrelo contra tres screenshots de páginas de login que hayas capturado en tu propio lab. Confirma que el output JSON distingue bien los tres mensajes de error distintos que insinúan enumeración de usernames.
2. **Diagrama a inventario.** Dibuja a mano en papel un diagrama de red de cinco hosts, fotografíalo y pásalo por `diagram_to_inventory.py`. Compara el JSON extraído con tu ground truth y anota cualquier error. Repite con la misma imagen a 640×480, 1280×960 y 2560×1920 para observar la curva de accuracy frente a costo en tokens.
3. **Local contra hosteado, cara a cara.** Toma veinte screenshots de tu carpeta de evidencia de Burp y corre `image-triage.py` una vez con `--backend hosted` y una vez con `--backend local`. Haz diff de los outputs JSON. Mide el wall-clock y el costo.

### Independientes

1. **Pipeline de OCR híbrido.** Extiende `image-triage.py` para que, cuando se pase `--hybrid`, corra Tesseract sobre cada imagen primero e inyecte el texto crudo del OCR en el prompt como contexto adicional. Mide si el JSON resultante tiene menos hostnames alucinados.
2. **Video a storyboard.** Dada una grabación de pantalla de 60 segundos de un flujo de login, usa `ffmpeg` para muestrear un frame por segundo y alimenta los frames a Claude Sonnet 5 con un prompt que devuelva un storyboard numerado de las acciones del usuario. Compáralo con una transcripción manual.
3. **Auditor de procedencia.** Escribe un script que recorra tu carpeta de evidencia, corra `c2patool` sobre cada imagen y marque cualquier archivo que haya sido re-codificado después de su última firma C2PA. Produce un report Markdown con una tabla de los archivos marcados.

## 17. Resumen y pasos siguientes

La IA multimodal pasó de curiosidad de investigación a herramienta diaria de pentest en el lapso de dos años. El workflow que acabas de armar — screenshots y diagramas entran, JSON estructurado sale, findings redactados en Markdown, todo ejecutable en local cuando el engagement lo exige — es ahora una expectativa de base para un tester senior, no un diferenciador.

Ideas clave de esta lección:

- Los vision-language models son arquitecturas de tres bloques (encoder + projector + LLM), y entender el costo en tokens por imagen es esencial para presupuestar.
- Los modelos hosteados (Claude Sonnet 5, GPT-4o) entregan la accuracy cruda más alta. Los modelos locales (Llama 3.2 Vision, Qwen2-VL, MiniCPM-V) entregan privacidad y repetibilidad.
- La estrategia correcta de OCR es híbrida: Tesseract para el ancla a nivel de carácter, un LLM para el razonamiento semántico.
- Los límites éticos no se negocian: los CAPTCHA son un tema solo defensivo, y el análisis de documentos queda acotado a artefactos autorizados por el cliente.
- La procedencia (C2PA, watermarks) es ahora parte del contrato de manejo de evidencia, tanto cuando recibes imágenes como cuando las produces.

Siguiente lección: Lección 11 — Retrieval-Augmented Generation for Vulnerability Research. Vas a armar un pipeline RAG local sobre la base de CVE, los feeds de NIST NVD, los archivos de exploit-db y los advisories de vendors, y después conectarlo a un skill de Claude Code para que "busca bugs conocidos en esta versión de software" sea un prompt de una línea.
