# Lección 05 — Building Report Generation Skills

Módulo 06: Building Pentest Skills with Claude and Codex · Penetration Testing with Kali Linux 1.0 · Red Team Leaders

## Introducción

Pregúntale a cualquier pentester senior cómo pasa la semana y la respuesta honesta es incómoda: la mayoría de las horas facturables no se gastan tecleando exploits en una shell; se gastan tecleando prosa en un report. Las capturas hay que recortarlas y ponerles pie. Las requests y las responses hay que sanitizarlas. Los vectores CVSS hay que discutirlos en Slack. Los executive summaries hay que reescribirlos porque el CISO quiere tres párrafos, no cinco. Las secciones de metodología hay que copiarlas del engagement anterior y luego editarlas para que el cliente no se dé cuenta.

Esa carga de reporting es la palanca de productividad más grande de una práctica de seguridad ofensiva. Si puedes comprimir un sprint de escritura de cinco días en un sprint de un día de revisión y envío, has duplicado en la práctica el número de engagements que un consultor puede entregar en un año, sin recortar ni una esquina de calidad. La forma de comprimir ese sprint es dejar de tratar el report como un documento Word monolítico armado a mano y empezar a tratarlo como un artefacto de build producido por un pipeline de skills pequeñas y de scope bien acotado.

Esta lección te enseña a construir ese pipeline como un conjunto de ocho skills de Claude Code, cada una con un contrato estrecho, entradas deterministas y salidas predecibles. El pipeline lee de una fuente de verdad en YAML, la enriquece con inteligencia de amenazas externa, la renderiza a través de una plantilla Typst y produce un PDF listo para el cliente. Cada paso es auditable. Cada párrafo redactado por un LLM pasa por una compuerta de revisión humana. Cada tarjeta de finding se ve idéntica a cualquier otra tarjeta de finding, engagement tras engagement, porque la plantilla —no el humor del tester a las 11 de la noche del día de entrega— decide el formato.

Al terminar esta lección tendrás un catálogo de skills que puedes copiar a cualquier engagement de Red Team Leaders y ejecutar de punta a punta, desde `findings/*/finding.yml` hasta `report.pdf`, en una sola sesión de Claude Code. También vas a entender dónde termina la asistencia del LLM y dónde empieza el juicio humano, que es el límite más importante de cualquier proceso de entregable asistido por IA.

## 1. El pipeline de findings — YAML como fuente de verdad

Antes de escribir cualquier skill, tienes que comprometerte con un modelo de datos. Cada finding del pipeline tiene exactamente una representación canónica: un archivo YAML en `./findings/<slug>/finding.yml`. Todo lo demás —la tarjeta Markdown, el recuadro del finding en el apéndice, la fila de la matriz de riesgo del executive summary, la entrada del export CSV que pidió el cliente— se deriva de ese YAML por transformación. Ninguna skill del pipeline puede inventar un campo que no aparezca en el esquema. Ningún humano puede editar a mano la tarjeta Markdown sin actualizar primero el YAML.

El pipeline tiene cuatro etapas:

```text
┌──────────────┐    ┌────────────────┐    ┌──────────────┐    ┌──────────┐
│  finding.yml │ ─► │ skill enrich   │ ─► │ Markdown     │ ─► │ Typst    │
│  (raw input) │    │ CVSS, CVE,     │    │ finding card │    │ compile  │
│              │    │ OWASP, ATT&CK  │    │ + summary    │    │ → PDF    │
└──────────────┘    └────────────────┘    └──────────────┘    └──────────┘
```

Cada etapa es idempotente. Ejecutar finding-enrich dos veces sobre el mismo YAML produce la misma salida. Ejecutar finding-card dos veces produce Markdown idéntico byte a byte. Esa propiedad es lo que hace seguro volver a ejecutar el pipeline en cualquier punto: si el cliente pide un cambio al executive summary a las 4 de la tarde del día de entrega, editas el YAML, vuelves a ejecutar las tres últimas etapas y el PDF se regenera en segundos.

El esquema canónico de `finding.yml` es:

```yaml
id: RTL-2026-0042
slug: apache-struts-rce-cve-2017-5638
title: Ejecución remota de código en la cabecera Content-Type de Apache Struts
severity: critical
status: draft
target:
  hostname: erp.example.internal
  ip: 10.20.30.40
  port: 8080
  url: https://erp.example.internal/struts2-showcase/showcase.action
  asset_class: production-web-application
discovered:
  by: joao.antonio
  date: 2026-09-04
  tool: burp-suite-scanner
  request_id: burp-req-000817
description: |
  La aplicación ejecuta Apache Struts 2.3.20, vulnerable a CVE-2017-5638.
  El parser Jakarta Multipart evalúa expresiones OGNL suministradas en la
  cabecera Content-Type, lo que conduce a ejecución remota de comandos
  no autenticada como la cuenta de servicio tomcat.
impact: |
  Un atacante no autenticado puede ejecutar comandos arbitrarios del
  sistema operativo con los privilegios de la cuenta de servicio Tomcat.
  La cuenta Tomcat tiene acceso de lectura a las credenciales de la base
  de datos de clientes almacenadas en /opt/tomcat/conf/db.properties, lo
  que otorga acceso directo a 2.4M registros de clientes, incluida PII
  regulada bajo la LGPD.
cvss:
  version: "4.0"
  vector: ""       # lo rellena cvss-score
  base_score: 0    # lo rellena cvss-score
  rationale: ""    # lo rellena cvss-score
references:
  cve: []          # lo rellena finding-enrich
  owasp_top_10: [] # lo rellena finding-enrich
  wstg: []         # lo rellena finding-enrich
  attack: []       # lo rellena finding-enrich
evidence:
  screenshots: []  # lo rellena evidence-collect
  requests: []     # lo rellena evidence-collect
  responses: []    # lo rellena evidence-collect
  tool_output: []  # lo rellena evidence-collect
remediation:
  short: ""        # autoría humana
  detailed: ""     # autoría humana
retest:
  status: not-yet-retested
  date: null
  outcome: null
```

Cada campo del esquema tiene un autor designado. Algunos son de autoría humana (`title`, `description`, `impact`, `remediation`). Algunos son de autoría de la skill (`cvss`, `references`, `evidence`). Algunos son de autoría de la máquina en el momento de la importación (`discovered`, `id`). El pipeline hace cumplir esto negándose a ejecutar skills de enriquecimiento sobre campos que debe escribir el humano, y negándose a dejar que los humanos sobrescriban campos que deben escribir las máquinas.

