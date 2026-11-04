# Lección 08 — Workflow Automation with AI Agents

Módulo 06: Construcción de skills de pentest con Claude y Codex · Penetration Testing con Kali Linux 1.0 · Red Team Leaders

## Introducción

Un solo agent, por capaz que sea, tiene techos duros. Arrastra una sola ventana de contexto, ejecuta las tools en serie, olvida lo que consideró irrelevante y —cuando una tarea crece más allá de unas pocas docenas de llamadas a tools— o se quema el budget en ramas que no producen nada, o se aleja del objetivo. En un pentest ese techo aparece rápido: el recon de un scope modesto implica enumerar decenas de subdominios, hacer fingerprint de cientos de servicios, spider de miles de URLs y hacer diff del resultado contra el snapshot de la semana pasada. Meter todo eso en un solo turno de un agent solo es posible con un objetivo de juguete.

La automatización de workflows es la respuesta. En lugar de un agent que hace todo, componemos agents especializados en un pipeline: cada agent tiene un trabajo estrecho, un prompt fijo, una allow-list de tools ajustada y una forma de salida predecible. Un script de workflow —la capa de control externa— decide quién corre y cuándo, qué estado pasa entre ellos, qué pasos corren en paralelo y dónde el pipeline une los resultados. El workflow es determinista donde el determinismo importa (fan-out, timeouts, reintentos, orden) y no determinista donde importa la creatividad (el razonamiento del agent dentro de una fase).

Esta lección te enseña a orquestar workflows de pentest multi-agent con la tool Workflow de Claude Code. Vas a construir un fan-out de recon que lanza cuatro enumeradores en paralelo, un workflow de review con un verificador adversarial, un router de triage que despacha findings a especialistas de web, cloud y cripto, y un script completo de unas 200 líneas, web-recon-triage, que escanea un dominio de punta a punta. Por el camino cubrimos budgets de costo, checkpointing, observabilidad, enrutamiento multi-proveedor y los anti-patrones que convierten un workflow en algo desbocado.

Al terminar vas a poder mirar un engagement de pentest, decidir si un workflow se justifica, elegir la topología correcta, escribir el script y hacer cumplir el scope ético dentro de cada fase.

## 1. Introducción — Por qué un solo agent no alcanza

### 1.1 El techo de un solo agent

Un agent de Claude Code tiene tres límites prácticos durante un pentest:

1. **Ventana de contexto.** Incluso una ventana de 200K tokens se llena rápido cuando entran listas de subdominios, XML de nmap y bundles de JavaScript. Cuando la ventana se compacta, la evidencia previa se vuelve borrosa.
2. **Ejecución serial de tools.** Un agent que necesita correr subfinder, httpx, nuclei y katana los corre uno detrás de otro. Cuatro escaneos de treinta segundos se convierten en dos minutos de tiempo de reloj antes de que empiece cualquier razonamiento.
3. **Deriva de la tarea.** Un prompt que pide "full recon on example.com" invita al agent a inventar scope. Puede hacer spider del sitio de marketing, adivinar paneles de admin o empezar a pedir subdominios que nunca autorizaste.

Un workflow levanta cada techo al sacar el control de la cabeza del agent y meterlo en un script que tú escribes, revisas y versionas.

### 1.2 La forma de un workflow

Un workflow es un archivo JavaScript que ejecuta la tool Workflow. Define fases, lanza agents, une sus salidas, aplica lógica determinista (filtros, ordenamientos, validación JSON) y solo vuelve a la sesión padre de Claude cuando el pipeline termina o falla.

```text
┌────────────────────────────────────────────────────────────┐
│ workflow: web-recon-triage.js                              │
│                                                            │
│   phase("scope")     → validar objetivo, cargar RoE       │
│   parallel([                                                │
│     agent("subs",   "enumerar subdominios"),               │
│     agent("ports",  "escanear puertos abiertos"),          │
│     agent("tech",   "fingerprint de stacks"),              │
│     agent("js",     "extraer endpoints del JS")            │
│   ])                                                        │
│   phase("aggregate") → fusionar, deduplicar, estructurar   │
│   agent("triage",   "ordenar findings por explotabilidad") │
│   phase("gate")     → aprobación humana antes del testing  │
│                         activo                             │
└────────────────────────────────────────────────────────────┘
```

La sesión padre de Claude solo ve el resultado final del workflow: una lista ordenada de findings y una decisión. Toda la charla intermedia de los agents se queda en el log del workflow.

### 1.3 Lo que te compra un workflow

| Propiedad | Un solo agent | Workflow |
|---|---|---|
| Paralelismo | Ninguno (tools en serie) | Fan-out de N vías |
| Reuso de contexto | Una ventana compartida | Aislado por agent |
| Determinismo | Depende del prompt | Flujo de control escrito en el script |
| Visibilidad de costo | Total al final de la corrida | Por fase, por agent |
| Reanudar tras un fallo | Reiniciar desde cero | Con checkpoint |
| Cumplimiento del scope | Esperanza puesta en el prompt | Compuerta en el script |

El precio es la complejidad. Un workflow es código, con toda la depuración que eso implica. Úsalo cuando la forma de la tarea pague el esfuerzo.

## 2. Topologías de workflow

Cuatro topologías cubren casi todos los casos de uso de un pentest. Aprende a reconocer cuál quiere una tarea antes de escribir una línea de script.

### 2.1 Pipeline secuencial

Cada fase depende de la anterior. La salida de A alimenta a B, que alimenta a C.

```text
[recon] → [filtro de activos] → [escaneo de vulns] → [triage] → [reporte]
```

Úsalo cuando cada etapa necesita el resultado de la etapa anterior y el tiempo de reloj total es aceptable. El ejemplo clásico: la salida de nmap alimenta el escaneo de vulnerabilidades, que alimenta la selección de exploits.

### 2.2 Fan-out paralelo

Una entrada, muchos workers independientes, una salida unida.

```text
                    ┌─ [subdominios] ─┐
[dominio objetivo] ─├─ [puertos] ─────┤──→ [agregador]
                    ├─ [tech-fp] ─────┤
                    └─ [js-endpoints] ┘
```

