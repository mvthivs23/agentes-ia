# 02 · Tu entorno de trabajo desde cero

> *Python, editor, terminal y claves de API*
>
> Capítulo 02 · *Agentes de IA con Python: guía práctica para diseñar, construir y desplegar sistemas agénticos*

> **Preparación rápida:** si solo quieres dejar el entorno listo, sigue el [paso a paso rápido para preparar el entorno](02-entorno-paso-a-paso-rapido.md).

---

## Introducción

Antes de que un agente de IA pueda ejecutar una herramienta, llamar a una API o mantener memoria entre conversaciones, necesita un entorno local fiable:

- un intérprete de Python,
- un editor que lo reconozca,
- una terminal manejable,
- claves de API guardadas de forma que el código pueda leerlas sin incrustarlas en archivos visibles.

Este capítulo no enseña programación en sentido amplio (eso lo aborda el capítulo 3). Enseña la instalación y las convenciones necesarias para que los ejemplos corran sin fricción.

---

## Python como intérprete, no como aplicación

Python no es una aplicación de escritorio con icono, doble clic y ventana. Es un **intérprete**: un ejecutable que recibe un archivo de texto con instrucciones y las ejecuta línea por línea.

VS Code, en cambio, es un editor de texto sofisticado. No incluye Python ni lo necesita para funcionar. La conexión entre ambos la aporta la **extensión oficial de Python** para VS Code, que se comunica con el intérprete instalado para ejecutar código, mostrar errores en línea y activar el autocompletado.

### Consecuencias prácticas

- Si instalas VS Code antes que Python, la extensión no encontrará ningún intérprete.
- Si instalas Python pero la extensión apunta al intérprete equivocado, los paquetes instalados con `pip` no estarán disponibles al ejecutar el código.
- Cuando hay varios Pythons en el sistema (algo frecuente en macOS, donde Apple instala el suyo propio con fines internos), el editor puede conectarse al incorrecto sin avisar de forma clara.

### El PATH

No hace falta memorizar rutas de archivo. Basta con entender el **PATH**: la lista ordenada de directorios donde el sistema operativo busca ejecutables cuando escribes un comando en la terminal.

Al escribir `python3` y presionar Enter, el sistema recorre esa lista de izquierda a derecha hasta encontrar un archivo llamado `python3`. El primero que encuentra es el que ejecuta.

- En macOS, si Python está instalado en `/usr/local/bin/`, ese directorio debe aparecer en el PATH antes que `/usr/bin/`, donde reside el Python del sistema.
- En Windows, conviene marcar la opción que agrega Python al PATH durante la instalación.

Para ver tu PATH actual:

```bash
echo $PATH        # macOS / Linux
```

```powershell
$env:Path         # PowerShell
```

No es necesario interpretar toda la salida: basta con confirmar que la ruta donde instalaste Python aparece antes que cualquier otra instalación.

---

## Instalar Python: macOS y Windows por separado

macOS y Windows tienen flujos de instalación distintos; mezclarlos suele producir errores difíciles de diagnosticar.

### macOS

El punto de partida es **python.org/downloads**. El instalador oficial coloca los ejecutables en `/usr/local/bin/` y crea los enlaces simbólicos `python3` y `pip3`.

El Python que Apple incluye o expone en macOS (accesible como `/usr/bin/python3` en muchas instalaciones) existe para uso interno del sistema. Las aplicaciones de terceros no deben depender de él ni modificarlo. Por eso los comandos de macOS usan `python3`, nunca `python` a secas, que en macOS moderno frecuentemente no existe o apunta a una versión heredada.

Verifica:

```bash
python3 --version
```

La salida debe mostrar la versión que acabas de instalar, por ejemplo `Python 3.12.4`. Si muestra una versión anterior a 3.10, el PATH todavía apunta al Python del sistema. Revisa el orden de directorios y, si es necesario, edita el archivo de configuración de tu shell (`~/.zshrc` en Zsh, la shell por defecto en macOS desde Catalina) para colocar `/usr/local/bin` al principio.