## 2. Skill 1 — finding-create

La primera skill es la más simple: toma un título, una descripción, punteros de evidencia y un objetivo, y produce en disco un YAML normalizado con el slug de nombre de archivo correcto y un ID generado.

### 2.1 Contrato

Entradas:

- `title` (string, obligatorio) — título del finding legible por humanos
- `description` (string, obligatorio) — uno a tres párrafos de descripción técnica
- `target_url` (string, obligatorio) — la URL, el host o el activo contra el que va el finding
- `evidence_paths` (lista de strings, opcional) — rutas a archivos de evidencia cruda que se organizarán después
- `severity_hint` (string, opcional) — `critical` | `high` | `medium` | `low` | `info`, usado solo como punto de partida

Salidas:

- Crea `./findings/<slug>/finding.yml` con el esquema de arriba
- Crea `./findings/<slug>/evidence/` como directorio vacío
- Devuelve el slug a quien llama

Garantías:

- El slug se deriva del título mediante un slugifier determinista: minúsculas, alfanumérico más guiones, máximo 60 caracteres
- El ID es `RTL-<año>-<contador>`, donde el contador es el siguiente entero no usado bajo `./findings/`
- El archivo no se creará si ya existe un finding con el mismo slug: la skill rechaza en lugar de sobrescribir

### 2.2 SKILL.md completo

````markdown
---
name: finding-create
description: |
  Crea un finding nuevo de pentest como archivo YAML en
  ./findings/<slug>/finding.yml. Úsala siempre que una vulnerabilidad
  nueva haya sido confirmada y deba entrar en el pipeline de reporting.
inputs:
  title:
    type: string
    required: true
  description:
    type: string
    required: true
  target_url:
    type: string
    required: true
  evidence_paths:
    type: array
    required: false
  severity_hint:
    type: string
    required: false
    enum: [critical, high, medium, low, info]
allowed_tools: [Read, Write, Bash]
---

# finding-create

## Cuándo invocarla

El usuario dice una de estas frases:

- "log a new finding"
- "create finding for <thing>"
- "add this vuln to the report"
- pega una descripción en bruto y te pide archivarla

## Procedimiento

1. Deriva el slug a partir del título:
   - Pasa a minúsculas.
   - Sustituye cualquier carácter no alfanumérico por un guion.
   - Colapsa los guiones consecutivos.
   - Quita los guiones iniciales y finales.
   - Trunca a 60 caracteres.
   - Si el slug resultante está vacío, aborta con un error.

2. Calcula el siguiente ID:
   - Ejecuta `ls ./findings/ | wc -l` para contar los findings existentes.
   - Suma uno; formatea como `RTL-<año-actual>-<contador con relleno de ceros a 4>`.

3. Comprueba colisiones:
   - Si `./findings/<slug>/finding.yml` ya existe, aborta y dile al
     usuario que use `finding-update` en su lugar.

4. Escribe el YAML usando el esquema canónico.
   - `severity` = `severity_hint` si se proporcionó; si no, `medium`.
   - `status` = `draft`.
   - `discovered.by` = el `user.email` actual de git si está disponible; si no, `unknown`.
   - `discovered.date` = hoy (YYYY-MM-DD).
   - Todos los campos de autoría de la skill quedan en blanco, como muestra el esquema.

5. Crea `./findings/<slug>/evidence/` como directorio vacío.

6. Si se proporcionó `evidence_paths`, NO muevas todavía los archivos: eso
   es trabajo de `evidence-collect`. Limítate a registrar las rutas en un
   archivo temporal `.pending-evidence` dentro del directorio del finding.

7. Devuelve: "Created finding <ID> at ./findings/<slug>/finding.yml".

## No-objetivos

- Esta skill nunca calcula CVSS. Eso es `cvss-score`.
- Esta skill nunca obtiene metadatos de CVE. Eso es `finding-enrich`.
- Esta skill nunca renderiza Markdown. Eso es `finding-card`.
````

### 2.3 Ejemplo de invocación

```text
Usuario: log a new finding — Apache Struts RCE on erp.example.internal port 8080,
      unauthenticated content-type OGNL injection, evidence in ~/burp/struts.zip

Claude (invoca finding-create):
  slug     = "apache-struts-rce-on-erp-example-internal"
  id       = "RTL-2026-0042"
  → wrote ./findings/apache-struts-rce-on-erp-example-internal/finding.yml
  → created ./findings/apache-struts-rce-on-erp-example-internal/evidence/
  → recorded pending evidence: ~/burp/struts.zip
```

Fíjate en que la skill nunca escribe el texto de `description`, `impact` ni `remediation` a partir de su propia comprensión de Struts. Esas son entradas humanas. Registra lo que se le da, al pie de la letra.

## 3. Skill 2 — cvss-score

La puntuación CVSS es la parte más contenciosa de un report de pentest. Dos testers que miran el mismo finding pueden producir vectores que difieren en dos o tres niveles. La skill cvss-score no resuelve ese desacuerdo —los humanos siguen en desacuerdo— pero impone un punto de partida consistente y obliga al tester a justificar cada métrica.

### 3.1 Qué hace la skill

- Lee los campos `description`, `impact` y `target` del finding.
- Usa el LLM para redactar una cadena de vector CVSS 4.0.
- Valida el vector contra la especificación CVSS 4.0: se niega a escribir combinaciones inválidas.
- Calcula el base score a partir del vector con una fórmula determinista (no el LLM).
- Escribe un párrafo de fundamento que explica cada elección de métrica.
- Marca el bloque `cvss` del finding como `pending-review` para que un humano tenga que firmar.

### 3.2 SKILL.md completo

````markdown
---
name: cvss-score
description: |
  Calcula un vector CVSS 4.0, el base score y el fundamento de un finding.
  Usa asistencia del LLM para redactar el vector a partir de la descripción,
  luego lo valida contra la especificación CVSS 4.0 y calcula el base score
  de forma determinista.
inputs:
  slug:
    type: string
    required: true
allowed_tools: [Read, Edit, Bash]
---

# cvss-score

## Cuándo invocarla

- Después de que `finding-create` haya producido un YAML con `description` e `impact`.
- Cuando el usuario pide "score this finding" o "add CVSS".
- Rechaza si `description` o `impact` están vacíos: no puedes puntuar el aire.

## Procedimiento

1. Lee `./findings/<slug>/finding.yml`.

2. Si `cvss.vector` ya está relleno y `cvss.status` es `approved`,
   rechaza y dile al usuario que ejecute `cvss-score --force` (un flujo aparte).

