# Lección 01 — Python Environment and Language Fundamentals

Las tools de estantería cubren la mayor parte del terreno en un engagement moderno, pero el último veinte por ciento — el endpoint que ningún scanner entiende, el exploit que nadie armó todavía, el payload que tiene que caber en un WAF o un filtro específico — siempre termina en código. Python te da velocidad de iteración y un ecosistema rico; C te da el control a nivel de sistemas que exigen el shellcode, los kernels y los primitivos de exploit.

Este módulo enseña los dos lenguajes lado a lado porque resuelven problemas complementarios y porque leerlos bien es un prerrequisito para leer advisories, gadgets y write-ups de CTF. Vas a terminar el módulo capaz de escribir scanners multihilo y web fuzzers en Python, y programas de buffer overflow, format string y carga de shellcode en C — desde cero, sin que un framework te lleve de la mano. Todo lo que armes acá se vuelve parte permanente de tu toolkit y se vuelve a usar en módulos posteriores.

*Pipeline de compilación de C frente al modelo de runtime de Python — los dos motores entre los que un pentester alterna cada semana.*

Módulo 03: Programming — Python and C · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Python es el lenguaje de trabajo de la seguridad ofensiva. Desde exploits a nivel de socket hasta frameworks de C2 enormes, desde un barrido de puertos de cinco líneas hasta el arsenal de Active Directory de Impacket de más de 200.000 líneas, Python es a lo que llegan los pentesters cuando una tool específica todavía no existe. Es lo bastante rápido para armar, lo bastante expresivo para prototipar en minutos, y su biblioteca estándar sola contiene casi cada primitivo que un atacante necesita — sockets, ejecución con subprocess, hashing criptográfico, clientes HTTP, expresiones regulares y pools de threads.

Esta lección establece la base de Python sobre la que se construye el resto del módulo. Vas a instalar un intérprete moderno de Python 3.12+ en Kali Linux 2026, aprender a aislar las dependencias del proyecto con un virtualenv, dominar las features centrales del lenguaje que aparecen en cada script de pentesting, y terminar armando un enumerador real de headers HTTP con nada más que la biblioteca estándar.

Python 3.12 y 3.13 trajeron mejoras que importan para el trabajo ofensivo. El intérprete es más rápido (intérprete adaptativo especializado de PEP 659), los mensajes de error son dramáticamente mejores (tracebacks de grano fino apuntan a la expresión exacta que falló), el typing es más expresivo (sentencia `type`, sintaxis de parámetros genéricos), y la integración con el profiler perf te da visibilidad real a nivel de CPU de los scans lentos. Si la última vez que tocaste Python fue en 3.8 o 3.9, ahora todo es más rápido y más ergonómico.

Al final de esta lección vas a poder levantar un proyecto de Python limpio, escribir código idiomático que otros operadores puedan leer, y producir una tool chica que hace trabajo de reconnaissance de verdad.

## 1. Por qué Python para pentesters

El dominio de Python en la seguridad ofensiva no es un accidente. Cuatro factores lo mantienen arriba.

### 1.1 El ecosistema

El Python Package Index (PyPI) hospeda más de 500.000 paquetes. Para casi cualquier tarea de pentesting, alguien ya escribió el primitivo que necesitas:

| Tarea | Biblioteca | Qué hace |
|---|---|---|
| Crafting de paquetes crudos | scapy | Construye, envía, sniffea y disecta paquetes |
| Requests HTTP | requests, httpx | Cliente HTTP limpio con sessions y control de TLS |
| Active Directory | impacket | SMB, Kerberos, NTLM, DCE/RPC, LDAP |
| Web crawling | beautifulsoup4, playwright | Parsing de HTML y control de browser headless |
| Fuzzing | boofuzz, atheris | Fuzzing de protocolos y fuzzing guiado por cobertura |
| Criptografía | cryptography, pycryptodome | Primitivos criptográficos modernos |
| Helpers de exploit dev | pwntools | Toolkit de explotación de binarios estilo CTF |
| Windows COM/WMI | pywin32, impacket | Post-explotación de Windows |

### 1.2 Velocidad de iteración

Entre una idea fresca y un proof of concept corriendo, Python comprime la brecha a minutos. No hay paso de compilación, no hay sistema de build que negociar, y el REPL significa que puedes prototipar cualquier interacción de protocolo de forma interactiva antes de envolverla en un script.

### 1.3 Alcance multiplataforma

El mismo script de Python corre en Kali, en un host Windows comprometido que tenga Python instalado, en una laptop de red team con macOS, y en un target Linux embebido que tenga Python 3. Por eso tantos frameworks de post-explotación — Empire, helpers de Covenant, el scripting de beacons de Sliver — o usan Python o emiten Python.

### 1.4 Leer las tools de otros

Casi cada tool de seguridad open-source que vale la pena leer es Python o tiene una interfaz Python. Si no puedes leer Python con fluidez, la mitad del universo de tooling ofensivo es opaco para ti.

> **Pentester Insight:** Rust y Go están ganando terreno para implantes de sigilo (binarios más chicos, más difíciles de reversar para los defensores), pero Python sigue siendo el lenguaje de reconnaissance, scripting y código de pegamento entre tools. Aprende Python primero, y después suma un lenguaje de sistemas cuando lo necesites.

## 2. Instalar Python 3.12+ en Kali 2026

Kali 2026 trae Python 3.12 como el `python3` por defecto. Verifica lo que tienes antes de instalar nada nuevo.

### 2.1 Chequear el Python del sistema

```bash
# Chequea la versión del intérprete por defecto:
python3 --version
# Python 3.12.7

# Path completo y confirma que es el paquete de Kali:
which python3
# /usr/bin/python3

dpkg -S /usr/bin/python3
# python3-minimal: /usr/bin/python3

# Mira cada intérprete de Python instalado en el sistema:
ls /usr/bin/python*
# /usr/bin/python3  /usr/bin/python3.12
```

### 2.2 Instalar intérpretes adicionales con apt

El gestor de paquetes de Kali entrega intérpretes actuales firmados por la distribución. Es la forma más segura de agregar o actualizar Python.