### Windows

Python no forma parte del sistema operativo. Si escribes `python` en una terminal recién instalada, Windows puede abrir la Microsoft Store o reportar que el comando no existe.

La documentación oficial recomienda obtenerlo desde **python.org/downloads** o desde la Microsoft Store; ambas rutas son válidas. El instalador de python.org incluye además el **Python Launcher for Windows**, que instala el comando `py` y permite seleccionar versión explícitamente:

```powershell
py -3.12 --version
```

Si solo tienes una versión instalada, `py` y `python` son equivalentes.

Durante la instalación, **activa "Add Python to PATH"**. Sin esa opción, tendrás que escribir la ruta completa al ejecutable cada vez, lo que hace inviable el trabajo diario.

Verifica la instalación con:

```powershell
py --version
```

o, si elegiste la ruta de Microsoft Store:

```powershell
python --version
```

### Verificar pip

`pip` es el gestor de paquetes. Conviene invocarlo **como módulo del intérprete** en lugar de llamar al ejecutable `pip` directamente. Así se garantiza que el gestor de paquetes pertenece al mismo Python que ejecutará el código. Este hábito evita una categoría entera de errores confusos.

```bash
python3 -m pip --version    # macOS
py -m pip --version         # Windows
```

---

## El entorno virtual: dependencias sin contaminación cruzada

Cada proyecto de agentes —el agente de investigación del capítulo 23, el agente de automatización personal del capítulo 24, el asistente con MCP del capítulo 25— necesitará paquetes y versiones concretas. Si los instalas directamente en el Python del sistema, tarde o temprano dos proyectos pedirán versiones incompatibles de la misma librería. **Los entornos virtuales evitan esa contaminación cruzada.**

Un entorno virtual es una carpeta dentro del proyecto que contiene su propio Python y sus propios paquetes, aislados del resto del sistema. Técnicamente, `venv` crea un directorio con un archivo `pyvenv.cfg`, un subdirectorio `bin/` en macOS/Linux o `Scripts/` en Windows con copias o enlaces simbólicos al intérprete, y una carpeta `site-packages/` donde se instalan los paquetes. Cualquier `pip install` ejecutado con ese entorno activo deposita los archivos ahí, no en el Python global.

### Crear el entorno

Sitúate en la carpeta raíz de tu proyecto y ejecuta:

```bash
# macOS / Linux
python3 -m venv .venv
```

```powershell
# Windows
py -m venv .venv
```

El punto al inicio del nombre (`.venv`) indica que es una carpeta auxiliar, no código de la aplicación. VS Code la reconoce automáticamente. Puedes llamarla `venv`, pero `.venv` es lo que usan la mayoría de las herramientas modernas y lo que verás en los ejemplos.

### Excluirla de Git

La carpeta `.venv` no debe subirse a Git. Agrégala a `.gitignore` antes de tu primer commit:

```bash
echo ".venv/" >> .gitignore
```

La razón es práctica: `.venv` puede superar los cien megabytes según los paquetes instalados, contiene binarios específicos del sistema operativo y puede regenerarse en cualquier momento con `pip install -r requirements.txt`. Subirla a un repositorio equivale a subir artefactos compilados: es innecesario, pesado y propenso a conflictos.

### Activar el entorno

Para activar el entorno —paso que debes hacer cada vez que abres una nueva terminal antes de trabajar en el proyecto—:

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

Cuando el entorno está activo, el prompt de la terminal muestra `(.venv)` al principio. Cualquier `python` o `pip` que uses a partir de ese momento apunta al entorno, no al sistema. Para desactivarlo, escribe `deactivate`.

### Mantener `requirements.txt`

Mantener un `requirements.txt` actualizado simplifica el trabajo a largo plazo. Después de instalar un paquete nuevo:

