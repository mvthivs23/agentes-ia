# Objetivo 19 — Guardrails for LLM Agents

Esta sección cubre la implementación de guardrails para agents LLM en entornos de seguridad ofensiva. Estos mecanismos imponen límites éticos y operativos, evitan el abuso, reducen el riesgo y aseguran que los agents de IA operen dentro de parámetros aceptables. Los temas incluyen filtrado de prompts, restricción de la salida, command gating y control de ejecución basado en políticas.

## 6.7.1 Qué son los guardrails

Los guardrails son políticas, reglas o lógica pensadas para:

- Evitar el mal uso de los LLM (p. ej., generar ransomware)
- Exigir compliance (p. ej., ninguna operación contra objetivos no autorizados)
- Filtrar entradas o salidas dañinas (p. ej., autolesión, extremismo)
- Controlar el uso de tools (p. ej., restringir wget, nmap y bash a alcances predefinidos)

Los guardrails operan en tres niveles:

- Entrada (prompt)
- Procesamiento (política de ejecución)
- Salida (filtro de postprocesado)

## 6.7.2 Filtros de prompt

El filtrado de prompts asegura que las instrucciones que llegan a un LLM sean seguras, estén validadas y autorizadas.

**Técnicas:**

- Coincidencia por regex / listas de palabras clave:
  - Detectar intentos de inyección (ignore, override, bypass, sudo)
- Clasificación de intención (con un modelo ligero o con reglas):
  - Clasificar el prompt como "exploit-gen", "info", "dangerous", etc.
- Casos de uso en lista blanca:
  - Permitir solo ciertos patrones o plantillas de prompt

```python
if "format" in prompt or "self-delete" in prompt:
    raise Exception("Unsafe prompt content detected.")
```

## 6.7.3 Filtros de salida

Filtra lo que el modelo puede devolver:

- Bloquea rutas que contengan `rm -rf`, `shutdown`, `kill -9` o código ofuscado
- Quita o redacta comandos si superan un umbral de riesgo
- Limita la generación de archivos (p. ej., solo `.txt`, `.ps1`, `.py`)

Ejemplo de enmascarado de la salida:

```python
def sanitize_output(text):
    banned = ["format", "dd", "shutdown", "fork bomb"]
    for word in banned:
        if word in text:
            return "[Filtered Output]"
    return text
```

## 6.7.4 Command gating y capas de política

Introduce capas de ejecución basadas en política para cualquier comando que el agent intente correr.

Política de ejemplo:

- Tools permitidos: `curl`, `ping`, `nmap`, `python3`
- Tools no permitidos: `bash`, `sh`, `del`, `wget`

Implementación:

```json
{
  "tool": "curl",
  "policy": "allowed",
  "require_human_review": false
}
```

Para comandos o scripts de alto riesgo:

- Pásalos por verificación human-in-the-loop (HITL)
- Exige 2FA o una aprobación firmada para ejecutar

## 6.7.5 Firewalls éticos

Un firewall ético es un componente lógico pensado para:

- Impedir el uso del agent en escenarios black hat
- Limitar el apuntamiento a activos no autorizados
- Detectar mal uso de patrones a lo largo del tiempo (p. ej., escaneos excesivos, despliegue de C2)

**Buenas prácticas:**

- Exige “Red Team Simulation Mode” solo sobre el alcance permitido
- Mantén logs de auditoría de entradas, salidas y acciones
- Despliega los agents detrás de VPN o sandboxes segmentados

## 6.7.6 Integración con frameworks de guardrails de código abierto

### GuardrailsAI

- Valida prompts y salidas mediante especificaciones XML/YAML
- Puede envolverse alrededor de LangChain o de completions de OpenAI

Especificación de guardrails de ejemplo:

```yaml
output:
  type: string
  regex: "^curl.*"
```

### Rebuff / Giskard / ReAct Guardrails

- Tools adicionales para auditoría, pruebas adversariales o para incrustar la política en la memoria del agent

## 6.7.7 Registro y monitoreo del comportamiento del agent

Conviene:

- Guardar logs de prompt y respuesta con marcas de tiempo
- Vigilar la frecuencia de prompts de alto riesgo
- Alertar ante anomalías (p. ej., comandos de fuerza bruta, intentos de bypass)

Usa soluciones de logging centralizado (p. ej., Elastic, Splunk) para agregar.

## 6.7.8 Resumen

Los guardrails, los filtros de prompt y los firewalls éticos son imprescindibles para mantener la seguridad, la legalidad y la integridad en entornos de red team impulsados por IA. Estas restricciones no limitan la creatividad: imponen responsabilidad. Todo despliegue de IA ofensiva debe incluir políticas embebidas que impidan el uso dañino o no autorizado, incluso cuando los agents son muy autónomos.
