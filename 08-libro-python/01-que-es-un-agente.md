# 01 · Qué es un agente de IA

> *Del chat que responde al sistema que actúa*
>
> Capítulo 01 · *Agentes de IA con Python: guía práctica para diseñar, construir y desplegar sistemas agénticos*

---

## La distinción que importa

La diferencia entre un chatbot y un agente de IA no es de calidad, sino de **arquitectura**.

- Un **chatbot**, por sofisticado que sea, opera dentro de una interacción por turnos: el usuario escribe, el modelo responde, el intercambio concluye.
- Un **agente** opera dentro de un modelo de consecución de objetivos: recibe una meta, formula o adapta un plan, actúa sobre el entorno mediante herramientas, observa los resultados y decide si continuar, corregir o escalar al usuario.

El turno de conversación da paso al ciclo de acción.

Esa diferencia cambia tres cosas:

1. **Quién evalúa el resultado.** En un chat, el usuario lee una respuesta. En un sistema que actúa, se verifica si un archivo existe, si el código pasa las pruebas o si la tarea se completó según criterios definidos.
2. **Los criterios de diseño.** Una respuesta útil exige claridad y relevancia. Una acción útil exige corrección, trazabilidad y reversibilidad.
3. **El perfil de riesgo.** Cuando un sistema puede actuar sobre el entorno, el desarrollador asume responsabilidades que van más allá de la calidad del texto generado.

### Predecir texto frente a completar tareas

Los modelos de lenguaje se entrenaron, en gran parte, para predecir la continuación más probable de un texto dado. Esa capacidad produce respuestas notablemente útiles: explicar un concepto, resumir un documento, redactar un borrador, responder una pregunta técnica.

Pero predecir la continuación de un texto y completar una tarea en el mundo son operaciones de naturaleza distinta.

### Un ejemplo: la misma pregunta, dos sistemas

Supón que el usuario pide localizar la documentación más reciente del SDK de agentes de OpenAI.

- **Un chatbot** sin acceso a herramientas dirá que no puede hacerlo o, en el peor caso, generará información plausible pero desactualizada. Su entorno es el contexto de la conversación: no puede leer un archivo que no se le ha proporcionado, ejecutar código para verificar sus afirmaciones ni enviar una solicitud HTTP aunque el usuario se lo pida.
- **Un agente** puede recibir esa misma solicitud, invocar una herramienta de búsqueda web, leer resultados de varias fuentes, comparar fechas de publicación, identificar la versión más reciente del parámetro que el usuario busca y devolver una respuesta verificada.

El ciclo no termina cuando el modelo genera texto. Termina cuando:

- el objetivo se considera cumplido,
- el sistema reconoce que no puede cumplirlo con los recursos disponibles, o
- el diseño del agente exige escalar al usuario antes de proceder.

### La diferencia no es solo de inteligencia

Un modelo muy capaz encerrado en una arquitectura de chatbot seguirá siendo un chatbot: podrá razonar dentro de la conversación, pero no actuar fuera de ella. **Lo que define a un agente es la arquitectura que conecta el modelo con el entorno y le permite intervenir sobre él.**

---

## Cuatro arquetipos en el mismo espectro

Las categorías rígidas sirven para definir los extremos del espectro, aunque los sistemas reales rara vez encajan con precisión en una sola. Cuatro arquetipos permiten situar cualquier sistema:

### 1. Chatbot

Responde en lenguaje natural dentro de una conversación, sin acceso a información externa ni capacidad para modificar su entorno. Su valor está en la calidad de las respuestas generadas a partir del entrenamiento y el contexto del hilo.

Es potente para explicar, redactar, analizar o conversar. Su acción queda confinada al texto que produce.

### 2. Asistente con herramientas

Amplía esa capacidad con acceso a recursos externos: puede buscar en la web, consultar un calendario, ejecutar una calculadora o llamar a una API.

Suele hacerlo en el marco de una interacción corta y dirigida: el usuario solicita una acción específica, el modelo la ejecuta y devuelve el resultado. La cadena es breve (a menudo un único paso de herramienta), y el modelo no decide por su cuenta cuántas herramientas usar ni en qué orden.

### 3. Flujo de trabajo automatizado

Representa el extremo opuesto en previsibilidad: ejecuta una secuencia de pasos predefinidos con alta fiabilidad cuando el camino está cerrado y bien especificado.

Un pipeline que descarga un archivo, lo transforma y lo carga en una base de datos es un flujo de trabajo. Su fortaleza es la reproducibilidad; su limitación, la falta de criterio ante lo no previsto. Si una fuente devuelve un error inesperado o los datos llegan con otro formato, el flujo se detiene o falla silenciosamente.

