# Preparar el entorno de trabajo (paso a paso detallado)

Guía para dejar listo un entorno de Python con acceso a modelos de lenguaje. Sirve tanto para usar el SDK de un proveedor (Anthropic, OpenAI, Google) como para usar **OpenRouter**, que da acceso a modelos de varios proveedores con una sola clave y tiene opciones gratuitas.

Para una versión corta, consulta [entorno-rapido.md](entorno-rapido.md).

---

## Índice

1. [Requisitos](#1-requisitos)
2. [Elegir tu ruta: proveedor directo u OpenRouter](#2-elegir-tu-ruta-proveedor-directo-u-openrouter)
3. [Verificar Python](#3-verificar-python)
4. [Crear la carpeta del proyecto](#4-crear-la-carpeta-del-proyecto)
5. [Crear el entorno virtual](#5-crear-el-entorno-virtual)
6. [Activar el entorno](#6-activar-el-entorno)
7. [Instalar las dependencias](#7-instalar-las-dependencias)
8. [Crear `.gitignore`](#8-crear-gitignore)
9. [Obtener la clave de API](#9-obtener-la-clave-de-api)
10. [Crear el archivo `.env`](#10-crear-el-archivo-env)
11. [Verificar las claves](#11-verificar-las-claves)
12. [Primer mensaje al modelo](#12-primer-mensaje-al-modelo)
13. [Guardar las dependencias](#13-guardar-las-dependencias)
14. [Conectar VS Code](#14-conectar-vs-code)
15. [Solución de problemas](#15-solución-de-problemas)
16. [Antes de subir a GitHub](#16-antes-de-subir-a-github)

---

## 1. Requisitos

- **Python 3.10 o superior.**
- **VS Code** con la extensión oficial de Python (publicada por Microsoft).
- **Una clave de API**: de un proveedor directo o de OpenRouter.
- Una terminal: Terminal o iTerm en macOS, PowerShell o Símbolo del sistema en Windows, cualquier shell en Linux.

Por qué importan estas piezas:

- **Python** es el intérprete que ejecuta tu código. VS Code no lo incluye; la extensión solo se comunica con el Python que tengas instalado.
- **La clave de API** identifica tu cuenta. Cada llamada puede tener costo, así que trátala como una contraseña.

---

## 2. Elegir tu ruta: proveedor directo u OpenRouter

| | **Ruta A: proveedor directo** | **Ruta B: OpenRouter** |
|---|---|---|
| Qué instalas | SDK del proveedor (`anthropic`, `openai` o `google-genai`) | `openai` apuntando a OpenRouter, o `requests` |
| Clave | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` o `GOOGLE_API_KEY` | `OPENROUTER_API_KEY` |
| Costo | Pago por uso | Hay modelos gratuitos (sufijo `:free`) |
| Modelos | Solo los de ese proveedor | Muchos proveedores con la misma API |
| Cuándo elegirla | Producción, funciones exclusivas del proveedor, datos sensibles | Aprender, experimentar, comparar modelos sin pagar |

Puedes seguir ambas rutas en el mismo proyecto. Los pasos 1 a 8 son iguales; cambian solo las dependencias, la clave y el código del primer mensaje.

---

## 3. Verificar Python

Comprueba que tienes Python 3.10 o superior.

```bash
python3 --version      # macOS / Linux
py --version           # Windows
```

Si la versión es menor a 3.10, instala una más reciente desde **python.org/downloads**.

**macOS:** usa siempre `python3`. El comando `python` a secas puede no existir o apuntar a una versión antigua.

**Windows:** durante la instalación, activa **Add Python to PATH**. Si no lo hiciste, vuelve a ejecutar el instalador y márcala.

---

## 4. Crear la carpeta del proyecto

```bash
mkdir -p ~/Documentos/agentes-ia
cd ~/Documentos/agentes-ia
```

Trabaja siempre desde esta carpeta. Todos los comandos siguientes se ejecutan aquí.

---

## 5. Crear el entorno virtual

Un entorno virtual es una carpeta dentro del proyecto con su propio Python y sus propias librerías. Así, las versiones que instales para un proyecto no afectan a otro.

```bash
python3 -m venv .venv        # macOS / Linux
py -m venv .venv             # Windows
```

El nombre `.venv` es el convencional. El punto inicial marca que es una carpeta auxiliar.

---

## 6. Activar el entorno

Hazlo cada vez que abras una terminal nueva dentro del proyecto.

```bash
source .venv/bin/activate              # macOS / Linux
.venv\Scripts\Activate.ps1             # Windows (PowerShell)
.venv\Scripts\activate.bat             # Windows (Símbolo del sistema)
```

Debe aparecer `(.venv)` al inicio del prompt. Para salir, escribe `deactivate`.

Verifica que `pip` pertenece al entorno:

```bash
python -m pip --version
```

La ruta que muestra debe contener `.venv`.

---

## 7. Instalar las dependencias

Elige según tu ruta.

**Ruta A: proveedor directo** (instala solo el SDK que vayas a usar):

```bash
python -m pip install anthropic python-dotenv       # Anthropic (Claude)
python -m pip install openai python-dotenv          # OpenAI (GPT)
python -m pip install google-genai python-dotenv    # Google (Gemini)
```

**Ruta B: OpenRouter** (con SDK de OpenAI, o sin SDK):

```bash
python -m pip install openai python-dotenv          # recomendado
python -m pip install requests python-dotenv        # alternativa sin SDK
```

Puedes instalar varias si quieres comparar las dos formas.

---

## 8. Crear `.gitignore`

El archivo `.gitignore` indica a Git qué no debe subir al repositorio. Como mínimo, excluye el entorno y las claves.

```bash
printf ".venv/\n.env\n" > .gitignore
cat .gitignore
```

Debe mostrar:

```
.venv/
.env
```

---

## 9. Obtener la clave de API

**Ruta A: proveedor directo**

| Proveedor | Dónde obtenerla | Variable |
|---|---|---|
| Anthropic | console.anthropic.com → API Keys | `ANTHROPIC_API_KEY` |
| OpenAI | platform.openai.com → API keys | `OPENAI_API_KEY` |
| Google | Google AI Studio → Get API key | `GOOGLE_API_KEY` |

**Ruta B: OpenRouter**

1. Entra a [openrouter.ai](https://openrouter.ai) y crea una cuenta.
2. Ve a **Keys** y crea una clave. Empieza por `sk-or-v1-`.
3. Si quieres usar modelos gratuitos, revísalos en [openrouter.ai/models](https://openrouter.ai/models) filtrando por **Free**. Su identificador termina en `:free`.

> Los paneles suelen mostrar la clave completa **solo al crearla**. Cópiala en ese momento.

---

## 10. Crear el archivo `.env`

Crea un archivo llamado exactamente `.env` en la raíz del proyecto. Sin comillas y sin espacios alrededor del `=`.

**Ruta A:**

```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
```

**Ruta B:**

```env
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx
```

Puedes tener varias claves en el mismo `.env`, una por línea.

> **Nunca subas `.env` a GitHub.** Si una clave se expone, revócala en el panel del proveedor y genera otra.

---

## 11. Verificar las claves

Crea `verificar_claves.py`. Este script solo confirma que la variable existe; no imprime el valor.

```python
import os
from dotenv import load_dotenv

load_dotenv()

claves = ["ANTHROPIC_API_KEY", "OPENAI_API_KEY", "GOOGLE_API_KEY", "OPENROUTER_API_KEY"]
for clave in claves:
    valor = os.environ.get(clave, "")
    estado = "OK" if valor else "FALTA"
    print(f"{clave}: {estado}")
```

```bash
python verificar_claves.py
```

Salida esperada: la línea de la clave que usas debe decir `OK`. Las demás pueden decir `FALTA` sin problema.

---

## 12. Primer mensaje al modelo

El mismo prompt para ambas rutas:

> *Explica en tres frases qué hace un agente de IA cuando usa una herramienta.*

### Ruta A: Anthropic

Crea `primer_mensaje.py`:

```python
from anthropic import Anthropic

cliente = Anthropic()  # lee ANTHROPIC_API_KEY del entorno

respuesta = cliente.messages.create(
    model="claude-sonnet-5",   # confirma el nombre vigente en la documentación
    max_tokens=256,            # obligatorio en Anthropic
    messages=[{
        "role": "user",
        "content": "Explica en tres frases qué hace un agente de IA cuando usa una herramienta.",
    }],
)

print(respuesta.content[0].text)
print(f"Entrada: {respuesta.usage.input_tokens} / Salida: {respuesta.usage.output_tokens}")
```

### Ruta A: OpenAI

```python
from openai import OpenAI

cliente = OpenAI()  # lee OPENAI_API_KEY del entorno

respuesta = cliente.responses.create(
    model="gpt-5.6-terra",     # confirma el nombre vigente
    input="Explica en tres frases qué hace un agente de IA cuando usa una herramienta.",
)

print(respuesta.output_text)
print(f"Entrada: {respuesta.usage.input_tokens} / Salida: {respuesta.usage.output_tokens}")
```

### Ruta B: OpenRouter (SDK de OpenAI)

Crea `primer_mensaje.py`:

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

MODELO = "recursive/flint-v1.0:free"   # identificador exacto de openrouter.ai/models

cliente = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)

respuesta = cliente.chat.completions.create(
    model=MODELO,
    messages=[{
        "role": "user",
        "content": "Explica en tres frases qué hace un agente de IA cuando usa una herramienta.",
    }],
    max_tokens=256,
)

print(respuesta.choices[0].message.content)
print(f"Entrada: {respuesta.usage.prompt_tokens} / Salida: {respuesta.usage.completion_tokens}")
```

### Ruta B: OpenRouter (HTTP directo, sin SDK)

```python
import os
import requests
from dotenv import load_dotenv

load_dotenv()

respuesta = requests.post(
    "https://openrouter.ai/api/v1/chat/completions",
    headers={
        "Authorization": f"Bearer {os.getenv('OPENROUTER_API_KEY')}",
        "Content-Type": "application/json",
    },
    json={
        "model": "recursive/flint-v1.0:free",
        "messages": [{
            "role": "user",
            "content": "Explica en tres frases qué hace un agente de IA cuando usa una herramienta.",
        }],
        "max_tokens": 256,
    },
    timeout=60,
)

if respuesta.status_code != 200:
    raise SystemExit(f"Error {respuesta.status_code}: {respuesta.text}")

datos = respuesta.json()
print(datos["choices"][0]["message"]["content"])
print(f"Entrada: {datos['usage']['prompt_tokens']} / Salida: {datos['usage']['completion_tokens']}")
```

Ejecuta cualquiera con:

```bash
python primer_mensaje.py
```

### Cambiar de modelo

En OpenRouter, cambia solo la variable `MODELO`. Usa siempre el identificador exacto que aparece en la página del modelo. En el proveedor directo, cambia el parámetro `model` por uno vigente de ese proveedor.

---

## 13. Guardar las dependencias

Registra las librerías instaladas con sus versiones:

```bash
python -m pip freeze > requirements.txt
```

Para replicar el entorno en otra máquina:

```bash
python -m pip install -r requirements.txt
```

Vuelve a generar el archivo cada vez que instales o desinstales una librería.

---

## 14. Conectar VS Code

1. Abre la carpeta `agentes-ia` en VS Code.
2. Instala la extensión **Python** (Microsoft) si no la tienes.
3. Presiona `Ctrl+Shift+P` (o `Cmd+Shift+P` en Mac) y elige **Python: Select Interpreter**.
4. Selecciona:
   - macOS / Linux: `.venv/bin/python`
   - Windows: `.venv\Scripts\python.exe`
5. Abre una terminal integrada con `` Ctrl+` ``. Si VS Code activa el entorno solo, verás `(.venv)`.

Si VS Code no activa el entorno, revisa en la configuración que esté habilitado `python.terminal.activateEnvironment`.

---

## 15. Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `python3 --version` muestra menos de 3.10 | El PATH apunta a un Python antiguo | Instala una versión nueva y revisa el orden del PATH |
| `ModuleNotFoundError: No module named 'openai'` | Se instaló en otro Python o el entorno no está activo | Activa `.venv` e instala de nuevo |
| `FALTA` en `verificar_claves.py` | `.env` no está en la carpeta desde la que ejecutas, o el nombre de la variable está mal escrito | Revisa la ubicación y el nombre exacto |
| 401 / `AuthenticationError` | La clave es incorrecta o no se cargó | Confirma que `load_dotenv()` se ejecuta antes de leer la clave; genera una nueva si hace falta |
| 404 / modelo no encontrado | Identificador mal escrito o modelo retirado | Copia el ID exacto desde la página del modelo |
| 429 / `RateLimitError` | Cuota agotada, común en modelos gratuitos | Espera o cambia de modelo |
| Respuesta cortada | `max_tokens` demasiado bajo | Revisa `finish_reason` (OpenRouter) o `stop_reason` (Anthropic) y súbelo si corresponde |
| VS Code no encuentra los paquetes | Intérprete equivocado | Vuelve a seleccionar `.venv` en *Select Interpreter* |

---

## 16. Antes de subir a GitHub

```bash
git status
```

No deben aparecer `.env` ni `.venv/`. Si aparecen, revisa `.gitignore` antes de hacer commit.

Si ya subiste una clave por error, revócala en el panel del proveedor. Borrar el archivo en un commit posterior no la invalida, porque queda en el historial.