Úsalo cuando las subtareas no comparten estado y pueden correr a la vez. El recon es el encaje canónico: ninguno de los enumeradores necesita saber qué encontraron los otros.

### 2.3 Supervisor jerárquico

Un agent supervisor descompone el trabajo, lo delega a especialistas y después lo reintegra.

```text
              [supervisor]
             /    |    |    \
      [web]  [api]  [mobile] [cloud]
             \    |    |    /
              [supervisor]
```

Úsalo cuando la descomposición misma es una tarea de razonamiento: no sabes de antemano qué especialistas lanzar. El triage encaja: el supervisor lee un finding y elige al experto correcto.

### 2.4 Map-reduce

Parte una entrada grande en trozos, corre el mismo agent sobre cada trozo y después reduce los resultados.

```text
   [10.000 URLs] ─split─→ [trozo 1] ─┐
                          [trozo 2] ─┼─ [reduce → dedup + rank]
                          [trozo 3] ─┤
                          [trozo N] ─┘
```

Úsalo cuando la entrada es homogénea y grande. Clasificar 10.000 URLs por superficie de ataque probable es un map-reduce; enumerar cuatro fuentes de datos distintas es un fan-out.

### 2.5 Hoja de trucos de topologías

| Tarea | Topología |
|---|---|
| Recon de un solo dominio | Fan-out paralelo |
| Campaña de pentest de punta a punta | Pipeline secuencial |
| Enrutar una bolsa mixta de findings | Supervisor jerárquico |
| Clasificar una lista gigante de URLs | Map-reduce |
| Revisar un PR por bugs, estilo y tests | Fan-out paralelo y después agregador |
| Escaneo multi-objetivo sobre una lista de scope | Map-reduce a través de los objetivos |

## 3. Cuándo preferir un workflow frente a un solo agent

No toda tarea de pentest se beneficia de un workflow. El costo de escribir, probar y revisar un script tiene que ganarse.

### 3.1 El filtro de tres preguntas

Pregunta, en orden:

1. ¿El scope está acotado y es enumerable? Si no puedes listar las fases de antemano, no estás listo para un workflow. Escribe primero un plan.
2. ¿Un flujo de control determinista reduciría los modos de fallo? Si un solo agent tiende a saltarse pasos, agotar su budget en una rama u olvidar el scope, una lista de fases escrita en el script corrige cada uno de esos fallos.
3. ¿Hay un budget real de paralelismo? Si las tareas son inherentemente seriales o están limitadas por una sola API con rate limit, el fan-out no gana nada.

Dos respuestas afirmativas es el umbral. Tres es una señal fuerte.

### 3.2 Cuándo quedarte con un solo agent

- **Exploración.** No sabes qué estás buscando. Deja que un agent husmee.
- **Depurar un error concreto.** El contexto es chico y el loop es cerrado.
- **Recon de una sola vez.** Una cadena `subfinder | httpx | nuclei` es un pipeline de bash, no un workflow.
- **Cualquier cosa de menos de dos minutos de tiempo de reloj.** El overhead del workflow no vale la pena.

### 3.3 Cuándo ir a un workflow

- **Campañas repetibles.** La misma forma cada semana: diff de subdominios, escaneo de vulns, triage, reporte.
- **Agregación de varias fuentes.** Cuatro APIs, cuatro agents, una vista unida.
- **Review adversarial.** Agent productor más agent verificador, los dos escritos en el script.
- **Escaneos de larga duración.** Quieres checkpoints y reanudación.
- **Scope multi-objetivo.** Los mismos pasos aplicados a una lista de objetivos.
- **Compuertas con humano en el loop.** Quieres pausar para pedir aprobación en puntos definidos.

## 4. Panorama de la tool Workflow de Claude Code

La tool Workflow ejecuta un archivo JavaScript que describe fases, agents y uniones. Tú escribes el script; la tool lo corre.

### 4.1 La API del script

Cuatro helpers cubren la mayoría de los workflows:

- `pipeline(...phases)` — declara una secuencia ordenada.
- `parallel(...agents)` — declara un bloque de fan-out que corre sus argumentos de forma concurrente.
- `agent(name, prompt, options)` — lanza un subagent con un nombre, un prompt y opciones (tools, model, budget, output_schema).
- `phase(name, fn)` — un paso escrito en el script que corre tu JavaScript, no un agent. Úsalo para filtrar, unir y validar.

### 4.2 Un script mínimo

```javascript
// hello-workflow.js
export default pipeline(
  phase("prepare", (ctx) => {
    ctx.target = ctx.input.domain;
    return ctx;
  }),
  agent("subs",
    (ctx) => `Enumera los subdominios de ${ctx.target}. Devuelve JSON.`,
    { tools: ["Bash"], budget: { max_tokens: 20_000 } }
  ),
  phase("summary", (ctx) => {
    return { subdomains: ctx.subs.output };
  })
);
```

La tool carga este archivo, llama al pipeline exportado con la entrada del padre y devuelve el valor de retorno de la última fase.

### 4.3 Paso de contexto

Cada fase y cada agent recibe un objeto `ctx`. Lo que devuelve un agent queda colgado en `ctx.<name>`. Una fase que muta `ctx` lo muta para los pasos posteriores. Una fase que devuelve un objeto nuevo reemplaza el contexto.

### 4.4 Opciones de agent que vas a usar

| Opción | Para qué sirve |
|---|---|
| `tools` | Allow-list de tools que este agent puede llamar |
| `model` | Sobrescribe el modelo (opus, sonnet, haiku, ollama:llama3) |
| `budget.max_tokens` | Techo duro de tokens para este agent |
| `budget.max_iterations` | Tope de turnos de uso de tools |
| `output_schema` | JSON schema que el agent debe cumplir |
| `timeout_ms` | Timeout de tiempo de reloj |
| `retry` | `{ max: 2, backoff: "exp" }` |
| `scope` | Objeto de scope heredado (ver la sección 15) |

