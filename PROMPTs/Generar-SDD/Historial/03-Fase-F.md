# Fase F P2 y P3 corregidos; sin referencias colgadas a la categoría 10.

---

Fase F cerrada. APROBADO, observaciones corregidas. Me detengo por la detención entre fases.

Fase F — completada y auditada
Entregable	Archivos	Contenido
09-Devops	6	Pipeline CI/CD (10 stages, ejecuta la DoD de 08 como gates), estrategia de versionado (SemVer + Conventional Commits + MinVer), entornos DEV/PROD, guía de publicación de imagen Docker, supply-chain (SBOM + cosign), README
10-Developer-Guide	—	Omitida (registrada en ADR-23); onboarding se consolida en el README raíz de la Fase H
05 · nuevos ADR	2	ADR-23 (omisión de 10) + ADR-24 (ambientes DEV/PROD sin staging); índice a 24 ADR
Audit Fase F	1	APROBADO (0 P0, 0 P1)
Estado del SDD

✓ A · ✓ B · ✓ B2 · ✓ C · ✓ D · ✓ E · ✓ F
· Fase G  (11-Examples)  ← siguiente
· Fase H  (README raíz + handoff a codificación)
La Fase G genera 11-Examples. Para este proyecto, el intake §18 declara un único sample —samples/ingesta-gmao/, que ilustra el contrato completo de la API de ingesta (los cuatro caminos 201/200/409/422)—; el panel Blazor no produce sample porque se demuestra ejecutando el propio servicio. Para web-monolith el mínimo del framework son 2 samples, así que evaluaré si sumo un segundo (datos seed) o registro la desviación — te lo confirmo al planificarla.

Después de la G queda solo la Fase H (README raíz + resumen ejecutivo + handoff a codificación), que cierra el SDD.

¿Commiteo la Fase F y sigo con la G?

---

Qué es la categoría 11-Examples
En el SDD, la 11-Examples produce ejemplos ejecutables y reproducibles que muestran cómo un tercero consume las superficies públicas del sistema. Ojo con la distinción: no es el producto, ni tests, ni documentación — son demos consumidor-facing. Viven en samples/ y cada una se puede correr y ver funcionar. Su propósito es que alguien externo (un integrador, o vos mismo dentro de seis meses) entienda cómo se usa el sistema sin leer el código.

Qué "sample" declara tu intake (§18)
Tu intake §18 fija un solo sample: samples/ingesta-gmao/. Ilustra el único contrato formal hacia terceros que tiene el sistema: la API REST de ingesta POST /api/v1/intervenciones, con sus cuatro caminos de respuesta:

201 — creado (clave nueva, cuerpo válido)
200 con creado=false — reintento idempotente (la clave ya se procesó con el mismo cuerpo)
409 — conflicto de idempotencia (misma clave, cuerpo distinto)
422 — invariante roto (costos que no cuadran, dinero sin moneda/fecha, etc.)
El intake es explícito en que el panel Blazor no produce sample: "se demuestra ejecutando el propio servicio, que es el entregable de cada etapa". Tiene lógica — el panel no lo consume un tercero por código; lo abrís y lo mirás. El único que sí se consume programáticamente (el GMAO empujando intervenciones) es la API. Por eso hay exactamente un sample.

De dónde sale la tensión
La tabla de adaptabilidad por tipo del framework (master-prompt §14) fija para web-monolith un mínimo de 2 samples: literalmente "datos seed + tema custom si hay extensión visual". Entonces:

Intake: 1 sample (la API de ingesta).
Piso del framework: 2 samples.
Un sample corto. De ahí las dos salidas que mencioné.

Las dos opciones
Opción A — sumar un segundo sample (datos seed).
El "tema custom" del framework no aplica (este proyecto no tiene extensión visual/theming). Pero "datos seed" sí encaja y aporta valor real: un sample que precarga el sistema con datos de ejemplo —equipos, baterías, historial, usando los escenarios E-1…E-8 del §20 del intake— para que cualquiera pueda explorar el panel y las consultas sin tener el SAI real conectado. Es la misma filosofía que el adaptador simulado: probar sin hardware ni riesgo. Onboarding y demo sin equipo físico.

Opción B — registrar la desviación como ADR.
Dejar el único sample de la API (como manda el intake) y anotar un ADR de omisión del segundo, justificado por §18: el panel no es superficie consumida por un tercero, así que un sample suyo sería artificial. Es lo mismo que hicimos con la omisión de la categoría 10 (ADR-23) y con los ambientes (ADR-24).

Mi recomendación: la Opción A. El sample de datos seed no es relleno para cumplir el piso — es genuinamente útil (explorar el sistema sin hardware) y barato de especificar, y evita otro ADR de omisión. Pero es tu llamada; por eso dije que te lo confirmo al planificar la fase.

Qué es la Fase H (la última)
Cierra el SDD. Produce tres cosas:

README raíz (SDD/Docs/README.md) — el índice maestro de toda la documentación: presenta la solución, enlaza las categorías 00→11, y —por el ADR-23— incluye la sección de onboarding del desarrollador que dijimos que se consolidaba acá.
Resumen ejecutivo del handoff — una tabla con todo lo generado: proyectos, documentos por categoría, cobertura de la cadena de trazabilidad D6 (Visión→NB→CU→RN→ADR→US→BT→Sprint→Test→Pipeline), ítems del Sprint 1 listos para codear, audits aprobados, decisiones pendientes (los 9 PENDIENTE + los 4 ADR de Sprint 0), y la línea de base del sensado de deriva.
El punto de handoff a codificación — el orquestador se detiene y NO escribe código hasta que vos confirmes explícitamente arrancar el Sprint 1. Es el cierre formal del trabajo de documentación.
¿Avanzo con el commit de la Fase F y, para la G, voy por la Opción A (sumo el sample de datos seed)? ¿O preferís la B?

---

Respuesta
ok hace el comit para la G y seguir con la opcion A