```bash
python -m pip freeze > requirements.txt
```

Este archivo lista los paquetes instalados con sus versiones exactas. Cuando compartes el proyecto o lo clonas en otra máquina, basta con crear un entorno nuevo e instalar desde ese archivo:

```bash
python -m pip install -r requirements.txt
```

> **Nota:** `freeze` reemplaza el contenido completo del archivo, así que siempre refleja lo instalado en ese momento. Si instalas un paquete antes de generar el archivo, aparecerá; si ejecutas el `freeze` antes de instalar nada, quedará vacío. Vuelve a ejecutarlo cada vez que cambies las dependencias.

---

## VS Code: el editor que necesita saber dónde está Python

VS Code será el editor de referencia porque integra bien Python, tiene soporte nativo para Jupyter notebooks (útiles en el capítulo 20) y ofrece una terminal integrada.

1. Descárgalo desde **code.visualstudio.com**.
2. Instala la extensión oficial **Python** (publicada por Microsoft) desde el panel de extensiones.

### Seleccionar el intérprete

Después debes indicarle qué intérprete usar. Con el proyecto abierto:

1. Presiona `Ctrl+Shift+P` (o `Cmd+Shift+P` en Mac).
2. Escribe y selecciona **Python: Select Interpreter**.
3. Elige la ruta del entorno virtual:
   - macOS/Linux: `.venv/bin/python`
   - Windows: `.venv\Scripts\python.exe`

La extensión listará todos los Pythons que detecta, incluido el del entorno virtual si ya lo creaste.

Desde ese momento, el botón de ejecutar, el depurador, el linter y el autocompletado usarán ese intérprete. Si seleccionas el Python global, el autocompletado no encontrará los paquetes instalados en `.venv` y aparecerán errores de importación ajenos al código.

### La terminal integrada

Se abre con `` Ctrl+` `` (el acento grave). En la primera apertura permite elegir el perfil de shell:

- **Windows:** PowerShell, Command Prompt, Git Bash o WSL.
- **macOS:** Zsh o Bash.

Cuando detecta `.venv` en el directorio del proyecto, VS Code puede activar automáticamente el entorno virtual: el prompt aparece con `(.venv)` sin intervención manual. Así se evita el error habitual de instalar paquetes con el entorno desactivado y luego ver fallar `import anthropic`.

Si la activación automática no ocurre, verifica en `settings.json` que esté habilitada la opción:

```json
"python.terminal.activateEnvironment": true
```

Por defecto está en `true`, aunque algunas instalaciones corporativas pueden desactivarla por política.

---

## Variables de entorno y claves de API

### Por qué variables de entorno

Los agentes se comunican con APIs externas:

- **Anthropic** para Claude,
- **OpenAI** para sus modelos,
- **Google** para Gemini.

Cada una requiere una clave de autenticación obtenida en su portal de desarrollador. Esas claves son **credenciales privadas**: quien las tenga puede hacer llamadas a la API **a tu costa**.

El mecanismo correcto para que el código acceda a ellas sin incrustarlas en archivos de código son las **variables de entorno**. Una variable de entorno es un par nombre-valor que el sistema operativo pone a disposición de los procesos que corren en él. Cuando Python ejecuta:

```python
import os
os.environ["ANTHROPIC_API_KEY"]
```

no lee ningún archivo de código: pregunta al sistema operativo por el valor de esa variable.

### El archivo `.env`

Para el desarrollo local, la forma más práctica de definir variables de entorno por proyecto es un archivo llamado **`.env`** en la raíz del proyecto:

```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxxx
GOOGLE_API_KEY=AIzaxxxxxxxxxxxxx
```

- El ejemplo usa tres proveedores. Si solo vas a trabajar con Claude, basta con la línea de Anthropic.
- No uses comillas ni espacios alrededor del `=`.

> **Este archivo jamás debe subirse a Git.** Agrégalo a `.gitignore`:
>
> ```gitignore
> .venv/
> .env
> ```

### Cargar `.env` desde Python

Para que Python pueda leer ese archivo y cargar las variables, instala la librería `python-dotenv`:

```bash
python -m pip install python-dotenv
```

Y al inicio de cualquier script que necesite las claves:

```python
from dotenv import load_dotenv
load_dotenv()
```

A partir de ahí, `os.environ["ANTHROPIC_API_KEY"]` devuelve el valor correcto. Los SDK de Anthropic, OpenAI y Google buscan automáticamente sus variables (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY` y `GOOGLE_API_KEY`; el SDK de Google también acepta `GEMINI_API_KEY`). Por eso muchos scripts ni siquiera necesitan leer la variable explícitamente: basta con ejecutar `load_dotenv()` antes de instanciar el cliente.