### 4.5 Invocar un workflow desde Claude Code

Desde una sesión de Claude Code:

```text
Workflow(
  file_path: "/opt/pentest/workflows/web-recon-triage.js",
  input: { domain: "example.com", scope_file: "/opt/pentest/scope.json" }
)
```

La sesión padre se pausa hasta que el workflow regresa. El valor de retorno es un objeto JSON.

## 5. Escribir un primer workflow — recon-fan-out

Aquí tienes un workflow completo y real que lanza cuatro agents de recon en paralelo y une sus salidas.

### 5.1 El script

```javascript
// recon-fan-out.js
// Lanza cuatro agents de recon contra un solo dominio y une sus salidas JSON.

const RECON_TOOLS = ["Bash", "Read", "Write", "Grep"];

const subdomainAgent = (ctx) => agent(
  "subdomains",
  `Eres un agent de enumeración de subdominios. Objetivo: ${ctx.target}.
   Usa solo subfinder y amass en modo pasivo.
   No envíes consultas DNS activas más allá de los servidores autoritativos del objetivo.
   Devuelve JSON: { "subdomains": [ { "name": "...", "source": "..." } ] }`,
  {
    tools: RECON_TOOLS,
    budget: { max_tokens: 30_000, max_iterations: 12 },
    timeout_ms: 180_000,
    output_schema: {
      type: "object",
      required: ["subdomains"],
      properties: {
        subdomains: {
          type: "array",
          items: {
            type: "object",
            required: ["name", "source"],
            properties: {
              name:   { type: "string" },
              source: { type: "string" }
            }
          }
        }
      }
    }
  }
);

const portAgent = (ctx) => agent(
  "ports",
  `Eres un agent de escaneo de puertos. Objetivo: ${ctx.target}.
   Corre: nmap -T3 -sS --top-ports 100 --open -oX -.
   Parsea el XML y devuelve JSON: { "hosts": [ { "ip": "...", "ports": [ { "port": N, "proto": "tcp", "service": "..." } ] } ] }`,
  {
    tools: RECON_TOOLS,
    budget: { max_tokens: 40_000, max_iterations: 15 },
    timeout_ms: 300_000,
    output_schema: {
      type: "object",
      required: ["hosts"],
      properties: {
        hosts: { type: "array" }
      }
    }
  }
);

const techAgent = (ctx) => agent(
  "tech",
  `Eres un agent de fingerprint de tecnología. Objetivo: ${ctx.target}.
   Usa httpx -tech-detect contra el dominio raíz y los 5 subdominios principales de ctx.subdomains.
   Devuelve JSON: { "stacks": [ { "url": "...", "technologies": [ "..." ] } ] }`,
  {
    tools: RECON_TOOLS,
    budget: { max_tokens: 30_000 },
    timeout_ms: 180_000
  }
);

const jsAgent = (ctx) => agent(
  "js",
  `Eres un extractor de endpoints de JavaScript. Objetivo: ${ctx.target}.
   Usa katana -js-crawl y linkfinder sobre los 5 subdominios principales.
   Devuelve JSON: { "endpoints": [ { "url": "...", "source_file": "..." } ] }`,
  {
    tools: RECON_TOOLS,
    budget: { max_tokens: 40_000 },
    timeout_ms: 240_000
  }
);

export default pipeline(
  phase("validate", (ctx) => {
    if (!ctx.input.target || !/^[a-z0-9.-]+$/i.test(ctx.input.target)) {
      throw new Error(`objetivo no válido: ${ctx.input.target}`);
    }
    ctx.target = ctx.input.target;
    return ctx;
  }),

  // Paso 1: primero los subdominios (tech y js dependen de ellos)
  subdomainAgent,

  // Paso 2: el resto puede abrirse en fan-out
  parallel(portAgent, techAgent, jsAgent),

  phase("join", (ctx) => ({
    target: ctx.target,
    subdomains: ctx.subdomains.subdomains,
    hosts:      ctx.ports.hosts,
    stacks:     ctx.tech.stacks,
    endpoints:  ctx.js.endpoints,
    generated_at: new Date().toISOString()
  }))
);
```

### 5.2 Lo que esto te compra

- La enumeración de subdominios corre primero porque los agents de tech y de JS necesitan su salida.
- Puertos, tech y JS corren de verdad en paralelo: tres tareas pesadas en tools se comprimen de lo serial (unos 12 minutos) a lo concurrente (unos 5 minutos).
- Cada agent tiene un techo de tokens, un tope de iteraciones y un timeout de tiempo de reloj. Nada se desboca.
- `output_schema` en el agent de subdominios hace que el código posterior pueda confiar en la forma.
- El scope se hace cumplir en el prompt (fuentes pasivas, top-5 de subdominios) y otra vez en la allow-list de tools.

### 5.3 Cómo correrlo

```text
Workflow(
  file_path: "/opt/pentest/workflows/recon-fan-out.js",
  input: { target: "lab.example.com" }
)
```

La sesión padre recibe un solo objeto JSON con las cuatro fuentes de datos fusionadas.

## 6. Construir un workflow pentest-review

Este workflow revisa un lote de cambios de código en busca de problemas de seguridad, limpieza y cobertura de tests, y después verifica de forma adversarial cada finding para reducir falsos positivos.

### 6.1 La forma

Para cada dimensión (bugs, limpieza, tests), un agent productor genera findings. Un agent verificador toma después cada finding e intenta refutarlo. Solo los findings que el verificador no puede refutar entran en el reporte final.

```text
              [diff]
                │
     ┌──────────┼──────────┐
     ↓          ↓          ↓
  [bugs]    [limpieza]   [tests]
     │          │          │
     └────┬─────┴─────┬────┘
          ↓           ↓
    por finding: [verificador] adversarial
                     │
                     ↓
              [reporte fusionado]
```

### 6.2 El script