```bash
# Actualiza la caché de paquetes primero:
sudo apt update

# Instala Python 3.13 al lado de 3.12 (disponible en los backports de Kali 2026):
sudo apt install python3.13 python3.13-venv python3.13-dev

# Verifica:
python3.13 --version
# Python 3.13.1

# El python3 por defecto no cambia — esto importa, no lo alteres:
python3 --version
# Python 3.12.7
```

Nunca borres ni pongas un symlink encima de `/usr/bin/python3`. Decenas de tools de Kali tienen shebangs `#!/usr/bin/env python3` y se van a romper si la versión del intérprete por defecto cambia debajo de ellas.

### 2.3 Instalar varias versiones con pyenv

Para pentesters que apuntan a entornos legacy o necesitan reproducir una versión exacta del intérprete, pyenv compila y administra instalaciones lado a lado sin tocar el Python del sistema.

```bash
# Instala dependencias de build:
sudo apt install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
  libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
  xz-utils tk-dev libffi-dev liblzma-dev

# Instala pyenv:
curl https://pyenv.run | bash

# Agrega pyenv a tu shell — append a ~/.zshrc o ~/.bashrc:
cat >> ~/.zshrc <<'EOF'
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
EOF

# Recarga la shell:
exec zsh

# Lista versiones disponibles:
pyenv install --list | grep " 3.1[23]"

# Instala Python 3.12.7 y 3.13.1:
pyenv install 3.12.7
pyenv install 3.13.1

# Pinea una versión por directorio para un engagement específico:
cd ~/engagements/acme-corp
pyenv local 3.13.1
python --version
# Python 3.13.1
```

> **Pentester Insight:** Algunos entornos de cliente todavía corren Python 2.7 o 3.6. Si escribes una tool que tiene que correr en el host del target después de caer ahí, pinea tu entorno de desarrollo a esa versión vieja con pyenv para cazar sintaxis incompatible durante el desarrollo y no en el medio de un engagement.

### 2.4 Confirmar que pip está instalado

```bash
python3 -m pip --version
# pip 24.3.1 from /usr/lib/python3/dist-packages/pip (python 3.12)

# Si falta, instálalo:
sudo apt install python3-pip
```

## 3. Virtualenvs

Cada proyecto serio de Python corre dentro de un virtualenv. Esto no es opcional. El `pip install` global en Kali 2026 está bloqueado por defecto (PEP 668 marca el Python del sistema como administrado externamente) y se va a negar con `error: externally-managed-environment`. Incluso si lo fuerzas con `--break-system-packages`, tarde o temprano vas a crear conflictos de dependencias entre tools.

### 3.1 El venv integrado

`venv` viene con Python. Es la opción aburrida y confiable.

```bash
# Crea un directorio de proyecto:
mkdir -p ~/tools/recon-scanner
cd ~/tools/recon-scanner

# Crea un virtualenv llamado .venv:
python3 -m venv .venv

# Actívalo:
source .venv/bin/activate

# Tu prompt ahora muestra (.venv):
# (.venv) kali@kali:~/tools/recon-scanner$

# Verifica que el intérprete es el del venv:
which python
# /home/kali/tools/recon-scanner/.venv/bin/python

# Instala un paquete — sin sudo, sin --break-system-packages:
pip install requests

# Desactiva cuando termines:
deactivate
```

Agrega siempre `.venv/` al `.gitignore`. Los virtualenvs contienen paths absolutos hardcodeados y no deben entrar al control de versiones.

### 3.2 pipx para tools de CLI independientes

Algunas tools — impacket, sqlmap, bloodyAD — las quieres como comandos en tu `PATH`, no como dependencias de un proyecto. pipx instala cada tool en su propio venv aislado y expone solo los entry points.

```bash
# Instala pipx vía apt (recomendado en Kali):
sudo apt install pipx
pipx ensurepath

# Instala una tool de forma global sin contaminar nada:
pipx install impacket

# Ahora úsala desde cualquier lado:
GetUserSPNs.py -h

# Lista todo lo que pipx tiene instalado:
pipx list

# Actualiza todas las tools administradas por pipx:
pipx upgrade-all
```

### 3.3 uv — la alternativa moderna y rápida

uv es un gestor de paquetes y proyectos de Python escrito en Rust, de Astral (el mismo equipo detrás de ruff). Es 10–100 veces más rápido que pip y combina los roles de pip, venv, pip-tools y pyenv en un solo binario. Para pentesters que manejan muchos proyectos chicos de tools, es el workflow más rápido disponible en 2026.

```bash
# Instala uv:
curl -LsSf https://astral.sh/uv/install.sh | sh
# O vía pipx:
pipx install uv

uv --version
# uv 0.5.11

# Levanta un proyecto nuevo — esto crea pyproject.toml, .python-version, .venv y un README:
uv init recon-scanner
cd recon-scanner

# Agrega una dependencia (crea el venv en la primera llamada):
uv add requests

# Agrega una dependencia solo de desarrollo:
uv add --dev pytest ruff mypy

# Corre un script dentro del entorno administrado (sin activación manual):
uv run python scanner.py

# Instala un intérprete de Python específico y pinea el proyecto a él:
uv python install 3.13
uv python pin 3.13

# Sincroniza el lockfile — installs reproducibles entre máquinas:
uv sync
```

La feature más útil de uv para pentesters es `uv run --script`, que lee metadata de dependencias inline del header de un script (PEP 723) y crea un entorno efímero al vuelo:

```python
# guarda como quick_probe.py
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx"]
# ///
import httpx
print(httpx.get("https://example.com").status_code)
```

```bash
uv run --script quick_probe.py
# 200
```

Sin virtualenv manual, sin `pip install`, sin estado residual.

## 4. Gestión de paquetes

### 4.1 Bases de pip

```bash
# Instala un paquete:
pip install requests

# Instala una versión específica:
pip install "requests==2.31.0"

# Instala un rango de versiones:
pip install "requests>=2.28,<3.0"

# Actualiza:
pip install --upgrade requests

# Lista paquetes instalados:
pip list

# Muestra detalles de un paquete:
pip show requests

# Desinstala:
pip uninstall requests
```