3. Redacta el vector.
   Lee los campos `description`, `impact` y `target`. Para cada una de las once
   métricas base de CVSS 4.0, elige un valor y registra una justificación
   de una frase. Las once métricas son:

   ```text
   AV (Attack Vector)     — N | A | L | P
   AC (Attack Complexity) — L | H
   AT (Attack Requirements) — N | P
   PR (Privileges Required) — N | L | H
   UI (User Interaction)  — N | P | A
   VC (Vuln Confidentiality Impact) — H | L | N
   VI (Vuln Integrity Impact)       — H | L | N
   VA (Vuln Availability Impact)    — H | L | N
   SC (Subsequent Confidentiality Impact) — H | L | N
   SI (Subsequent Integrity Impact)       — H | L | N
   SA (Subsequent Availability Impact)    — H | L | N
   ```

4. Ensambla la cadena del vector en el orden canónico:

   ```text
   CVSS:4.0/AV:.../AC:.../AT:.../PR:.../UI:.../VC:.../VI:.../VA:.../SC:.../SI:.../SA:...
   ```

5. Valida.
   Ejecuta `cvss4-validate "<vector>"`: una tool pequeña de Python incluida
   que comprueba el vector contra la especificación CVSS 4.0. Si devuelve
   distinto de cero, el borrador del LLM está roto; regenera una vez y luego
   aborta con el error de validación si vuelve a fallar.

6. Calcula el base score.
   Ejecuta `cvss4-score "<vector>"`: otra vez una tool incluida, no el LLM.
   Al LLM no se le permite calcular el número; solo se le permite
   redactar el vector. Este es el límite entre lo determinista y lo redactado.

7. Escribe de vuelta en el YAML:

   ```yaml
   cvss:
     version: "4.0"
     vector: "<vector validado>"
     base_score: <calculado>
     status: pending-review
     rationale: |
       AV:N — la vulnerabilidad está expuesta en una interfaz de red alcanzable
       desde la LAN corporativa sin autenticación.
       AC:L — no se requieren condiciones especiales para el exploit.
       ...
   ```

8. Devuelve: "Scored finding <slug> at <base_score> (<severity>). Vector marked pending-review; human sign-off required before ship."

## Notas

- Nunca adivines métricas que no puedas sostener a partir de la descripción. Pregúntale al usuario.
- Si VC/VI/VA son todas `N` y SC/SI/SA son todas `N`, el score es 0.0. Eso es
  casi siempre un error en la descripción; pídele al usuario que la amplíe.
````

### 3.3 El límite determinista

Esta skill es el ejemplo más claro del pipeline del límite entre lo determinista y el LLM. El LLM redacta el vector porque leer inglés y mapearlo a valores de métrica es exactamente lo que los LLM hacen bien. El score lo calcula una tool de Python incluida porque la aritmética es exactamente lo que los LLM hacen mal. Si dejas que el LLM calcule el score, de vez en cuando obtendrás un vector de 7,4 informado como 8,1, y eso acabará en un PDF del cliente. No hagas eso.

## 4. Skill 3 — finding-enrich

Una vez puntuado un finding, hay que atarlo al panorama de amenazas más amplio: números de CVE, categorías de OWASP Top 10, IDs de prueba WSTG y técnicas de MITRE ATT&CK. Ese contexto importa por dos razones: permite que el blue team del cliente mapee tu finding a detecciones que ya tiene, y le demuestra al auditor del cliente que la evaluación cubrió marcos de amenaza reconocidos.

### 4.1 SKILL.md completo

````markdown
---
name: finding-enrich
description: |
  Adjunta metadatos de CVE, categorías de OWASP Top 10, IDs de prueba WSTG
  e IDs de técnica de MITRE ATT&CK a un finding. Tira de NVD y de tablas
  de referencia incluidas; se niega a inventar identificadores.
inputs:
  slug:
    type: string
    required: true
  cve_hint:
    type: string
    required: false
    description: "Si el finding es una CVE conocida, nómbrala (p. ej. CVE-2017-5638)."
allowed_tools: [Read, Edit, Bash, WebFetch]
---

# finding-enrich

## Cuándo invocarla

- Después de que `cvss-score` haya producido un vector validado.
- El usuario pide "enrich" un finding, "add references" o "add ATT&CK mapping".

## Procedimiento

1. Lee `./findings/<slug>/finding.yml`.

2. Resuelve los metadatos de la CVE.
   Si se pasó `cve_hint`, verifica que el formato coincide con `CVE-YYYY-NNNN+`
   y obtén los datos de NVD:

   ```text
   WebFetch https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=<CVE>
   ```

   Extrae:
   - Fecha de publicación
   - ID(s) de CWE
   - Productos afectados (`cpeMatch`)
   - URLs de referencia

   Si la CVE no está en NVD, anota ese hecho, pero no inventes una descripción.

3. Mapea a OWASP Top 10 (edición 2021).
   Lee la tabla incluida `refs/owasp-top-10-2021.yml`. Para cada entrada,
   comprueba si la descripción del finding o el CWE coincide con su lista
   de CWE incluidos. Adjunta cada categoría coincidente como:

   ```yaml
   references.owasp_top_10:
     - id: A03:2021
       name: Injection
       reason: "CWE-77 command injection coincide con la categoría A03."
   ```

4. Mapea a WSTG.
   Lee `refs/wstg-4.2.yml`. Empareja por CWE o por palabra clave. Adjunta:

   ```yaml
   references.wstg:
     - id: WSTG-INPV-12
       name: "Testing for Command Injection"
       url: https://owasp.org/.../WSTG-INPV-12/
   ```

5. Mapea a MITRE ATT&CK.
   Lee `refs/attack-enterprise-14.yml`. Empareja por palabras clave de técnica
   extraídas de la sección `impact` del finding (no de la descripción).
   Adjunta hasta tres IDs de técnica de mayor relevancia:

   ```yaml
   references.attack:
     - id: T1190
       name: "Exploit Public-Facing Application"
       tactic: "Initial Access"
   ```

6. Escribe de vuelta en el YAML. Nunca sobrescribas entradas existentes; anexa.

7. Devuelve un resumen de un párrafo de lo que se adjuntó, para que un humano
   pueda revisar los mapeos por muestreo.

## Rechazos

- No adjuntes una CVE a menos que se haya pasado como `cve_hint` Y se haya
  resuelto con éxito en NVD. Los números de CVE alucinados en un report
  de cliente son un evento que acaba con una carrera.