```javascript
// pentest-review.js

const producer = (name, focus) => (ctx) => agent(
  name,
  `Eres un revisor de ${focus}. Lee el diff en ${ctx.diff_path}.
   Devuelve JSON: { "findings": [ { "id": "...", "file": "...", "line": N,
                                   "severity": "low|med|high", "claim": "..." } ] }`,
  {
    tools: ["Read", "Grep", "Bash"],
    budget: { max_tokens: 40_000 },
    output_schema: {
      type: "object",
      required: ["findings"],
      properties: { findings: { type: "array" } }
    }
  }
);

const verifier = (finding) => agent(
  `verify-${finding.id}`,
  `Eres un verificador adversarial. Un revisor afirma:
   FILE: ${finding.file}
   LINE: ${finding.line}
   CLAIM: ${finding.claim}
   Intenta refutar la afirmación. Lee el archivo, traza el flujo, propone un contraejemplo.
   Devuelve JSON: { "id": "${finding.id}", "verdict": "confirmed|refuted|uncertain",
                  "reasoning": "..." }`,
  {
    tools: ["Read", "Grep"],
    budget: { max_tokens: 20_000, max_iterations: 8 },
    output_schema: {
      type: "object",
      required: ["id", "verdict", "reasoning"]
    }
  }
);

export default pipeline(
  phase("prep", (ctx) => {
    ctx.diff_path = ctx.input.diff_path;
    if (!ctx.diff_path) throw new Error("se requiere diff_path");
    return ctx;
  }),

  parallel(
    producer("bugs",    "security-bug"),
    producer("cleanup", "code-cleanup"),
    producer("tests",   "test-coverage")
  ),

  phase("gather", (ctx) => {
    ctx.all_findings = [
      ...ctx.bugs.findings,
      ...ctx.cleanup.findings,
      ...ctx.tests.findings
    ];
    return ctx;
  }),

  // Fan-out: un verificador por finding
  phase("verify", async (ctx) => {
    const jobs = ctx.all_findings.map(f => verifier(f));
    const results = await Promise.all(jobs.map(j => j.run(ctx)));
    ctx.verdicts = results;
    return ctx;
  }),

  phase("report", (ctx) => {
    const confirmed = ctx.all_findings
      .map(f => ({ ...f, verdict: ctx.verdicts.find(v => v.id === f.id) }))
      .filter(f => f.verdict && f.verdict.verdict === "confirmed");
    return { confirmed, all: ctx.all_findings, verdicts: ctx.verdicts };
  })
);
```

### 6.3 Por qué ayuda la división adversarial

Los productores reportan de más. Todo revisor basado en un LLM marca cosas que una mirada de cinco segundos descartaría. Un verificador al que se le pide refutar —con solo Read y Grep, así que tiene que apoyar su argumento en el código— filtra ese ruido. En la práctica esto corta las tasas de falsos positivos más o menos a la mitad en diffs que no son triviales.

## 7. Triage multi-agent — Enrutar findings a especialistas

El recon y la revisión de código producen bolsas mixtas. El triage enruta cada finding al especialista correcto.

### 7.1 El patrón router

```javascript
// triage.js

const webSpec = (finding) => agent(
  `web-${finding.id}`,
  `Eres un experto en seguridad de aplicaciones web. Evalúa: ${JSON.stringify(finding)}.
   Devuelve JSON con explotabilidad, impacto y pasos de reproducción.`,
  { tools: ["Bash", "Read", "Write"], budget: { max_tokens: 30_000 } }
);

const cloudSpec = (finding) => agent(
  `cloud-${finding.id}`,
  `Eres un experto en seguridad cloud (AWS/GCP/Azure). Evalúa: ${JSON.stringify(finding)}.
   Devuelve JSON con explotabilidad específica de cloud y radio de impacto.`,
  { tools: ["Bash", "Read"], budget: { max_tokens: 30_000 } }
);

const cryptoSpec = (finding) => agent(
  `crypto-${finding.id}`,
  `Eres un experto en criptografía. Evalúa: ${JSON.stringify(finding)}.
   Devuelve JSON con la clase de debilidad del algoritmo y el impacto en el mundo real.`,
  { tools: ["Read"], budget: { max_tokens: 20_000 } }
);

const CATEGORY_MAP = {
  web:    webSpec,
  cloud:  cloudSpec,
  crypto: cryptoSpec
};

const classifier = (finding) => agent(
  `classify-${finding.id}`,
  `Clasifica este finding en una de: web, cloud, crypto, other.
   FINDING: ${JSON.stringify(finding)}.
   Devuelve JSON: { "category": "web|cloud|crypto|other" }`,
  { tools: [], model: "haiku", budget: { max_tokens: 2_000 } }
);

export default pipeline(
  phase("classify", async (ctx) => {
    const classifications = await Promise.all(
      ctx.input.findings.map(f => classifier(f).run(ctx))
    );
    ctx.classified = ctx.input.findings.map((f, i) => ({
      ...f, category: classifications[i].category
    }));
    return ctx;
  }),

  phase("dispatch", async (ctx) => {
    const dispatched = ctx.classified
      .filter(f => f.category !== "other")
      .map(f => CATEGORY_MAP[f.category](f).run(ctx));
    ctx.assessments = await Promise.all(dispatched);
    return ctx;
  }),

  phase("rank", (ctx) => {
    ctx.assessments.sort((a, b) =>
      severityScore(b.severity) - severityScore(a.severity)
    );
    return { assessments: ctx.assessments };
  })
);

function severityScore(s) {
  return { critical: 4, high: 3, med: 2, low: 1 }[s] || 0;
}
```

### 7.2 Por qué un clasificador barato primero

El clasificador usa Haiku (u Ollama) porque la tarea es chica y la forma de su salida es un solo campo. Mandar cada finding directo a Opus saldría de cuatro a cinco veces más caro. Enruta barato y trabaja caro solo en las rutas que importan.

## 8. Salidas estructuradas de los agents del workflow

El texto suelto entre fases garantiza problemas. Haz cumplir JSON schemas.

### 8.1 Adjuntar un schema

Cada agent acepta `output_schema`. La tool Workflow valida el mensaje final del agent contra el schema antes de colgarlo en `ctx`. Si falla, reintenta con el error inyectado en el prompt siguiente.

