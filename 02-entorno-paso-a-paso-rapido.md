# Preparar el entorno: paso a paso rápido

> Versión corta del Capítulo 02, pensada para dejar el entorno listo en unos minutos. Para entender el porqué de cada paso, consulta [02-entorno-de-trabajo.md](02-entorno-de-trabajo.md).

**Requisitos:** Python 3.10 o superior, VS Code con la extensión oficial de Python y una clave de API de Anthropic.

---

## 1. Crear la carpeta del proyecto

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

## 3. Crear el entorno virtual

```bash
python3 -m venv .venv  # macOS / Linux
py -m venv .venv       # Windows
```

## 4. Activar el entorno

Hazlo cada vez que abras una terminal nueva dentro del proyecto.

```bash
source .venv/bin/activate              # macOS / Linux
.venv\Scripts\Activate.ps1             # Windows (PowerShell)
```

El prompt debe mostrar `(.venv)` al inicio.

## 5. Instalar las dependencias

```bash
python -m pip install anthropic python-dotenv
```

## 6. Crear `.gitignore`

Escribe el nombre exacto, **sin espacios**:

```bash
printf ".venv/\n.env\n" > .gitignore
cat .gitignore
```

Debe mostrar:

```
.venv/
.env
```

## 7. Crear el archivo `.env`

Crea un archivo llamado `.env` en la raíz del proyecto con tu clave (sin comillas ni espacios alrededor del `=`):

```env
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
```

> Si todavía no tienes la clave: crea una cuenta en `console.anthropic.com`, ve a **API Keys** y genera una. Cópiala en ese momento, porque no se vuelve a mostrar.

## 8. Crear el script de verificación

Crea `verificar_claves.py` con este contenido:

```python
import os
from dotenv import load_dotenv

load_dotenv()

claves = ["ANTHROPIC_API_KEY"]
for clave in claves:
    valor = os.environ.get(clave, "")
    estado = "OK" if valor else "FALTA"
    print(f"{clave}: {estado}")
```

## 9. Ejecutarlo

```bash
python verificar_claves.py
```

Salida esperada:

```
ANTHROPIC_API_KEY: OK
```

Si dice `FALTA`, revisa que el `.env` esté en la misma carpeta desde donde ejecutas el script y que el nombre de la variable esté bien escrito.

## 10. Guardar las dependencias

```bash
python -m pip freeze > requirements.txt
```

Para replicar el entorno en otra máquina:

```bash
python -m pip install -r requirements.txt
```

## 11. Conectar VS Code con el entorno

1. Abre la carpeta `agentes-ia` en VS Code.
2. Presiona `Ctrl+Shift+P` (o `Cmd+Shift+P` en Mac).
3. Elige **Python: Select Interpreter**.
4. Selecciona `.venv/bin/python` (macOS / Linux) o `.venv\Scripts\python.exe` (Windows).

Abre una terminal integrada con `` Ctrl+` ``. Si VS Code activa el entorno solo, verás `(.venv)` sin escribir nada más.

---

## Lista rápida

- [ ] Python 3.10 o superior
- [ ] `(.venv)` aparece en la terminal
- [ ] `.gitignore` contiene `.venv/` y `.env`
- [ ] `.env` tiene `ANTHROPIC_API_KEY`
- [ ] `verificar_claves.py` muestra `OK`
- [ ] VS Code usa el intérprete de `.venv`

## Antes de subir a GitHub

Revisa `git status`. No deben aparecer `.env` ni `.venv/`. Si aparecen, revisa el `.gitignore` antes de hacer commit.
