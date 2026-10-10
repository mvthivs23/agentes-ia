# Guía paso a paso: entorno de trabajo con Python para agentes de IA

> Basado en el **Capítulo 02 · Tu entorno de trabajo desde cero: Python, editor, terminal y claves de API** del libro *Agentes de IA con Python: guía práctica para diseñar, construir y desplegar sistemas agénticos*.

Este entorno es la base para ejecutar los ejemplos de los capítulos siguientes. Si algo falla aquí, los errores aparecerán más adelante como `ModuleNotFoundError` o `AuthenticationError`, que parecen errores de código pero no lo son.

---

## Índice

1. [Entender qué es Python en tu sistema](#1-entender-qué-es-python-en-tu-sistema)
2. [Instalar Python (macOS y Windows)](#2-instalar-python-macos-y-windows)
3. [Instalar y configurar VS Code](#3-instalar-y-configurar-vs-code)
4. [Crear el entorno virtual `.venv`](#4-crear-el-entorno-virtual-venv)
5. [Activar el entorno y gestionar dependencias](#5-activar-el-entorno-y-gestionar-dependencias)
6. [Obtener y guardar las claves de API](#6-obtener-y-guardar-las-claves-de-api)
7. [Verificar todo antes de continuar](#7-verificar-todo-antes-de-continuar)
8. [Subir este material a GitHub](#8-subir-este-material-a-github)
9. [Solución de problemas comunes](#9-solución-de-problemas-comunes)

---

## 1. Entender qué es Python en tu sistema

- Python **no es una aplicación con ventana**: es un intérprete que lee archivos de texto con instrucciones y las ejecuta línea por línea.
- VS Code es un editor. La conexión con Python la aporta la **extensión oficial de Python**.
- Tu sistema puede tener **varios Python** instalados (en macOS, Apple incluye uno propio). Si el editor se conecta al equivocado, los paquetes no se encontrarán.
- El **PATH** es la lista de carpetas donde el sistema busca ejecutables cuando escribes un comando. Para ver tu PATH:
  - macOS/Linux: `echo $PATH`
  - PowerShell: `$env:Path`

---

## 2. Instalar Python (macOS y Windows)

### Versión recomendada

Python **3.10 o superior** (el libro usa ejemplos con Python 3.12).

### 2.1 macOS

1. Descarga el instalador desde **python.org/downloads**.
2. Instala. Coloca los ejecutables en `/usr/local/bin/` y crea los enlaces `python3` y `pip3`.
3. Verifica:

```bash
python3 --version
```

Debe mostrar algo como `Python 3.12.4`. Si muestra una versión anterior a 3.10, revisa el orden de directorios del PATH y, si es necesario, edita `~/.zshrc` (shell por defecto en macOS desde Catalina) para colocar `/usr/local/bin` al principio.

> En macOS usa siempre `python3`, no `python` a secas.

### 2.2 Windows

Hay dos rutas válidas:

- **python.org/downloads** (recomendado). Durante la instalación:
  - ✅ Activa **Add Python to PATH**. Sin esto tendrás que escribir la ruta completa cada vez.
  - El instalador incluye el **Python Launcher for Windows**, que instala el comando `py`.
- **Microsoft Store**.

Verifica:

```powershell
py --version
```

(Si instalaste desde Microsoft Store, usa `python --version`.)

Para seleccionar una versión concreta con el launcher:

```powershell
py -3.12 --version
```

### 2.3 Verificar pip

`pip` es el gestor de paquetes. **Invócalo siempre como módulo del intérprete**, así garantizas que pertenece al mismo Python que ejecutará tu código:

```bash
python3 -m pip --version    # macOS
py -m pip --version         # Windows
```

---

## 3. Instalar y configurar VS Code

1. Descarga VS Code desde **code.visualstudio.com**.
2. En el panel de extensiones, instala la extensión oficial **Python** (publicada por Microsoft).
3. Abre la carpeta de tu proyecto.
4. Selecciona el intérprete:
   - Presiona `Ctrl+Shift+P` (o `Cmd+Shift+P` en Mac).
   - Escribe y elige **Python: Select Interpreter**.
   - Elige la ruta del entorno virtual:
     - macOS/Linux: `.venv/bin/python`
     - Windows: `.venv\Scripts\python.exe`
   - Si aún no creaste `.venv` (paso 4), vuelve aquí después.
5. Abre la terminal integrada con `Ctrl+ç` o `Ctrl+acento grave` (`` Ctrl+` ``). En la primera apertura elige el perfil de shell:
   - Windows: PowerShell, Command Prompt, Git Bash o WSL.
   - macOS: Zsh o Bash.
6. Si la activación automática no ocurre, revisa en `settings.json` que esté habilitada la opción:

```json
"python.terminal.activateEnvironment": true
```

(Por defecto está en `true`, pero algunas instalaciones corporativas pueden desactivarla por política.)

---

## 4. Crear el entorno virtual `.venv`

Un entorno virtual es una carpeta dentro del proyecto con su propio Python y sus propios paquetes, aislados del resto del sistema. Así evitas conflictos de versiones entre proyectos.

Sitúate en la carpeta raíz del proyecto y ejecuta:

```bash
# macOS / Linux
python3 -m venv .venv
```

```powershell
# Windows
py -m venv .venv
```

Notas:
- El punto inicial (`.venv`) indica que es una carpeta auxiliar, no código de la aplicación. VS Code la reconoce automáticamente.
- Puedes llamarla `venv`, pero `.venv` es el nombre que usan la mayoría de herramientas modernas y el que verás en los ejemplos.

### Excluir `.venv` de Git

La carpeta `.venv` **no debe subirse a Git**: pesa más de cien megabytes, contiene binarios específicos del sistema operativo y se regenera en cualquier momento con `requirements.txt`.

```bash
echo ".venv/" >> .gitignore
```

---

## 5. Activar el entorno y gestionar dependencias

### 5.1 Activar

Cada vez que abras una **nueva terminal** antes de trabajar en el proyecto:

```bash
# macOS / Linux
source .venv/bin/activate
```

```powershell
# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (Command Prompt)
.venv\Scripts\activate.bat
```

Cuando el entorno está activo, el prompt muestra `(.venv)` al principio. Cualquier `python` o `pip` que uses apunta al entorno, no al sistema.

Para desactivarlo:

```bash
deactivate
```

### 5.2 Instalar paquetes

Con el entorno activo:

```bash
python -m pip install nombre-del-paquete
```

### 5.3 Mantener `requirements.txt`

Después de instalar paquetes, congela las versiones exactas:

```bash
python -m pip freeze > requirements.txt
```

Para replicar el entorno en otra máquina o al clonar el proyecto:

```bash
python -m pip install -r requirements.txt
```

---

## 6. Obtener y guardar las claves de API

### 6.1 Por qué variables de entorno

Los agentes se comunican con APIs externas: **Anthropic** (Claude), **OpenAI** y **Google** (Gemini). Cada una requiere una clave privada. Quien tenga tu clave puede hacer llamadas a la API **a tu costa**, así que nunca va dentro del código.

La forma correcta es una **variable de entorno**, que el código lee con `os.environ`.

### 6.2 Obtener cada clave

| Proveedor | Dónde | Sección |
|---|---|---|
| Anthropic | `console.anthropic.com` | "API Keys" → generar nueva clave |
| OpenAI | `platform.openai.com` | "API keys" |
| Google (Gemini) | `aistudio.google.com` | "Get API Key" |

> ⚠️ Las tres plataformas muestran la clave completa **solo en el momento de crearla**. Cópiala inmediatamente al archivo `.env`. Normalmente no hay forma de recuperarla después; tendrías que crear una nueva.

### 6.3 Crear el archivo `.env`

En la raíz del proyecto crea un archivo llamado `.env` con este formato:

```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxxx
GOOGLE_API_KEY=AIzaxxxxxxxxxxxxx
```

Reemplaza las `x` por tus claves reales. Nada de comillas ni espacios alrededor del `=`.

### 6.4 Proteger `.env`

**Este archivo jamás debe subirse a Git.** Agrégalo al `.gitignore`:

```bash
echo ".env" >> .gitignore
```

Tu `.gitignore` debe contener, como mínimo:

```gitignore
.venv/
.env
```

### 6.5 Cargar `.env` desde Python

Instala la librería `python-dotenv`:

```bash
python -m pip install python-dotenv
```

Y al inicio de cualquier script que necesite las claves:

```python
from dotenv import load_dotenv
load_dotenv()
```

A partir de ahí, `os.environ["ANTHROPIC_API_KEY"]` devuelve el valor. Los SDK de Anthropic, OpenAI y Google buscan automáticamente sus variables (el SDK de Google también acepta `GEMINI_API_KEY`), así que muchos scripts ni siquiera necesitan leerlas explícitamente: basta con ejecutar `load_dotenv()` antes de instanciar el cliente.

### 6.6 Regla de seguridad

> **Una clave expuesta es una clave comprometida.** Si la pegas en un mensaje, un issue, una captura de pantalla o un commit, considérala pública.
>
> Si ocurre: **no** basta con borrar el commit (puede estar en el historial de otros clones). Revoca la clave en el portal del proveedor y genera una nueva. GitHub tiene un sistema de *secret scanning* que detecta claves conocidas en commits públicos y notifica al proveedor para que las invalide.

---

## 7. Verificar todo antes de continuar

### 7.1 Script de verificación de claves

Crea un archivo `verificar_claves.py`. **Este script no imprime las claves**; solo confirma que existen:

```python
import os
from dotenv import load_dotenv

load_dotenv()

claves = ["ANTHROPIC_API_KEY", "OPENAI_API_KEY", "GOOGLE_API_KEY"]

for clave in claves:
    valor = os.environ.get(clave, "")
    estado = "OK" if valor else "FALTA"
    print(f"{clave}: {estado}")
```

Ejecútalo:

```bash
python verificar_claves.py
```

La salida esperada son tres líneas con `OK`. Si alguna dice `FALTA`, revisa el nombre de la variable en `.env` y que estés en la carpeta correcta.

### 7.2 Lista de verificación

Marca cada punto antes de avanzar al capítulo 3:

- [ ] `python3 --version` (macOS) o `py --version` (Windows) muestra **3.10 o superior**
- [ ] `python -m pip --version` muestra pip asociado al intérprete correcto
- [ ] La carpeta `.venv` existe en el directorio del proyecto
- [ ] Aparece `(.venv)` en el prompt al abrir la terminal integrada de VS Code
- [ ] VS Code muestra el intérprete de `.venv` en la barra inferior
- [ ] `.env` existe con las tres claves y aparece en `.gitignore`
- [ ] `verificar_claves.py` reporta `OK` para las tres claves

> Si algún punto falla, el síntoma más común no será un error explícito de configuración, sino un `ModuleNotFoundError` o un `AuthenticationError` que parecen errores de código. Resuelve la configuración antes de avanzar.

---

## 8. Subir este material a GitHub

### 8.1 Requisitos

- Cuenta en GitHub.
- Git instalado: `git --version`. Si no está, descárgalo desde **git-scm.com**.

### 8.2 Primera vez: configurar Git

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu@correo.com"
```

### 8.3 Crear el repositorio en GitHub

1. Entra a **github.com** → botón **New repository**.
2. Ponle un nombre (por ejemplo `mi-entorno-agentes-ia`).
3. Déjalo **Private** si contiene material propio.
4. **No** marques "Add a README" ni `.gitignore` (ya los tienes localmente).
5. Crear.

### 8.4 Preparar la carpeta local

Desde la raíz del proyecto:

```bash
git init
git add .
git status
```

Antes de hacer commit, **revisa `git status`**: `.venv/` y `.env` **no** deben aparecer. Si aparecen, revisa tu `.gitignore`. Si `.env` ya estuvo en un commit anterior, deberás revocar esas claves (ver sección 6.6).

### 8.5 Primer commit y subida

```bash
git commit -m "Primer commit: guía de entorno y configuración"
git branch -M main
git remote add origin https://github.com/TU-USUARIO/mi-entorno-agentes-ia.git
git push -u origin main
```

Sustituye `TU-USUARIO` por tu usuario de GitHub. La primera vez, Git te pedirá autenticarte (token personal o GitHub CLI).

### 8.6 Subidas posteriores

```bash
git add .
git commit -m "Descripción del cambio"
git push
```

---

## 9. Solución de problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `python` no se reconoce (Windows) | No marcaste *Add Python to PATH* | Reinstala con esa opción o usa `py` |
| `python3 --version` muestra una versión antigua (macOS) | El PATH apunta a otro Python | Revisa el orden de directorios; edita `~/.zshrc` y pon `/usr/local/bin` primero |
| VS Code no encuentra los paquetes instalados | Seleccionaste el Python global | *Python: Select Interpreter* → elige `.venv` |
| `ModuleNotFoundError` al ejecutar | Paquete instalado fuera del entorno virtual, o entorno no activo | Activa `.venv` y reinstala con `python -m pip install ...` |
| `AuthenticationError` al llamar a la API | Clave ausente, mal escrita o revocada | Ejecuta `verificar_claves.py`; revisa `.env` o genera una clave nueva |
| No aparece `(.venv)` en la terminal | Activación automática desactivada | Activa manualmente o revisa `python.terminal.activateEnvironment` |
| Git quiere subir `.venv` o `.env` | Falta la entrada en `.gitignore` | Agrega ambas líneas y ejecuta `git rm -r --cached .venv` (o el archivo afectado) |

---

## Material del capítulo

El código canónico del capítulo, organizado por archivo, está en el repositorio del libro:

**https://github.com/studio35post/agentes-ia-python**

El Anexo A explica cómo descargarlo paso a paso y el Anexo B resuelve los errores comunes.