```javascript
agent(
  "cve-lookup",
  (ctx) => `Busca CVEs para ${ctx.tech.stacks[0].technologies.join(", ")}.
            Devuelve JSON estricto.`,
  {
    output_schema: {
      type: "object",
      required: ["cves"],
      properties: {
        cves: {
          type: "array",
          items: {
            type: "object",
            required: ["id", "cvss", "affected"],
            properties: {
              id:       { type: "string", pattern: "^CVE-\\d{4}-\\d+$" },
              cvss:     { type: "number", minimum: 0, maximum: 10 },
              affected: { type: "string" }
            }
          }
        }
      }
    },
    retry: { max: 2, backoff: "exp" }
  }
)
```

### 8.2 Agregar salida estructurada

Cuando la salida de cada agent es un objeto validado, la agregación es JavaScript plano:

```javascript
phase("aggregate", (ctx) => {
  const flat = ctx.results.flatMap(r => r.cves);
  const dedup = new Map(flat.map(c => [c.id, c]));
  return { cves: [...dedup.values()].sort((a, b) => b.cvss - a.cvss) };
});
```

### 8.3 Diseño de prompts con el schema primero

Una disciplina útil: escribe el schema antes de escribir el prompt. Entonces el prompt se vuelve: "produce datos que coincidan con este schema". El trabajo del agent se estrecha y la forma de su salida deja de sorprenderte.

## 9. Costo y budget de tokens

Los workflows pueden quemar tokens rápido. Presupuestálos de forma explícita.

### 9.1 Las tres palancas de budget

- `max_tokens` — tokens totales que el agent puede consumir a lo largo de todos los turnos.
- `max_iterations` — máximo de turnos de uso de tools.
- `model` — modelos más baratos para fases más baratas.

### 9.2 Una tabla de budgets

Para un workflow de recon típico, los siguientes budgets son conservadores:

| Agent | Modelo | max_tokens | max_iterations |
|---|---|---|---|
| classifier | haiku | 2.000 | 1 |
| subdomains | sonnet | 30.000 | 12 |
| ports | sonnet | 40.000 | 15 |
| tech | sonnet | 30.000 | 10 |
| js | sonnet | 40.000 | 15 |
| triage | opus | 20.000 | 5 |
| verifier | sonnet | 20.000 | 8 |

Total: unos 180K tokens por corrida. Llévalo la cuenta: pon un alert si una sola fase quema más del 150% de su budget en una semana.

### 9.3 Techos globales

Agrega un techo a nivel de workflow para atrapar desbordes:

```javascript
export default pipeline(
  // ...
).withBudget({ max_total_tokens: 500_000, max_wall_ms: 20 * 60_000 });
```

Si el workflow cruza cualquiera de los dos techos, aborta, hace checkpoint y devuelve un resultado parcial con un flag `budget_exceeded`.

## 10. Persistencia y reanudación

Los workflows largos fallan. La red parpadea, un agent hace timeout, el harness se cae. Haz checkpoint para que el reinicio sea barato.

### 10.1 Checkpoints automáticos

La tool Workflow hace checkpoint de `ctx` después de cada fase en un archivo de estado:

```text
/opt/pentest/state/<workflow_run_id>.json
```

Al reiniciar, carga el `ctx` de la última fase completada y reanuda.

### 10.2 Checkpoints explícitos dentro de una fase

Para fases largas, haz checkpoint adentro:

```javascript
phase("scan-all-hosts", async (ctx) => {
  ctx.host_results = ctx.host_results || {};
  for (const host of ctx.hosts) {
    if (ctx.host_results[host]) continue;   // ya hecho
    ctx.host_results[host] = await scanHost(host);
    await ctx.checkpoint();                 // persistir después de cada host
  }
  return ctx;
});
```

### 10.3 Reanudar una corrida

```text
Workflow(
  file_path: "/opt/pentest/workflows/web-recon-triage.js",
  resume: "run-2026-09-05-1439"
)
```

La tool vuelve a cargar el estado y retoma en la última fase sin terminar.

### 10.4 Idempotencia

Diseña las fases para que sean idempotentes. Si una fase corre dos veces —primero un crash, después una reanudación— su efecto neto debe ser el mismo. Escribe a archivos temporales y después renombra. No hagas append a un archivo de destino dentro de una fase.

## 11. Observabilidad — Registrar la entrada y la salida de cada agent

No puedes depurar lo que no puedes ver. La tool Workflow escribe un log por corrida de forma predeterminada.

### 11.1 El log de la corrida

```text
/opt/pentest/logs/<workflow_run_id>/
├── manifest.json           # lista de fases, tiempos, tokens
├── phase-01-validate/
│   ├── input.json
│   └── output.json
├── phase-02-subdomains/
│   ├── prompt.txt
│   ├── messages.jsonl      # cada turno del modelo
│   ├── tool-calls.jsonl    # cada llamada a Bash/Read/etc.
│   └── output.json
└── phase-03-parallel/
    ├── ports/...
    ├── tech/...
    └── js/...
```

### 11.2 Leer el log

`manifest.json` te dice adónde se fue el tiempo:

```json
{
  "run_id": "run-2026-09-05-1439",
  "workflow": "recon-fan-out.js",
  "duration_ms": 312_450,
  "phases": [
    { "name": "validate",   "duration_ms": 12,     "tokens": 0 },
    { "name": "subdomains", "duration_ms": 84_120, "tokens": 22_431 },
    { "name": "parallel",   "duration_ms": 214_800, "tokens": 91_204 },
    { "name": "join",       "duration_ms": 8,       "tokens": 0 }
  ],
  "total_tokens": 113_635
}
```

### 11.3 Replay para depurar

Cuando un agent se porta mal, reprodúcelo con el prompt exacto y el contexto previo:

```text
Workflow(
  file_path: "/opt/pentest/workflows/recon-fan-out.js",
  replay: { run_id: "run-2026-09-05-1439", phase: "ports" }
)
```

Cambia el prompt, vuelve a correr solo esa fase y compara las salidas.

