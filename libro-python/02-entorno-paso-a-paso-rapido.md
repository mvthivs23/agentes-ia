# Preparar el entorno: paso a paso rápido

Versión corta para dejar el entorno listo en unos minutos. Para entender el porqué de cada paso, consulta [entorno-detallado.md](entorno-detallado.md).

**Requisitos:** Python 3.10 o superior, VS Code con la extensión oficial de Python y una clave de API de un proveedor (Anthropic, OpenAI, Google) **o** de [OpenRouter](https://openrouter.ai).

---

## 1. Crear la carpeta y entrar

```bash
mkdir -p ~/Documentos/agentes-ia
cd ~/Documentos/agentes-ia
```

## 2. Verificar Python

```bash
python3 --version      # macOS / Linux
py --version           # Windows
```

Debe mostrar **3.10 o superior**.

## 3. Crear y activar el entorno virtual

```bash
python3 -m venv .venv                  # macOS / Linux (Windows: py -m venv .venv)
source .venv/bin/activate              # macOS / Linux
.venv\Scripts\Activate.ps1             # Windows (PowerShell)
```

Debe aparecer `(.venv)` al inicio del prompt. Hazlo cada vez que abras una terminal nueva.

## 4. Crear `.gitignore`

```bash
printf ".venv/\n.env\n" > .gitignore
```

## 5. Instalar dependencias

Elige una ruta:

```bash
python -m pip install anthropic python-dotenv    # Anthropic
python -m pip install openai python-dotenv       # OpenAI, o OpenRouter (recomendado)
python -m pip install google-genai python-dotenv # Google Gemini
```

## 6. Crear el archivo `.env`

Crea `.env` en la raíz del proyecto, sin comillas ni espacios alrededor del `=`. Usa una línea según tu proveedor:

```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxxx
GOOGLE_API_KEY=AIzaxxxxxxxxxxxxx
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx
```

Obtén la clave en: `console.anthropic.com` · `platform.openai.com` · `aistudio.google.com` · `openrouter.ai` (sección **Keys**). Cópiala al momento, porque no se vuelve a mostrar.

## 7. Verificar las claves

Crea `verificar_claves.py`:

```python
import os
from dotenv import load_dotenv

load_dotenv()

claves = ["ANTHROPIC_API_KEY", "OPENAI_API_KEY", "GOOGLE_API_KEY", "OPENROUTER_API_KEY"]
for clave in claves:
    valor = os.environ.get(clave, "")
    print(f"{clave}: {'OK' if valor else 'FALTA'}")
```

```bash
python verificar_claves.py
```

La clave que vas a usar debe decir `OK`. Las demás pueden decir `FALTA`.

## 8. Primer mensaje

**OpenRouter con SDK de OpenAI:**

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

cliente = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.getenv("OPENROUTER_API_KEY"),
)

respuesta = cliente.chat.completions.create(
    model="recursive/flint-v1.0:free",   # modelo gratuito; cambia por el ID que quieras
    messages=[{"role": "user", "content": "Explica en tres frases qué hace un agente de IA."}],
    max_tokens=256,
)
print(respuesta.choices[0].message.content)
```

**Anthropic:**

```python
from anthropic import Anthropic

cliente = Anthropic()
respuesta = cliente.messages.create(
    model="claude-sonnet-5",   # confirma el nombre vigente
    max_tokens=256,
    messages=[{"role": "user", "content": "Explica en tres frases qué hace un agente de IA."}],
)
print(respuesta.content[0].text)
```

```bash
python primer_mensaje.py
```

## 9. Guardar dependencias

```bash
python -m pip freeze > requirements.txt
```

Para replicar en otra máquina: `python -m pip install -r requirements.txt`

## 10. Conectar VS Code

1. Abre la carpeta `agentes-ia`.
2. `Ctrl+Shift+P` (o `Cmd+Shift+P`) → **Python: Select Interpreter**.
3. Elige `.venv/bin/python` (macOS / Linux) o `.venv\Scripts\python.exe` (Windows).

---

## Lista rápida

- [ ] Python 3.10 o superior
- [ ] `(.venv)` aparece en la terminal
- [ ] `.gitignore` contiene `.venv/` y `.env`
- [ ] `.env` tiene la clave de tu proveedor u OpenRouter
- [ ] `verificar_claves.py` muestra `OK` para la clave que usas
- [ ] VS Code usa el intérprete de `.venv`

## Antes de subir a GitHub

Revisa `git status`. No deben aparecer `.env` ni `.venv/`.