### Obtener las claves

- **Anthropic:** crea una cuenta en `console.anthropic.com`, ve a **API Keys** y genera una nueva clave. El nombre que le das es solo para tu organización interna.
- **OpenAI:** en `platform.openai.com`, sección **API keys**.
- **Google (Gemini):** en `aistudio.google.com`, sección **Get API Key**.

> Las tres plataformas muestran la clave completa **solo en el momento de crearla**. Cópiala de inmediato al archivo `.env`. Normalmente no hay forma de recuperarla después; hay que generar una nueva.

### Seguridad: una clave expuesta es una clave comprometida

La seguridad de las claves descansa en un principio simple: **una clave expuesta es una clave comprometida**, sin importar por cuánto tiempo ni en qué contexto. Si pegas una clave en un mensaje de chat, en un issue de GitHub, en una captura de pantalla o en un archivo que luego haces commit y push, esa clave debe considerarse pública.

GitHub tiene un sistema de *secret scanning* que detecta claves conocidas en commits públicos y notifica automáticamente al proveedor para que las invalide.

**La reacción correcta no es eliminar el commit** (que puede estar ya en el historial de otros clones), sino **revocar la clave comprometida** en el portal del proveedor y generar una nueva.

### Verificar que las claves están disponibles

Antes de trabajar con cualquier API, verifica que las tres claves están disponibles. Este script **no las imprime**; solo confirma su presencia. Guárdalo como `verificar_claves.py`:

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

```bash
python verificar_claves.py
```

La salida esperada son tres líneas con `OK`. Si alguna dice `FALTA`, el SDK correspondiente fallará con un error de autenticación al primer intento de llamada, no con un error de importación.

---

## Lista de verificación antes de continuar

Antes de avanzar al capítulo 3, confirma estos puntos:

- [ ] `python3 --version` (macOS) o `py --version` (Windows) muestra **3.10 o superior**
- [ ] `python -m pip --version` muestra pip asociado al intérprete correcto
- [ ] La carpeta `.venv` existe en el directorio del proyecto
- [ ] `(.venv)` aparece en el prompt al abrir la terminal integrada de VS Code
- [ ] VS Code muestra el intérprete de `.venv` en la barra inferior
- [ ] `.env` existe con las claves necesarias y aparece en `.gitignore`
- [ ] `verificar_claves.py` reporta `OK` para las claves que vas a usar

Si algún punto falla, el síntoma más común no será un error explícito de configuración, sino un `ModuleNotFoundError` o un `AuthenticationError` que pueden parecer errores de código. **Resolver la configuración antes de avanzar evita diagnósticos innecesarios en los capítulos siguientes.**

Con el entorno operativo, el capítulo 3 puede centrarse en el subconjunto de Python necesario para construir agentes.

---

## Material del capítulo

El código canónico de este capítulo, completo y organizado por archivo, está en el repositorio del libro:

**https://github.com/studio35post/agentes-ia-python**

El Anexo A explica cómo descargarlo paso a paso y el Anexo B resuelve los errores comunes.