### 4. Agente

Sintetiza las capacidades de los tres anteriores y añade una que ninguno tiene: **criterio de avance**.

Puede conversar, usar herramientas y seguir un plan, pero también reconocer cuándo el plan no funciona y replanificarse, cuándo necesita información adicional y pedirla, y cuándo ha alcanzado el objetivo y detenerse sin que el usuario se lo indique. Opera con un grado de autonomía que no requiere aprobación externa en cada decisión intermedia, aunque su diseño puede —y en muchos casos debe— incluir supervisión humana para decisiones de alto impacto.

Claude Code, por ejemplo, puede comportarse como asistente con herramientas en tareas simples (responder una pregunta sobre el código, buscar un símbolo en el repositorio) y como agente con replanificación en tareas complejas: depurar un fallo cuya causa raíz requiere inspeccionar varios archivos, ejecutar las pruebas, interpretar el output de error, formular una hipótesis, aplicar la corrección y confirmar el resultado. La misma infraestructura se usa de distinta forma según la naturaleza del objetivo.

---

## El bucle percepción-razonamiento-acción

La teoría clásica de la inteligencia artificial define al agente como una entidad que percibe su entorno mediante sensores y actúa sobre él mediante actuadores. Esta formulación, concebida para robots y sistemas simbólicos, se traslada con notable fidelidad a los agentes de lenguaje modernos: los sensores y actuadores han cambiado de forma, pero la lógica es la misma.

- **Sensores**: todo aquello que puede leer o recibir como entrada. Instrucciones del sistema que definen su rol y sus límites, mensajes del usuario, resultados de herramientas, contenido de archivos, respuestas de API, capturas de pantalla de un navegador o eventos de una cola de mensajes.
- **Actuadores**: todo aquello que puede producir como salida con efecto en el mundo. Llamadas a funciones, edición de archivos, comandos de terminal, envío de mensajes, navegación web o acciones en interfaces de aplicaciones.

Entre percepción y acción opera el modelo de lenguaje: razona sobre el estado actual, evalúa qué sabe y qué le falta, decide qué acción tiene más probabilidades de acercarlo al objetivo y emite esa acción como salida estructurada. El resultado de la acción vuelve al sistema como nueva entrada, cerrando el ciclo.

```text
Objetivo
    ↓
Percepción  → leer instrucciones, archivos, estado, resultados anteriores
    ↓
Razonamiento → evaluar situación, decidir siguiente acción
    ↓
Acción       → llamar herramienta / editar archivo / ejecutar código / navegar / etc.
    ↓
Observación  → leer resultado de la acción
    ↓
¿Objetivo cumplido?
   Sí → entregar resultado
   No → volver a Percepción
```

Este bucle puede iterar decenas de veces en una tarea compleja. Un agente que recibe la instrucción *"corrige el error en el módulo de exportación y añade una prueba que lo cubra"* seguirá, por ejemplo, este recorrido:

1. leer archivos relevantes del repositorio,
2. identificar la función problemática,
3. ejecutar las pruebas existentes para confirmar el fallo,
4. interpretar la salida de error,
5. formular una hipótesis sobre la causa raíz,
6. editar el archivo,
7. ejecutar las pruebas de nuevo,
8. escribir la nueva prueba y confirmar el resultado.

Cada vuelta produce información nueva que orienta la siguiente decisión.

El ciclo depende de la infraestructura que conecta el modelo con el entorno: herramientas, memoria persistente y permisos que delimitan su campo de acción. Sin esa infraestructura, el modelo más capaz no puede completar un objetivo que requiera más de un turno de conversación.

---

## El espectro de autonomía

La autonomía de un agente no es una propiedad binaria. Se distribuye a lo largo de un espectro que va desde la respuesta pura hasta la operación con supervisión delegada. Cualquier posición es legítima según el contexto, el riesgo y el nivel de confianza que el diseñador decide otorgar al sistema.

Una escala de cinco posiciones permite concretar ese espectro:

| Nivel | Qué hace el sistema | Ejemplo | Quién decide |
|---|---|---|---|
| 1. **Responder** | Explica, describe o analiza sin ejecutar ninguna acción fuera del texto | Diferencia entre RAG y ajuste fino | El usuario |
| 2. **Sugerir** | Propone un plan con pasos concretos, pero no ejecuta ninguno | Secuencia para implementar un agente de investigación con la API de Anthropic | El usuario ejecuta |
| 3. **Usar una herramienta** | Ejecuta una o pocas acciones acotadas para producir un resultado | Buscar tres fuentes y resumirlas; consultar el estado de un repositorio | El usuario revisa antes de usarlo |
| 4. **Ejecutar una tarea** | Completa un objetivo de múltiples pasos sin aprobación intermedia | Crear un archivo de configuración, ejecutar pruebas, corregir errores y confirmar que todo pasa | El usuario define el objetivo al inicio y revisa al final |
| 5. **Operar con supervisión delegada** | Opera de forma continua o recurrente, con permisos amplios y puntos de escalado explícitos | Revisar cada mañana los pull requests abiertos y pedir aprobación antes de fusionar | El humano conserva las decisiones de alto impacto |