- No adjuntes un ID de WSTG o de ATT&CK que no esté en la tabla incluida.
  Las tablas son la verdad de base.
````

### 4.2 Por qué tablas incluidas

La skill se apoya en tres archivos YAML incluidos: `refs/owasp-top-10-2021.yml`, `refs/wstg-4.2.yml`, `refs/attack-enterprise-14.yml`. Se entregan con el catálogo de skills y se versionan en git. La skill no golpeará los sitios en vivo de OWASP o MITRE para estos mapeos: usa las tablas congeladas. ¿Por qué? Porque un report de cliente tiene que ser reproducible dentro de seis meses, y los sitios en vivo cambian. Las tablas se actualizan en una cadencia programada (una vez por trimestre) y la actualización es un acto deliberado, no un accidente.

Las búsquedas de CVE son la excepción porque las entradas de NVD son estables una vez publicadas y los clientes esperan datos actuales.

## 5. Skill 4 — evidence-collect

Cada finding necesita evidencia, y organizar la evidencia es la segunda actividad que más tiempo consume en el reporting, después de escribir prosa. La skill evidence-collect toma archivos sueltos —capturas, exports de Burp, salida de curl, resultados de Nmap— y los organiza en una estructura canónica con un manifiesto.

### 5.1 Diseño de directorio que produce

```text
./findings/<slug>/evidence/
├── manifest.yml
├── screenshots/
│   ├── 001-login-page.png
│   ├── 002-post-injection.png
│   └── 003-shell-output.png
├── requests/
│   ├── 001-poc-request.http
│   └── 002-followup-request.http
├── responses/
│   ├── 001-poc-response.http
│   └── 002-followup-response.http
└── tool-output/
    ├── nmap-scan.txt
    ├── burp-scanner.xml
    └── curl-verify.log
```

### 5.2 SKILL.md completo

````markdown
---
name: evidence-collect
description: |
  Organiza archivos de evidencia cruda de un finding en el diseño de
  directorio canónico bajo ./findings/<slug>/evidence/ y escribe un
  manifest.yml que describe cada archivo, su origen y su hash de contenido.
inputs:
  slug:
    type: string
    required: true
  paths:
    type: array
    required: false
    description: "Archivos de evidencia adicionales a importar; por defecto lee .pending-evidence."
allowed_tools: [Read, Write, Bash]
---

# evidence-collect

## Procedimiento

1. Lee `./findings/<slug>/.pending-evidence` si existe; combínalo con
   la entrada `paths`.

2. Para cada ruta de origen:
   a. Clasifica por extensión y contenido:
      - `.png`, `.jpg`, `.jpeg`, `.gif` → `screenshots/`
      - `.http`, `.req`, `.raw` (o texto de cabeceras y luego cuerpo) → `requests/` o `responses/`
      - `.xml`, `.txt`, `.log`, `.json` → `tool-output/`
   b. Copia (no muevas) al directorio de destino con un prefijo numerado
      en el orden de inserción.
   c. Calcula el hash SHA-256 del archivo.
   d. Registra en `manifest.yml`:

      ```yaml
      - name: 001-login-page.png
        source: ~/burp/login.png
        type: screenshot
        sha256: e3b0c4...
        added: 2026-09-05T14:22:03-03:00
        note: ""    # editable por el humano
      ```

3. Si se incluye una sesión de Burp Suite `.zip` o `.xml`, desempáquetala y
   divídela en pares request/response.

4. Nunca borres ni modifiques un archivo de origen. La evidencia original
   del pentester permanece intacta.

5. Al terminar, imprime el manifiesto para que el humano pueda añadir notas.

## No-objetivos

- No redactes secretos ni PII de forma automática. Eso es `evidence-redact`
  (no forma parte de esta lección).
- No incrustes evidencia en Markdown. La skill de tarjeta referencia rutas.
````

### 5.3 Por qué los hashes

El hash SHA-256 del manifiesto no es decoración. Es un control de cadena de custodia. Si el cliente pregunta alguna vez «¿esta es la misma captura que tomaste el 2026-09-04?», puedes producir el hash y demostrar que el archivo no ha sido alterado. Algunos engagements regulados (servicios financieros, salud, gobierno) exigen este nivel de integridad de la evidencia.

## 6. Skill 5 — finding-card

La tarjeta de finding es la representación de cara al cliente de un solo finding. Es un documento Markdown que se renderiza dentro del PDF final como una sección autocontenida. La skill de tarjeta convierte el YAML en Markdown usando una plantilla fija: nada de reinterpretación creativa, solo sustitución de datos en la plantilla.

### 6.1 La plantilla de tarjeta de Red Team Leaders

````markdown
## <ID> — <Title>

**Severity:** <Severity> (CVSS 4.0 <Score> — `<Vector>`)
**Status:** <Status>
**Target:** `<Hostname>` (<IP>) — <URL>
**Discovered:** <Date> by <Tester>

### Description

<Description paragraph(s)>

### Impact

<Impact paragraph(s)>

### Evidence

**Screenshot — <caption from manifest note>**

![<caption>](evidence/screenshots/001-...png)

**Request:**

```http
<contents of requests/001-...http>
```

**Response:**

```http
<contents of responses/001-...http, truncated to 40 lines>
```

### References

- CVE: <linked CVE list, or "None assigned">
- OWASP Top 10 (2021):
- OWASP WSTG:
- MITRE ATT&CK:

### Remediation

Short-term: <remediation.short>

Long-term: <remediation.detailed>
````

### 6.2 SKILL.md completo

````markdown
---
name: finding-card
description: |
  Renderiza un YAML de finding en una tarjeta Markdown siguiendo la plantilla
  de Red Team Leaders. Sustitución determinista: nada de reescritura, nada
  de parafraseo, nada de suavizado.
inputs:
  slug:
    type: string
    required: true
  out:
    type: string
    required: false
    default: "./build/cards/<slug>.md"
allowed_tools: [Read, Write, Bash]
---

# finding-card

## Procedimiento

1. Lee `./findings/<slug>/finding.yml`.
2. Lee `./findings/<slug>/evidence/manifest.yml`.
3. Niégate a renderizar si falta o está vacío cualquiera de estos elementos:
   - `title`, `description`, `impact`
   - `cvss.vector`, `cvss.base_score`, `cvss.status == approved`
   - `remediation.short`
4. Carga la plantilla incluida en `templates/finding-card.md.tmpl`.
5. Sustituye los campos usando las etiquetas estilo Jinja de la plantilla.
6. Para las capturas, itera el manifiesto en orden numérico. Para cada una,
   emite una línea de pie usando `manifest.note` y luego la referencia de la imagen.