### 4.2 requirements.txt

La forma clásica de registrar dependencias. Un archivo de texto plano, un paquete por línea.

```text
# requirements.txt
requests==2.32.3
beautifulsoup4==4.12.3
dnspython==2.7.0
colorama==0.4.6
```

```bash
# Instala todo de una vez:
pip install -r requirements.txt

# Congela el conjunto instalado ahora a un archivo:
pip freeze > requirements.txt
```

`pip freeze` registra cada dependencia transitiva con una versión exacta. Eso es bueno para la reproducibilidad pero ruidoso — para una tool que distribuyes, prefiere un `requirements.txt` curado a mano con solo tus dependencias directas.

### 4.3 pyproject.toml — el estándar moderno

PEP 621 estandarizó la metadata del proyecto en `pyproject.toml`. Tanto pip como uv lo entienden, y reemplaza a `setup.py`, `setup.cfg` y a menudo a `requirements.txt`.

```toml
# pyproject.toml
[project]
name = "recon-scanner"
version = "0.1.0"
description = "Fast reconnaissance scanner"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.32",
    "dnspython>=2.7",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[project.scripts]
recon-scan = "recon_scanner.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```bash
# Instala el proyecto en modo editable — los cambios de código surten efecto sin reinstalar:
pip install -e .

# Instala con dependencias de dev:
pip install -e ".[dev]"

# Con uv, esto se vuelve:
uv sync --all-extras
```

## 5. Tipos de datos en profundidad

Cada valor de Python tiene un tipo, y elegir el correcto suele ser la diferencia entre código limpio y un lío de conversiones. Esta sección recorre cada tipo central con la mirada de un pentester.

### 5.1 int

Los enteros de Python tienen precisión arbitraria — nunca desbordan. Esto importa cuando calculas hashes, factorizas keys o trabajas con offsets grandes.

```python
port = 8080
timeout_ms = 5_000            # guiones bajos como separadores de miles (legibles, el parser los ignora)
big = 2 ** 4096                # nunca hay overflow
print(big.bit_length())        # 4097

# Conversiones de base — por todos lados en pentesting:
hex(255)                       # '0xff'
bin(0b1010_1010)               # '0b10101010'
int("deadbeef", 16)            # 3735928559
int("0b1101", 2)               # 13
```

### 5.2 float

IEEE 754 de 64 bits. Bien para timings y delays, mal para dinero o comparaciones exactas.

```python
timeout = 2.5
import time
start = time.perf_counter()
# ... trabajo ...
elapsed = time.perf_counter() - start
print(f"took {elapsed:.3f}s")
```

### 5.3 str

Texto. Inmutable, siempre Unicode en Python 3.

```python
banner = "SSH-2.0-OpenSSH_9.6p1"
banner.startswith("SSH-2.0")         # True
banner.split("-")[2]                  # 'OpenSSH_9.6p1'
banner.upper()                        # 'SSH-2.0-OPENSSH_9.6P1'

# f-strings — la elección moderna de formateo:
target = "10.10.10.50"
port = 22
print(f"connecting to {target}:{port}")     # connecting to 10.10.10.50:22

# Atajo de debug de f-string (3.8+):
print(f"{port=}")                            # port=22

# Raw strings para regex o paths de Windows:
pattern = r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"
```

### 5.4 bytes

Datos binarios crudos. Todo lo que sale de un socket, de un archivo abierto en modo binario o de un pipe de subprocess es `bytes`, no `str`.

```python
# Literal de bytes — el prefijo b:
payload = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"

# Desde un string hex (útil para shellcode):
shellcode = bytes.fromhex("31c050682f2f7368682f62696e89e3505389e199b00bcd80")
print(len(shellcode))                # 24

# De vuelta a hex para mostrarlo:
shellcode.hex()                      # '31c050682f2f7368682f62696e89e3505389e199b00bcd80'

# bytes a str — decode con un encoding explícito:
raw = b"admin:hunter2"
raw.decode("utf-8")                  # 'admin:hunter2'

# str a bytes — especifica siempre el encoding:
"admin:hunter2".encode("utf-8")      # b'admin:hunter2'
```

Ejemplo de pentester — leer de un socket devuelve bytes:

```python
import socket
s = socket.socket()
s.settimeout(3)
s.connect(("example.com", 80))
s.sendall(b"HEAD / HTTP/1.0\r\n\r\n")
response = s.recv(4096)              # response es bytes, no str
s.close()

# La primera línea de una response HTTP es segura en ASCII, así que el decode está bien:
first_line = response.split(b"\r\n")[0].decode("ascii")
print(first_line)                    # HTTP/1.1 200 OK
```

Nunca llames `.decode()` sobre datos de red arbitrarios sin un `try`/`except` — un byte mal formado puede levantar `UnicodeDecodeError` y tumbar tu scanner a mitad del loop.

### 5.5 list

Ordenada, mutable, indexable. La colección de propósito general de Python.

```python
targets = ["10.10.10.1", "10.10.10.2", "10.10.10.3"]
targets.append("10.10.10.4")
targets[0]                           # '10.10.10.1'
targets[-1]                          # '10.10.10.4'
targets[1:3]                         # ['10.10.10.2', '10.10.10.3']
len(targets)                         # 4

# Sort in-place vs lista sorted nueva:
ports = [443, 22, 80, 8080]
ports.sort()                         # ports ahora es [22, 80, 443, 8080]
new_list = sorted([443, 22, 80])     # devuelve [22, 80, 443], el original no cambia
```

### 5.6 tuple

Ordenada, inmutable. Úsala cuando importa la identidad de la colección — coordenadas, pares de dirección, keys de diccionario.

```python
target = ("10.10.10.50", 22)         # par (host, port)
host, port = target                   # desempaquetado de tuple

