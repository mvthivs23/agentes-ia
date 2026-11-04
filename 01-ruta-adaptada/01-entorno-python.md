# 01 — Entorno para automatizar

Fuente: `03-python-for-security/01-python-environment-and-language-fundamentals.md` (secciones 2, 3 y 12, ejercicios 3 y 4) y el scanner de headers de `03-python-for-security/05-python-tool-development-building-real-pentest-tools.md` (sección 12, `REQUIRED_HEADERS`).

El ejercicio 1 de Python es un port scanner y el 2 es un filtro de wordlist. No los uses como automatización de la consultora. Los que sí encajan son el entorno, el enumerador de headers y el merge de reportes JSON.

## Paso 1 — El entorno

Sigue la sección 3 de la lección 01: un virtualenv por proyecto, no paquetes en el Python del sistema.

```bash
mkdir -p ~/devsecops-lab && cd ~/devsecops-lab
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Si en tu máquina el material de la lección 01 usa `uv` en vez de `venv`, usa ese camino. No mezcles los dos en el mismo directorio.

## Paso 2 — Headers del servicio propio

La lección 01, sección 12, arma un enumerador de headers HTTP con la biblioteca estándar. La lección 05 fija qué headers faltantes son un hallazgo:

| Header | Hallazgo si falta |
| --- | --- |
| `Strict-Transport-Security` | HSTS not set |
| `Content-Security-Policy` | CSP not set |
| `X-Frame-Options` | clickjacking possible |
| `X-Content-Type-Options` | MIME sniff |
| `Referrer-Policy` | sensitive URLs may leak |

Apunta el enumerador a un servicio que sea tuyo (el de laboratorio o el del producto). El script imprime los headers que la respuesta trae. Después compara esa lista con la tabla. El entregable es un JSON:

```json
{"url": "https://tu-servicio", "missing": ["Content-Security-Policy"]}
```

No lo apuntes a un host que no sea tuyo. El timeout y el manejo de `OSError` son los de la sección 8 de la lección 01.

## Paso 3 — Juntar dos reportes

El ejercicio 3 de la lección 01 pide `merge_scans.py`: dos JSON `{"host": str, "open_ports": list[int]}` se fusionan por host, uniendo los puertos en un set, y se escriben con `json.dump(..., indent=2)`.

Haz ese script tal cual. En esta ruta el `host` es el nombre del servicio o de la imagen que escaneaste, y `open_ports` es la lista de hallazgos que ya normalizaste a enteros de severidad o a puertos del propio lab. La función no cambia. Cambia el archivo que le pasas.

## Paso 4 — Tiempo del gate

El ejercicio 4 pide `@timed(threshold_ms=500)`: una factory de decorators que avisa por stderr solo si la llamada pasa el umbral.

Envuelve la función que llama al scanner o a la API de NVD del archivo `07-cve-y-nvd.md`. El umbral del material es 500 ms. Si tu llamada de red lo supera siempre, súbelo y anota el número en el informe: el decorator es el mismo, el umbral es el del job.

## Entregable

- El virtualenv creado.
- El JSON de headers faltantes de un servicio tuyo.
- `merge_scans.py` corriendo sobre dos reportes.
- Una función decorada con `@timed(threshold_ms=500)` que avisa solo cuando se pasa.
