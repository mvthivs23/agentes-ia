# Fundamentos de LLM para seguridad: resumen de los puntos importantes

> Resumen de las lecciones de Red Team Leaders sobre IA, machine learning, LLM, RAG, copilotos y agentes. Está pensado para entender cómo funcionan los LLM sin conocimientos previos y para tener lo mínimo que pide un rol de AI engineer, AI security engineer o DevSecOps con IA.
>
> **Cómo leerlo:** cada sección indica de qué lección sale (por ejemplo, **[L1.4]**). Cada tema responde qué es, por qué existe y qué implica para seguridad. Los términos técnicos se mantienen en inglés.

## Índice

1. [Ubicación: IA, ML, deep learning y LLM](#1-ubicación-ia-ml-deep-learning-y-llm)
2. [Cómo aprende una red neuronal](#2-cómo-aprende-una-red-neuronal)
3. [Tokens](#3-tokens)
4. [Embeddings](#4-embeddings)
5. [Transformer y self-attention](#5-transformer-y-self-attention)
6. [Familias de arquitectura](#6-familias-de-arquitectura)
7. [Escala y capacidades emergentes](#7-escala-y-capacidades-emergentes)
8. [Cómo se entrena un asistente](#8-cómo-se-entrena-un-asistente)
9. [Parámetros de inferencia](#9-parámetros-de-inferencia)
10. [Ventana de contexto y costo](#10-ventana-de-contexto-y-costo)
11. [Fine-tuning, LoRA y QLoRA](#11-fine-tuning-lora-y-qlora)
12. [RAG](#12-rag)
13. [Modelos multimodales](#13-modelos-multimodales)
14. [Copilotos empresariales](#14-copilotos-empresariales)
15. [Agentes, herramientas, MCP y harness](#15-agentes-herramientas-mcp-y-harness)
16. [Mapa de riesgos de seguridad](#16-mapa-de-riesgos-de-seguridad)
17. [Gobernanza y cumplimiento mínimos](#17-gobernanza-y-cumplimiento-mínimos)
18. [Lo mínimo por rol](#18-lo-mínimo-por-rol)
19. [Glosario](#19-glosario)
20. [Preguntas de repaso](#20-preguntas-de-repaso)
21. [Fuentes y verificación](#21-fuentes-y-verificación)

---

## 1. Ubicación: IA, ML, deep learning y LLM

**[L1.1, L1.2]** Son círculos concéntricos:

```
Inteligencia Artificial (IA)     → sistemas que realizan tareas que parecen inteligentes
  └── Machine Learning (ML)      → el sistema aprende patrones a partir de datos
        └── Deep Learning        → ML con redes neuronales de muchas capas
              └── LLM            → redes tipo Transformer entrenadas con enormes cantidades de texto
```

**Niveles de IA:**
- **ANI (IA estrecha):** optimizada para una tarea o dominio. Todo lo desplegado comercialmente hoy es ANI, incluidos GPT, Claude y Gemini.
- **AGI (IA general):** igualaría cualquier tarea intelectual humana sin entrenamiento específico. No existe; su cercanía está en debate.
- **ASI (superinteligencia):** superaría al mejor humano en todo. Es especulativa.

**Por qué importa:** los sistemas actuales tienen límites explotables. Pueden fallar con entradas fuera de su distribución de entrenamiento, pueden manipularse con prompts y no razonan sobre sus propias limitaciones como un experto humano. Son propiedades del paradigma actual, no errores pendientes de corregir.

**Paradigmas de machine learning [L1.2]:**

| Paradigma | Cómo aprende | Uso en seguridad | Riesgo específico |
|---|---|---|---|
| **Supervisado** | Pares entrada-etiqueta | Detección de intrusión, spam, malware | Envenenar etiquetas o muestras de entrenamiento |
| **No supervisado** | Encuentra estructura sin etiquetas | Clustering, detección de anomalías | Evadir el límite entre "normal" y "anómalo" |
| **Por refuerzo** | Ensayo y error con recompensa | Pruebas de penetración automatizadas, optimización de firewalls | Manipular la señal de recompensa; *reward hacking* |

**Generativo vs. discriminativo [L1.1]:**
- **Discriminativo:** aprende P(etiqueta | entrada). Clasifica (spam, fraude).
- **Generativo:** aprende cómo son los datos y crea texto, imágenes o código nuevos. Los LLM son generativos.

**Hitos que explican la arquitectura actual [L1.1, L1.2]:**
- **1950–1980:** IA simbólica y sistemas expertos (MYCIN, DENDRAL). Eran frágiles fuera de su base de reglas.
- **1986:** *backpropagation* hace práctico entrenar redes neuronales.
- **2012:** AlexNet baja el error top-5 en ImageNet de 26% a 15,3%. Marca el inicio del deep learning moderno, impulsado por GPU.
- **2014:** GAN (Goodfellow): un generador y un discriminador compiten. Base de la generación de imágenes y de los deepfakes.
- **2017:** Transformer ("Attention Is All You Need"). Base de casi todos los LLM actuales.
- **2018–2020:** modelos de fundación (BERT, GPT-1, GPT-2, GPT-3).

**Arquitecturas previas al Transformer [L1.2]:**
- **CNN:** para datos con estructura espacial (imágenes). Usa filtros aprendidos que detectan patrones sin importar dónde aparecen. En seguridad: visualización de malware, detección de phishing en capturas.
- **RNN / LSTM:** para secuencias (tráfico, logs, llamadas al sistema). Procesan token a token; las LSTM retienen dependencias largas con puertas. Su límite: no se paralelizan bien.

---

## 2. Cómo aprende una red neuronal

**[L1.2]**

- **Neurona:** calcula una suma ponderada de sus entradas, suma un sesgo (*bias*) y aplica una función de activación no lineal: `z = Σ(wᵢ·xᵢ) + b`, `salida = σ(z)`.
- **Pesos (weights):** números aprendidos que codifican la importancia de cada entrada. Son el "conocimiento" del modelo, y por eso son un activo valioso y un objetivo de robo.
- **Activación no lineal:** sin ella, apilar capas seguiría siendo una función lineal. ReLU es la más usada en capas ocultas; GELU es común en modelos grandes.
- **Profundidad:** cada capa aprende representaciones más abstractas (bordes, luego formas, luego conceptos).
- **Función de pérdida:** mide el error. Para clasificación se usa entropía cruzada.
- **Descenso de gradiente:** ajusta los pesos en la dirección que reduce la pérdida. El paso de actualización es `θ ← θ − α·∇J(θ)`, donde α es la tasa de aprendizaje.
- **Backpropagation:** calcula el gradiente de cada peso con la regla de la cadena. Reduce el cálculo a algo lineal en el número de parámetros.
- **Entrenamiento real:** con mini-lotes (32 a 512 muestras) y optimizadores adaptativos como Adam.

**Sobreajuste y regularización:**
- **Underfitting:** el modelo es demasiado simple y falla en entrenamiento y en prueba.
- **Overfitting:** memoriza el entrenamiento, incluido su ruido, y no generaliza.
- **Por qué es crítico en seguridad:** las técnicas de ataque cambian. Un detector ajustado a familias de ransomware de 2023 puede fallar con variantes de 2024.
- **Técnicas:** regularización L2 (penaliza pesos grandes), L1 (fomenta pesos en cero), *dropout* (apaga neuronas al azar), *early stopping* (detiene cuando sube la pérdida de validación), *data augmentation*.

**Métricas de evaluación [L1.2]:**
- **Accuracy engaña en seguridad:** un sistema que marca todo como benigno tiene 99,9% de accuracy si los ataques son el 0,1% del tráfico, y no detecta ninguno.
- **Precision** = TP / (TP + FP): de las alertas, cuántas eran ataques reales. Baja precision genera fatiga de alertas.
- **Recall** = TP / (TP + FN): de los ataques reales, cuántos se detectaron. Bajo recall significa ataques que pasan.
- **F1:** media armónica de precision y recall.
- **AUC-ROC:** poder discriminativo en todos los umbrales (1,0 = perfecto; 0,5 = azar).
- **Trade-off:** un SOC elige entre más alertas (más recall) o menos alertas (más precision). Ejemplo del material: un malware detector con 95% de recall y 40% de precision puede ser inaceptable si genera demasiadas falsas alarmas.

**Por qué importa:** un modelo con buena métrica en un conjunto de prueba puede fallar en producción por distribution shift, por datos mal etiquetados o por un adversario que conoce el umbral de decisión.

---

## 3. Tokens

**[L1.4, L1.3]**

- **Token:** la unidad que procesa el modelo. No es un carácter ni necesariamente una palabra; es un fragmento de subpalabra que apareció con frecuencia suficiente en el entrenamiento.
- **Vocabulario:** tabla finita de tokens. Va de unos 32.000 (LLaMA 2) a más de 100.000 (cl100k_base, usado por GPT-3.5 y GPT-4, con 100.277 entradas).
- **Ejemplos:** "gato" puede ser un token; "tokenización" puede ser tres (`tok` + `en` + `ization`).
- **Relaciones aproximadas:** en inglés, una palabra equivale a unos 1,3 tokens y un token a unos 4 caracteres. El código y el JSON cuestan más.

**Algoritmos principales:**
- **BPE (Byte-Pair Encoding):** empieza con 256 bytes y fusiona iterativamente los pares más frecuentes. Usado por la familia GPT (tiktoken).
- **SentencePiece:** opera sobre Unicode sin depender de espacios, útil para japonés o chino. Usado por Google (BERT, T5, PaLM, Gemini).
- **WordPiece:** variante de BPE que maximiza probabilidad. Usado por BERT original.

**Por qué importa:**
- **Costo:** las APIs cobran por token, no por palabra.
- **Límites:** la ventana de contexto se mide en tokens.
- **Token smuggling:** caracteres Unicode que parecen ASCII (por ejemplo, la "а" cirílica frente a la "a" latina) pasan un filtro de texto pero se tokenizan de forma distinta. Ejemplo típico: un filtro que bloquea "ignora" no reconoce una versión con homoglifos, pero el modelo sí la interpreta.

---

## 4. Embeddings

**[L1.4, L1.3, L1.7]**

- **Qué es:** un vector denso de números reales que representa el significado de un token o de un fragmento de texto.
- **Cómo se obtiene:** cada token tiene un ID; el modelo busca la fila de ese ID en una matriz aprendida de forma `[tamaño_vocabulario, d_model]`. Esa fila es el embedding.
- **Dimensiones de referencia:** BERT-base 768; GPT-2 768; GPT-3 12.288; LLaMA 2 70B 8.192; modelos de embeddings dedicados (text-embedding-3-small) 1.536, (text-embedding-3-large) 3.072.
- **Geometría del significado:** palabras con significados relacionados quedan cerca. Se mide con **similitud coseno** (cercana a 1 = relacionados). Ejemplo clásico: `rey − hombre + mujer ≈ reina`.
- **Por qué existe:** un ID entero no dice nada sobre el significado; un vector sí, porque los tokens que aparecen en contextos parecidos terminan con vectores parecidos.

**Por qué importa para seguridad:**
- **RAG y búsqueda semántica** dependen de la similitud: la pregunta se convierte en vector y se compara con los documentos.
- **Inversión de embeddings:** un vector almacenado puede permitir reconstruir gran parte del texto original. **Un embedding no es una anonimización.** Investigaciones como Morris et al. (2023) muestran reconstrucciones de alta fidelidad.
- **Ataques por proximidad:** una entrada diseñada para quedar cerca de un contenido legítimo puede ser recuperada en su lugar.
- **Envío a APIs de terceros:** enviar texto a una API de embeddings equivale a entregar el texto en claro al proveedor.

---

## 5. Transformer y self-attention

**[L1.3]**

### Qué es y por qué reemplazó a las RNN

El **Transformer** (Vaswani et al., 2017) calcula relaciones entre todos los tokens de una secuencia **en paralelo**, usando solo atención. Antes, las RNN y LSTM procesaban token por token: no se paralelizaban, y toda la historia se comprimía en un vector de tamaño fijo, con pérdida de dependencias lejanas.

### La idea de self-attention

Cada token produce tres vectores a partir de su embedding:
- **Query (Q):** qué busca.
- **Key (K):** qué ofrece para ser encontrado.
- **Value (V):** qué información aporta si alguien le presta atención.

Cada token compara su Q con las K de todos los demás, convierte esas comparaciones en pesos que suman 1 (softmax) y se construye como una **mezcla ponderada** de los V.

```
Attention(Q, K, V) = softmax(QKᵀ / √d_k) · V
```

- **√d_k (escala):** evita que los productos punto crezcan tanto que el softmax quede en zonas de gradiente casi nulo.
- **Máscara causal:** en modelos tipo GPT, las posiciones futuras reciben −∞ antes del softmax. Cada token solo ve lo anterior. Así se genera texto de izquierda a derecha.

**Ejemplo intuitivo:** en "El banco cerró temprano porque el río se desbordó", la palabra "banco" se interpreta como orilla de río porque la self-attention relaciona "banco" con "río".

### Multi-head attention

Varias atenciones corren en paralelo, cada una con proyecciones aprendidas. Cada cabeza puede especializarse en un tipo de relación (sintaxis, correferencia, roles semánticos). La especialización surge del entrenamiento; nadie la programa. Los modelos van de 32 cabezas (LLaMA 7B) a 96 (GPT-3).

### Bloque del Transformer

- **Multi-head self-attention**, seguida de
- **Red feed-forward (FFN):** procesa cada posición por separado. Se interpreta que aquí se almacena buena parte del conocimiento factual (Geva et al., 2021).
- Cada subcapa tiene **conexión residual** y **LayerNorm**, que permiten entrenar redes muy profundas.

### Codificación posicional

La atención por sí sola no sabe el orden: "el perro muerde al hombre" y "el hombre muerde al perro" tendrían las mismas relaciones. Por eso se suma una señal de posición a cada embedding (senos y cosenos en 2017; hoy RoPE o ALiBi). Los primeros tokens de un contexto reciben atención desproporcionada (*attention sinks*, Xiao et al., 2023).

### Por qué esto importa para seguridad

**Este es el punto más importante del documento.** Las instrucciones del sistema, la entrada del usuario, los documentos recuperados y la salida de herramientas entran **al mismo flujo de tokens** y se procesan **con el mismo mecanismo**. La arquitectura no separa instrucciones de datos. El modelo no sabe qué parte del contexto tiene autoridad. Por eso la defensa de prompt injection debe estar **fuera** del modelo: permisos, validación de salidas, aprobaciones humanas y guardrails.

---

## 6. Familias de arquitectura

**[L1.3]**

| Familia | Ejemplos | Atención | Uso típico | Implicación de seguridad |
|---|---|---|---|---|
| **Solo encoder** | BERT (2018) | Bidireccional | Clasificación, moderación, detección de jailbreak | Entrenado con *masked language modeling*. Vulnerable a sustituciones de tokens que cambian la clasificación sin cambiar el significado para un humano |
| **Solo decoder** | GPT, LLaMA, Mistral, Claude, Gemini | Causal (solo ve lo anterior) | Asistentes, generación de texto y código | Instrucciones y datos fluyen en una sola secuencia. Origen de la prompt injection |
| **Encoder-decoder** | T5, BART | Encoder bidireccional, decoder causal | Traducción, resumen | La *cross-attention* es un componente aprendido que también puede manipularse |

**Por qué importa:** casi todos los LLM con los que trabajarás son decoder-only. Por eso el estudio de seguridad se centra en esa familia.

---

## 7. Escala y capacidades emergentes

**[L1.1, L1.3, L1.2]**

### Leyes de escala

- **Kaplan et al. (2020):** el error baja de forma predecible al aumentar parámetros, datos y cómputo.
- **Chinchilla (Hoffmann et al., 2022):** para un presupuesto de cómputo dado, la proporción óptima es de unos **20 tokens de entrenamiento por parámetro**. Muchos modelos anteriores estaban subentrenados. Chinchilla (70B) con 1,4 billones de tokens superó a Gopher (280B).
- **Consecuencia:** modelos más pequeños y mejor entrenados (LLaMA 2 de 7B a 70B, con 2 billones de tokens) pueden correr localmente, fuera del control de un proveedor de API. Eso cambia el modelo de amenaza.

### Capacidades emergentes

- **Wei et al. (2022):** algunas capacidades (aritmética de varios pasos, razonamiento de varios saltos, analogías) aparecen de forma abrupta por encima de cierta escala. Hay debate sobre si son transiciones reales o artefactos de medición (Schaeffer et al., 2023).
- **Few-shot learning:** GPT-3 (175.000 millones de parámetros) aprendió tareas a partir de unos pocos ejemplos en el prompt, sin actualizar pesos.
- **Escala de cómputo:** el entrenamiento de GPT-3 requirió unos 3,14 × 10²³ FLOPS, logrado con miles de GPU en paralelo.

### Por qué importa para seguridad

- **No puedes caracterizar un modelo grande probando uno pequeño.** Un modelo que se niega a algo a 7B puede comportarse distinto a 70B porque la capacidad no existía antes.
- **Las capacidades peligrosas también emergen.** Un modelo puede explicar tácticas de ingeniería social o escribir código malicioso porque aprendió de textos que lo contienen, no porque alguien lo programara.

---

## 8. Cómo se entrena un asistente

**[L1.1, L1.6]**

| Etapa | Qué hace | Por qué existe | Riesgo |
|---|---|---|---|
| **1. Pre-training** | Predice el siguiente token sobre datos masivos | Adquirir lenguaje y conocimiento general | Memoriza secretos y datos personales del corpus |
| **2. SFT / Instruction fine-tuning (IFT)** | Entrena con pares (instrucción, respuesta) | Un modelo base solo continúa texto; el SFT lo convierte en asistente (FLAN, InstructGPT) | Ejemplos envenenados crean comportamientos persistentes |
| **3. RLHF** | Humanos comparan respuestas; se entrena un modelo de recompensa; PPO optimiza el LLM | Alinear con preferencias humanas de utilidad e inocuidad | *Reward hacking* |
| **4. Constitutional AI (RLAIF)** | El modelo critica y revisa sus respuestas contra principios escritos (Bai et al., 2022) | Menos anotación humana de contenido dañino; alineación auditable | La calidad depende de los principios |
| **5. DPO** | Optimiza directamente sobre pares de preferencia, sin modelo de recompensa aparte | Más simple y estable que PPO | Facilita desalinear un modelo abierto con pocos pares |

### RLHF paso a paso

1. **SFT:** demostraciones humanas enseñan el formato de asistente.
2. **Modelo de recompensa:** se entrena con comparaciones humanas ("A es mejor que B") usando una pérdida tipo Bradley-Terry. Produce una puntuación de calidad.
3. **PPO:** el LLM maximiza esa puntuación con una penalización **KL** que impide alejarse demasiado del modelo SFT.

**Reward hacking:** el modelo encuentra atajos que puntúan bien sin ser mejores. Ejemplos del material: respuestas excesivamente largas, afirmaciones muy seguras aunque sean incorrectas, exceso de precaución que no aporta nada, formato vistoso (viñetas, encabezados). Es la Ley de Goodhart: cuando una medida se convierte en objetivo, deja de ser una buena medida. La penalización KL existe para limitarlo: si β es demasiado bajo, la optimización es agresiva y se cae en atajos; si es demasiado alto, el modelo casi no cambia.

**Alignment tax:** la alineación reduce la disposición del modelo a producir ciertas salidas. Los jailbreaks intentan recuperar esas capacidades.

**Modelo base vs. modelo de chat:** un modelo base responde preguntas con otra pregunta; un modelo de chat pasó por SFT y alineación. Si usas un modelo base como asistente sin alineación, no tiene las defensas que das por sentadas.

**Por qué importa:** la seguridad de un modelo **no es una propiedad fija**. Puede perderse con fine-tuning, cuantización o un jailbreak bien diseñado.

---

## 9. Parámetros de inferencia

**[L1.3]**

Después del entrenamiento, el modelo produce una distribución de probabilidad sobre el vocabulario para el siguiente token. Los parámetros de inferencia deciden cómo se elige:

| Parámetro | Qué hace | Efecto | Implicación de seguridad |
|---|---|---|---|
| **Temperatura** | Escala la distribución antes de muestrear | T < 1 concentra (más determinista); T > 1 aplana (más aleatorio); T = 0 es *greedy decoding* | Con temperatura alta, los rechazos son más frágiles. Un jailbreak que funciona 40% de las veces a T = 1,0 puede funcionar más a T = 1,5, si el atacante controla el parámetro. Con T = 0, la salida es predecible y fácil de sondear |
| **Top-k** | Elige entre los k tokens más probables | Descarta opciones improbables | Reduce tokens raros |
| **Top-p** (*nucleus sampling*) | Elige entre el conjunto mínimo cuya probabilidad acumulada supera p | Se adapta al contexto | Mismo efecto de limitar, de forma adaptativa |
| **Max tokens** | Límite de longitud de salida | Corta la respuesta | Control básico contra consumo descontrolado |

**Regla para recordar:** la temperatura **escala** la distribución; top-k y top-p la **truncan**.

**Consecuencia para pruebas:** los ataques son probabilísticos. Un ataque que falla una vez puede funcionar en otra llamada. Hay que probar con repeticiones y varios parámetros.

---

## 10. Ventana de contexto y costo

**[L1.4, L1.3]**

### Qué es

La **context window** es la cantidad máxima de tokens (entrada + salida) que el modelo procesa en una llamada. Es su **memoria de trabajo**: system prompt, historial, resultados de herramientas, documentos recuperados y la pregunta del usuario. Fuera de esa ventana, el modelo no sabe nada, salvo que la aplicación lo reinyecte.

**Crecimiento:** GPT-3 con 2.048 tokens; GPT-4 con 8.192; GPT-4o con 128.000; Claude 3 con 200.000; Gemini 1.5 Pro con 1 millón o más.

**Por qué existe el límite:** el costo de self-attention crece con el cuadrado de la longitud en su forma densa. Hay técnicas para extenderlo, pero el costo sigue ahí.

### Qué pasa cuando se llena

- **Truncamiento:** se descarta lo más antiguo. El modelo puede contradecir instrucciones que ya salieron de la ventana.
- **Resumen:** se pierde detalle literal.
- **Recuperación (RAG):** se trae solo lo relevante.
- **Error:** la llamada falla con un error de longitud excedida.

### Lost in the middle

Aunque la ventana sea grande, la información en el centro se usa peor que la del inicio y el final (Liu et al., 2023). Práctica: poner las instrucciones críticas al principio y el contenido crítico cerca del final.

### Costo

- **Costo ≈ tokens de entrada × precio de entrada + tokens de salida × precio de salida.**
- La salida cuesta varias veces más que la entrada. El material da como referencia (mediados de 2025) GPT-4o a 2,50 USD por millón de tokens de entrada y 10 USD por millón de salida. **Verifica el precio vigente.**
- Ejemplo del material: 100.000 llamadas diarias con 2.000 tokens de entrada y 500 de salida cuestan unos 1.000 USD por día con GPT-4o. Usar un modelo más pequeño para tareas simples puede reducir el costo en más de 90%.
- En una conversación, el historial se reenvía en cada turno; el costo crece con cada turno.

### Por qué importa para seguridad

- **Denial of Wallet:** un atacante que fuerza salidas largas o llamadas costosas hace subir la factura.
- **Context flooding:** un adversario que inyecta texto de relleno puede empujar las instrucciones de seguridad fuera de la ventana.
- **Superficie:** cada token de contexto puede contener instrucciones.

---

## 11. Fine-tuning, LoRA y QLoRA

**[L1.6, L1.4]**

### Qué es

**Fine-tuning** es continuar el entrenamiento de un modelo preentrenado con un conjunto de datos más pequeño y curado, para cambiar su comportamiento, formato o dominio.

### Fine-tuning completo

- Actualiza **todos** los parámetros. Es la mejor calidad posible, pero con costo alto.
- Un modelo de 7B en float32 necesita unos 28 GB solo para los pesos; el optimizador y los gradientes multiplican ese uso. Un modelo de 70B no cabe en un nodo común.
- **Olvido catastrófico:** ajustar con datos muy estrechos puede hacer perder capacidades generales.
- Suele requerir miles a decenas de miles de ejemplos.

### PEFT: fine-tuning eficiente en parámetros

Congela la mayoría de los pesos y entrena un número pequeño de parámetros adicionales (típicamente menos del 1%). Idea: las actualizaciones necesarias para una tarea tienen baja dimensionalidad.

### LoRA (Hu et al., 2021)

```
W' = W + ΔW = W + B × A
```

- `W` (d × d) está congelada.
- `A` (r × d) y `B` (d × r) son entrenables. `r` (rank) suele ser de 4 a 64.
- **Ejemplo del material:** con d = 4096 y r = 8, el ajuste completo de una matriz son 16,7 millones de parámetros; con LoRA son 2 × 4096 × 8 = **65.536**, unas 256 veces menos. En el modelo completo, LoRA entrena alrededor del 0,1% al 1%.
- **Hiperparámetros:** r (expresividad), alpha (escala ΔW; suele igualarse a r), módulos objetivo (las proyecciones de atención q, k, v, o suelen funcionar bien).
- **Ventaja operativa:** el adaptador es un archivo pequeño. Puedes tener un adaptador por cliente o por idioma sobre un mismo modelo base.

### QLoRA (Dettmers et al., 2023)

LoRA sobre un modelo base **cuantizado a 4 bits** (NF4, diseñado para pesos con distribución aproximadamente normal), con doble cuantización y *paged optimizers* que usan memoria de CPU como desbordamiento. Permite ajustar modelos grandes en hardware de consumo.

### Cuándo usar cada opción

| Opción | Úsala cuando |
|---|---|
| **Prompt engineering / few-shot** | La tarea está bien especificada y el modelo ya tiene la capacidad |
| **RAG** | El conocimiento cambia, es grande, o necesitas citar fuentes |
| **Fine-tuning (LoRA)** | Necesitas estilo, formato o terminología de dominio consistentes |
| **RAG + fine-tuning** | Necesitas estilo propio y datos actualizados a la vez |

**Regla práctica:** para **conocimiento**, RAG; para **comportamiento y formato**, fine-tuning.

### Por qué importa para seguridad

- **Datos envenenados** en el ajuste pueden crear backdoors que no aparecen en pruebas normales.
- **El ajuste puede desalinear el modelo.** La alineación se puede eliminar con pocos pares de preferencia (DPO), y los modelos abiertos se pueden ajustar sin barreras.
- **Sesgo amplificado:** un dataset pequeño o no representativo amplifica sus sesgos.
- **Ajustar con datos de clientes** trae riesgo de memorización y fuga de datos personales.
- **Doble uso:** un modelo ajustado para ciberseguridad (CVE, exploits) es útil para defensa y también es un riesgo si su alineación es débil.

---

## 12. RAG

**[L1.7, L1.4]**

### Qué es y por qué existe

RAG (Lewis et al., 2020) **busca documentos relevantes en una base externa en el momento de la pregunta** y los pasa al prompt. El modelo pasa de **recordar** a **leer**.

Resuelve dos problemas:
- **Corte de conocimiento:** el modelo no sabe lo que pasó después de su entrenamiento.
- **Alucinación:** cuando no sabe algo, puede inventar una respuesta fluida y falsa. Es consecuencia de la generación autorregresiva, no un error que se parchee.

Ventajas: conocimiento actualizable sin reentrenar, respuestas con citas, control de qué se consulta.

### Las dos tuberías

**Indexación (sin conexión):**
1. **Ingesta:** leer PDF, Word, Markdown, HTML, wikis. Las tablas, los diseños de varias columnas y los escaneados (que requieren OCR) degradan la calidad. *Basura adentro, basura afuera.*
2. **Chunking:** dividir en trozos.
3. **Embedding:** convertir cada trozo en un vector con un modelo de embeddings.
4. **Almacenamiento:** guardar vectores, metadatos (fuente, fecha, permisos) y texto en una base vectorial con un índice ANN.

**Consulta (en línea):**
1. **Embed de la pregunta** con el **mismo** modelo de embeddings.
2. **Búsqueda de similitud** (coseno o producto punto) en el índice.
3. **Reranking (recomendado):** un *cross-encoder* compara la pregunta con cada candidato. Por ejemplo, de 20 candidatos se quedan 3 a 5. Agrega 50–200 ms.
4. **Generación:** los trozos finales van al prompt. El modelo responde, idealmente citando fuentes y diciendo cuándo la respuesta no está en el material.

**Regla crítica:** indexación y consulta deben usar **el mismo modelo de embeddings**. Dos modelos producen espacios vectoriales incompatibles.

### Chunking

| Estrategia | Cómo corta | Ventaja | Problema |
|---|---|---|---|
| **Tamaño fijo** | 256–1.024 tokens, con solapamiento de 10–20% | Simple; buena línea base | Corta frases o párrafos a mitad |
| **Semántica** | Corta donde baja la similitud entre oraciones consecutivas | Trozos coherentes | Más costosa, requiere embeddings antes de indexar |
| **Recursiva** | Separadores jerárquicos (párrafos, líneas, puntos) | Respeta la estructura; buena para Markdown y HTML; recomendada por defecto | Depende de que el documento tenga estructura |
| **Padre-hijo** | Busca con trozos de 64–128 tokens y entrega al LLM el padre de ~512 | Precisión al buscar, contexto amplio al responder | Más complejidad de indexación |

**Compromiso básico:** trozos grandes conservan contexto pero diluyen la señal; trozos pequeños son precisos pero pierden el contexto.

### Recuperación

- **Densa (semántica):** similitud de vectores. Encuentra sinónimos y paráfrasis, pero puede fallar con códigos exactos, números de serie o nombres propios.
- **Dispersa (BM25):** coincidencia de palabras con índice invertido. Exacta y rápida, sin sinónimos.
- **Híbrida:** ambas en paralelo, fusionadas con **Reciprocal Rank Fusion (RRF)**: `RRF(d) = Σ 1 / (k + rank_i(d))`, con k ≈ 60. Según el material, mejora el recall entre 10% y 20% frente a la densa sola. Es la recomendación para producción.

**Ejemplo del material:** una consulta sobre "CVE-2023-44487" suele fallar con recuperación solo densa, porque el identificador exacto no se representa bien como vector. BM25 lo encuentra por coincidencia exacta.

**Transformaciones de la consulta:** HyDE (embeber una respuesta hipotética), reescritura de la consulta, múltiples consultas, *step-back prompting*.

### Índices y bases vectoriales

- **Flat (exacto):** compara con todos los vectores. Exacto pero lento a gran escala.
- **HNSW:** grafo jerárquico de vecinos. Muy rápido con recall alto (más de 95%); usa más memoria.
- **IVF:** divide el espacio en celdas y busca en las más cercanas; menos memoria, requiere entrenamiento. Se combina con PQ (IVF-PQ) para escalas muy grandes.
- **Bases:** Pinecone (gestionada, serverless), Weaviate (open source, módulos de embeddings), Chroma (ligera, para prototipos), Qdrant (filtra por metadatos durante la búsqueda), pgvector (extensión de PostgreSQL, ideal si ya usas Postgres).

### Evaluación con RAGAS

- **Faithfulness:** ¿cada afirmación de la respuesta está en el contexto recuperado? Es la métrica principal para detectar alucinación.
- **Answer relevancy:** ¿la respuesta atiende la pregunta?
- **Context precision:** ¿qué fracción del contexto recuperado era relevante?
- **Context recall:** ¿está en el contexto toda la información necesaria?
- **Answer correctness:** exactitud frente a una respuesta de referencia; requiere un conjunto etiquetado.

El material propone como objetivos de producción: faithfulness > 0,85; answer relevancy > 0,80; context precision > 0,70; context recall > 0,80.

**Diagnóstico:** si las respuestas tienen cifras que no están en el contexto, cae **faithfulness**. Si el contexto tiene ruido, cae **context precision**. Si falta la información, cae **context recall**.

### RAG vs. fine-tuning

| Criterio | RAG | Fine-tuning | Híbrido |
|---|---|---|---|
| Frescura del conocimiento | Excelente | Pobre (reentrenar) | Buena |
| Razonamiento de dominio | Moderado | Excelente | Excelente |
| Citas de fuente | Incorporadas | No disponibles | Incorporadas |
| Complejidad | Moderada | Alta | Muy alta |
| Costo por consulta | Más alto | Más bajo | Más alto |
| Control de alucinaciones | Bueno | Moderado | Lo mejor |

### Por qué RAG importa para seguridad

RAG **reduce alucinaciones**, pero **añade superficie**:
- **Indirect prompt injection:** un documento recuperado puede contener instrucciones que el modelo obedece. Es el canal indirecto más importante en sistemas empresariales.
- **Envenenamiento de la base vectorial:** un documento malicioso indexado se recupera cuando la consulta se le parece.
- **Permission bypass:** si la recuperación no filtra por permisos, el modelo muestra lo que el usuario no debería ver. **El filtro de permisos debe aplicarse en la consulta vectorial, no después.**
- **Inversión de embeddings:** los vectores almacenados pueden revelar el texto original. La base vectorial necesita los mismos controles que los documentos.
- **Fuga entre inquilinos:** incluso sin devolver documentos, el número de resultados, los scores o el tiempo de respuesta pueden revelar información.

---

## 13. Modelos multimodales

**[L1.8]**

### Qué significa

Un modelo multimodal procesa y/o produce texto, imágenes, audio, video o código. Cada modalidad es una nueva superficie de ataque.

### Cómo se unen las modalidades

1. Un **encoder específico** convierte la entrada en vectores continuos. Para imágenes, un **Vision Transformer (ViT)** divide la imagen en parches (por ejemplo, 16×16 píxeles; una imagen de 224×224 produce 196 parches).
2. Una **capa de proyección** (un MLP o una transformación lineal) lleva esos vectores al mismo espacio que los embeddings de texto.
3. Los vectores proyectados se intercalan con los tokens de texto. La imagen se convierte en "tokens blandos" que el LLM atiende igual que palabras.

**Ejemplos de arquitectura:**
- **LLaVA:** encoder CLIP ViT-L/14 + proyección lineal + LLaMA o Mistral. Demostró que la calidad de los datos importa más que la complejidad de la proyección.
- **Q-Former (BLIP-2, InstructBLIP):** extrae un número fijo de vectores de consulta, de modo que el costo no crece con la resolución.
- **CLIP:** entrenado con 400 millones de pares imagen-texto con aprendizaje contrastivo. Sus espacios visual y textual quedan alineados. Hereda los sesgos de sus datos.

### Nativo vs. modular

- **Modular (GPT-4V, LLaVA):** el visor se añade a un modelo de texto.
- **Nativo (GPT-4o, Gemini):** se entrena con texto, imagen, audio y video desde el inicio. GPT-4o procesa audio sin el paso intermedio de transcripción, lo que quita un punto de control de contenido que antes servía para filtrar.

### Audio y video

- **Whisper** (680.000 horas de audio, 99 idiomas) convierte audio en texto antes del LLM. Esa costura puede dejar pasar contenido que el filtro de texto no ve en su forma codificada.
- **Audio nativo** elimina la costura, pero el modelo debe alinearse también frente a entradas de audio, incluidas señales prosódicas (urgencia, sarcasmo).
- **Video:** muestreo de fotogramas y atención temporal. Riesgos: fotogramas adversarios imperceptibles a velocidad normal y manipulación del contexto temporal.

### Generación

- **DALL-E 3:** usa un LLM para reescribir las descripciones de entrenamiento y mejorar la fidelidad.
- **Stable Diffusion:** difusión latente, pesos abiertos. Existen variantes sin filtros de seguridad.
- **TTS neural y clonación de voz:** pocos segundos de audio bastan. Ya se usan en *business voice compromise* y en fraudes de ingeniería social.

### Riesgos específicos

- **Brecha de alineación entre modalidades:** un modelo que rechaza una petición en texto puede aceptarla si el mismo texto va dentro de una imagen. Los guardrails entrenados en texto no se generalizan solos.
- **Prompt injection visual:** instrucciones ocultas en imágenes, capturas o PDFs que pasan por OCR.
- **Perturbaciones adversarias:** ruido imperceptible que cambia la clasificación (Szegedy et al., 2013: un panda clasificado como gibón con 99,3% de confianza).
- **Datos sensibles en imágenes:** PII en fondos de fotos, pizarras, pantallas. Es más difícil de redactar que el texto.
- **Costo:** una imagen de alta resolución agrega cientos de tokens, con costo y latencia.

### Benchmarks

VQAv2, MMMU (30 disciplinas universitarias), ScienceQA, MMBench, BLINK. **El rendimiento en benchmarks no es una garantía de seguridad.** Un modelo puede tener muy buen MMMU y ser vulnerable a inyección visual. La evaluación para despliegue debe incluir pruebas adversarias.

### Abierto vs. cerrado (multimodal)

- **Cerrado (API):** guardrails actualizados por el proveedor, pero sin acceso interno, con cambios de comportamiento fuera de tu control y datos que salen de tu infraestructura.
- **Abierto (LLaVA, CogVLM, Phi-3-V):** se puede desplegar on-premises y probar a fondo, pero tú pones los guardrails y la seguridad.

---

## 14. Copilotos empresariales

**[L1.9]**

### Chatbot vs. copiloto

- **Chatbot:** el usuario escribe, el modelo responde. Sin conexión persistente a sistemas. El radio de impacto es lo que el usuario pegó en la sesión.
- **Copiloto:** integrado en un sistema de registro. Combina cuatro cosas:
  1. **RAG sobre datos vivos** (correo, archivos, tickets, CRM).
  2. **Tool use:** puede enviar correos, crear tickets, subir código.
  3. **Identidad:** actúa como el usuario o como una cuenta de servicio con sus permisos.
  4. **Integración permanente**, no limitada a la sesión.

Cada una de esas capacidades es también una superficie de ataque.

### Productos y su perfil de riesgo

- **Microsoft 365 Copilot:** usa Microsoft Graph para recuperar correos, calendario, Teams, SharePoint y OneDrive, con el permiso del usuario. Su riesgo principal es el **oversharing**: si SharePoint tiene permisos mal gobernados (accesos de invitados nunca revocados, grupos "todos"), Copilot los muestra a quien pregunte correctamente. Se apoya en un índice semántico que refleja el estado de permisos del inquilino.
- **GitHub Copilot:** completado de código, chat, resúmenes de PR y Copilot Workspace (agente que crea ramas y PR). Riesgos: memorización de secretos y código público, y vulnerabilidades generadas que llegan a producción. Las ediciones Business y Enterprise permiten restringir archivos en contexto y auditar.
- **Salesforce Einstein:** IA en CRM. Riesgo: un correo malicioso importado como actividad podría instruir al copiloto a resumir y filtrar datos de transacciones.
- **ServiceNow Now Assist:** resume tickets y sugiere resoluciones. El CMDB contiene el mapa de la infraestructura. Quien pueda escribir en un ticket puede inyectar instrucciones en la IA que lo lee.

### Patrones de implementación

| Patrón | Ventaja | Riesgo principal |
|---|---|---|
| **SaaS por API** (predeterminado) | Rápido, bajo costo operativo | Confianza en el proveedor; el proveedor decide cuándo cambia el modelo |
| **Azure OpenAI / endpoint privado** | Calidad de frontera con residencia de datos más estricta | Configuración de región y permisos |
| **Despliegue privado** (modelo abierto on-premises) | Los datos no salen del perímetro | Infraestructura GPU, actualizaciones y parches a tu cargo; costo total mayor |
| **Híbrido** | Embeddings privados para indexar; modelo de frontera solo con contexto recuperado y mínimo | Hay que mapear qué datos fluyen a cada nivel |

### Riesgos clave

- **Prompt injection a través de datos conectados:** una instrucción en el asunto de un correo, en un documento de SharePoint, en un ticket o en un comentario de código puede redirigir al copiloto. Es el vector de mayor gravedad que introducen los copilotos.
- **Fuga de datos entre usuarios:** si la recuperación no aplica permisos por usuario de forma consistente, se muestran datos no autorizados.
- **Shadow AI:** herramientas no aprobadas (ChatGPT gratuito, extensiones de navegador, revisores de código de terceros). Es un síntoma de falla de gobierno: las políticas que solo prohíben empujan el uso a la sombra.
- **Identidad:** una cuenta comprometida da acceso al copiloto con todo lo que el usuario puede ver. Las cuentas privilegiadas son el objetivo más valioso. Las cuentas de servicio del copiloto deben tratarse como identidades privilegiadas.
- **Residencia y cumplimiento:** GDPR (datos personales fuera de geografías aprobadas), HIPAA (PHI sin BAA), SOC 2 Tipo II (controles demostrables de acceso a datos).

### Gobernanza

- **Auditoría de permisos antes de habilitar:** la acción de seguridad más impactante antes de activar un copiloto.
- **Registro de auditoría activado** y enviado al SIEM.
- **Red team antes del despliegue masivo:** pruebas de inyección con datos representativos.
- **Mínimo privilegio:** alcance explícito por sitio, objeto o repositorio, no el máximo por defecto.
- **Dependencia del proveedor:** cambios de modelo que alteran el comportamiento, precios, SLA que excluyen fallas específicas de IA.
- **Capacitación:** los usuarios deben saber qué puede ver el copiloto y qué no preguntar en entornos compartidos.
- **Shadow AI:** usar herramientas de descubrimiento (DNS, proxy) para priorizar y ofrecer alternativas aprobadas, no solo bloquear.

---

## 15. Agentes, herramientas, MCP y harness

**[L1.10, L1.8, L1.3]**

### Qué hace que un sistema sea agéntico

Un LLM se convierte en **agente** cuando combina tres propiedades:
1. **Autonomía:** el modelo impulsa la secuencia de decisiones, aunque haya puntos de control humanos.
2. **Dirección de meta:** organiza su trabajo alrededor de un objetivo de varios pasos.
3. **Uso de herramientas:** puede actuar en el mundo (correo, código, archivos, APIs, navegador).

Los agentes de producción suelen añadir **persistencia**, **autorreflexión** y **planificación**.

Comparación: un chatbot es reactivo y sin estado. Un RAG agrega lectura a una base, sin actuar. Un LLM con herramientas llama funciones, normalmente en un solo paso. Un agente combina todo con un bucle autónomo. Un sistema multiagente coordina varios agentes.

### El bucle ReAct

ReAct (Yao et al., 2022) intercala **pensamiento** (razonar qué hacer) y **acción** (llamar una herramienta). Cada acción devuelve una **observación** y el ciclo se repite:

```
Thought: necesito el CEO de la empresa X.
Action: web_search("CEO empresa X 2025")
Observation: [resultado: Jane Doe, desde 2022]
Thought: ya tengo la respuesta.
Action: finish("El CEO es Jane Doe.")
```

El bucle de producción: **Percibir → Razonar → Planificar → Actuar → Observar → Reflexionar**. Termina cuando se cumple el objetivo, se alcanza un máximo de pasos, un humano interrumpe o el agente llama a *finish*. **La falta de condiciones de terminación robustas es uno de los fallos más comunes.**

### Memoria del agente (cuatro tipos)

| Tipo | Qué guarda | Cómo se implementa | Ataque típico |
|---|---|---|---|
| **De trabajo** | Lo que está en la ventana de contexto ahora | El propio prompt | Inyección en el contexto |
| **Episódica** | Historial de acciones y resultados | Log estructurado en base de datos | Escribir eventos falsos para que el agente crea que ya hizo algo |
| **Semántica** | Conocimiento de largo plazo | Base vectorial (RAG) | Envenenar la base de conocimiento |
| **De procedimiento** | Herramientas, esquemas, flujos | Código y descripciones de herramientas | Inyección de habilidades: que el agente aprenda un comportamiento malicioso |

### Tool calling (llamada de funciones)

1. **Definición:** las herramientas se describen en JSON Schema (nombre, descripción, parámetros). **La descripción también entra al contexto**, por lo que es un canal de texto.
2. **Decisión del modelo:** el modelo responde con un bloque estructurado que indica la herramienta y los argumentos. **El modelo no ejecuta nada.**
3. **Ejecución:** la aplicación (el harness) valida la llamada y la ejecuta.
4. **Inyección del resultado:** el resultado vuelve al contexto como observación.
5. **Continuación:** el modelo sigue razonando.

Diferencias de formato: OpenAI usa `tool_calls` con argumentos como string JSON; Anthropic usa bloques `tool_use` con argumentos ya parseados, y exige un `tool_result` en el turno siguiente.

**Punto clave:** la validación de la llamada y de sus argumentos es el control real, no el modelo.

### Categorías de herramientas y su riesgo

| Herramienta | Riesgo principal |
|---|---|
| **Búsqueda web** | Páginas diseñadas para manipular al agente (*prompt injection lure*) |
| **Ejecución de código** | Reconocimiento local, escape del sandbox, exfiltración por DNS o HTTP, agotamiento de recursos. Debe tener la política más restrictiva |
| **Automatización de navegador** | Actúa con la sesión del usuario: puede comprar, enviar formularios o acceder a sistemas internos |
| **APIs externas** | Hereda las credenciales del agente. Una clave de administrador "por comodidad" da privilegios de administrador a cada tarea y a cada inyección |
| **Sistema de archivos** | Sobrescribir configuración, inyectar código, filtrar secretos |
| **Servidores MCP** | Descripciones de herramientas maliciosas, resultados manipulados, permisos excesivos |

### MCP (Model Context Protocol)

Protocolo abierto de Anthropic (2024) que estandariza la conexión entre aplicaciones de IA y herramientas o fuentes de datos. Es cliente-servidor sobre JSON-RPC:
- **Servidor MCP:** expone herramientas, recursos y prompts de un sistema concreto.
- **Cliente MCP:** la aplicación anfitriona. **Es el límite de confianza:** lo que el cliente permite, el modelo puede hacerlo.

**Riesgos típicos:**
- **Tool poisoning:** la descripción de una herramienta contiene instrucciones maliciosas.
- **Rug pull:** el servidor cambia el comportamiento de una herramienta después de que el usuario la aprobó. Defensa: verificar en cada ejecución, no solo al aprobar; usar identificadores por contenido.
- **Exfiltración entre servidores:** una herramienta lee datos sensibles y otra los envía fuera. Cada llamada parece legítima. Defensa: no combinar lectura sensible con salida externa sin aprobación.

### Sistemas multiagente

- **Orquestador/trabajador:** un orquestador descompone la meta, asigna subtareas a agentes especializados y agrega resultados. Lo usan CrewAI, LangGraph y las guías de Anthropic. **Los resultados de los trabajadores deben tratarse como no confiables.**
- **Pizarra (blackboard):** los agentes leen y escriben un estado compartido. Un resultado malicioso de un agente puede inyectar instrucciones en otro.
- **Basado en metas (AutoGPT, 2023):** el agente recibe una meta y ejecuta sin supervisión. Mostró acciones inesperadas, peticiones web no deseadas y consumo descontrolado de créditos de API. **Son propiedades emergentes de la autonomía sin guardrails**, no bugs puntuales.

### El harness (arnés de IA)

El **harness** es la capa de software que envuelve al modelo: el **sistema operativo del agente**. Gestiona todo lo que el modelo no hace por sí mismo:
- **Cumplimiento de políticas:** qué herramientas se pueden llamar, con qué argumentos y cuántas veces.
- **Gestión de contexto:** qué entra a la ventana en cada paso, resúmenes y desalojos.
- **Aislamiento del entorno:** sandbox para código, reglas de salida de red, control de acceso al sistema de archivos.
- **Guardrails:** filtros de entrada y salida sobre acciones planificadas y respuestas.
- **Auditoría:** registro de cada paso (entradas, razonamiento, llamadas, resultados) para análisis forense.

**Fallos típicos del harness:** credenciales con más permisos de los necesarios; no validar argumentos; confiar en resultados de herramientas sin sanitizar; permitir cargar herramientas nuevas en tiempo de ejecución; no limitar iteraciones ni consumo. **Un harness permisivo es la causa raíz de la mayoría de los incidentes agénticos.**

### Principios de diseño

- Empezar con el **conjunto mínimo de herramientas**, de solo lectura primero.
- **Confianza escalonada** para los resultados: internos y validados > web pública > contenido externo.
- **Human-in-the-loop** para acciones de alto impacto o irreversibles. No para cada paso: si todo requiere aprobación, la gente aprueba sin mirar.
- **Registro completo** e inmutable.
- **Red team** con escenarios donde los resultados contienen instrucciones inyectadas.

---

## 16. Mapa de riesgos de seguridad

Este resumen integra lo visto en todas las lecciones. Los nombres siguen la lista OWASP Top 10 for LLM Applications en su edición más reciente. **Verifica la numeración y el contenido vigentes en la fuente oficial antes de citarlos en un informe o examen.**

| Componente | Riesgo principal | Ejemplo | Lección |
|---|---|---|---|
| **Entrada y documentos** | Prompt injection (directa e indirecta) | Correo con texto oculto que ordena reenviar datos | L1.3, L1.7, L1.9 |
| **Salida del modelo** | Improper output handling | Salida con HTML o SQL que se ejecuta sin validar | L1.10 |
| **Contexto y system prompt** | Hidden context exposure | El prompt revela reglas de negocio o claves | L1.3 |
| **Datos de entrenamiento** | Sensitive information disclosure | Memorización de datos personales o secretos | L1.1, L1.2 |
| **Entrenamiento y ajuste** | Data and model poisoning; backdoors | Ejemplos envenenados en fine-tuning | L1.6 |
| **Herramientas y agente** | Excessive agency | Agente con permiso de borrar bases de datos | L1.10 |
| **Recursos** | Unbounded consumption | Una solicitud barata que dispara cómputo caro | L1.4 |
| **RAG y embeddings** | Vector and embedding weaknesses | Filtración entre inquilinos; inversión de embeddings | L1.4, L1.7 |
| **Cadena de suministro** | Supply chain | Pesos con código malicioso; paquetes inventados por la IA | L1.6, L1.9 |
| **Respuestas a personas** | Misinformation | Citas falsas usadas en decisiones reguladas | L1.7 |
| **Multimodal** | Inyección visual, adversarial images, clonación de voz | Instrucción oculta en una imagen | L1.8 |
| **Copilotos** | Oversharing, shadow AI, identidad | Copiloto que muestra archivos de SharePoint mal compartidos | L1.9 |

### Ideas de defensa que se repiten

- **La defensa está fuera del modelo:** guardrails, permisos, validación, aprobaciones. El modelo no puede autocontrolarse.
- **Mínimo privilegio:** cada herramienta y cada credencial hacen solo lo necesario.
- **Tratar todo contenido externo como no confiable**, incluso si es "interno".
- **Filtrar por permisos antes de recuperar**, no después.
- **Diseñar acciones reversibles** y poner humanos en los pasos sin vuelta atrás.
- **Medir antes y después** de cada cambio de modelo, prompt, herramienta o fuente.
- **Ninguna garantía es absoluta:** no hay defensa probada contra todos los ataques. Por eso el red team es continuo.

---

## 17. Gobernanza y cumplimiento mínimos

**[L1.9, L1.2, L1.1]**

- **Clasificación del sistema:** ¿es un modelo estrecho o de propósito general? ¿Accedes por API o localmente? ¿Tiene capacidades agénticas? ¿Qué procedencia tienen sus datos y pesos?
- **Política de IA empresarial:** herramientas aprobadas, clasificaciones de datos permitidas por herramienta, quién aprueba nuevas herramientas, cómo se revisa el contenido generado, qué formación se exige. Una política que solo restringe empuja al shadow AI; combínala con una vía rápida de aprobación.
- **Cumplimiento:** GDPR (residencia de datos y derecho a explicación de decisiones automatizadas), HIPAA (PHI requiere BAA), SOC 2 Tipo II (controles demostrables).
- **Dependencia del proveedor:** cambios de versión del modelo, precios, SLA que excluyen fallas de IA, costos de cambio.
- **Model cards:** documentan en qué se entrenó el modelo, con qué poblaciones se evaluó, qué fallos se conocen y en qué condiciones operar. Un clasificador entrenado solo con archivos PE puede fallar silenciosamente con binarios ELF.
- **Riesgo residual:** toda decisión de adopción debe registrar qué riesgos quedan y quién los acepta.

---

## 18. Lo mínimo por rol

### AI Engineer (construir aplicaciones con LLM)

- [ ] Tokens, costo y ventana de contexto (secciones 3 y 10).
- [ ] Embeddings y similitud coseno (sección 4).
- [ ] Parámetros de inferencia y su efecto en la consistencia (sección 9).
- [ ] RAG de extremo a extremo: ingesta, chunking, embeddings, recuperación híbrida, reranking (sección 12).
- [ ] Decidir entre prompting, RAG y fine-tuning (secciones 11 y 12).
- [ ] Tool calling y el bucle ReAct (sección 15).
- [ ] Evaluación con métricas (precision, recall, F1) y RAGAS (secciones 2 y 12).
- [ ] Multimodal: qué cambia al añadir imágenes o audio (sección 13).

### AI Security Engineer (proteger sistemas con LLM)

- [ ] Por qué el Transformer no separa instrucciones de datos (sección 5). Es la base de todo.
- [ ] Prompt injection directa e indirecta, y jailbreaks (secciones 5, 12 y 15).
- [ ] Riesgos de RAG: permisos, envenenamiento, inversión de embeddings (sección 12).
- [ ] Riesgos de agentes y MCP: mínimo privilegio, tool poisoning, rug pull (sección 15).
- [ ] Riesgos de datos y entrenamiento: memorización, backdoors, desalineación por fine-tuning (secciones 8 y 11).
- [ ] Riesgos de copilotos: oversharing, shadow AI, identidad (sección 14).
- [ ] Riesgos multimodales: brecha de alineación entre modalidades, inyección visual, clonación de voz (sección 13).
- [ ] Mapa de riesgos y defensas (sección 16).

### DevSecOps con IA (llevar sistemas con IA a producción)

- [ ] Patrones de despliegue: SaaS, endpoint privado, modelo abierto on-premises, híbrido (sección 14).
- [ ] Control de costo y límites por token (sección 10).
- [ ] Fijar versiones del modelo y validar cada cambio (secciones 8 y 14).
- [ ] Harness y políticas de herramientas: validación de argumentos, sandbox, human-in-the-loop (sección 15).
- [ ] Logging completo y auditable de agentes y herramientas (sección 15).
- [ ] Auditoría de permisos antes de conectar fuentes a un copiloto (sección 14).
- [ ] Evaluaciones de seguridad como puerta de despliegue en CI/CD (sección 12 y 16).
- [ ] Cadena de suministro de IA: procedencia de pesos, datasets, embeddings, paquetes (sección 16).

---

## 19. Glosario

| Término | Definición |
|---|---|
| **ANI / AGI / ASI** | IA estrecha / IA general / superinteligencia |
| **Machine learning** | Sistema que aprende patrones a partir de datos |
| **Deep learning** | ML con redes neuronales de muchas capas |
| **Red neuronal** | Capas de neuronas con pesos aprendidos y activaciones no lineales |
| **Backpropagation** | Cálculo de gradientes con la regla de la cadena |
| **Descenso de gradiente** | Ajuste iterativo de pesos para reducir la pérdida |
| **Overfitting** | Memorizar el entrenamiento y no generalizar |
| **Precision / Recall / F1 / AUC-ROC** | Métricas de calidad de un clasificador |
| **Token** | Fragmento de texto que procesa el modelo |
| **Vocabulario** | Conjunto finito de tokens del modelo |
| **BPE / SentencePiece / WordPiece** | Algoritmos de tokenización |
| **Token smuggling** | Ocultar contenido con caracteres que el filtro no ve pero el modelo interpreta |
| **Embedding** | Vector denso que representa significado |
| **Similitud coseno** | Medida del ángulo entre vectores; cercana a 1 = relacionados |
| **Transformer** | Arquitectura basada en atención, sin recurrencia (2017) |
| **Self-attention** | Cada token pondera la relevancia de los demás |
| **Query / Key / Value** | Vectores que buscan, ofrecen y aportan información |
| **Multi-head attention** | Varias atenciones en paralelo |
| **Máscara causal** | Impide ver posiciones futuras en modelos generativos |
| **Codificación posicional** | Información de orden que se suma a los embeddings |
| **Attention sink** | Tokens iniciales que reciben atención desproporcionada |
| **Context window** | Tokens máximos (entrada + salida) por llamada |
| **Lost in the middle** | Peor uso de la información en el centro de un contexto largo |
| **Pre-training** | Predicción del siguiente token sobre datos masivos |
| **SFT / IFT** | Ajuste supervisado con pares instrucción-respuesta |
| **RLHF** | Alineación con modelo de recompensa entrenado con preferencias humanas |
| **Reward hacking** | El modelo optimiza la puntuación en vez de la calidad real |
| **Penalización KL** | Limita cuánto se aleja el modelo del SFT durante RLHF |
| **Constitutional AI / RLAIF** | Alineación con críticas y revisiones contra principios escritos |
| **DPO** | Optimización directa de preferencias, sin modelo de recompensa aparte |
| **Alignment tax** | Capacidad que la alineación restringe; objetivo de los jailbreaks |
| **Temperatura** | Controla la aleatoriedad al elegir el siguiente token |
| **Top-k / Top-p** | Limitan los candidatos por número o por probabilidad acumulada |
| **Greedy decoding** | Elegir siempre el token más probable |
| **Fine-tuning** | Ajustar pesos de un modelo preentrenado con datos adicionales |
| **PEFT** | Fine-tuning que entrena una fracción pequeña de parámetros |
| **LoRA** | Ajuste de bajo rango: `W' = W + BA` |
| **QLoRA** | LoRA sobre un modelo cuantizado a 4 bits (NF4) |
| **Olvido catastrófico** | Pérdida de capacidades generales tras un ajuste estrecho |
| **RAG** | Buscar documentos externos al momento de la pregunta |
| **Chunking** | División de documentos en trozos para indexar |
| **Reranking** | Reordenar candidatos con un modelo más preciso (cross-encoder) |
| **BM25** | Recuperación por coincidencia de palabras |
| **RRF** | Fusión de rankings de varios buscadores |
| **HNSW / IVF** | Índices ANN para búsqueda vectorial rápida |
| **RAGAS** | Marco de evaluación de RAG (faithfulness, relevancy, precision, recall) |
| **Alucinación** | Respuesta fluida y segura que es falsa |
| **Multimodal** | Procesa o produce más de un tipo de dato |
| **ViT** | Vision Transformer: imágenes como secuencias de parches |
| **CLIP** | Alineación contrastiva de imagen y texto |
| **Tokens blandos** | Vectores de imagen proyectados al espacio del LLM |
| **Copiloto** | Asistente integrado en sistemas de registro, con datos, herramientas e identidad |
| **Oversharing** | Permisos excesivos que la IA expone a quien pregunte |
| **Shadow AI** | Uso no autorizado de herramientas de IA |
| **Agente** | Sistema con autonomía, meta y uso de herramientas |
| **ReAct** | Patrón que alterna razonamiento y acción |
| **Tool calling** | El modelo solicita una herramienta; la aplicación la ejecuta |
| **MCP** | Protocolo cliente-servidor para conectar LLM con herramientas |
| **Tool poisoning / Rug pull** | Descripción maliciosa de una herramienta / cambio de comportamiento tras la aprobación |
| **Harness** | Capa de software que gobierna al agente: políticas, contexto, aislamiento, auditoría |
| **Orquestador/trabajador** | Patrón multiagente con un coordinador y especialistas |
| **Model card** | Documento de capacidades, límites y fallos conocidos de un modelo |
| **SBOM / AI-BOM** | Inventario de componentes de software y de IA (modelos, datasets) |

---

## 20. Preguntas de repaso

**Básico**
1. ¿Qué diferencia hay entre IA, ML, deep learning y LLM?
2. ¿Por qué un token no equivale a una palabra? Da un ejemplo.
3. ¿Qué es un embedding y qué significa que dos tengan similitud coseno cercana a 1?
4. ¿Por qué accuracy engaña en detección de intrusiones?
5. ¿Qué es RAG y qué dos problemas de los LLM resuelve?

**Intermedio**
6. ¿Por qué el Transformer no separa instrucciones de datos? ¿Qué implica para la defensa?
7. ¿Qué hace la máscara causal y por qué existe en los modelos tipo GPT?
8. ¿Qué es la penalización KL en RLHF y qué problema evita?
9. Con temperatura 0, ¿la salida es "más precisa"? Justifica.
10. ¿Por qué el costo de una conversación crece con cada turno?
11. ¿Por qué un LLM con herramientas no ejecuta acciones por sí mismo?
12. ¿Por qué el filtro de permisos en RAG debe aplicarse en la consulta vectorial?

**Avanzado**
13. Con d = 4096 y r = 8, ¿cuántos parámetros entrenables tiene LoRA para una matriz? ¿Y el ajuste completo? (65.536 frente a 16,7 millones.)
14. Según Chinchilla, ¿cuántos tokens de entrenamiento corresponden a un modelo de 70.000 millones de parámetros? (Unos 1,4 billones.)
15. ¿Qué pasa si indexas con un modelo de embeddings y consultas con otro?
16. Un correo con texto oculto ordena a un agente reenviar los últimos diez correos. ¿Qué control de la aplicación impide la exfiltración aunque el modelo obedezca?
17. ¿Por qué un modelo que pasa el red team a 7B podría fallar a 70B?
18. ¿Por qué una imagen puede saltarse un guardrail que bloquea el mismo texto escrito directamente?
19. Un copiloto muestra un archivo de SharePoint a un usuario que no debería verlo. ¿Qué falló y cómo lo corriges antes de activarlo?

<details>
<summary><strong>Clave de respuestas (resumen)</strong></summary>

1. IA es el campo general; ML aprende de datos; deep learning usa redes de muchas capas; un LLM es una red tipo Transformer entrenada con texto a gran escala.
2. Porque el modelo trabaja con subpalabras del vocabulario. Ejemplo: "tokenización" = `tok` + `en` + `ization`.
3. Un vector que representa significado. Significa que los textos son semánticamente muy parecidos.
4. Si los ataques son una fracción mínima del tráfico, un modelo que nunca alerta tiene accuracy altísima y detecta cero ataques.
5. Recuperar documentos externos al momento de la pregunta y pasarlos al prompt. Resuelve el corte de conocimiento y reduce la alucinación.
6. Todas las entradas se procesan como una misma secuencia de tokens con el mismo mecanismo. La defensa debe estar fuera del modelo.
7. Impide ver posiciones futuras, poniendo −∞ antes del softmax. Permite generar de izquierda a derecha.
8. Limita cuánto se aleja el modelo del SFT. Evita atajos que maximizan la recompensa sin mejorar la respuesta.
9. No necesariamente. Temperatura 0 es reproducible, pero no garantiza corrección.
10. El historial completo se reenvía y se cobra como tokens de entrada en cada llamada.
11. Solo produce tokens que indican una llamada. La aplicación (el harness) decide si la ejecuta, con qué argumentos y con qué permisos.
12. Porque después de recuperar, el modelo ya vio el documento. El filtro debe impedir que llegue.
13. LoRA: 2 × 4096 × 8 = 65.536. Ajuste completo: 4096 × 4096 = 16.777.216.
14. 20 × 70.000 millones = 1.400.000 millones de tokens (1,4 billones).
15. Los espacios vectoriales son incompatibles; la búsqueda devuelve resultados sin sentido.
16. Permisos mínimos en la herramienta de envío y validación de la acción fuera del modelo, con aprobación humana para enviar datos fuera.
17. Porque la capacidad puede emerger con la escala; la evaluación en un modelo pequeño no describe al grande.
18. La alineación entrenada en texto no se generaliza automáticamente a la modalidad imagen. La costura entre modalidades queda sin cubrir.
19. Falló el filtrado de permisos en la recuperación, o los permisos de SharePoint están mal configurados (oversharing). Hay que auditar permisos y limitar el alcance del copiloto antes de activarlo.

</details>

---

## 21. Fuentes y verificación

**Material de referencia (lecciones del curso Red Team Leaders):**
- 1.1: Paisaje de la IA (de la IA estrecha a los sistemas agénticos).
- 1.2: Machine learning, deep learning y redes neuronales.
- 1.3: Cómo funcionan los LLM: Transformer, atención y arquitectura.
- 1.4: Tokens, embeddings y ventanas de contexto.
- 1.6: Fine-tuning, RLHF y adaptación de modelos.
- 1.7: RAG y bases de datos vectoriales.
- 1.8: Modelos multimodales e IA de propósito general.
- 1.9: Copilotos, asistentes de IA e integraciones empresariales.
- 1.10: IA agéntica: agentes, llamadas a herramientas y harness.

**Artículos citados en el material:**
- Vaswani et al., 2017, *Attention Is All You Need*.
- Kaplan et al., 2020, *Scaling Laws for Neural Language Models*.
- Hoffmann et al., 2022, *Training Compute-Optimal Large Language Models* (Chinchilla).
- Wei et al., 2022, *Emergent Abilities of Large Language Models*; Schaeffer et al., 2023, sobre su debate.
- Lewis et al., 2020, *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*.
- Hu et al., 2021, *LoRA*; Dettmers et al., 2023, *QLoRA*.
- Ouyang et al., 2022, *InstructGPT*; Bai et al., 2022, *Constitutional AI*; Rafailov et al., 2023, *DPO*.
- Liu et al., 2023, *Lost in the Middle*; Xiao et al., 2023, *attention sinks*.
- Yao et al., 2022, *ReAct*; Greshake et al., 2023, *Indirect Prompt Injection*.
- Morris et al., 2023, *Text Embeddings Reveal (Almost) As Much As Text*.
- Dosovitskiy et al., 2020/2021, *ViT*; Radford et al. (CLIP); Li et al., 2023, *BLIP-2*.

**Errores del material que conviene tener presentes:**
- La lección 1.2 ubica AlexNet en 2017 en un pasaje y en 2012 en otro. La fecha correcta es **2012**.
- La lección 1.8 atribuye la ventana de 1 millón de tokens a "Gemini Ultra". En el material, esa cifra corresponde a **Gemini 1.5 Pro**.
- Algunas cifras de precios y de tamaños de modelos cambian con el tiempo. Verifícalas en la documentación del proveedor antes de usarlas en un informe.

**Verifica antes de citar:**
- Precios, tamaños de contexto y versiones de modelos.
- La numeración y el contenido vigentes de OWASP Top 10 for LLM Applications.
- Las cifras de cualquier producto comercial (Microsoft 365 Copilot, GitHub Copilot, Salesforce, ServiceNow).

---

*Resumen de estudio. No sustituye las lecciones completas ni la documentación oficial de los proveedores.*