# Un patrón común de pentesting — lista de targets (host, port):
scan_queue = [
    ("10.10.10.1", 22),
    ("10.10.10.1", 80),
    ("10.10.10.1", 443),
]
```

Las tuples son hasheables si todos sus elementos lo son, así que pueden ser keys de diccionario o miembros de un set. Las lists no.

### 5.7 dict

Mapa clave-valor. Ordenado desde Python 3.7. El caballo de batalla de los datos estructurados.

```python
host = {
    "ip": "10.10.10.50",
    "hostname": "dc01.corp.local",
    "os": "Windows Server 2022",
    "open_ports": [88, 135, 389, 445, 636, 3389],
}

host["ip"]                            # '10.10.10.50'
host.get("mac", "unknown")            # 'unknown' — default seguro, sin KeyError

# Itera keys, values, items:
for port in host["open_ports"]:
    print(port)

for key, value in host.items():
    print(f"{key} = {value}")

# Merge de dicts (sintaxis 3.9+):
enriched = host | {"ttl": 128, "confidence": "high"}
```

### 5.8 set

Colección no ordenada de elementos únicos y hasheables. Tests de pertenencia en O(1).

```python
# Deduplica una lista de subdominios de varias fuentes:
from_crt_sh = {"www.acme.com", "mail.acme.com", "vpn.acme.com"}
from_dns_brute = {"vpn.acme.com", "dev.acme.com", "staging.acme.com"}

all_subs = from_crt_sh | from_dns_brute       # unión
overlap = from_crt_sh & from_dns_brute        # intersección
only_brute = from_dns_brute - from_crt_sh     # diferencia

# Test de pertenencia rápido — esto es O(1) contra el O(n) de una list:
if "vpn.acme.com" in all_subs:
    print("known target")
```

> **Pentester Insight:** Cuando dedupliques una wordlist o un resultado de enumeración de subdominios, pásalo siempre por un set. En una wordlist de 10 millones de líneas, `set()` termina en segundos mientras un loop ingenuo `if x not in seen: seen.append(x)` tarda horas.

## 6. Control de flujo

### 6.1 if / elif / else

```python
status_code = 401

if status_code == 200:
    print("OK")
elif status_code in (301, 302, 307, 308):
    print("redirect")
elif status_code == 401:
    print("auth required")
elif 400 <= status_code < 500:
    print("client error")
elif 500 <= status_code < 600:
    print("server error")
else:
    print("unknown")
```

### 6.2 while

```python
import socket
attempts = 0
while attempts < 3:
    try:
        s = socket.create_connection(("10.10.10.50", 22), timeout=2)
        s.close()
        print("open")
        break
    except OSError:
        attempts += 1
else:
    # Corre solo si el while termina sin un break:
    print("port closed or filtered after 3 attempts")
```

### 6.3 for

```python
# Itera una lista de puertos:
for port in [22, 80, 443, 8080]:
    print(f"scanning {port}")

# range para secuencias numéricas:
for port in range(1, 1025):
    pass  # escanea cada puerto well-known

# enumerate cuando necesitas el índice:
targets = ["host1", "host2", "host3"]
for i, host in enumerate(targets, start=1):
    print(f"{i}. {host}")

# zip para recorrer dos iterables en paralelo:
hosts = ["10.10.10.1", "10.10.10.2"]
names = ["gateway", "dc01"]
for ip, name in zip(hosts, names, strict=True):
    print(f"{ip} = {name}")
```

`strict=True` (3.10+) levanta `ValueError` si los iterables tienen distinta longitud — caza bugs temprano.

### 6.4 Comprehensions

Las comprehensions son loops compactos que arman una colección en una expresión. Se leen más rápido que el `for` equivalente una vez que te acostumbras.

```python
# List comprehension — puertos well-known:
well_known = [p for p in range(1, 1025)]

# Con un filtro:
high_ports = [p for p in range(1, 65536) if p > 49152]

# Transforma y filtra:
banners = ["SSH-2.0-OpenSSH_9.6p1", "220 vsFTPd 3.0.5", "HTTP/1.1 200"]
ssh_banners = [b.split("-")[2] for b in banners if b.startswith("SSH-")]
# ['OpenSSH_9.6p1']

# Dict comprehension — mapa de puerto a servicio:
services = {22: "ssh", 80: "http", 443: "https"}
inverted = {name: port for port, name in services.items()}
# {'ssh': 22, 'http': 80, 'https': 443}

# Set comprehension — dominios únicos de una lista de URLs:
urls = ["https://a.com/1", "https://b.com/2", "https://a.com/3"]
domains = {u.split("/")[2] for u in urls}
# {'a.com', 'b.com'}
```

### 6.5 El operador walrus

`:=` asigna y devuelve un valor en una expresión. Especialmente útil en loops que leen hasta agotar.

```python
import socket
s = socket.create_connection(("example.com", 80), timeout=3)
s.sendall(b"GET / HTTP/1.0\r\n\r\n")

# Lee chunks hasta que el socket cierra — el walrus captura cada chunk una sola vez:
chunks = []
while chunk := s.recv(4096):
    chunks.append(chunk)
s.close()
response = b"".join(chunks)
```

Sin el walrus, tendrías que llamar `s.recv()` dos veces por iteración o reestructurar el loop de forma incómoda.

### 6.6 match / case

El pattern matching estructural, agregado en Python 3.10, es más poderoso que un `switch` de C. Matchea tipos, formas y captura valores.

```python
def classify_response(resp):
    match resp:
        case {"status": 200, "body": body}:
            return f"OK, {len(body)} bytes"
        case {"status": code} if 300 <= code < 400:
            return f"redirect ({code})"
        case {"status": 401 | 403}:
            return "auth required"
        case {"status": code} if code >= 500:
            return f"server error ({code})"
        case _:
            return "unhandled"

classify_response({"status": 401})            # 'auth required'
classify_response({"status": 200, "body": b"hello"})  # 'OK, 5 bytes'
```

El operador `|` matchea varios valores en un brazo; `case _` es el default; los guards (`if code >= 500`) agregan matching condicional.

## 7. Funciones

### 7.1 Bases

```python
def is_open(host: str, port: int, timeout: float = 2.0) -> bool:
    """Devuelve True si una conexión TCP a host:port tiene éxito dentro de timeout segundos."""
    import socket
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return True
    except OSError:
        return False