7. Para requests/responses, incluye los tres primeros pares al pie de la letra.
   Trunca las responses de más de 40 líneas con `... (truncated, N more lines)`.
8. Escribe en la ruta de salida (por defecto `./build/cards/<slug>.md`).
9. Imprime las primeras 20 líneas para que el usuario pueda revisar por muestreo.

## No-objetivos

- No reescribas la descripción. No «mejores» la redacción del impacto.
  El YAML es la fuente de verdad. Si la prosa está mal, edita el YAML.
- No añadas referencias que no estén presentes en el YAML.
- No calcules ni vuelvas a derivar la severity a partir del score. Usa lo que
  dice el YAML. Si entran en conflicto, es un bug que el humano debe resolver.
````

### 6.3 Por qué rechazar si faltan campos

El rechazo del paso 3 es deliberado. Si el CVSS sigue en `pending-review`, la tarjeta renderizaría un documento de cara al cliente con un score sin revisar. Si `remediation.short` está vacío, el cliente va a recibir un finding sin guía de corrección. La skill rechaza en lugar de emitir en silencio una tarjeta rota. El tester ve el rechazo y llena el hueco.

## 7. Skill 6 — exec-summary

El executive summary es la sección que lee cada ejecutivo y que cada tester teme. Tiene que comprimir un engagement entero en tres párrafos de lenguaje sobre el que pueda actuar un CISO o un COO no técnico. La skill exec-summary lo redacta a partir de los YAML de findings, con advertencias fuertes sobre la revisión humana.

### 7.1 SKILL.md completo

````markdown
---
name: exec-summary
description: |
  Redacta el executive summary de tres párrafos de un report leyendo
  cada YAML de finding bajo ./findings/ y sintetizando contexto, riesgos
  principales y acciones recomendadas.
inputs:
  engagement:
    type: string
    required: true
    description: "El nombre o el ID del engagement."
  scope_notes:
    type: string
    required: false
    description: "Cualquier contexto de scope que deba dar forma al resumen."
allowed_tools: [Read, Write]
---

# exec-summary

## Procedimiento

1. Enumera `./findings/*/finding.yml`.
2. Carga cada uno y extrae: `title`, `severity`, `base_score`, `target.asset_class`,
   `remediation.short`.
3. Agrupa por severity. Cuenta cada cubo.
4. Redacta tres párrafos, en este orden fijo:

   Párrafo 1 — Contexto.
   Enuncia el nombre del engagement, la ventana de la evaluación y el scope en
   una o dos frases. Menciona el número de activos probados y el
   número de findings por severity. Este párrafo contiene solo hechos
   extraídos de los YAML y de la entrada `scope_notes`; nada de interpretación.

   Párrafo 2 — Riesgos principales.
   Nombra los tres findings principales por `base_score` (o por severity si los scores
   empatan). Para cada uno, enuncia en una frase qué podría hacer un atacante.
   No nombres la CVE. No nombres la tool. Usa lenguaje de negocio.

   Párrafo 3 — Acciones recomendadas.
   Agrupa los findings por tema de remediación (parcheo, endurecimiento de configuración,
   revisión de control de acceso, monitoreo). Recomienda los tres temas principales
   en orden de prioridad. El esfuerzo estimado no es trabajo de la skill: no
   incluyas estimaciones de tiempo.

5. Escribe en `./build/exec-summary.md`.

6. Marca el archivo con una cabecera de front-matter:

   ```yaml
   ---
   drafted_by: exec-summary skill
   drafted_at: <marca de tiempo ISO>
   review_status: pending
   ---
   ```

   El campo `review_status` pasa a `approved` solo después de que un humano
   edite y vuelva a guardar.

## Rechazos

- Si hay cero findings, no redactes un resumen que diga "no
  issues were identified". Rechaza y pregúntale al humano qué escribir.
- Si algún finding tiene `cvss.status != approved`, advierte pero continúa con
  una nota en la cabecera.
````

### 7.2 Por qué tres párrafos

Tres párrafos es un estándar de casa de Red Team Leaders. Cada resumen que entrega la firma, en cada engagement, en cada industria, son tres párrafos en el mismo orden. Los clientes aprenden a hojearlos de la misma manera. Los auditores aprenden a encontrar los hechos en el mismo sitio. El tester no decide la estructura: la decide la plantilla.

## 8. Skill 7 — report-assemble

La skill report-assemble es la orquestadora. Lee las tarjetas, el executive summary, la sección de metodología y los apéndices, y produce un solo archivo Markdown listo para renderizar.

### 8.1 El esqueleto del report

```text
# <Engagement Name> — Penetration Test Report
| Prepared by Red Team Leaders | Confidential

## 1. Executive Summary          ← salida de exec-summary
## 2. Scope and Rules of Engagement  ← desde engagement.yml
## 3. Methodology                ← desde templates/methodology.md
## 4. Findings                   ← cada finding-card, ordenada por severity
## 5. Recommendations Summary    ← generado a partir de finding.remediation
## 6. Appendix A — Tools and Versions  ← desde tool-inventory.yml
## 7. Appendix B — Evidence Manifest    ← lista plana de cada manifest
## 8. Appendix C — Retest Results      ← desde los bloques retest de los YAML
```

### 8.2 SKILL.md completo

````markdown
---
name: report-assemble
description: |
  Ensambla cada tarjeta de finding, el executive summary y los apéndices
  estándar en un solo archivo Markdown build/report.md, listo para el
  renderizado Typst.
inputs:
  engagement:
    type: string
    required: true
  out:
    type: string
    required: false
    default: "./build/report.md"
allowed_tools: [Read, Write, Bash]
---

# report-assemble

## Procedimiento

1. Lee `./engagement.yml` para obtener el nombre del engagement, las fechas, el scope,
   el contacto del cliente y las rules of engagement.
2. Lee `./build/exec-summary.md`. Si `review_status != approved`, advierte
   en voz alta pero continúa.
3. Enumera `./build/cards/*.md`. Ordena por severity (`critical` → `info`),
   y luego por ID dentro de cada cubo de severity.
4. Lee `./templates/methodology.md`. Este archivo se edita una vez al año
   y se copia al pie de la letra.
5. Lee `./tool-inventory.yml`. Renderiza el Apéndice A como una tabla.
6. Recorre `./findings/*/evidence/manifest.yml`. Renderiza el Apéndice B como una
   fila por archivo de evidencia: slug, nombre de archivo, sha256, origen.
