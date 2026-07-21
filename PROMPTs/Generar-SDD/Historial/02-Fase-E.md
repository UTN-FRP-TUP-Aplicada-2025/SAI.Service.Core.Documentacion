Renombrado: US-25 ahora es US-25-Adaptador-De-Conexion-Directo-v1.0.md, "Sin-Nut" queda en 0 en toda la doc. El Product-Backlog referencia US-25 por ID (no por link de filename), así que no se rompió ningún enlace. 06 quedó completamente limpio de nombres de producto, incluido el filename.

Fase D cerrada. El P1 (bloqueante) está corregido y verificado por grep; P2 también. Los 3 P3 quedan anotados. Me detengo por la detención entre fases.

Fase D — completada y auditada (P1 corregido)
Categoría	Archivos	Contenido
06-Backlog-Tecnico	4 + 26 US	Product Backlog (7 épicas, 26 US, MoSCoW 18/6/2), Backlog Técnico (30 BT), DoR, README
07-Plan-Sprint	2	Mini-Plan (6 etapas, 17 riesgos) + README — variante 1 dev, sin artefactos de equipo
Audit Fase D	1	P1 (stack en 06) corregido; cobertura NB↔CU↔US↔BT sin huérfanos
La cadena D6 llega al plan de iteración: … → ADR → arquitectura → US/BT → Mini-Plan, con los IDs del plan todos existentes en el backlog.

Estado del SDD

✓ A · ✓ B · ✓ B2 · ✓ C · ✓ D
· Fase E  (08-Calidad-Y-Pruebas)  ← siguiente
· Fase F  (09-Devops, 10-Developer-Guide)
· Fase G  (11-Examples)
· Fase H  (README raíz + handoff a codificación)
La Fase E (08-Calidad-Y-Pruebas) genera la estrategia de calidad/testing, plan de pruebas, matriz de cobertura, casos de prueba, criterios de validación y la Definition-of-Done — y además emite la Matriz-Sensado-Deriva que quedó pendiente de la Fase B2 (va en 08).

¿Commiteo la Fase D y sigo con la E?