is_open("example.com", 80)                    # True
is_open("example.com", 22, timeout=1.0)       # probablemente False
```

### 7.2 *args y **kwargs

`*args` junta los argumentos posicionales extra en una tuple; `**kwargs` junta los argumentos por keyword extra en un dict.

```python
def scan(*ports, **options):
    verbose = options.get("verbose", False)
    for port in ports:
        if verbose:
            print(f"scanning {port}")

scan(22, 80, 443, verbose=True)

# Desempaquetado en el sitio de la llamada:
port_list = [22, 80, 443]
opts = {"verbose": True, "timeout": 3.0}
scan(*port_list, **opts)
```

### 7.3 Type hints

Los type hints son documentación que las tools pueden chequear. El comportamiento en runtime no cambia, pero mypy y los IDEs cazan errores de categoría antes de que corran.

```python
def http_get(url: str, timeout: float = 5.0) -> str:
    import urllib.request
    with urllib.request.urlopen(url, timeout=timeout) as resp:
        return resp.read().decode("utf-8", errors="replace")

# Sintaxis de función genérica de 3.12+:
def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

first([1, 2, 3])                              # 1
first(["a", "b"])                             # 'a'
first([])                                     # None
```

Los type hints no imponen nada en runtime — son para humanos y para el tooling.

### 7.4 lambda

Funciones anónimas de una sola expresión. Mejor usadas como callbacks cortos.

```python
# Ordena hosts por cantidad de puertos abiertos:
hosts = [
    {"ip": "10.10.10.1", "open": [80]},
    {"ip": "10.10.10.2", "open": [22, 80, 443, 3389]},
    {"ip": "10.10.10.3", "open": [22, 443]},
]
hosts.sort(key=lambda h: len(h["open"]), reverse=True)
```

Si una lambda ocupa más de una línea o recibe un nombre, usa `def`.

### 7.5 Decorators

Un decorator envuelve una función para agregar comportamiento — logging, timing, retries, rate limiting. Para pentesting, cronometrar cada función de scan es un patrón común.

```python
import time
from functools import wraps