7. Recorre los bloques `retest`. Renderiza el Apéndice C. Si cada `retest.status` es
   `not-yet-retested`, omite el apéndice por completo.
8. Emite el report en el orden de secciones hacia la ruta de salida.
9. Imprime la tabla de contenidos.

## No-objetivos

- No modifiques ninguna tarjeta. Las tarjetas ya están renderizadas.
- No modifiques el executive summary. Si necesita ediciones, el humano
  edita la fuente y esta skill se vuelve a ejecutar.
````

### 8.3 El principio de un solo archivo

El report es un archivo Markdown. No un directorio de archivos que se pegan en el momento del PDF: un archivo. Es una regla dura porque significa que el humano puede, en cualquier momento, hacer `less build/report.md` y ver exactamente lo que verá el cliente. No hay una capa invisible de composición en el momento del render. Lo que ves en el Markdown es lo que se entrega.

## 9. Skill 8 — typst-render

Typst es un sistema moderno de composición tipográfica que produce PDFs a partir de un lenguaje fuente simple, parecido a LaTeX. Red Team Leaders usa Typst en lugar de Word o LaTeX porque las plantillas se pueden diferenciar en git, la compilación es determinista y rápida, y la salida es tipografía de grado de producción. La skill typst-render ejecuta `typst compile` contra la plantilla incluida de Red Team Leaders.

### 9.1 La plantilla incluida

La plantilla vive en `templates/rtl-report.typ` y, en esquema, se ve así:

```typst
#import "components/cover.typ": cover
#import "components/section.typ": section
#import "components/finding.typ": finding-block

#let engagement = yaml("../engagement.yml")

#cover(
  client: engagement.client,
  engagement: engagement.name,
  window: engagement.window,
  version: engagement.version,
)

#include "../build/report.md"
```

El `#include` es donde entra el Markdown ensamblado. Typst renderiza Markdown de forma nativa mediante sus funciones `read` y `raw`, más un adaptador pequeño de Markdown a Typst que se entrega con la plantilla.

### 9.2 SKILL.md completo

````markdown
---
name: typst-render
description: |
  Compila el Markdown del report ensamblado en un PDF listo para el cliente
  usando la plantilla Typst de Red Team Leaders.
inputs:
  engagement:
    type: string
    required: true
  out:
    type: string
    required: false
    default: "./build/report.pdf"
allowed_tools: [Read, Bash]
---

# typst-render

## Procedimiento

1. Verifica que `typst` está en el PATH. Si no, imprime el comando de instalación y aborta.
2. Verifica que existe `./build/report.md`. Si no, dile al usuario que ejecute
   primero `report-assemble`.
3. Ejecuta:

   ```bash
   typst compile templates/rtl-report.typ <out>
   ```

4. Si la compilación falla, imprime las últimas 40 líneas de stderr y aborta.
5. Si tiene éxito, imprime:

   ```text
   PDF written to <out> (<N> pages, <M> KB).
   ```

6. De forma opcional, ejecuta `pdfinfo <out>` e imprime los metadatos como
   comprobación de cordura.

## No-objetivos

- No modifiques la plantilla Typst. La plantilla es un artefacto de build
  bajo control de cambios; los cambios pasan por un pull request.
- No abras el PDF. Eso es una acción humana.
````

### 9.3 Por qué Typst

Typst se elige porque la compilación es reproducible byte a byte dadas las mismas entradas, la plantilla es un archivo pequeño y legible que puede mantener alguien que no es diseñador, y la salida está compuesta con criterio profesional. LaTeX funcionaría, pero tiene un coste de mantenimiento mucho más alto. Word funcionaría, pero no se puede diferenciar y no es reproducible. Un Markdown-a-PDF impulsado por Pandoc funcionaría, pero produce documentos que parecen salida de Pandoc, lo cual delata que el report se generó de forma automática.

## 10. Contenido determinista frente a contenido redactado por el LLM

Cada skill del pipeline tiene un límite entre lo que redacta el LLM y lo que produce un cálculo determinista. Contrasta tu catálogo de skills con esta tabla antes de entregar:

| Contenido | Autor | Motivo |
|-----------|-------|--------|
| Título, descripción e impacto del finding | Humano | Prosa de cara al cliente; juicio profesional del tester |
| Remediación del finding | Humano | Exige conocimiento del stack tecnológico del cliente |
| Estructura del YAML del finding | Skill (finding-create) | Aplicación mecánica del esquema |
| Slug e ID | Skill (determinista) | Unicidad y cumplimiento del formato |
| Vector CVSS | LLM (cvss-score) | Leer inglés y mapearlo a métricas |
| Base score CVSS | Determinista (cvss4-score) | Aritmética: los LLM se equivocan en la aritmética |
| Metadatos de CVE | Fuente externa (NVD) | Verdad de base, no creencia del modelo |
| Mapeos OWASP / WSTG / ATT&CK | Emparejamiento determinista de tabla | Tablas congeladas de verdad de base |
| Diseño de archivos de evidencia | Skill (evidence-collect) | Operaciones mecánicas de archivos |
| Hashes de evidencia | Determinista (SHA-256) | Cadena de custodia |
| Markdown de la tarjeta de finding | Determinista (plantilla) | Nada de variación creativa entre engagements |
| Párrafos del executive summary | LLM (exec-summary) | Síntesis de muchos YAML en prosa |
| Ensamblado del esqueleto del report | Determinista (report-assemble) | Orden fijo de secciones |
| Maquetación del PDF | Determinista (plantilla Typst) | Tipografía reproducible |

El patrón es este: los LLM redactan prosa e interpretan inglés; el código determinista hace cumplir la estructura, calcula números y produce artefactos reproducibles. Siempre que te tiente dejar que el LLM decida un número, un nombre de archivo o un campo del esquema, da un paso atrás y escribe la pieza determinista. No es porque el LLM sea torpe: es porque el cliente necesita un report en el que pueda confiar dentro de seis meses, y la reproducibilidad es lo que hace posible la confianza.

## 11. Compuertas de revisión humana

Cada párrafo redactado por el LLM en el pipeline tiene una compuerta de revisión. La compuerta se hace cumplir con un campo `review_status` en el artefacto:

```yaml
review_status: pending    # el LLM redactó; el humano todavía no ha revisado
review_status: approved   # el humano ha revisado y ha firmado
review_status: rejected   # el humano rechazó; el artefacto debe volver a redactarse
```

Las skills que consumen un artefacto con compuerta se comportan de forma distinta según el estado:

- finding-card se niega a renderizar si `cvss.status` no es `approved`.
- report-assemble advierte pero continúa si `exec-summary.review_status` no es `approved`.
- typst-render imprime un banner grande en el pie del PDF de cualquier build de vista previa en el que alguna compuerta no esté `approved`. El banner dice:

```text
DRAFT — CONTAINS UNREVIEWED LLM-DRAFTED CONTENT. NOT FOR CLIENT DELIVERY.
```

El banner solo se puede quitar volviendo a ejecutar `typst-render --final`, que a su vez rechaza a menos que cada compuerta esté `approved`.

### 11.1 El flujo de firma

El flujo de firma es un comando de shell de una línea:

```bash
rtl approve exec-summary
rtl approve finding <slug> cvss
```

Esto voltea el campo de estado, registra al aprobador (desde `git config user.email`) y la marca de tiempo, y escribe una entrada de auditoría en `./audit.log`. El log de auditoría se hace commit en el repositorio del engagement junto con el report y se conserva como evidencia de que un humano cualificado revisó cada párrafo redactado por el LLM.

### 11.2 Por qué esto importa

La compuerta no es una medida de paranoia. Es una medida legal y de responsabilidad profesional. Si el LLM redacta un executive summary que dice «no se identificaron findings critical» y el tester lo entrega sin leerlo —porque confía en el pipeline— y resulta que se pasó por alto un finding critical, el responsable es el pentester, no el LLM. La compuerta hace imposible saltarse el paso de revisión.

## 12. Cumplimiento de la guía de estilo

Red Team Leaders hace cumplir un estilo de casa. El estilo tiene tres reglas:

- Nada de lenguaje evasivo. «Might be vulnerable», «could potentially allow», «appears to be exposed»: estas frases están prohibidas. El tester confirmó la vulnerabilidad o no la confirmó. Si está confirmada, enúnciala. Si no está confirmada, quita el finding.
- Nada de primera persona. «I found» pasa a ser «the assessment identified». «We tested» pasa a ser «testing was performed». El report se escribe desde la perspectiva de la evaluación, no desde la del tester.
- Nada de relleno de marketing. «State-of-the-art», «best-in-class», «leveraging synergy»: prohibidos de plano.

La skill style-lint hace cumplir esto:

````markdown
---
name: style-lint
description: |
  Pasa el lint del report ensamblado contra la guía de estilo de casa de
  Red Team Leaders. Marca lenguaje evasivo, voz en primera persona y frases
  de marketing prohibidas. Se niega a emitir un PDF final mientras existan
  violaciones.
inputs:
  path:
    type: string
    required: false
    default: "./build/report.md"
allowed_tools: [Read, Bash]
---

# style-lint

## Procedimiento

1. Lee el archivo objetivo.

2. Busca patrones prohibidos usando el `style/forbidden.yml` incluido:

   ```yaml
   hedging:
     - "might be"
     - "may be vulnerable"
     - "could potentially"
     - "appears to"
     - "seems to"
     - "possibly"
   first_person:
     - "\\bI \\b"
     - "\\bwe \\b"
     - "\\bour \\b"
     - "\\bmy \\b"
   marketing:
     - "state-of-the-art"
     - "best-in-class"
     - "leveraging"
     - "synergy"
     - "cutting-edge"
     - "world-class"
   ```

3. Por cada acierto, imprime el número de línea, la frase coincidente y una
   reescritura sugerida.

4. Sal con código distinto de cero si se encontró algún acierto. `typst-render --final` se condiciona a
   este código de salida.
````

El lint es estricto a propósito: de vez en cuando marcará un uso legítimo de «we» (por ejemplo, en una descripción de CVE citada). El tester lidia con los falsos positivos reescribiendo o añadiendo un marcador `<!-- style-lint: allow-line -->`.

## 13. Workflow completo — de YAML a PDF en una sesión

Así se ve el pipeline de punta a punta desde la silla del tester, usando seis de las skills en una sesión de Claude Code. El tester acaba de terminar un engagement de dos semanas y tiene notas en bruto, capturas y exports de Burp en `~/engagement-notes/`.

### 13.1 La sesión

```text
Usuario: Terminé el engagement de ACME Corp. Tengo doce findings en
      ~/engagement-notes/. Armemos el report.

Claude: Claro. Empecemos importando cada finding. Pega el primero
        o indícame un archivo que liste los títulos y las descripciones.

Usuario: (pega el finding 1 — un RCE critical de Apache Struts)

Claude: (invoca finding-create)
  → Created RTL-2026-0042 at findings/apache-struts-rce-on-erp/finding.yml
  → Pending evidence recorded from ~/engagement-notes/struts-*.zip

Usuario: (pega los findings 2, 3, ..., 12)

Claude: (invoca finding-create once veces más)
  → Created RTL-2026-0043 through RTL-2026-0053.

Usuario: Puntúalos todos.

Claude: (invoca cvss-score para cada slug)
  → 12 findings scored. All marked pending-review.
  → Summary: 2 critical, 4 high, 5 medium, 1 low.

Usuario: Enriquécelos.

Claude: (invoca finding-enrich para cada slug)
  → CVE metadata attached where available (3 CVEs resolved).
  → OWASP Top 10 categories attached to all findings.
  → WSTG test IDs attached to 10 findings.
  → MITRE ATT&CK techniques attached to all findings.

Usuario: Organiza la evidencia.

Claude: (invoca evidence-collect para cada slug)
  → Evidence for 12 findings organized. Manifests written.

Usuario: Ahora voy a revisar los vectores CVSS.

(El usuario abre cada finding.yml, revisa el vector y el fundamento, edita
donde hace falta y ejecuta `rtl approve finding <slug> cvss` para cada uno.)

Usuario: Aprobaciones hechas. Redacta las tarjetas.

Claude: (invoca finding-card para cada slug)
  → 12 cards written to build/cards/.

Usuario: Redacta el executive summary.

Claude: (invoca exec-summary)
  → build/exec-summary.md written. review_status: pending.

(El usuario lee el resumen, edita el párrafo dos para reflejar el contexto
de la industria del cliente y ejecuta `rtl approve exec-summary`.)

Usuario: Ensambla el report.

Claude: (invoca report-assemble)
  → build/report.md written. 87 sections, 6 appendices.

Usuario: Pásale el lint.

Claude: (invoca style-lint)
  → 3 hedging violations found in finding cards. Fix and re-run.

(El usuario edita tres descripciones YAML para quitar "may be" y "appears to",
vuelve a ejecutar finding-card para esos slugs, vuelve a ejecutar report-assemble,
vuelve a ejecutar style-lint. Limpio.)

Usuario: Renderiza el final.

Claude: (invoca typst-render --final)
  → build/report.pdf written. 64 pages, 2.1 MB.
  → No draft banner emitted (all gates approved).

Usuario: Envíalo.
```

