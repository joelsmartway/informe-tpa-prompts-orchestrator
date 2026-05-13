# Agente TPA — Nueva arquitectura V10 y V11

**Audiencia:** Product Manager
**Resumen:** Hemos reconstruido el agente de informes del TPA sobre un pipeline modular y auditable. **V10** es la versión en la nueva arquitectura del antiguo informe único **V9**. **V11** es la versión en la nueva arquitectura del antiguo informe dual **V8**. Mismo output de negocio, mejora drástica en ingeniería y operación.

---

## 1. Qué cambia en un párrafo

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

### Nivel 1 — Contexto del sistema

```mermaid
C4Context
    title Agente TPA — Contexto del sistema

    Person(coach, "Coach TPA / Team Lead", "Lanza sesiones y revisa los informes")
    Person(admin, "Admin SmartWay", "Ajusta prompts y monitoriza calidad")

    System(tpa, "Plataforma TPA", "Evaluación de desempeño de equipos con informes generados por IA (V10/V11)")

    System_Ext(openai, "OpenAI API", "Proveedor LLM para los sub-agentes")
    System_Ext(pg, "PostgreSQL", "Sesiones, respuestas, informes y traza de auditoría")

    Rel(coach, tpa, "Lanza cuestionarios, lee informes, descarga PDF")
    Rel(admin, tpa, "Edita prompts, revisa la auditoría pipeline_steps")
    Rel(tpa, openai, "Llamadas de sub-agente (JSON-mode)")
    Rel(tpa, pg, "Persiste pasos, informes y chunks RAG")
```

### Nivel 2 — Contenedores

```mermaid
C4Container
    title Agente TPA — Contenedores

    Person(coach, "Coach / TL")

    Container_Boundary(tpa, "Plataforma TPA") {
        Container(spa, "Frontend SPA", "Vue 3 + TS", "Vistas de resultados V8/V9/V10/V11 y descarga PDF")
        Container(api, "API", "Go (Gin)", "REST API, rutas /v2/agent/{v8,v9,v10,v11}")
        Container(orch, "Orquestador del agente", "Paquete Go", "Ejecuta el grafo del pipeline y persiste los pasos")
        Container(skills, "Registro de skills", "Paquete Go", "Cálculos deterministas (TPI, Lencioni, Tuckman, Role, Systemic)")
        Container(rag, "Thesis RAG", "Go + Postgres", "Recupera fragmentos del marco TPA por tema")
        Container(pdf, "Renderizador PDF", "Go (gofpdf)", "Renderiza los informes V10/V11 a PDF con marca")
    }

    System_Ext(openai, "OpenAI API")
    SystemDb_Ext(pg, "PostgreSQL")

    Rel(coach, spa, "HTTPS")
    Rel(spa, api, "REST + Blob (PDF)")
    Rel(api, orch, "Ejecuta pipeline V10/V11")
    Rel(orch, skills, "Invoca skill")
    Rel(orch, rag, "Recupera chunks de grounding")
    Rel(orch, openai, "Llamada LLM al sub-agente")
    Rel(orch, pg, "pipeline_steps + agent_results")
    Rel(api, pdf, "Genera PDF (variante)")
```

### Nivel 3 — Componentes (orquestador)

```mermaid
C4Component
    title Orquestador del agente — Componentes (V10 único / V11 dual)

    Container_Boundary(orch, "Orquestador del agente") {
        Component(ctrl, "Controlador V10/V11", "Go", "Punto de entrada; cablea sub-agentes, overrides de modelo y registries")
        Component(graph, "Pipeline Runner", "Go", "Ejecuta el grafo de pasos; reanudar-en-fallo; cachea por paso")
        Component(diag, "Sub-agente: Diagnóstico", "LLM JSON", "Extrae findings de los scores brutos")
        Component(thesis, "Sub-agente: Tesis", "LLM JSON", "Construye una hipótesis fundamentada usando hits del RAG")
        Component(interp, "Sub-agente: Interpretación", "LLM JSON", "Solo V11 — puente entre diagnóstico y narrativa")
        Component(repTech, "Sub-agente: Informe técnico", "LLM markdown", "V10: informe completo. V11: informe de profundidad técnica")
        Component(repTeam, "Sub-agente: Informe de equipo", "LLM markdown", "Solo V11 — informe orientado a acción")
        Component(qaTech, "Sub-agente: QA técnico", "LLM JSON", "Puntúa el informe, marca issues y dispara regen")
        Component(qaTeam, "Sub-agente: QA equipo", "LLM JSON", "Solo V11 — QA independiente para la variante de equipo")
        Component(proj, "Proyector de estado", "Go", "Mapea las salidas de pasos a AgentReportResponse{V10,V11}")
    }

    ComponentDb(steps, "pipeline_steps", "Postgres", "Auditoría por paso (input, output, modelo, tokens, latencia)")
    ComponentDb(results, "agent_results", "Postgres", "Informes finales por versión (v8/v9/v10/v11)")

    Rel(ctrl, graph, "Run(pipeline_def)")
    Rel(graph, diag, "1. diagnóstico")
    Rel(graph, thesis, "2. tesis")
    Rel(graph, interp, "3. interpretación (V11)")
    Rel(graph, repTech, "4. informe técnico")
    Rel(graph, qaTech, "5. QA técnico — bucle ≤2 si falla")
    Rel(graph, repTeam, "6. informe equipo (V11)")
    Rel(graph, qaTeam, "7. QA equipo — bucle ≤2 si falla (V11)")
    Rel(graph, steps, "Persiste cada paso")
    Rel(graph, proj, "Proyecta el estado final")
    Rel(proj, results, "Guarda el informe")
```

### Nivel 4 — Flujo del pipeline

```mermaid
flowchart LR
    A[Respuestas de sesión] --> B[Skills: TPI / Lencioni / Tuckman / Role / Systemic]
    B --> C[Diagnóstico]
    C --> D[Tesis + Hits RAG]
    D -- V11 --> E[Interpretación]
    D -- V10 --> F1[Informe]
    E --> F1[Informe técnico]
    F1 --> G1{QA}
    G1 -- falla ≤2 --> F1
    G1 -- pasa --> H1[(Persistir)]
    E --> F2[Informe de equipo]
    F2 --> G2{QA}
    G2 -- falla ≤2 --> F2
    G2 -- pasa --> H2[(Persistir)]
    H1 --> I[Renderizador PDF]
    H2 --> I
    I --> J[Descarga frontend]
```

---

## 5. Notas de riesgo y coste para el PM

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
