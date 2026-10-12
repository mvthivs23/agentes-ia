# Seguridad de LLM: resumen del manual de estudio

> Notas de estudio en español sobre los nueve módulos del manual *Seguridad de LLM* (Red Team Leaders, versión 1.0, 2026). Autor de las notas: Matías Salazar.

## Índice

1. [Módulo 01: Fundamentos y arquitectura de LLM](#módulo-01-fundamentos-y-arquitectura-de-llm)
2. [Módulo 02: OWASP Top 10 para aplicaciones LLM](#módulo-02-owasp-top-10-para-aplicaciones-llm)
3. [Módulo 03: Ingeniería de prompts y jailbreaks](#módulo-03-ingeniería-de-prompts-y-jailbreaks)
4. [Módulo 04: Gobernanza y gestión de riesgos](#módulo-04-gobernanza-y-gestión-de-riesgos)
5. [Módulo 05: Privacidad y tratamiento de datos](#módulo-05-privacidad-y-tratamiento-de-datos)
6. [Módulo 06: Seguridad de MCP](#módulo-06-seguridad-de-mcp-model-context-protocol)
7. [Módulo 07: Agentes de IA, orquestación y vibe coding](#módulo-07-agentes-de-ia-orquestación-y-vibe-coding)
8. [Módulo 08: Seguridad de aplicaciones para productos de IA](#módulo-08-seguridad-de-aplicaciones-para-productos-de-ia)
9. [Módulo 09: Identidad, acceso, memoria y respuesta a incidentes](#módulo-09-identidad-acceso-memoria-y-respuesta-a-incidentes)
10. [Idea central del manual](#idea-central-del-manual)

---

## Módulo 01: Fundamentos y arquitectura de LLM

### 1.1 Arquitectura Transformer y atención

- Todos los LLM de frontera (GPT-4, Claude, Gemini, LLaMA) usan la arquitectura Transformer (Vaswani et al., 2017), que reemplazó a las RNN y LSTM con autoatención.
- Fórmula de atención:

  ```
  Attention(Q, K, V) = softmax(QKᵀ / √d_k) · V
  ```

- **Q** (consulta): qué busca el token. **K** (clave): qué contiene. **V** (valor): qué aporta.
- La división por √d_k evita que el softmax caiga en zonas de gradiente muy pequeño.
- **Atención causal**: en modelos tipo GPT, cada token solo ve posiciones anteriores. Es la razón por la que el modelo predice el siguiente token.
- **Atención multicabeza**: varias operaciones en paralelo, cada una con proyecciones propias. Los modelos modernos usan desde decenas hasta más de cien cabezas.
- **Implicación de seguridad**: el prompt de sistema, la entrada del usuario y los documentos recuperados pasan por el mismo mecanismo. No hay separación arquitectónica entre contenido confiable y no confiable. Esa es la causa raíz de la inyección de prompts.

### 1.2 Pipeline de entrenamiento

| Etapa | Qué hace | Consideración de seguridad |
|---|---|---|
| Pre-entrenamiento | Predice el siguiente token sobre datos masivos (CLM) | Memoriza datos, incluida información sensible |
| SFT | Ajusta con pares instrucción-respuesta | Moldea formato y tono; los pares envenenados persisten |
| RLHF | Modelo de recompensa entrenado con preferencias humanas + PPO | Mecanismo principal de alineación; el objetivo de los jailbreaks |
| Constitutional AI | El modelo se autocritica contra principios escritos | Alineación más consistente y escalable |

- **Impuesto de alineación (alignment tax)**: la seguridad reduce la disposición del modelo a producir ciertas salidas. Los jailbreaks buscan acceder a capacidades que la alineación restringió.

### 1.3 Parámetros de inferencia y tokenización

- **Temperatura (0,0–2,0)**: con 0, la decodificación es determinista (codiciosa). Valores altos aumentan la aleatoriedad.
  - Ventaja: salidas reproducibles para pruebas.
  - Riesgo: salidas predecibles, más fáciles de sondear.
- **Top-p (núcleo)**: umbral de probabilidad acumulada. Con p = 0,9, el conjunto de candidatos cubre el 90% de la masa de probabilidad.
- **Tokens máximos**: límite de longitud de salida. Es el control principal contra la denegación de servicio del modelo.
- **Tokenización**: BPE o SentencePiece. "impredecible" puede dividirse en "im" + "predec" + "ible". Cada modelo tokeniza distinto, así que el mismo texto cuesta distinta cantidad de tokens.
- **Ventana de contexto**: entrada más salida por llamada. Va desde 4K hasta más de 200K tokens. Ventanas más grandes amplían la superficie de manipulación.

### 1.4 Panorama de modelos y postura de seguridad

- **OpenAI (GPT-4, ChatGPT)**: plugins, GPT Actions, Code Interpreter y navegación. Cada plugin es un vector de envenenamiento de herramientas.
- **Anthropic (Claude)**: Constitutional AI, contextos largos y documentación pública de prompts de sistema.
- **Google (Gemini)**: multimodal nativo e integración con Workspace, Android y Search. Un compromiso puede propagarse por correo, documentos y calendario.
- **Ollama**: pesos abiertos en hardware local. No hay filtrado de un proveedor; el operador asume toda la seguridad.
- **GitHub Copilot**: entrenado en repositorios públicos, con código vulnerable. Puede sugerir inyección SQL o secretos codificados. Es el riesgo central del *vibe coding*.

### 1.5 RAG, fine-tuning y cuantización

- **RAG**: recupera documentos en la inferencia. Reduce la alucinación, pero añade inyección indirecta, embeddings envenenados y riesgo si los permisos por documento no se aplican en la recuperación.
- **Fine-tuning**: completo o eficiente (LoRA). Datos envenenados pueden incrustar puertas traseras o sesgos persistentes, difíciles de detectar.
- **Cuantización**: de FP16/FP32 a INT8/INT4. Reduce memoria y costo, con posible pérdida de calidad y cambios en el comportamiento de seguridad.

> **Puntos clave del módulo**
> - La atención procesa todos los tokens igual; no existe un límite de confianza dentro del contexto.
> - Temperatura, top-p y tokens máximos afectan la seguridad y el consumo de recursos.
> - Cada modelo de despliegue (API, Ollama, Copilot) tiene trade-offs distintos.

---

## Módulo 02: OWASP Top 10 para aplicaciones LLM

El OWASP Top 10 para LLM identifica riesgos específicos de los sistemas de IA, no cubiertos por la seguridad web tradicional.

### LLM01: Inyección de prompts

- **Directa**: el atacante escribe en la entrada hacia el LLM para anular el prompt de sistema.
- **Indirecta**: instrucciones maliciosas en contenido que el LLM procesa (páginas web, documentos RAG, correos, reseñas). El atacante no interactúa con el modelo, así que es más difícil de defender.
- **Diputado confundido**: un LLM con herramientas tiene autoridad para actuar, pero no criterio para reconocer que lo manipulan.
- **Defensa en profundidad**: preprocesamiento de entrada, diseño estructural del prompt (delimitadores y jerarquía), validación de salida, restricción de privilegios y monitoreo.

### LLM02: Manejo inseguro de salidas

Las salidas del LLM son datos no confiables. Si se pasan sin sanitizar a sistemas posteriores:
- **XSS**: contenido renderizado en el navegador sin codificar.
- **Inyección SQL**: consultas generadas e interpoladas como cadenas.
- **Inyección de comandos**: salida usada para construir comandos de shell.
- **Path traversal**: rutas generadas que acceden a directorios no autorizados.

**Mitigación**: codificar según el destino (HTML para web, consultas parametrizadas para SQL, validación de rutas para archivos).

### LLM03: Envenenamiento de datos de entrenamiento

- Vectores: comprometer datasets públicos, contribuir datos envenenados a corpus de código abierto o manipular datos de ajuste con acceso interno.
- **Mitigación**: procedencia de los datos, análisis estadístico de anomalías, pruebas adversariales de los modelos ajustados y fuentes verificadas.

### LLM04: Denegación de servicio del modelo

- Entradas muy largas, estructuras recursivas o prompts diseñados para maximizar la salida.
- **Mitigación**: límites de tokens de entrada y salida, rate limiting por usuario o clave, monitoreo del consumo de tokens y alertas de costo.

### LLM05: Vulnerabilidades de la cadena de suministro

Incluye pesos preentrenados, datasets, modelos de embeddings, bases vectoriales, servidores MCP, plugins y frameworks de orquestación.
- **Mitigación**: checksums y procedencia del modelo, fuentes confiables, escaneo de dependencias y un SBOM de IA.

### LLM06: Divulgación de información sensible

- Vectores: memorización de datos de entrenamiento, extracción del prompt de sistema y recuperación RAG sin control de acceso.
- **Mitigación**: sanitizar datos de entrenamiento, filtrar salidas con patrones sensibles (DNI, tarjetas, claves), aplicar controles de acceso en RAG y **nunca incrustar secretos en el prompt de sistema**.

### LLM07: Diseño inseguro de plugins

- Plugins que aceptan entradas sin sanitizar, se ejecutan con permisos excesivos o carecen de autenticación.
- **Mitigación**: denegación por defecto, concesión explícita y acotada de permisos, validación de cada parámetro y registro de cada invocación.

### LLM08: Agencia excesiva

Cuando el LLM puede emitir reembolsos, borrar cuentas, enviar correos o ejecutar código sin aprobación, una inyección o una alucinación puede causar acciones irreversibles.
- **Mitigación**: human-in-the-loop para acciones de alto impacto, herramientas con el mínimo de capacidades y diseño reversible siempre que sea posible.

### LLM09: Dependencia excesiva

Confiar en salidas sin verificarlas (citas legales alucinadas, código con vulnerabilidades, datos inventados).
- **Mitigación**: revisión humana obligatoria en decisiones reguladas, divulgación de que el contenido es generado por IA y contraste con fuentes autorizadas.

### LLM10: Robo de modelo

- Acceso no autorizado a los pesos, o reconstrucción del comportamiento mediante consultas sistemáticas (extracción).
- **Mitigación**: controles de acceso al almacenamiento, rate limiting, detección de patrones de extracción y marcas de agua en las salidas.

---

## Módulo 03: Ingeniería de prompts y jailbreaks

### 3.1 Prompts de sistema seguros

Principios de diseño:
- **Asumir la extracción**: tratar el prompt de sistema como potencialmente extraíble. Nunca incrustar claves de API, credenciales ni lógica de negocio sensible.
- **Jerarquía clara**: delimitadores explícitos entre instrucciones del sistema y entrada del usuario.
- **Restricciones de comportamiento**: definir lo que el modelo *no* debe hacer.
- **Formato de salida**: esquemas JSON o respuestas estructuradas.
- **Cadena de razonamiento**: en producción, ocultar o filtrar los pasos de razonamiento al usuario final, conservándolos para depuración.

### 3.2 Taxonomía de jailbreaks

| Técnica | Mecanismo |
|---|---|
| DAN (Do Anything Now) | Pide interpretar una IA sin restricciones |
| Crescendo | Escalada gradual en varios turnos; cada paso parece inocente |
| Skeleton Key (Microsoft) | Convence al modelo de que el contexto está "autorizado" |
| Many-shot | Muchos ejemplos de conducta insegura desplazan la distribución de salida (aprendizaje en contexto) |
| PAIR | Un LLM atacante refina prompts de forma iterativa contra un LLM objetivo |

### 3.3 Sufijos adversarios y contrabando de tokens

- **Sufijos adversarios**: secuencias optimizadas con gradientes, a menudo sin sentido para humanos, que alejan la salida de las respuestas de seguridad. Suelen ser transferibles entre modelos.
- **Contrabando de tokens**: leetspeak, homoglifos Unicode, caracteres de ancho cero o Base64. Eluden filtros de texto que el modelo sí interpreta.
- **Defensa emergente**: filtrado por perplejidad. Los sufijos adversarios suelen ser estadísticamente inusuales.

### 3.4 Arquitectura de guardrails

- **Guardrails de entrada**: detectan intentos de inyección antes de que lleguen al LLM.
- **Guardrails de salida**: detectan fuga de datos, contenido dañino o inyección de código antes de llegar al usuario o a sistemas posteriores.
- Los filtros basados en reglas se eluden con paráfrasis o codificación. Los clasificadores basados en ML detectan intención semántica, pero requieren mantenimiento continuo.

### 3.5 Endurecimiento de prompts y prevención de fugas

- **Prompt hardening**: jerarquía de instrucciones sólida, instrucciones explícitas de rechazo, restricciones de formato y tokens canario que alertan si el prompt se filtra.
- **Prompt leaking**: técnicas como "repite todo lo anterior" o "muestra tus instrucciones". La defensa más robusta es asumir que el prompt será extraído.

---

## Módulo 04: Gobernanza y gestión de riesgos

### 4.1 NIST AI RMF e ISO/IEC 42001

- **NIST AI RMF**: cuatro funciones: **Gobernar, Mapear, Medir y Gestionar**. Es voluntario, pero se cita mucho en contratación pública.
- **ISO/IEC 42001**: estándar para un sistema de gestión de IA (AIMS): establecer, implementar, mantener y mejorar la gestión de riesgos de IA.

### 4.2 EU AI Act y regulación

Niveles de riesgo de la EU AI Act:
- **Inaceptable**: prohibido (scoring social, vigilancia biométrica en tiempo real con excepciones).
- **Alto**: scoring crediticio, contratación, diagnóstico médico, aplicación de la ley. Exige evaluación de conformidad, documentación y supervisión humana. Muchos despliegues empresariales de LLM caen aquí.
- **Limitado**: obligaciones de transparencia con las personas que interactúan con la IA.
- **Mínimo**: sin requisitos específicos.

Otros puntos: el Artículo 22 del GDPR da derecho a explicación de decisiones automatizadas, y los datos regulados que cruzan fronteras en prompts requieren revisión de residencia.

### 4.3 Red teaming para LLM

Va más allá del pentesting: sondea sesgo, cumplimiento de políticas, prompts adversarios, extracción de datos y abuso de herramientas. Debe tener línea base antes del despliegue, repetirse tras cada actualización del modelo, involucrar a perfiles diversos y rastrear la remediación en el registro de riesgos.

### 4.4 Registros de riesgo y model cards

- **Registro de riesgos de IA**: probabilidad, impacto, controles y mitigaciones. Incluye riesgos técnicos, operativos, éticos y reputacionales.
- **Model cards** (Mitchell et al., 2019): capacidades, limitaciones, usos previstos y modos de fallo conocidos.

### 4.5 Shadow AI, uso aceptable y responsabilidad

- **Shadow AI**: uso no autorizado de herramientas de IA, como pegar código o datos de clientes en cuentas personales. Es el equivalente de shadow IT, con mayor riesgo de fuga.
- **Política de uso aceptable**: casos aprobados, actividades prohibidas, manejo de datos y herramientas aprobadas, tanto de consumo como empresariales.
- **Responsabilidad algorítmica**: pistas de auditoría, explicabilidad y responsabilidad humana clara.
- **Divulgación responsable**: notificar al proveedor, dar tiempo razonable de remediación y luego divulgar públicamente.

---

## Módulo 05: Privacidad y tratamiento de datos

### 5.1 Riesgos de datos de entrenamiento

- **Extracción de datos de entrenamiento**: prompts que recuperan contenido memorizado textualmente (PII, claves, código propietario). Es más probable con datos repetidos o con sobreajuste.
- **Inferencia de membresía**: determinar si una muestra formó parte del entrenamiento.
- **Inversión de embeddings**: reconstruir texto desde vectores almacenados, aunque los documentos originales tengan control de acceso.

### 5.2 Regulaciones

- **GDPR**: el derecho al olvido es difícil de aplicar en modelos ya entrenados. Los requisitos de residencia de datos afectan dónde pueden procesarse los prompts y la inferencia.
- **CCPA**: derecho de los residentes de California a saber qué información se recopila y a pedir su eliminación, incluso en logs y bases vectoriales.
- **HIPAA**: enviar datos de salud a una API de LLM sin un Acuerdo de Asociado Comercial (BAA) constituye una violación.

### 5.3 Tecnologías de mejora de la privacidad (PETs)

- **Privacidad diferencial**: ruido calibrado con ε, que controla el trade-off entre privacidad y utilidad. Limita la memorización, pero suele reducir la calidad del modelo.
- **Aprendizaje federado**: cada participante entrena localmente; solo se comparten actualizaciones (gradientes).
- **Cifrado homomórfico**: inferencia sobre datos cifrados. Hoy es computacionalmente prohibitivo para un LLM completo.

### 5.4 Minimización, redacción y consentimiento

- **Minimización**: enviar solo los datos necesarios; reducir el contexto innecesario y el historial retenido.
- **Redacción**: eliminar o enmascarar PII antes de que llegue al modelo. Debe considerarse el riesgo de reidentificación desde el contexto.
- **Consentimiento**: informar si los prompts pueden usarse para entrenar, y ofrecer exclusión (opt-out).
- **Datos sintéticos**: útiles para pruebas y ajuste cuando los datos reales están restringidos.

---

## Módulo 06: Seguridad de MCP (Model Context Protocol)

### 6.1 Arquitectura: clientes, servidores y transportes

- **Cliente MCP**: la aplicación anfitriona (Claude Code, Cursor). Descubre herramientas, las presenta al LLM y ejecuta las llamadas. Es el límite de confianza.
- **Servidor MCP**: expone herramientas, recursos y prompts. Cada servidor suele dar acceso a un sistema concreto.
- **Transportes**:
  - `stdio`: servidores locales, por entrada/salida estándar.
  - `Streamable HTTP`: servidores remotos; requiere TLS y autenticación.
- **Primitivas**: *Tools* (funciones invocables), *Resources* (datos para el contexto) y *Sampling* (el servidor pide completions al LLM a través del cliente).

### 6.2 Envenenamiento de herramientas y rug pull

- **Tool poisoning**: la descripción de la herramienta contiene instrucciones que engañan al modelo (por ejemplo, enviar datos sensibles al servidor).
- **Rug pull**: el servidor cambia el comportamiento de la herramienta *después* de que el usuario la aprobó. Ejemplo: se aprueba "lee datos públicos del clima" y luego la herramienta exfiltra datos.
- **Exfiltración entre servidores**: una herramienta lee datos sensibles y otra los envía fuera; el LLM orquesta el flujo y parece legítimo.
- **Defensa**: verificar en el momento de la ejecución, no solo en la aprobación. Usar descripciones direccionadas por contenido y monitorear el comportamiento.

### 6.3 Autenticación, autorización y capacidades

- **OAuth 2.1** para servidores remotos: rotación de tokens, alcances (scopes) y almacenamiento seguro. Evitar credenciales hardcodeadas.
- **Negociación de capacidades**: en la inicialización se acuerda qué funciones soporta cada parte; si el servidor no necesita *sampling*, debe deshabilitarse.
- **Logging de seguridad**: invocaciones, parámetros, marcas de tiempo, resultados e identidad del solicitante.

### 6.4 Endurecimiento y aislamiento

- **Contenedores**: cada servidor en su propio contenedor, con red mínima. Un servidor comprometido no debe acceder a otros servidores ni al sistema anfitrión.
- **Path traversal**: sandbox de directorios; validar rutas contra `../` y symlinks.
- **Bases de datos**: consultas parametrizadas, credenciales de solo lectura y tablas acotadas. Nunca permitir SQL arbitrario.
- **Inyección de comandos**: validar y sanitizar cada parámetro antes de usarlo en operaciones del sistema.
- **Superficie mínima**: una herramienta que "ejecuta cualquier SQL" es mucho más riesgosa que una que "consulta el estado de un pedido por número".
- **Operaciones sensibles**: correos, pagos o cambios de datos requieren confirmación explícita e idempotencia.

---

## Módulo 07: Agentes de IA, orquestación y vibe coding

### 7.1 Arquitecturas de agentes

- **ReAct**: alterna razonamiento y acción (Pensamiento → Acción → Observación). La transparencia es visible, pero la cadena de razonamiento puede revelar lógica de negocio.
- **Planificación**: el agente genera un plan y lo ejecuta paso a paso. Un paso manipulado se propaga a todas las acciones siguientes.
- **Multi-agente**: un router distribuye tareas. La salida de un agente se convierte en entrada de otro, creando vectores de inyección entre agentes.

### 7.2 Seguridad de agentes: sandboxing, circuit breakers y HITL

- **Sandboxing**: ejecución de código en contenedores con límites de recursos, tiempo y red. Sistema de archivos de solo lectura o acotado a un directorio de trabajo.
- **Circuit breakers**: detienen la ejecución ante comportamientos anómalos (acceso inusual a datos, intentos de escalada, desviación de patrones de salida).
- **Human-in-the-loop (HITL)**: aprobación humana para operaciones irreversibles (eliminar datos, transacciones financieras), comunicaciones externas y cambios de privilegios. No para cada invocación; si fuera así, los agentes serían impracticables.
- **Rate limiting y presupuestos**: máximo de acciones por sesión, límites de tokens y topes monetarios.
- **Memoria del agente**: cifrada en reposo, controlada por usuario o sesión, validada en su integridad y revisada en busca de entradas envenenadas.

### 7.3 Seguridad de la orquestación

- **Validación de límites**: en LangChain o LangGraph, cada transición entre cadenas, herramientas y servicios es un punto de inyección. Validar y sanitizar en cada límite.
- **Seguridad del router**: un enrutador comprometido puede redirigir consultas sensibles a agentes maliciosos.
- **Autorización por herramienta**: validación de entrada y registro de cada ejecución (agente, parámetros, marca de tiempo, resultado).
- **Observabilidad**: trazas completas de razonamiento, llamadas, decisiones y transiciones, almacenadas a prueba de manipulación.

### 7.4 Vibe coding: riesgos y SDLC seguro

El *vibe coding* es aceptar código generado por IA con revisión mínima, confiando en que "parece correcto, compila y pasa los tests".

Riesgos principales:
- **Vulnerabilidades heredadas**: los modelos aprenden patrones buenos y malos del código público (inyección SQL, secretos hardcodeados, deserialización insegura).
- **Errores lógicos sutiles**: el código pasa los tests pero contiene fallas de seguridad.
- **Riesgos de dependencias**: paquetes desactualizados, obsoletos o maliciosos (typosquatting).
- **Falta de modelo de amenazas**: la IA no conoce tus requisitos de seguridad.

**Mitigación**: el código generado pasa las mismas puertas de revisión que el escrito por personas (SAST/DAST en CI/CD), y las rutas críticas exigen revisión humana.

---

## Módulo 08: Seguridad de aplicaciones para productos de IA

Los productos de IA siguen siendo software: las vulnerabilidades web aplican, y la integración de LLM agrega nuevas superficies.

### 8.1 Vulnerabilidades tradicionales en contexto de IA

- **SSRF**: el LLM solicita URLs o recursos internos (endpoints de metadatos en la nube).
- **XSS**: contenido generado que se renderiza sin codificar. Defensa clave: Content-Security-Policy.
- **Inyección SQL**: SQL generado por el LLM sin parametrizar. Aunque se pida SQL seguro, puede manipularse con prompts.
- **Path traversal**: rutas generadas sin validación.

### 8.2 Seguridad de API

- **Autenticación**: OAuth 2.0 con tokens de vida corta y rotación de refresh tokens. Evitar claves de larga duración en clientes.
- **Rate limiting**: por tokens consumidos, conexiones concurrentes y costo, no solo por solicitudes por minuto.
- **Streaming**: el filtrado de contenido debe operar sobre salidas parciales en tiempo real.
- **Errores**: mensajes genéricos al usuario; diagnósticos detallados solo en logs. Nunca exponer stack traces ni configuración.

### 8.3 DevSecOps para IA

- **CI/CD**: además de SAST/DAST, pruebas de inyección de prompts, regresión del comportamiento del modelo y validación de guardrails.
- **Pentesting específico**: pruebas de inyección, jailbreaks, extracción de datos, abuso de herramientas e inyección indirecta vía RAG.
- **Fuzzing**: sobre la API tradicional y sobre los límites de entrada del modelo.
- **Monitoreo**: un pico repentino en el consumo de tokens combinado con salidas inusuales es la señal más clara de incidente.

### 8.4 Contenedores, cadena de suministro y logging

- **Contenedores**: imágenes mínimas, sin privilegios de root, sistema de archivos de solo lectura y límites de recursos. Compartir GPU entre contenedores exige cuidado.
- **Cadena de suministro**: pesos, datasets, embeddings y servidores MCP. Cada componente requiere procedencia, integridad y monitoreo de vulnerabilidades.
- **Logging seguro**: auditar eventos (disparos de guardrails, abuso de herramientas) sin almacenar el contenido completo de prompts sensibles.
- **WebSocket**: TLS, validación de origen, autenticación por conexión e integridad a nivel de mensaje.

---

## Módulo 09: Identidad, acceso, memoria y respuesta a incidentes

### 9.1 RBAC, ABAC y Zero Trust

- **RBAC**: permisos por rol. Adecuado para roles simples.
- **ABAC**: permisos por atributos del usuario, del recurso y del entorno (hora, ubicación, dispositivo). Preferido en plataformas multi-tenant.
- **Zero Trust**: verificar cada solicitud, cada llamada a API, cada herramienta y cada acceso a datos, sin confianza implícita por ubicación de red.
- **Rotación de credenciales**: automatizada, para reducir la ventana de exposición.
- **Tokens de capacidad para agentes**: acceso acotado en alcance y tiempo. "Leer datos de clientes durante 15 minutos" es más seguro que una credencial permanente.

### 9.2 Bases de datos vectoriales y ataques a embeddings

- **Amenazas**: embeddings envenenados que manipulan la recuperación, e inversión de embeddings que reconstruye texto.
- **Controles de acceso**: aplicar permisos por documento en la capa de recuperación, para que el RAG solo devuelva lo que el usuario puede ver. Así se evita la escalada de privilegios a través del pipeline.
- Las bases vectoriales necesitan los mismos controles que los documentos que representan.

### 9.3 Ventana de contexto y persistencia de memoria

- **Envenenamiento de la ventana de contexto**: en conversaciones multi-turno, contenido inyectado en turnos anteriores influye en los siguientes. El modelo no distingue el historial legítimo del inyectado.
- **Defensas**: resumen periódico con verificación de integridad, reinyección de instrucciones de sistema a intervalos y detección de anomalías. Para aplicaciones de alta seguridad, limitar la duración de la conversación.
- **Memoria persistente**: cifrada, controlada por acceso, validada y revisada en busca de entradas envenenadas.

### 9.4 Respuesta a incidentes para sistemas de IA

Además de la respuesta tradicional:
- **Análisis forense del modelo**: qué produjo el modelo y por qué.
- **Análisis de prompts**: qué entradas dispararon el incidente.
- **Efectividad de guardrails**: qué controles fallaron.
- **Impacto downstream**: qué sistemas recibieron salidas comprometidas.
- **Break-glass**: deshabilitación de emergencia, cuarentena de salidas y redirección a sistemas de respaldo, con aprobación documentada y notificación inmediata al liderazgo de seguridad.
- **Línea base de comportamiento**: distribuciones de salida, uso de herramientas, consumo de tokens y tasas de rechazo. Un cambio repentino en la tasa de rechazo puede indicar un jailbreak exitoso.

### Orden de la defensa en profundidad (punta a punta)

1. Seguridad de red: firewall, WAF, protección contra DDoS.
2. Autenticación y autorización: identidad, RBAC/ABAC.
3. Guardrails de entrada: inyección de prompts, filtrado de contenido.
4. Restricciones de comportamiento del modelo: prompt de sistema, entrenamiento de seguridad.
5. Validación de salida: sanitización, formato, filtrado de datos sensibles.
6. Monitoreo y observabilidad: líneas base, detección de anomalías, logging de auditoría.

---

## Idea central del manual

- El LLM no separa instrucciones de datos: la inyección de prompts es el riesgo principal.
- Ningún control aislado basta: se necesita defensa en profundidad con capas independientes.
- Los agentes y las integraciones MCP elevan el impacto, por eso la aprobación humana es necesaria para acciones irreversibles.
- Las herramientas, plugins y dependencias pueden cambiar después de aprobados: verificar en la ejecución.
- La gobernanza que solo prohíbe empuja el uso a la sombra; hay que ofrecer alternativas sancionadas.

---

*Notas de estudio basadas en el manual de Red Team Leaders (Seguridad de LLM, v1.0, 2026). Las referencias externas deben verificarse en sus fuentes originales.*