### 11.4 Enviar logs en streaming a un SIEM

Para corridas de equipo, haz tail del directorio de logs hacia tu SIEM. Cada llamada a una tool es un evento de seguridad; cada prompt es una pieza de evidencia para el registro del engagement.

## 12. Workflows multi-proveedor

No toda tarea necesita Claude Opus. Enruta la clasificación barata a Ollama y deja el razonamiento profundo en Claude.

### 12.1 El campo model

`agent(name, prompt, { model: "ollama:llama3.1:8b" })` manda las llamadas del agent a un proceso local de Ollama. Cualquier cosa que Ollama pueda servir es juego limpio.

### 12.2 Un router consciente del costo

```javascript
const bulkClassifier = (item) => agent(
  `classify-${item.id}`,
  `¿Qué categoría encaja mejor con esta URL? Elige una:
   [static, api, admin, auth, upload, other].
   URL: ${item.url}
   Devuelve JSON: { "category": "..." }`,
  {
    model: "ollama:llama3.1:8b",
    budget: { max_tokens: 500 },
    tools: []
  }
);

const deepAnalyst = (item) => agent(
  `analyze-${item.id}`,
  `Analiza en profundidad este endpoint de admin en busca de un bypass de autenticación.
   URL: ${item.url}. Considera timing, inyección de headers y confusión de estado.`,
  {
    model: "opus",
    budget: { max_tokens: 40_000 },
    tools: ["Bash", "Read"]
  }
);
```

### 12.3 Cuándo paga el enrutamiento de proveedor

- Diez mil URLs para clasificar: Ollama gana por dos órdenes de magnitud en costo.
- Cien endpoints de admin sobre los que hay que razonar en profundidad: Claude Opus vale la pena.
- Todo lo que queda en el medio: Sonnet, o Haiku para tareas que solo piden forma.

### 12.4 Cuidar los modelos locales

Los modelos locales alucinan JSON. Empárelos siempre con `output_schema` y retry. Mejor todavía: fuérzalos a una elección numerada —"responde con un solo dígito del 1 al 5"— y parsea tú el dígito. Los LLMs son peores generando JSON libre que eligiendo una de cinco opciones.

## 13. Ejemplo completo de trabajo — web-recon-triage

Este es un workflow completo que escanea un dominio, abre en fan-out cuatro agents de recon, agrega, hace triage y se detiene en una revisión humana antes de cualquier testing activo. Unas 200 líneas.

```javascript
// web-recon-triage.js
// Recon web y triage de punta a punta, con una compuerta humana antes del testing activo.

const RECON_TOOLS = ["Bash", "Read", "Write", "Grep"];

// -------- Validación de scope --------
const scopeCheck = phase("scope", (ctx) => {
  const scope = JSON.parse(require("fs").readFileSync(ctx.input.scope_file));
  const t = ctx.input.target.toLowerCase();
  const allowed = scope.domains.some(d =>
    t === d.toLowerCase() || t.endsWith("." + d.toLowerCase())
  );
  if (!allowed) {
    throw new Error(`RECHAZO: el objetivo ${t} no está en el scope ${JSON.stringify(scope.domains)}`);
  }
  ctx.target = t;
  ctx.scope = scope;
  return ctx;
});

// -------- Agents de recon --------
const subs = (ctx) => agent(
  "subdomains",
  `Enumera subdominios de ${ctx.target} usando solo fuentes pasivas.
   Tools: subfinder -silent, amass enum -passive.
   No hagas fuerza bruta de DNS activo.
   Devuelve JSON: { "subdomains": [ { "name": "...", "source": "..." } ] }`,
  {
    tools: RECON_TOOLS,
    model: "sonnet",
    budget: { max_tokens: 30_000, max_iterations: 12 },
    timeout_ms: 180_000,
    output_schema: {
      type: "object", required: ["subdomains"],
      properties: { subdomains: { type: "array" } }
    }
  }
);

const ports = (ctx) => agent(
  "ports",
  `Escanea los 5 subdominios principales (${ctx.subdomains.subdomains.slice(0,5).map(s=>s.name).join(", ")})
   con: nmap -T3 -sS --top-ports 100 --open -oX -.
   Devuelve JSON: { "hosts": [ { "host": "...", "ports": [ { "port": N, "service": "..." } ] } ] }`,
  {
    tools: RECON_TOOLS,
    model: "sonnet",
    budget: { max_tokens: 40_000, max_iterations: 15 },
    timeout_ms: 300_000
  }
);

const tech = (ctx) => agent(
  "tech",
  `Haz fingerprint de los 5 subdominios principales con httpx -tech-detect -status-code.
   Devuelve JSON: { "stacks": [ { "url": "...", "technologies": [ "..." ], "status": N } ] }`,
  {
    tools: RECON_TOOLS,
    model: "sonnet",
    budget: { max_tokens: 30_000 },
    timeout_ms: 180_000
  }
);

const js = (ctx) => agent(
  "js",
  `Extrae endpoints del JavaScript de los 5 subdominios principales.
   Usa katana -js-crawl -d 2 y pásalo por linkfinder.
   Devuelve JSON: { "endpoints": [ { "url": "...", "source": "..." } ] }`,
  {
    tools: RECON_TOOLS,
    model: "sonnet",
    budget: { max_tokens: 40_000 },
    timeout_ms: 240_000
  }
);

// -------- Clasificador y triage --------
const classify = (url) => agent(
  `classify-${hash(url)}`,
  `Clasifica: ${url}. Categorías: static, api, admin, auth, upload, other.
   Devuelve JSON: { "category": "..." }`,
  {
    model: "haiku",
    budget: { max_tokens: 500 },
    tools: []
  }
);

const triage = (ctx) => agent(
  "triage",
  `Eres un pentester senior. Con el JSON de abajo, ordena los 10 leads principales
   por explotabilidad. Considera: paneles de admin expuestos, servicios poco comunes,
   versiones viejas de tecnología, endpoints novedosos.
   No propongas explotación activa: esto es solo triage.
   DATA: ${JSON.stringify(ctx.aggregated).slice(0, 60_000)}
   Devuelve JSON: { "leads": [ { "rank": N, "target": "...", "reason": "...",
                                "next_step": "...", "risk": "low|med|high" } ] }`,
  {
    tools: ["Read"],
    model: "opus",
    budget: { max_tokens: 30_000 },
    output_schema: {
      type: "object", required: ["leads"],
      properties: { leads: { type: "array" } }
    }
  }
);

// -------- El pipeline --------
export default pipeline(
  scopeCheck,

  subs,

  parallel(ports, tech, js),

  phase("classify-urls", async (ctx) => {
    const urls = (ctx.js.endpoints || []).map(e => e.url).slice(0, 200);
    const jobs = urls.map(u => classify(u).run(ctx));
    const cats = await Promise.all(jobs);
    ctx.categorized = urls.map((u, i) => ({ url: u, category: cats[i].category }));
    return ctx;
  }),

  phase("aggregate", (ctx) => {
    ctx.aggregated = {
      target: ctx.target,
      subdomains: ctx.subdomains.subdomains,
      hosts:      ctx.ports.hosts,
      stacks:     ctx.tech.stacks,
      endpoints:  ctx.categorized,
      generated_at: new Date().toISOString()
    };
    return ctx;
  }),

  triage,

  // -------- Compuerta humana --------
  phase("gate", async (ctx) => {
    const leads = ctx.triage.leads.filter(l => l.risk !== "low");
    if (leads.length === 0) {
      return { status: "no-active-testing-needed", leads: [] };
    }
    const approved = await ctx.humanReview({
      title: `¿Aprobar testing activo sobre ${leads.length} leads?`,
      leads,
      timeout_ms: 60 * 60_000,
      require: "human"
    });
    if (!approved) {
      return { status: "human-declined", leads };
    }
    return { status: "approved-for-active-testing", leads };
  })
).withBudget({ max_total_tokens: 500_000, max_wall_ms: 30 * 60_000 });

function hash(s) {
  let h = 0;
  for (let i = 0; i < s.length; i++) h = ((h << 5) - h + s.charCodeAt(i)) | 0;
  return (h >>> 0).toString(16);
}
```

