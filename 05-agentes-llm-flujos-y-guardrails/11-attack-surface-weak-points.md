# Objetivo 11 — Attack Surface Weak Points

**Objetivo:** identificar puntos débiles en la superficie de ataque de una empresa objetivo.

1. **[Paso 1] Agent de usuario:** «Empieza un reconocimiento completo de acme.com».
2. **[Paso 2]** El agent de Subfinder se ejecuta y registra los resultados.
3. **[Paso 3]** El agent de HTTPX enriquece con metadatos.
4. **[Paso 4]** El agent analizador de tecnología marca WordPress desactualizado.
5. **[Paso 5]** El agent analista resume el riesgo y sugiere una cadena de exploit.
6. **[Paso 6]** El planificador de red team recomienda un escenario de phishing a partir de filtraciones de correo del objetivo.

## 4.6.8 Consideraciones de OPSEC y ética

- Usa un entorno anonimizado (proxychains, VPNs).
- Guarda los logs en almacenamiento local cifrado.
- Limita la recogida de datos a objetivos autorizados o disponibles de forma pública.
- No permitas que agents autónomos ejecuten exploits o acciones sin un humano en el bucle.

## 4.6.9 Resumen

| Idea |
| --- |
| Los agents LLM pueden replicar y automatizar flujos de OSINT completos |
| LangChain destaca en pipelines estructurados y cadenas de tools |
| AutoGen es potente para el encadenamiento al estilo diálogo |
| CrewAI es ideal para repartir tareas entre agents que colaboran |
| La integración con tools de código abierto permite operaciones offline completas |
