# Agente TPA — Nueva arquitectura V10 y V11

## 1. Qué cambia 

Los agentes V8/V9 eran una sola llamada LLM monolítica con un prompt gigante. Los nuevos V10/V11 son **pipelines orquestados**: una secuencia de sub-agentes pequeños y especializados, cada uno con una única responsabilidad (diagnóstico → tesis → informe → auto-auditoría), apoyados en **skills deterministas** para cálculos y lookups y en una capa **Thesis-RAG** que fundamenta la narrativa con el marco teórico TPA. Cada paso se **persiste**, lo que permite reanudar, auditar y regenerar de forma selectiva.

```
V9  (monolito)  ──►  V10 (orquestado, informe único)
V8  (monolito)  ──►  V11 (orquestado, informe dual: técnico + equipo)
```

---

## 2. Qué ganamos

| Capacidad                   | Antes (V8 / V9)                          | Ahora (V10 / V11)                                                       |
| --------------------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| **Control de calidad**      | Ninguno — lo que devuelva el LLM         | Sub-agente QA audita cada informe; hasta 2 regeneraciones automáticas   |
| **Fiabilidad**              | Fallo → reintento total, coste total     | Reanudar-en-fallo por paso; solo se reejecuta el paso roto              |
| **Coste / eficiencia tokens** | Cada reintento = prompt completo de nuevo | Salidas cacheadas por paso; solo regenera secciones marcadas por QA   |
| **Auditabilidad**           | Caja negra                               | Tabla `pipeline_steps`: input, output, latencia y modelo de cada paso   |
| **Precisión / grounding**   | Solo prompt, propenso a drift            | Thesis-RAG recupera fragmentos del marco TPA; citas en la auditoría     |
| **Determinismo**            | El LLM hacía cálculos                    | Las skills (código Go) calculan scores/lookups; el LLM solo narra       |
| **Iteración de prompts**    | Editar un archivo gigante, desplegar a ciegas | Prompts por paso; cambias uno sin tocar el resto                   |
| **Exportación PDF**         | Captura manual                           | PDF nativo (V10: único; V11: técnico / equipo / ambos)                  |
| **Versionado**              | Difícil de A/B                           | V8, V9, V10 y V11 conviven; los usuarios cambian de ruta                |

**Impacto de negocio:**
- Menos incidencias del tipo "el informe sale raro" — el gate de QA las detecta antes de que llegue al usuario.
- Menor factura de OpenAI — los pasos que fallan no queman todo el pipeline otra vez.
- Experimentación de prompts más rápida — el Prompt Lab edita un sub-agente cada vez.
- Listo para cumplimiento — cada informe tiene una traza de auditoría completa.

---

## 3. V10 vs V11 — qué produce cada uno

| Versión | Sustituye a | Salida                                                       | Caso de uso                                |
| ------- | ----------- | ------------------------------------------------------------ | ------------------------------------------ |
| **V10** | V9          | **Un** informe narrativo integrado                            | Lectura ejecutiva, consumo rápido          |
| **V11** | V8          | **Dos** informes: **Técnico** (profundidad) + **Equipo** (orientado a acción) | Sesión de coach + debrief de equipo en una sola ejecución |

Ambos usan **la misma** arquitectura nueva por debajo — V11 simplemente ejecuta la etapa "informe + QA" dos veces (una por audiencia) con bucles de regeneración independientes.

---


## 4. Modelo C4 — Nueva arquitectura

**Contexto:** quién usa el sistema y con qué se integra.

```mermaid
C4Context
    Person(coach, "Coach / Team Lead", "Lanza sesiones y lee informes")
    System(tpa, "Plataforma TPA", "Informes generados por IA (V10/V11)")
    System_Ext(openai, "OpenAI API", "LLM")
    System_Ext(pg, "PostgreSQL", "Datos y auditoría")
    Rel(coach, tpa, "Usa")
    Rel(tpa, openai, "Llama sub-agentes")
    Rel(tpa, pg, "Persiste")
```

**Contenedores:** las piezas internas de la plataforma.

```mermaid
C4Container
    Person(coach, "Coach / TL")
    Container(spa, "Frontend SPA", "Vue 3", "Vistas + descarga PDF")
    Container(api, "API", "Go (Gin)", "REST /v2/agent")
    Container(orch, "Orquestador", "Go", "Ejecuta pipeline + skills + RAG + PDF")
    SystemDb_Ext(pg, "PostgreSQL")
    System_Ext(openai, "OpenAI API")
    Rel(coach, spa, "HTTPS")
    Rel(spa, api, "REST")
    Rel(api, orch, "Ejecuta V10/V11")
    Rel(orch, openai, "Sub-agentes")
    Rel(orch, pg, "Pasos + informes")
```

**Flujo del pipeline:** qué hace cada ejecución.

```mermaid
flowchart LR
    A[Respuestas] --> B[Skills + Diagnóstico + Tesis RAG]
    B -- V10 --> R1[Informe] --> Q1{QA}
    Q1 -- falla ≤2 --> R1
    Q1 -- pasa --> P[(Persistir + PDF)]
    B -- V11 --> R2[Informe técnico] --> Q2{QA}
    Q2 -- falla ≤2 --> R2
    Q2 -- pasa --> P
    B -- V11 --> R3[Informe equipo] --> Q3{QA}
    Q3 -- falla ≤2 --> R3
    Q3 -- pasa --> P
```

---

## 5. Notas de riesgo y coste

- **Presupuesto de tokens:** una ejecución V11 ≈ 7 llamadas LLM frente a 1 en V8. Lo compensamos con (a) prompts más pequeños por paso, (b) pasos cacheados en los reintentos, y (c) regeneración solo del informe que falla, no de todo el pipeline. El coste neto es comparable; **el coste del fallo cae bruscamente**.
- **Latencia:** V11 es secuencial por diseño (cada paso necesita el anterior). El tiempo total ronda 1,5–2× la pared de V8, pero el usuario ve una sola UI de progreso; en una iteración futura podemos paralelizar diagnóstico y skills.
- **Seguridad de rollout:** las rutas V8/V9 permanecen vivas e intactas. V10/V11 son **aditivas**. Riesgo de migración para clientes existentes: cero.
- **Operabilidad:** cada informe trae una huella de auditoría (pasos, modelos, tokens, latencia, fuentes RAG) visible en la UI — soporte puede diagnosticar incidencias en segundos.

---

## 6. Siguientes pasos sugeridos

1. A/B entre V8 y V11 sobre una muestra de sesiones recientes (puntuaciones QA + feedback del coach).
2. Paralelizar pasos independientes (diagnóstico ‖ skills) para recortar ~25% de la latencia de V11.
3. Exponer el Prompt Lab a los admins TPA para experimentación de prompts por paso sin necesidad de dev.
4. Añadir dashboards de coste basados en `pipeline_steps.tokens_in/out` por versión.