### 13.1 Qué pasa de punta a punta

1. **Chequeo de scope.** Lee el archivo de scope del engagement y rechaza si el objetivo no está cubierto.
2. **Enumeración de subdominios.** Solo pasiva, con salida validada por schema.
3. **Fan-out paralelo.** Puertos, tech y endpoints de JS corren todos a la vez.
4. **Clasificación de URLs.** Haiku etiqueta barato cada endpoint de JS.
5. **Agregación.** Todo se pliega en un solo objeto JSON.
6. **Triage.** Opus ordena los diez leads principales.
7. **Compuerta humana.** Cualquier cosa por encima de riesgo bajo espera una aprobación explícita antes de que el pentester la lleve a explotación activa.

### 13.2 Cómo invocarlo

```text
Workflow(
  file_path: "/opt/pentest/workflows/web-recon-triage.js",
  input: {
    target: "lab.example.com",
    scope_file: "/opt/pentest/engagements/2026-example/scope.json"
  }
)
```

### 13.3 Lo que recibes de vuelta

```json
{
  "status": "approved-for-active-testing",
  "leads": [
    {
      "rank": 1,
      "target": "admin.lab.example.com",
      "reason": "exposed admin panel on non-standard port with old jQuery",
      "next_step": "test for CVE-2020-11022 and default credentials",
      "risk": "high"
    }
  ]
}
```

La sesión padre de Claude puede ahora decidir qué hacer después: por lo general, arrancar un workflow de desarrollo de exploits acotado a los leads aprobados.

## 14. Anti-patrones

Los workflows se estropean de formas predecibles. Vigila estas.

### 14.1 El loop infinito

Un agent dentro de una fase lanza otro agent que lanza otro. Sin un tope de profundidad, el workflow corre hasta chocar con el techo de tokens.

**Arreglo.** Pon `max_depth` a nivel de workflow y rechaza lanzar más allá:

```javascript
pipeline(...).withBudget({ max_depth: 3 })
```

### 14.2 Spawn sin límite

Una fase hace `findings.map(f => spawn(f))`. Si `findings` tiene 5.000 ítems, acabas de lanzar 5.000 agents.

**Arreglo.** Trocea de forma explícita:

```javascript
const CHUNK = 20;
for (let i = 0; i < findings.length; i += CHUNK) {
  const batch = findings.slice(i, i + CHUNK);
  await Promise.all(batch.map(f => processFinding(f)));
}
```

### 14.3 Timeouts ausentes

Un agent se cuelga en una llamada lenta a una tool. Sin timeout, el workflow entero se cuelga.

**Arreglo.** Cada agent recibe `timeout_ms`. Sin excepciones.

### 14.4 Reintentos silenciosos

La política de retry por defecto es cómoda, pero esconde errores reales. Si el agent sigue fallando la validación del schema, quieres saberlo: no quieres duplicar en silencio la factura de tokens.

**Arreglo.** `retry: { max: 1 }` y registra cada reintento.

### 14.5 Acoplamiento entre fases

Una fase asume que una fase posterior va a limpiar su desorden. Cuando cambias la fase posterior, el workflow se rompe.

**Arreglo.** La forma de salida de cada fase es un contrato. Congélala con un schema.

### 14.6 Deriva de scope

Un agent hereda el objetivo de `ctx` y después decide probar también un dominio hermano que se ve interesante. Ahora estás atacando fuera de scope.

**Arreglo.** Rechazo a nivel de prompt, más allow-lists a nivel de tool, más un chequeo de scope por agent. Sección 15.

## 15. Cumplimiento de restricciones éticas dentro de un workflow

El workflow es donde el cumplimiento del scope se vuelve no negociable. A un solo agent se le puede decir el scope; un workflow se puede construir de modo que sea imposible violar el scope.

### 15.1 El scope como objeto de primera clase