Toda esa sesión, de punta a punta, es más o menos una hora facturable del tiempo del tester: la mayor parte se gasta revisando borradores del LLM y aprobando compuertas, no escribiendo prosa. El trabajo de escritura de prosa que antes consumía cinco días se comprime en esa hora de revisión.

### 13.2 Lo que no ocurrió

Fíjate en lo que el pipeline no hizo:

- No inventó números de CVE.
- No calculó scores CVSS en la cabeza del LLM.
- No decidió por su cuenta si el executive summary era lo bastante bueno para enviar.
- No se saltó una compuerta de revisión humana.
- No se desvió de la plantilla de Red Team Leaders.

Cada uno de esos puntos es una tentación. Cada uno de esos puntos es exactamente lo que vuelve poco fiables los reports asistidos por IA cuando el límite se traza mal. El pipeline vale la pena construirlo precisamente porque obliga a trazar bien el límite.

## 14. Ejercicios

### 14.1 Ejercicios guiados

**Ejercicio 1 — Construye finding-create y haz un dry-run.**

Copia el `SKILL.md` de arriba en `~/.claude/skills/finding-create/SKILL.md`. En un directorio de borrador, ejecuta:

```text
Usuario: log a new finding — reflected XSS in the search parameter of
      https://juice-shop.local/#/search, discovered via Burp scanner.
```

Verifica que el `findings/reflected-xss-in-the-search-parameter/finding.yml` resultante contiene:

- Un ID de la forma `RTL-2026-0001`.
- Bloques `cvss`, `references` y `evidence` vacíos.
- Un `discovered.date` que coincida con hoy.
- Un `target.url` no vacío igual a la URL que proporcionaste.

Ahora prueba el mismo comando una segunda vez. La skill debe rechazar en lugar de sobrescribir.

**Ejercicio 2 — Puntúa un finding y verifica el límite determinista.**

Usando el finding del Ejercicio 1, pídele a Claude que invoque cvss-score. Lee los campos resultantes `cvss.vector` y `cvss.base_score`. Ahora edita `cvss.vector` a mano para cambiar `AV:N` por `AV:L`, guarda el archivo y vuelve a invocar cvss-score con el flag `--recompute-only` (que tendrás que añadir a la skill).

- Confirma que el `base_score` cambia en consecuencia.
- Confirma que el fundamento se regenera para reflejar el nuevo `AV`.
- Confirma que un vector completamente inválido (borra el trozo `AC:L`) hace que la skill rechace en lugar de emitir un score aleatorio.

**Ejercicio 3 — Pipeline completo en un engagement de dos findings.**

Elige dos vulnerabilidades de un entorno de laboratorio que ya tengas —DVWA, Juice Shop, máquinas retiradas de HackTheBox, cualquier cosa para la que tengas permiso escrito de prueba. Toma capturas reales, captura requests reales. Luego recorre el pipeline entero:

- finding-create para cada finding.
- cvss-score para cada uno.
- finding-enrich para cada uno.
- evidence-collect para cada uno.
- Aprueba las compuertas.
- finding-card para cada uno.
- exec-summary.
- Aprueba la compuerta de exec-summary.
- report-assemble.
- style-lint.
- `typst-render --final`.

Produce el PDF. Léelo de principio a fin. Anota cada lugar en el que la salida no se siente lista para el cliente e identifica si la corrección pertenece a una skill, a una plantilla o al paso de revisión humana.

### 14.2 Ejercicios independientes

**Ejercicio 4 — Diseña y construye la skill evidence-redact.**

Cada engagement real produce evidencia que contiene datos que el tester no debería enviar al cliente: hashes de contraseñas capturados a mitad de un exploit, PII expuesta en mensajes de error, tokens de sesión en cabeceras de request. Diseña una skill que escanee archivos de evidencia en busca de patrones bien conocidos (cookies de sesión, JWT, hashes codificados en hex, direcciones de correo, números de documento de identidad de tu locale) y produzca copias redactadas junto a los originales, actualizando el manifiesto para que referencie las versiones redactadas. Decide qué patrones están en el scope. Decide cómo se manejan los falsos positivos (una redacción que quita algo que el tester necesitaba conservar). Decide cómo aprueba el humano cada redacción antes de que la copia redactada se use en una tarjeta.

**Ejercicio 5 — Diseña el workflow de retest.**

Los engagements reales a menudo incluyen un retest: el tester vuelve de 30 a 90 días después, prueba los mismos ataques contra el sistema corregido e informa qué se corrigió de verdad. Extiende el esquema, añade una skill retest-run que recorra los findings, le pida al humano una respuesta por cada uno y actualice el bloque `retest`. Extiende report-assemble para renderizar el Apéndice C. Extiende typst-render para añadir una sección «Retest Summary» en la portada si hay algún dato de retest.

**Ejercicio 6 — Lint de estilo multilingüe.**

Red Team Leaders entrega reports en inglés, portugués y español. Extiende la skill style-lint para manejar los tres idiomas con tablas de patrones prohibidos por idioma. Decide cómo se detecta el idioma (un campo `language` en `engagement.yml`, o un sniffing del contenido). Decide cómo se manejan los reports de idioma mixto (un cliente que quiere el executive summary en portugués pero los findings en inglés).

## 15. Resumen y lección siguiente

Ahora tienes el diseño de un pipeline de generación de reports de ocho skills que lee YAML, enriquece con inteligencia externa, aplica plantillas deterministas, redacta prosa solo donde la prosa de verdad ayuda, pone cada salida del LLM detrás de una revisión humana y produce un PDF listo para el cliente vía Typst. Has visto dónde corresponde el código determinista, dónde corresponde la redacción del LLM y por qué el límite no es negociable si el report tiene que sostenerse en una auditoría del cliente seis meses después de la entrega.

El pipeline que construiste en esta lección opera sobre findings que ya existen. La Lección 06 se desplaza al otro extremo del workflow: skills que te ayudan a encontrar los findings en primer lugar. Construirás skills recon-*, web-triage y smb-triage que convierten información cruda del objetivo en pistas estructuradas para la investigación manual. Esas skills alimentan el pipeline de reporting que construiste aquí: las dos lecciones se componen en un workflow de evaluación completo.

Continúa en Lesson 06 — Building Reconnaissance Skills.