### Ninguna posición es inherentemente superior

Una tarea de bajo riesgo y alta repetición (clasificar correos entrantes, actualizar un log de cambios, verificar que las pruebas pasan en cada commit) puede operar en los niveles 4 o 5.

Una acción que implica enviar comunicaciones a usuarios reales, modificar datos en producción o comprometer recursos económicos debe mantenerse en el nivel 3 o inferior, con aprobación explícita en cada paso sensible.

El nivel de autonomía adecuado es una decisión de diseño y de política de control, no una consecuencia automática de la capacidad del modelo.

### Cómo lo describen los proveedores

OpenAI describe sus agentes como modelos de lenguaje equipados con instrucciones, herramientas y primitivas de coordinación, incluidos *handoffs* entre agentes, *guardrails* de seguridad, sesiones con estado y trazas de auditoría.

Anthropic, en su documentación de *tool use*, distingue entre herramientas cuya ejecución ocurre en el entorno del desarrollador (este decide qué ejecutar y cuándo) y herramientas gestionadas por la infraestructura del proveedor.

Ambas aproximaciones comparten un principio: **la autonomía de un agente siempre está acotada por una arquitectura de control diseñada de forma explícita.**

---

## Las piezas que habilitan la agencia

La agencia no depende únicamente de la capacidad del modelo para razonar; depende de la infraestructura que lo rodea. Cuatro categorías de componentes transforman un modelo en un sistema capaz de perseguir objetivos.

### Herramientas

Son las funciones que el agente puede invocar para leer o modificar el entorno: una búsqueda web, una llamada a una API de gestión de tareas, un comando de terminal, la lectura o escritura de un archivo.

El modelo no ejecuta estas funciones directamente: emite una solicitud estructurada con la herramienta y sus argumentos; el sistema anfitrión ejecuta la llamada real y devuelve el resultado como nueva entrada. **Esta separación entre decisión y ejecución es fundamental para el control**: permite validar, registrar, limitar o bloquear cualquier invocación antes de que produzca efecto en el entorno.

### Memoria

Los modelos de lenguaje no retienen información entre conversaciones por diseño. Para operar en tareas largas o recordar sesiones anteriores, un agente necesita mecanismos explícitos:

- **contexto activo** en cada iteración del bucle,
- **almacenamiento externo** que puede leer y escribir durante la tarea (bases de datos, archivos, índices de búsqueda semántica),
- **estrategias de compresión** del historial cuando el contexto supera la ventana del modelo.

Sin memoria persistente, el agente comienza cada tarea desde cero y no acumula el estado necesario para objetivos que exceden un único contexto de conversación.

### Permisos y reglas de control

La autonomía sin límites no es un objetivo de diseño deseable. Todo agente bien construido opera dentro de restricciones explícitas:

- qué herramientas tiene disponibles,
- qué recursos puede leer o modificar,
- qué acciones requieren aprobación humana,
- qué condiciones lo obligan a detenerse y escalar.

Estas reglas no viven en el modelo —no se "programan" modificando sus pesos—, sino en el sistema que lo orquesta: el prompt de sistema que define su rol y límites, el código que conecta el modelo con las herramientas y la lógica de supervisión que intercepta acciones de riesgo antes de ejecutarlas.

### Orquestación

En sistemas con múltiples agentes, o en tareas que requieren coordinar varios modelos especializados, hace falta una capa que gestione el flujo entre componentes:

- qué agente trabaja en qué subproblema,
- cómo se transfieren los resultados,
- cómo se agregan y verifican las conclusiones antes de presentarlas al usuario.

La presencia de múltiples agentes no multiplica los riesgos de forma automática si cada componente opera con permisos mínimos y supervisión adecuada. **La capa de orquestación es tan parte del agente como el modelo mismo.**

---

## Resumen

Herramientas, memoria, permisos y orquestación determinan si un sistema es un agente en sentido pleno o un modelo con acceso puntual a una función externa.

La capacidad de razonamiento del modelo es condición necesaria, pero no suficiente. La infraestructura que lo rodea convierte ese razonamiento en acción controlada sobre el mundo.