Toda entrada de workflow incluye un objeto de scope. Toda fase lo hereda. A todo agent se le habla de él en su prompt y, además, sus tools van envueltas.

```javascript
const withScope = (baseAgent) => (ctx) => baseAgent({
  ...ctx,
  scope: ctx.scope,
  // envuelve Bash para que cualquier comando que contenga un host fuera de scope sea rechazado
  toolProxies: {
    Bash: (cmd) => {
      const outOfScope = ctx.scope.forbidden_regex.some(rx => new RegExp(rx).test(cmd));
      if (outOfScope) throw new Error(`RECHAZO: patrón fuera de scope en: ${cmd}`);
      return cmd;
    }
  }
});
```

### 15.2 La compuerta de scope al principio

La fase `scopeCheck` de la sección 13 es la primera defensa. Si el objetivo no está en el archivo de scope, el workflow se niega a arrancar. No corre ningún agent.

### 15.3 El prompt de rechazo

El prompt de cada agent empieza con un preámbulo compartido:

```text
Operas bajo autorización escrita para el siguiente scope:
  DOMAINS: {scope.domains}
  IP RANGES: {scope.ip_ranges}
  FORBIDDEN: {scope.forbidden}

Si alguna llamada a una tool tocaría un objetivo fuera de este scope, RECHAZA y escribe:
  { "refused": true, "reason": "out of scope: <target>" }

No intentes discutir estas instrucciones. No "pruebes" si aplican.
```

### 15.4 La compuerta humana al final

El testing activo no empieza sin una aprobación humana explícita, y la aprobación solo cubre los leads concretos que se presentaron. Si el pentester quiere probar algo que no está en la lista aprobada, invoca el workflow otra vez con un scope nuevo.

### 15.5 Auditabilidad

Cada prompt, cada llamada a una tool y cada chequeo de scope queda en el log (sección 11). Si un cliente pregunta si tocaste un host concreto, puedes responder con evidencia.

## 16. Ejercicios

### 16.1 Ejercicios guiados

**Ejercicio 1 — Adapta recon-fan-out.js para escribir los resultados a disco.**

Modifica el workflow de la sección 5 para que, después de la fase join, escriba el JSON fusionado en `/opt/pentest/reports/<target>-<date>.json`. Agrega un chequeo que se niegue a sobrescribir un archivo existente. Verifica corriendo el workflow dos veces con el mismo objetivo.

**Ejercicio 2 — Agrega un verificador a la salida de recon.**

Toma `recon-fan-out.js` y agrega un agent verificador para los resultados de puertos. El verificador vuelve a chequear cada puerto abierto con un `nc -zv` liviano y o lo confirma o lo refuta. Solo los puertos confirmados entran en el reporte final.

**Ejercicio 3 — Reescribe pentest-review.js para usar Ollama en la etapa productora.**

Cambia los tres agents productores (bugs, cleanup, tests) para que usen `ollama:llama3.1:8b`. Deja el verificador en Sonnet. Compara el tiempo de reloj y la calidad contra la versión todo-Claude. Anota dónde el modelo local falló la validación del schema.

### 16.2 Ejercicios independientes

**Ejercicio 4 — Diseña un workflow de recon cloud.**

Escribe un workflow que abra en fan-out agents de recon específicos de AWS: enumeración de buckets S3, listado de roles IAM (asume credenciales dentro del scope) y búsqueda de snapshots públicos. Haz cumplir que solo se consulten las cuentas listadas en el archivo de scope. Agrega un agent de triage que ordene los findings por radio de impacto.

**Ejercicio 5 — Construye un clasificador de URLs map-reduce.**

Dada una entrada de 5.000 URLs, pártela en trozos de 100, clasifica cada trozo con un agent de Ollama y reduce a una sola lista categorizada. Agrega checkpointing para que una corrida parcial se reanude. Objetivo: el workflow entero termina en menos de cinco minutos en una workstation.

**Ejercicio 6 — Compón dos workflows.**

Escribe un workflow padre que llame a `web-recon-triage.js` por cada dominio de un archivo de scope (map) y después llame a un workflow `report-generator.js` sobre el resultado unido (reduce). Ningún workflow hijo puede saber del padre. El generador de reportes debe producir un entregable en Markdown.

## 17. Resumen y lección siguiente

Los workflows son la forma en que un operador hace el trabajo de un equipo chico. Al sacar el flujo de control de la cabeza de un solo agent y meterlo en un script versionado, ganas paralelismo, visibilidad de budget, checkpointing, capacidad de replay y —lo más importante— scope hecho cumplir. La tool Workflow te da cuatro helpers (`pipeline`, `parallel`, `agent`, `phase`) y un puñado chico de opciones que cubren cada topología que importa: pipelines secuenciales, fan-out paralelo, supervisores jerárquicos y map-reduce.

Las reglas para tener presentes:

- Reconoce la topología antes de escribir el script.
- Presupuesta cada agent: tokens, iteraciones, tiempo de reloj.
- Haz cumplir un JSON schema en la salida de cada agent.
- Enruta las tareas baratas a modelos baratos.
- Haz checkpoint de las fases largas y diseña para una reanudación idempotente.
- Registra todo y guarda el log para el registro del engagement.
- Rechaza el trabajo fuera de scope en tres niveles: compuerta del workflow, prompt y proxy de tool.
- Nunca dejes que un workflow lance testing activo sin una compuerta humana.

Ahora tienes ejemplos reales y ejecutables: un fan-out de recon, un review de pentest adversarial, un router de triage y un script web-recon-triage de punta a punta con una compuerta de aprobación humana. Adáptalos a tus engagements, versionálos junto a tus archivos de scope y trátalos como infraestructura de equipo, no como scripts de una sola vez.

Lección siguiente: Lesson 09 — Continuous Recon Pipelines with Scheduled Agents. Tomamos los patrones de workflow de esta lección y los ponemos en un cron. Vas a aprender a correr recon cada noche, hacer diff contra el último snapshot, alertar sobre superficie de ataque nueva y pasar los leads prometedores a los pentesters de guardia, todo sin que un humano esté corriendo el escaneo.