def timed(func):
    """Decorator que imprime el tiempo de reloj de pared de la función envuelta."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            print(f"[timed] {func.__name__} took {elapsed*1000:.1f} ms")
    return wrapper

@timed
def sweep(subnet: str, ports: list[int]) -> list[tuple[str, int]]:
    import socket, ipaddress
    open_ports = []
    for ip in ipaddress.ip_network(subnet).hosts():
        for port in ports:
            try:
                with socket.create_connection((str(ip), port), timeout=0.3):
                    open_ports.append((str(ip), port))
            except OSError:
                pass
    return open_ports

# Uso — el decorator imprime el timing automáticamente:
sweep("192.168.56.0/29", [22, 80, 443])
# [timed] sweep took 1834.7 ms
```

`@wraps(func)` preserva el nombre y el docstring de la función envuelta para que los tracebacks sigan siendo útiles.

### 7.6 Generators

Una función generator usa `yield` para producir valores de a uno sin materializar la lista completa en memoria. Esencial para wordlists grandes.

```python
def read_wordlist(path: str):
    """Entrega una línea por vez — nunca carga el archivo entero en memoria."""
    with open(path, encoding="utf-8", errors="replace") as f:
        for line in f:
            line = line.rstrip("\n")
            if line:
                yield line

# Úsalo exactamente como una list:
for word in read_wordlist("/usr/share/wordlists/rockyou.txt"):
    if word.startswith("admin"):
        print(word)
```

Leer `rockyou.txt` (~14 millones de líneas) como list usaría ~130 MB de RAM. Como generator, usa ~4 KB.

## 8. Manejo de errores

El código de red falla todo el tiempo — timeouts, conexiones rechazadas, errores de DNS, responses mal formadas. El manejo robusto de errores es lo que separa un scanner que corre hasta el final de uno que muere en el primer host inalcanzable.

### 8.1 try / except / else / finally

```python
import socket

def probe(host: str, port: int) -> str:
    s = socket.socket()
    s.settimeout(2)
    try:
        s.connect((host, port))
    except socket.timeout:
        return "timeout"
    except ConnectionRefusedError:
        return "refused"
    except socket.gaierror:
        return "dns error"
    except OSError as e:
        return f"error: {e}"
    else:
        # Corre solo si try terminó sin excepción:
        return "open"
    finally:
        # Corre siempre — la limpieza va acá:
        s.close()
```

Nunca captures un `except:` pelado — se traga `KeyboardInterrupt` y `SystemExit` y convierte Ctrl+C en una pesadilla de debugging. Captura tipos de excepción específicos, o `except Exception` si tienes que tirar una red ancha.

### 8.2 Levantar excepciones

```python
def parse_port(s: str) -> int:
    n = int(s)
    if not 1 <= n <= 65535:
        raise ValueError(f"port out of range: {n}")
    return n
```

### 8.3 Excepciones custom

Las clases de excepción custom dejan que quien llama maneje tus modos de falla específicos de forma distinta.

```python
class ScanError(Exception):
    """Base de todos los errores del scanner."""

class HostUnreachable(ScanError):
    def __init__(self, host: str):
        super().__init__(f"host {host} is unreachable")
        self.host = host

class RateLimited(ScanError):
    def __init__(self, retry_after: float):
        super().__init__(f"rate limited, retry after {retry_after}s")
        self.retry_after = retry_after

# Quien llama puede capturar de forma estrecha:
try:
    do_scan()
except RateLimited as e:
    time.sleep(e.retry_after)
except HostUnreachable as e:
    log.warning(f"skipping {e.host}")
except ScanError:
    log.exception("scan failed")
```

### 8.4 Context managers

La sentencia `with` garantiza la limpieza incluso si el cuerpo levanta. Es la forma correcta de manejar sockets, archivos y cualquier recurso que necesite un close explícito.

```python
import socket

# El socket se cierra automáticamente, incluso si send/recv levanta:
with socket.create_connection(("example.com", 80), timeout=3) as s:
    s.sendall(b"HEAD / HTTP/1.0\r\n\r\n")
    print(s.recv(4096))
# el socket se cerró acá, garantizado
```

Puedes escribir tus propios context managers con `contextlib.contextmanager`:

```python
from contextlib import contextmanager
import socket

@contextmanager
def tcp(host: str, port: int, timeout: float = 3.0):
    s = socket.create_connection((host, port), timeout=timeout)
    try:
        yield s
    finally:
        s.close()

with tcp("example.com", 80) as s:
    s.sendall(b"HEAD / HTTP/1.0\r\n\r\n")
    data = s.recv(4096)
```

## 9. I/O de archivos y encoding

### 9.1 Modo texto vs modo binario

```python
# Modo texto — decodifica bytes a str usando el encoding dado:
with open("targets.txt", encoding="utf-8") as f:
    for line in f:
        print(line.rstrip())

# Modo binario — bytes crudos, sin decoding:
with open("shellcode.bin", "rb") as f:
    payload = f.read()

# Especifica siempre el encoding en modo texto. Nunca te apoyes en el default de la plataforma.
```

### 9.2 Escritura

```python
# Escribe texto — pisa cualquier archivo existente:
with open("scan.log", "w", encoding="utf-8") as f:
    f.write("scan started\n")

# Append en lugar de pisar:
with open("scan.log", "a", encoding="utf-8") as f:
    f.write("host 10.10.10.5 open\n")

# Escribe bytes crudos:
with open("payload.bin", "wb") as f:
    f.write(bytes.fromhex("9090909031c0"))
```

### 9.3 Manejo de wordlists

Las wordlists a menudo contienen bytes que no son UTF-8 (`rockyou.txt` es un ejemplo famoso). `errors="replace"` cambia los bytes no decodificables por el carácter de reemplazo Unicode en lugar de levantar.

```python
def load_wordlist(path: str) -> list[str]:
    with open(path, encoding="utf-8", errors="replace") as f:
        return [line.rstrip("\n") for line in f if line.strip()]

words = load_wordlist("/usr/share/wordlists/rockyou.txt")
print(len(words))
```

Para wordlists enormes, usa el patrón de generator de la sección 7.6 para no cargar todo en memoria.

### 9.4 Trampas de encoding

El bug de Python más común en scripts de pentesting es mezclar `str` y `bytes`. Reglas para memorizar:

- Los datos que cruzan un socket, un archivo abierto en modo binario o un pipe de subprocess son `bytes`.
- Los datos que se muestran en una terminal o se escriben a un archivo en modo texto son `str`.
- Convierte de forma explícita con `.encode(encoding)` y `.decode(encoding)`.
- Nunca asumas el encoding por defecto de la plataforma — nómbralo siempre.

```python
# Mal — va a crashear en cualquier byte que no sea ASCII:
data.decode()

# Bien — encoding explícito, manejo explícito de errores:
data.decode("utf-8", errors="replace")
```

## 10. Destacados de la biblioteca estándar para pentesters

La biblioteca estándar de Python sola es un toolkit de pentesting. Estos son los módulos a los que vas a llegar más seguido.

### 10.1 os y sys

```python
import os, sys

os.getenv("PATH")                    # variable de entorno
os.getcwd()                          # directorio de trabajo actual
os.listdir(".")                      # contenido del directorio
sys.argv                             # argumentos de línea de comandos
sys.exit(1)                          # sale con un código de estado
sys.stderr.write("error: ...\n")     # escribe a stderr
```

### 10.2 subprocess

Corre comandos externos de forma limpia. `subprocess.run` es el punto de entrada moderno.

```python
import subprocess

# Captura el output como texto:
result = subprocess.run(
    ["nmap", "-sS", "-p", "22,80,443", "10.10.10.50"],
    capture_output=True,
    text=True,
    timeout=60,
)
print(result.stdout)
print(result.returncode)

# Nunca pases shell=True con input no confiable — riesgo de inyección de comandos.
# Prefiere una lista de argumentos, que no invoca una shell.
```

### 10.3 pathlib

Paths orientados a objetos. Reemplaza la mayor parte del uso de `os.path`.

```python
from pathlib import Path

wordlist_dir = Path("/usr/share/wordlists")
for f in wordlist_dir.rglob("*.txt"):
    print(f.name, f.stat().st_size)

p = Path.home() / "loot" / "creds.txt"
p.parent.mkdir(parents=True, exist_ok=True)
p.write_text("admin:hunter2\n")
```

### 10.4 json

```python
import json

# Parsea un string JSON:
data = json.loads('{"host":"10.10.10.5","open":[22,80]}')
data["open"]                         # [22, 80]

# Serializa:
json.dumps(data, indent=2)

# Lee y escribe archivos JSON:
with open("report.json", encoding="utf-8") as f:
    report = json.load(f)

with open("report.json", "w", encoding="utf-8") as f:
    json.dump(report, f, indent=2)
```

### 10.5 base64

Los tokens web, los headers de HTTP basic auth y muchos desafíos de CTF usan base64.

```python
import base64

# Codifica credenciales para HTTP Basic Auth:
creds = base64.b64encode(b"admin:hunter2").decode("ascii")
header = f"Authorization: Basic {creds}"

# Decode:
base64.b64decode(b"YWRtaW46aHVudGVyMg==")    # b'admin:hunter2'

# Variante URL-safe (usada en JWTs):
base64.urlsafe_b64encode(b"payload")
```

### 10.6 hashlib

```python
import hashlib

# Calcula hashes de una wordlist para cracking offline:
def md5(s: str) -> str:
    return hashlib.md5(s.encode()).hexdigest()

md5("password123")                   # '482c811da5d5b4bc6d497ffa98491e38'
hashlib.sha256(b"admin").hexdigest()
hashlib.sha1(b"admin").hexdigest()

# Hash NTLM (usado en Windows AD):
import hashlib
def nt_hash(password: str) -> str:
    return hashlib.new("md4", password.encode("utf-16-le")).hexdigest()
```

### 10.7 hmac

Para validar firmas de JWT, cookies firmadas y tokens de API.

```python
import hmac, hashlib

secret = b"super-secret-key"
message = b"user=admin&role=user"
sig = hmac.new(secret, message, hashlib.sha256).hexdigest()

# Comparación en tiempo constante — resiste timing attacks:
hmac.compare_digest(sig, "expected-sig")
```

### 10.8 re

Expresiones regulares. Compila una vez, usa muchas.

```python
import re

IP_RE = re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")
EMAIL_RE = re.compile(r"[\w.-]+@[\w.-]+\.\w+")

text = "server 10.10.10.5, contact admin@corp.local"
IP_RE.findall(text)                  # ['10.10.10.5']
EMAIL_RE.findall(text)               # ['admin@corp.local']
```

### 10.9 itertools

Primitivos combinatorios que aparecen todo el tiempo en credential spraying y generación de permutaciones.

```python
import itertools

# Password spraying: cada user contra cada password:
users = ["alice", "bob", "carol"]
passwords = ["Winter2026!", "Spring2026!", "P@ssw0rd"]
for user, pwd in itertools.product(users, passwords):
    print(f"try {user}:{pwd}")

# Parte un iterable en grupos (3.12+ tiene itertools.batched):
for chunk in itertools.batched(range(10), 3):
    print(chunk)
# (0, 1, 2)
# (3, 4, 5)
# (6, 7, 8)
# (9,)
```

## 11. PEP 8 y estilo de código

PEP 8 es la guía de estilo de Python. Seguirla no es preferencia estética — el estilo consistente hace código que otros operadores pueden leer rápido, y cada proyecto serio corre tools automatizadas para imponerlo.

### 11.1 Las reglas que más importan

- 4 espacios de indentación, nunca tabs.
- Líneas de hasta 100 caracteres (PEP 8 dice 79, pero 100 es la elección moderna pragmática).
- `snake_case` para funciones y variables, `PascalCase` para clases, `UPPER_SNAKE` para constantes.
- Dos líneas en blanco entre funciones y clases de nivel superior, una línea en blanco entre métodos.
- Orden de imports: biblioteca estándar, terceros, local — cada grupo separado por una línea en blanco.
- Sin imports sin usar.
- Docstrings para cada función, clase y módulo públicos.

### 11.2 ruff — el linter de una sola parada

ruff es un linter y formatter escrito en Rust que reemplaza a flake8, isort, pyupgrade y black a 10–100 veces la velocidad. Es el estándar en proyectos nuevos de Python en 2026.

```bash
pip install ruff

# Lint:
ruff check .

# Auto-arregla lo que puede:
ruff check --fix .

# Formatea (reemplaza a black):
ruff format .
```

Config en `pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "S", "SIM"]
# E: errores de pycodestyle
# F: pyflakes (imports sin usar, nombres indefinidos)
# I: isort (orden de imports)
# N: pep8-naming
# UP: pyupgrade (sintaxis moderna)
# B: bugbear (patrones de bugs comunes)
# S: bandit (issues de seguridad)
# SIM: sugerencias de simplificación
```

### 11.3 black — si lo prefieres

black es el formatter más viejo y opinado. Sigue funcionando y está bien.

```bash
pip install black
black .
```

Elige un formatter por proyecto y quédate con él. Mezclar formatters produce ruido en cada diff.

### 11.4 mypy — chequeo estático de tipos

Si usas type hints, corre mypy para cazar bugs que el intérprete no va a cazar.

```bash
pip install mypy
mypy scanner.py

# el modo strict caza más:
mypy --strict scanner.py
```

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unused_ignores = true
```

> **Pentester Insight:** Para scripts de una sola vez, sáltate la ceremonia. Para cualquier tool que vayas a usar dos veces, o compartir con un compañero, cablea ruff y mypy desde el principio. Diez minutos de setup de tooling ahorran horas de debugging meses después.

## 12. Armar una tool chica — enumerador de headers HTTP

Hora de armar lo que aprendiste. Este es un enumerador de headers HTTP de unas 40 líneas que se conecta a un target, manda un request HEAD, imprime todos los headers de response y marca issues comunes de headers de seguridad. Usa solo la biblioteca estándar.

```python
#!/usr/bin/env python3
"""http_headers.py — enumera y evalúa los headers de response HTTP."""
from __future__ import annotations

import socket
import ssl
import sys
from urllib.parse import urlparse

SECURITY_HEADERS = {
    "strict-transport-security": "HSTS not set",
    "content-security-policy": "CSP not set",
    "x-frame-options": "clickjacking protection missing",
    "x-content-type-options": "MIME sniffing not disabled",
    "referrer-policy": "referrer policy not set",
    "permissions-policy": "permissions policy not set",
}


def fetch_headers(url: str, timeout: float = 5.0) -> dict[str, str]:
    parsed = urlparse(url)
    host = parsed.hostname
    if not host:
        raise ValueError(f"invalid URL: {url}")
    port = parsed.port or (443 if parsed.scheme == "https" else 80)
    path = parsed.path or "/"

    sock = socket.create_connection((host, port), timeout=timeout)
    if parsed.scheme == "https":
        ctx = ssl.create_default_context()
        sock = ctx.wrap_socket(sock, server_hostname=host)

    with sock:
        request = f"HEAD {path} HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n"
        sock.sendall(request.encode("ascii"))
        chunks: list[bytes] = []
        while chunk := sock.recv(4096):
            chunks.append(chunk)

    raw = b"".join(chunks).decode("iso-8859-1", errors="replace")
    header_block = raw.split("\r\n\r\n", 1)[0]
    lines = header_block.split("\r\n")
    status_line = lines[0]
    headers = {}
    for line in lines[1:]:
        if ":" in line:
            name, _, value = line.partition(":")
            headers[name.strip().lower()] = value.strip()
    headers["_status"] = status_line
    return headers


def main(argv: list[str]) -> int:
    if len(argv) != 2:
        print(f"usage: {argv[0]} <url>", file=sys.stderr)
        return 2
    try:
        headers = fetch_headers(argv[1])
    except (OSError, ValueError) as e:
        print(f"error: {e}", file=sys.stderr)
        return 1

    print(headers.pop("_status"))
    for name, value in sorted(headers.items()):
        print(f"  {name}: {value}")

    print("\n[security header audit]")
    for name, message in SECURITY_HEADERS.items():
        marker = "OK" if name in headers else "MISSING"
        print(f"  [{marker}] {name} — {message if marker == 'MISSING' else headers[name]}")
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Guarda el archivo, hazlo ejecutable y córrelo:

```bash
chmod +x http_headers.py
./http_headers.py https://example.com
# HTTP/1.1 200 OK
#   accept-ranges: bytes
#   age: 12345
#   cache-control: max-age=3600
#   content-length: 1256
#   content-type: text/html; charset=UTF-8
#   date: Fri, 04 Sep 2026 12:00:00 GMT
#   etag: "84238dfc8092e5d9c0dac8ef93371a07"
#   last-modified: Thu, 17 Oct 2024 00:00:00 GMT
#   server: ECAcc (nyd/D190)
#   vary: Accept-Encoding
#
# [security header audit]
#   [MISSING] strict-transport-security — HSTS not set
#   [MISSING] content-security-policy — CSP not set
#   [MISSING] x-frame-options — clickjacking protection missing
#   [MISSING] x-content-type-options — MIME sniffing not disabled
#   [MISSING] referrer-policy — referrer policy not set
#   [MISSING] permissions-policy — permissions policy not set
```

Fíjate qué demuestra esta tool de todo lo que aprendiste:

- Type hints (`dict[str, str]`, `list[bytes]`) — sección 7.3
- El operador walrus dentro del loop de `recv()` — sección 6.5
- Context manager (`with sock:`) — sección 8.4
- Manejo explícito de excepciones con una tuple estrecha — sección 8.1
- Diccionario como tabla de lookup para chequeos de seguridad — sección 5.7
- Cruce de la frontera bytes/str — sección 5.4 y sección 9.4
- Solo biblioteca estándar — sin dependencias, sin virtualenv requerido para correr

## 13. Ejercicios

### Ejercicios guiados

#### Ejercicio 1 — Port scanner

**Objetivo:** escribe un script `portscan.py` que tome un host y una lista de puertos separada por comas desde la línea de comandos e imprima qué puertos están abiertos. Usa `socket.create_connection` con un timeout de 1 segundo.

**Pista:** sigue el patrón de `is_open()` de la sección 7.1. Itera con un `for` sobre la lista de puertos y usa un bloque `try`/`except OSError`.

#### Ejercicio 2 — Filtro de wordlist

**Objetivo:** escribe un script que lea un archivo de wordlist desde stdin e imprima solo las líneas que tengan entre 8 y 16 caracteres y contengan al menos un dígito. Usa un generator para mantener bajo el uso de memoria.

**Pista:** lee de `sys.stdin` línea por línea, haz `.rstrip("\n")` de cada línea y usa `any(c.isdigit() for c in line)` para chequear dígitos.

#### Ejercicio 3 — Merger de reportes JSON

**Objetivo:** escribe un script `merge_scans.py` que lea dos archivos JSON, cada uno con la forma `{"host": str, "open_ports": list[int]}`, y produzca un reporte mergeado que agrupe hosts y una los conjuntos de puertos abiertos.

**Pista:** usa `json.load` para leer, un dict con key por host y valor set, y `json.dump(..., indent=2)` para escribir.

### Ejercicios independientes

#### Ejercicio 4 — Decorator de timing con umbral

**Objetivo:** extiende el decorator `@timed` de la sección 7.5 para que acepte un argumento de umbral. Si una llamada tarda más que el umbral, imprime un warning a stderr; si no, quédate callado.

**Pista:** necesitas una factory de decorators — una función que devuelve un decorator. El uso debería verse como `@timed(threshold_ms=500)`.

#### Ejercicio 5 — Deduplicador de subdominios

**Objetivo:** escribe una tool que tome cualquier cantidad de paths de archivos de texto en la línea de comandos, lea subdominios uno por línea de cada uno, los deduplique sin distinguir mayúsculas, los ordene e imprima el resultado.

**Pista:** un set de strings en minúsculas, después `sorted()` sobre el resultado. Usa `pathlib.Path` para iterar los argumentos.

#### Ejercicio 6 — Generador de header Basic Auth

**Objetivo:** escribe una utilidad chica que acepte un username y una password (como argumentos o desde stdin) e imprima un valor completo de header `Authorization: Basic ...`. Maneja caracteres que no son ASCII de forma correcta codificando como UTF-8 antes del base64.

**Pista:** `base64.b64encode(f"{user}:{pwd}".encode("utf-8")).decode("ascii")`. Considera qué pasa si el user o la password contienen dos puntos.

## 14. Resumen y lección siguiente

Ahora tienes cada fundamental de Python que necesitas para armar tooling ofensivo. Instalaste un intérprete moderno en Kali 2026, aislaste tus proyectos con virtualenvs (venv, pipx, uv) y recorriste el lenguaje central — tipos de datos, control de flujo, funciones, manejo de errores, I/O de archivos — desde un ángulo de pentesting. Viste los módulos de la biblioteca estándar que aparecen en cada script serio y terminaste armando un enumerador real de headers HTTP usando solo lo que viene con Python.

Ideas clave para llevar:

- Trabaja siempre dentro de un virtualenv. uv es la elección moderna más rápida; venv es el fallback confiable.
- `bytes` y `str` son tipos distintos. Cruza la frontera de forma explícita con `.encode()` y `.decode()`, siempre con un encoding nombrado.
- Cada recurso de socket, archivo y subprocess pertenece dentro de un bloque `with`.
- Captura tipos de excepción específicos, nunca un `except:` pelado.
- Type hints más ruff más mypy cazan bugs antes de que corras el código.
- Prefiere generators sobre lists para cualquier cosa que lea un archivo grande o un stream largo.

Lección siguiente — Lección 02: Python for Network Programming. Vas a construir sobre esta base para escribir clientes y servers de socket crudos, usar asyncio para scanning concurrente de miles de hosts, trabajar con scapy para armar paquetes custom en Layer 2 y Layer 3, e implementar un TCP SYN scanner que funcione, desde cero.
