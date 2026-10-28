# 11 — Estado y pendientes

> **Propósito:** decir dónde está parado el proyecto: versión, qué etapa del plan está cerrada según la evidencia (CHANGELOG + git), qué ramas y PR hay, y todo lo declarado **pendiente / diferido / fuera de alcance** en el SDD y en el código. También registra las **desincronizaciones documentales**, es decir, documentos cuyo estado declarado quedó atrás del código.
> **Fuente primaria:** `CHANGELOG.md` (lo declara fuente autoritativa del avance `SDD/Docs/07-Plan-Sprint/Mini-Plan-v1.0.md` §10) y el historial git de `main`. Fuentes secundarias: `SDD/Docs/07-Plan-Sprint/`, `SDD/Docs/06-Backlog-Tecnico/`, `SDD/Docs/05-Arquitectura-Tecnica/Decisiones-Arquitectura-v1.0.md`, grep de `TODO|FIXME|pendiente|diferido` en `src/` y `tests/`.
> **Corte de esta foto:** último commit de `main` `6fd23d5` del 2026-08-04. Todo lo posterior es «(no verificado)».

Índices vecinos: el *qué es* de cada etapa vive en `04_Aplicacion-Y-Casos-De-Uso.md` y `02_Dominio-Y-Reglas.md`; el pipeline y el contenedor, en `08_Devops-Despliegue-Y-Operacion.md`; las pruebas, en `07_Calidad-Y-Pruebas.md`; el árbol del SDD, en `09_Documentacion-Sdd.md`.

---

## 1. Foto rápida

| Dato | Valor | Fuente |
|---|---|---|
| Versión en el build | `0.1.0-alpha.1` (fija, «de referencia») | `Directory.Build.props` (`<Version>`) |
| Versión en el sello del panel | `0.1.0-alpha.1`, `FechaIteracion` 2026-07-20, `EsPreliminar: true` | `src/SAI.Service.Core/SAI.Service.Core.Web/appsettings.json` (`Sello`) |
| Última release del CHANGELOG | `[0.1.0] - 2026-07-20` (sólo documentación). Todo el código está en `[Unreleased]` | `CHANGELOG.md` |
| Tags git | **ninguno** | `git tag` |
| MinVer (versionado desde tags) | Previsto, **no incorporado**: no hay `PackageReference` a MinVer | `SDD/Docs/09-Devops/Estrategia-Versionado-v1.0.md` §3 vs `Directory.Build.props` |
| Último commit de `main` | `6fd23d5` 2026-08-04, *Merge pull request #29 … feature/politica-retardo-retorno* | `git log -1 main` |
| `main` vs `origin/main` | sincronizados | `git status -sb` |
| Commits en `main` | 81 (30 merges: 29 PR + 1 merge `origin/main` → `feature/ep04-politicas`) | `git log` |
| Etapa del plan | **Etapa 5 Realizada**: los 12 CU y los 10 flujos UF cerrados, «release v1.0 en alcance» | `SDD/Docs/07-Plan-Sprint/Mini-Plan-v1.0.md` §8, §10, v1.2 |
| Migraciones EF | 15 (de `EsquemaInicialAcceso` a `PoliticaTiempoRetorno`) | `src/SAI.Service.Core/SAI.Service.Core.Infrastructure/Persistencia/Migraciones/` |
| Pruebas (atributos `[Fact]/[Theory]/[HechoNutVivo]`, conteo por grep) | Domain 124 · Application 31 · Integration 115 (~270) | `tests/*/` (es un conteo aproximado, no la corrida) |
| ADR | 29: 24 Aceptado · 1 Propuesto (ADR-21) · 4 Superado (19, 20, 22, 23) | `SDD/Docs/05-Arquitectura-Tecnica/Adrs/*.md` (línea `**Estado:**`) |

---

## 2. Línea de tiempo (git + CHANGELOG)

```
2026-07-18  a65036f  Initial commit                                   → [0.0.1]
2026-07-20  PR #1    Intake + Fases A y B del SDD                      → [0.1.0]
2026-07-20/21 PR #2  Fases B2 (maqueta), C, D, E, F, G, H del SDD
2026-07-21  PR #3    Sprint 0 — andamiaje (5 assemblies, CI, panel base)
2026-07-21  PR #4    Etapa 1 — persistencia, acceso, API Bearer JWT (ROPC)
2026-07-21  PR #5–#8 Etapa 2 — A dominio equipos · B adaptador NUT · C1 persistencia · C2 panel alta
2026-07-21/22 PR #9–#13 Etapa 3 — A sondeo · B1 eventos · B2 panel en vivo · C prueba batería · D históricos
2026-07-22/23 PR #14–#17 Etapa 4 — A ventana mant. · B apagado ordenado · C recambio · E disparo+rearme
2026-07-23/25 PR #18–#19 Rediseño UX del Panel de verificaciones + ejercicio guiado (P-7) + fixes
2026-07-25  PR #20   Docs: referencias a reglas sin prefijo numérico
2026-07-25  PR #21   Etapa 4·D — sustitución del SAI (EP-06)
2026-07-26  PR #22   Políticas de apagado versionadas (EP-04)
2026-07-26  PR #23   Informe de período y comparación de marcas (EP-07, UF-9)
2026-07-26  PR #24   Ingesta idempotente POST /api/v1/intervenciones (UF-10) → cierra Etapa 5
2026-07-26  PR #25   Mini-plan v1.2 (cierra UF-10 y Etapa 5)
2026-07-26  PR #26   Reajuste SDD v3.0 Parte A (10↔11)
2026-07-27  PR #27   Categoría 11-Documentacion (Parte B) + AGENTS.md
2026-07-27  PR #28   Coherencia final del reajuste
2026-08-04  PR #29   Tiempo de retorno configurable en la política + Dockerfile multi-stage
```

Fuente: `git log --format='%h %ad %s'` sobre `main`, con los títulos de commit y de merge.

### 2.1 PR fusionados (mensajes de merge)

| PR | Rama | Commit de feature | Contenido | Entrada CHANGELOG |
|---|---|---|---|---|
| #1 | `feature/sdd-manifest-y-docs` | `27a5e72`, `a4c6479` | Intake v1.0 + Fases A y B | `[0.1.0]` y Unreleased/Notas |
| #2 | `feature/sdd-fase-b2-maqueta` | `2be1105` … `109ed97` | Fases B2, C, D, E, F, G, H | «Fase B2…H del SDD» |
| #3 | `feature/sprint-0-arranque` | `42157fe`, `72fc9ed`, `03ee7be` | Andamiaje, SQLitePCLRaw 2.1.12, tema | «Sprint 0 — Arranque» |
| #4 | `feature/etapa-1-persistencia-acceso` | `4df5e30` | Etapa 1 | «Etapa 1 — Persistencia y acceso» |
| #5 | `feature/etapa-2a-dominio-equipos` | `7ed677f` | Etapa 2·A (BT-11, BT-12) | «Etapa 2 · Incremento A» |
| #6 | `feature/etapa-2b-adaptador-nut` | `933fc8f` | Etapa 2·B (BT-15, US-03) | «Etapa 2 · Incremento B» |
| #7 | `feature/etapa-2c-alta-equipos` | `ef30868` | Etapa 2·C1 | «Etapa 2 · Incremento C1» |
| #8 | `feature/etapa-2c2-panel-alta` | `48ac12a` | Etapa 2·C2 | «Etapa 2 · Incremento C2» |
| #9 | `feature/etapa-3a-sondeo-muestras` | `859f7d3` | Etapa 3·A | «Etapa 3 · Incremento A» |
| #10 | `feature/etapa-3b-eventos-panel` | `a04e33f` | Etapa 3·B1 | «Etapa 3 · Incremento B1» |
| #11 | `feature/etapa-3b2-panel-en-vivo` | `96dd3eb` | Etapa 3·B2 | «Etapa 3 · Incremento B2» |
| #12 | `feature/etapa-3c-prueba-bateria` | `ddf5df2` | Etapa 3·C | «Etapa 3 · Incremento C» |
| #13 | `feature/etapa-3d-historicos` | `8523c5f` | Etapa 3·D | «Etapa 3 · Incremento D» |
| #14 | `feature/etapa-4a-ventana-mantenimiento` | `1799207` | Etapa 4·A | «Etapa 4 · Incremento A» |
| #15 | `feature/etapa-4b-apagado-ordenado` | `63e1b59` | Etapa 4·B + write path NUT | «Etapa 4 · Incremento B» |
| #16 | `feature/etapa-4c-recambio-bateria` | `d906b62` | Etapa 4·C | «Etapa 4 · Incremento C» |
| #17 | `feature/etapa-4e-disparo-rearme` | `cae3529` + 6 `fix(verificaciones)` | Disparo + rearme por reinicio | «Ventana de mantenimiento — disparo…» |
| #18 | `feature/ux-panel-verificaciones` | `55f4933`, `b163a11`, `b3bcdfc` | Rediseño UX fases 1-2 + P-7 | «Rediseño UX/UI…», «Ejercicio guiado…» |
| #19 | `fix/panel-verificaciones-ux` | `d60a59f`, `5b333cf`, `d56f2bf` | Fixes + indicador SAI en vivo | «Indicador de estado del SAI…» |
| #20 | `docs/referencias-reglas-sdd` | `3ee3bbf` | Nomenclatura de reglas | «nomenclatura de las reglas» |
| #21 | `feature/ep06-sustitucion-sai` | `af0142d` | Etapa 4·D | «Sustitución del SAI…» |
| #22 | `feature/ep04-politicas` | `4002625`, `b314bef` | Políticas versionadas | «Configuración de políticas…» |
| #23 | `feature/ep07-informe-periodo` | `e622549`, `6a0eeec` | Informe de período | «Informe de período…» |
| #24 | `feature/uf10-ingesta` | `8d232e7` | Ingesta idempotente | «Etapa 5 · Ingesta…» |
| #25 | `docs/miniplan-uf9` | `74725e3` | Mini-plan v1.2 | — |
| #26 | `docs/reajuste-sdd-cat-10-11` | `4f5c6bf` | Reajuste SDD v3.0 A | Unreleased/Cambiado |
| #27 | `feature/sdd-cat11-documentacion` | `4329aef` | Categoría 11 | Unreleased/Cambiado |
| #28 | `docs/reajuste-coherencia-final` | `54d08aa` | README maestro §4/§7 | — |
| #29 | `feature/politica-retardo-retorno` | `087ad71`, `baa9a54` | Tiempo de retorno + Dockerfile | «Empaquetado…», «Tiempo de retorno…» |

#17 y #18 se fusionaron **apilados** (merge `4841d88` y `5445a73` sobre la rama del PR siguiente) y llegaron a `main` recién con el merge de #19 (`17a89ae`, 2026-07-25). Por eso `git log --first-parent main` no los muestra. Los 6 `fix(verificaciones)` de #17 son `b33b779` … `e743e79`.

### 2.2 Ramas existentes

| Rama | Dónde | Estado |
|---|---|---|
| `main` | local + `origin/main` (HEAD remoto) | vigente, `6fd23d5` |
| `feature/politica-retardo-retorno` | sólo local | **ya fusionada** (aparece en `git branch --merged main`; punta `baa9a54` = PR #29). Se puede borrar |

En el remoto (`git ls-remote --heads origin`) sólo queda `refs/heads/main`. Las demás ramas de los PR #1–#28 ya no existen.

---

## 3. Estado por etapa del plan

Plan único para un solo desarrollador (Mini-Plan; no hay planes de iteración, ni review, ni velocity): `SDD/Docs/07-Plan-Sprint/README.md`, `Mini-Plan-v1.0.md` §1.

| Etapa | Épica | UF | SP | Release plan | Estado **declarado** en Mini-Plan | Estado **real** (CHANGELOG/git) |
|---|---|---|---|---|---|---|
| Sprint 0 — Arranque | EP-01+EP-02 | — | 24 | v0.2 | «Propuesto», ítems «Pendiente» (§3) | Cerrado, PR #3 |
| Etapa 1 — Persistencia, admin, sesión | EP-03 | — | 24 | v0.4 | «Propuesto» (§4) | Cerrado, PR #4 |
| Etapa 2 — Alta de equipos y políticas | EP-04 | UF-1→UF-2 | 58 | v0.6 | «Propuesto» (§5) | UF-1 cerrado PR #5–#8. UF-2 (políticas, BT-16/US-06) **recién en PR #22** (2026-07-26), fuera de orden |
| Etapa 3 — Monitoreo, salud, históricos | EP-05 | UF-3→UF-5→UF-4 | 79 | v0.9 | «Propuesto» (§6) | Cerrado, PR #9–#13 |
| Etapa 4 — Verificación y ciclo de vida | EP-06 | UF-8→UF-6→UF-7 | 91 | v0.12 | «Propuesto» (§7) | Cerrado, PR #14–#19, #21 |
| Etapa 5 — Integración e informes | EP-07 | UF-10→UF-9 | 47 | v1.0 | **«Realizada»**, ítems **«Realizado»** (§8) | Cerrado, PR #23, #24 |

La bitácora (§10 del Mini-Plan) agrupa «Sprint 0 → Etapa 4» en una sola fila: «CU-01..CU-10 entregados de forma incremental; ver `CHANGELOG.md`».

### 3.1 Ítems por etapa (BT/US) y su evidencia

| Etapa | Ítems | Incremento CHANGELOG que los cierra |
|---|---|---|
| Sprint 0 | BT-01, BT-02, BT-04 (spikes → ADR-25/26/27), BT-03 (→ diferido), BT-05, BT-06 | «Sprint 0 — Arranque» |
| 1 | BT-07..BT-10, US-01, US-02 | «Etapa 1 — Persistencia y acceso» |
| 2 | BT-11, BT-12 · BT-13 (configuración de despliegue, no código) · BT-14, BT-15, US-03 · US-04, US-05 · BT-16, US-06 | 2·A · 2·B · 2·C1/C2 · «Configuración de políticas» |
| 3 | BT-17, BT-18, US-08, US-10 · BT-19, BT-20, US-09 · US-07 · BT-21, US-12, US-13 · US-11 | 3·A · 3·B1 · 3·B2 · 3·C · 3·D |
| 4 | BT-23, BT-25, US-16, US-17 · BT-22, BT-24, US-14, US-15 · BT-26, BT-27, US-18, US-19 · US-20 | 4·A · 4·B · 4·C · «Sustitución del SAI» |
| 5 | BT-28, BT-29, US-21, US-22 · BT-30, US-23, US-24 | «Etapa 5 · Ingesta» · «Informe de período» |

El `CHANGELOG.md` no cita por ID BT-01..BT-06 (Sprint 0 nombra los ADR-25/26/27), BT-22 ni BT-29. Para esos, la asignación sale del Mini-Plan y no de una entrada que los cierre. BT-13 figura como «configuración de despliegue, no código» (2·B).

Las rutas de las historias están en `SDD/Docs/06-Backlog-Tecnico/historias-usuario/US-NN-…-v1.0.md`, y las BT, inline en `SDD/Docs/06-Backlog-Tecnico/Backlog-Tecnico-v1.0.md` §2.

### 3.2 Trabajo fuera del plan original (posterior al cierre de etapas)

| Cambio | Origen | Fuente |
|---|---|---|
| Rediseño UX/UI del Panel de verificaciones, fases 1 y 2 (estado `PorVencer`, `DialogoConfirmacionRiesgo`, `ConstructorVistaPanel`) | SPEC *Rediseño UX/UI del Panel de verificaciones* (no versionada en el repo, «(no verificado)» dónde vive) | `CHANGELOG.md` Unreleased/Cambiado |
| Ejercicio guiado P-7 (`SesionEjercicio`, `SecuenciaFisica`) | misma SPEC | `CHANGELOG.md` |
| Indicador del SAI en vivo en verificaciones (`DescriptorEnVivo`) | mejora UX | `CHANGELOG.md` |
| Tiempo de retorno configurable (`VersionPolitica.TiempoRetornoSeg`, migración `PoliticaTiempoRetorno`) | CU-03, ADR-09/ADR-27 | `CHANGELOG.md` Unreleased/Añadido |
| `Dockerfile` multi-stage + `.dockerignore` | ADR-18, ADR-20 | `CHANGELOG.md`, `Dockerfile` |
| Reajuste SDD v3.0 (10↔11) + categoría 11 + `AGENTS.md` | framework `IA.SDD` v3.0 | `CHANGELOG.md` Unreleased/Cambiado |

---

## 4. Backlog: alcance y lo diferido

| Concepto | Valor | Fuente |
|---|---|---|
| Historias | 26 (US-01..US-26) · 161 SP | `SDD/Docs/06-Backlog-Tecnico/Product-Backlog-v1.0.md` §3-§4 |
| Must / Should / Could | 18 (114 SP) / 6 (39 SP) / 2 (8 SP) | ídem §4 |
| Tareas técnicas | 30 (BT-01..BT-30) | `Backlog-Tecnico-v1.0.md` §2 |
| Alcance de v1 | 24 historias Must+Should (153 SP) | `Mini-Plan-v1.0.md` §1 |
| DoR | 7 criterios para US y 5 para BT | `Definition-Of-Ready-v1.0.md`, `06-Backlog-Tecnico/README.md` |

### 4.1 Diferido a v2 (Could, diseñado pero no implementado)

| ID | Qué | Fuente |
|---|---|---|
| US-25 | Adaptador de conexión **directo** (equipos que `nutdrv_qx` no cubre), F-27 | `historias-usuario/US-25-Adaptador-De-Conexion-Directo-v1.0.md`; `SDD/Docs/05-Arquitectura-Tecnica/Extensibilidad-v1.0.md` §3 |
| US-26 | Capa de **add-ons de dialecto** de protocolo, F-26 / E-07 | `historias-usuario/US-26-…-v1.0.md`; `Extensibilidad-v1.0.md` §4; `SDD/Docs/08-Calidad-Y-Pruebas/Guia-Testing-Extensibilidad-v1.0.md` |

### 4.2 Won't (v1.0), fuera del backlog

F-28 apagado de otros equipos · F-29 múltiples SAI · F-30 notificaciones externas primarias · F-31 gestión de usuarios y roles · F-32 lectura del ajuste de BIOS por software · F-33 traductor de protocolo propio. Fuente: `Product-Backlog-v1.0.md` §4 (remite a Intake §4).

---

## 5. Pendientes y diferidos declarados (en el SDD y el CHANGELOG)

| # | Pendiente | Estado declarado | Fuente |
|---|---|---|---|
| P-01 | **Endpoint de rectificación del 409** de la ingesta (BT-03) | ADR-21 **Propuesto**, el único ADR abierto. El 409 devuelve `accionSugerida` como texto | `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-21-…-v1.0.md`; `CHANGELOG.md` «Etapa 5 · Ingesta»; `src/SAI.Service.Core/SAI.Service.Core.Application/Ingesta/ServicioIngesta.cs` (texto «pendiente, ADR-21») |
| P-02 | **TLS de producción** | ADR-26 está Aceptado, pero sólo hay endpoint HTTP `:8080`. La «ranura» `_RanuraTls` queda sin implementar | `src/SAI.Service.Core/SAI.Service.Core.Web/appsettings.json` (`Kestrel._RanuraTls`); `SDD/Docs/11-Documentacion/Guia-Contenedor-v1.0.md` l.65 («hoy no está implementado») |
| P-03 | **RN-11 / I-13**: ligar cada `Accion` a su `VersionPolitica` | diferido | `CHANGELOG.md` «Configuración de políticas». Confirmado en el código: `src/SAI.Service.Core/SAI.Service.Core.Domain/Acciones/Accion.cs` no tiene referencia a versión de política |
| P-04 | Verificaciones requeridas **editables por versión** de política | diferido (se derivan de la modalidad) | `CHANGELOG.md` «Configuración de políticas» |
| P-05 | Ranura del **asistente de IA** en políticas | reservada | ídem |
| P-06 | **Purga por retención** de muestras (30 d muestra / 10 a agregado) | diferida porque choca con el interceptor append-only. Hoy `ServicioHistoricos` sólo **elige la fuente** según la retención, no purga | `CHANGELOG.md` 3·A y 3·D; `src/SAI.Service.Core/SAI.Service.Core.Application/Monitoreo/ServicioHistoricos.cs` (`DiasRetencionMuestras = 30`) |
| P-07 | **Corrección de fecha con reatribución del histórico** (CU-08 FA-2) | fuera del incremento | `CHANGELOG.md` 4·C y «Sustitución del SAI» |
| P-08 | **Auto-medición del tiempo de apagado** | fuera de alcance por ADR-25: el tiempo se carga a mano | `CHANGELOG.md` «Ventana de mantenimiento — disparo…» |
| P-09 | **Write path NUT real contra host físico** | «se valida contra un host físico en despliegue, no en desarrollo». El simulado es el banco de pruebas | `CHANGELOG.md` 4·B y «disparo…» |
| P-10 | **Verificación visual en navegador** del informe de período | «queda para la validación del despliegue» | `CHANGELOG.md` «Informe de período»; `Mini-Plan-v1.0.md` §8 |
| P-11 | Selector **multi-equipo** y theme global/sidebar | fuera de alcance del rediseño UX | `CHANGELOG.md` «Rediseño UX/UI» |
| P-12 | **i18n**: la interfaz es sólo en español | fuera de alcance | `src/SAI.Service.Core/SAI.Service.Core.Web/Components/Pages/PanelVerificaciones/FormatoRelativo.cs` |
| P-13 | **CI stages 5–10**: e2e (bUnit/Playwright), SCA, SBOM, imagen, firma cosign, publicación | «se agregan en etapas posteriores». `ci.yml` corre restore, format, build y test | `.github/workflows/ci.yml` (cabecera); `SDD/Docs/09-Devops/Pipeline-CI-CD-v1.0.md` §1 («Stages obligatorios») |
| P-14 | **MinVer** y tags `v*` | previsto, no implementado (versión fija, sin tags) | `Estrategia-Versionado-v1.0.md` §3; `Directory.Build.props` |
| P-15 | Ningún **tag ni release** del código (todo en `[Unreleased]`) | no se cortó release pese a «v1.0 en alcance» | `CHANGELOG.md`; `git tag` |
| P-16 | `HEALTHCHECK` de la imagen | se declara en el orquestador, no en la imagen (la imagen `aspnet` no trae curl/wget) | `CHANGELOG.md` «Empaquetado» |
| P-17 | Escenario de datos de **sustitución del SAI** (R-11) | mitigación prevista «cuando se implemente el flujo». No hay rastro de que se haya agregado, «(no verificado)» | `Mini-Plan-v1.0.md` §7.5 |
| P-18 | Validación de **volumen/retención** (~6,3 M filas/año) y tamaño del SQLite (R-07) | «antes de producción» | `Mini-Plan-v1.0.md` §6.5; `SDD/Intake/SOLUTION-INTAKE-Sai-Service-Core-v1.0.md` (PENDIENTE l.673) |
| P-19 | Medición del resto del apagado del SO y SLO del servicio | PENDIENTE en el intake | `SOLUTION-INTAKE-…-v1.0.md` l.651, l.689 |
| P-20 | NFR ausentes: tiempos de respuesta del panel, accesibilidad, i18n, rate limiting, límites de request, plan de recuperación ante desastre del propio servicio | PENDIENTE; el intake dice que «se cierran en la categoría 08» | `SOLUTION-INTAKE-…-v1.0.md` l.704 |
| P-21 | Riesgos físicos R-01 (trampa de firmware), R-02 (presupuesto de 540 s), R-03 (BIOS) | se mitigan con la ventana de mantenimiento **física**. No hay evidencia en el repo de que se haya ejecutado sobre el host real, «(no verificado)» | `Mini-Plan-v1.0.md` §7.5 |
| P-22 | Traza de auditoría de la Fase D | **resuelto**: §7 dice «NO APROBADO — BLOQUEADO», pero el §8 *Addendum de cierre (2026-07-21)* registra P1-01 y P2-01 corregidos y actualiza el veredicto a «APROBADO CON OBSERVACIONES». Lo que queda desactualizado es la Fase H (ver §7) | `SDD/Docs/Audit/Fase-D-Backlog-Plan-Sprint-v1.0.md` §7–§8; `SDD/Docs/Audit/Fase-H-Consolidado-Final-v1.0.md` P1-01 |

La bitácora `SDD/Docs/11-Documentacion/Bitacora-Eventualidades-v1.0.md` no aporta pendientes: sus 7 entradas (EVE-01..EVE-07) tienen **Resolución** y **Destino**.

---

## 6. Marcadores en el código (grep)

`git grep -E 'TODO|FIXME|HACK|XXX'` sobre `src/`, `tests/`, `scripts/`, `.github/`: **cero marcadores reales** (el único resultado es «TODOS» en un comentario de `Dockerfile` l.24). Los pendientes aparecen en prosa, en comentarios XML:

| Archivo | Texto | Vigencia |
|---|---|---|
| `src/SAI.Service.Core/SAI.Service.Core.Application/Ingesta/ServicioIngesta.cs` l.35 | «endpoint de rectificación (pendiente, ADR-21)» | **vigente** (P-01) |
| `src/SAI.Service.Core/SAI.Service.Core.Web/appsettings.json` l.43 | `_RanuraTls`: «ADR-20 (P-04, PENDIENTE)» | vigente en sustancia (P-02), pero cita ADR-20, que ya está Superado por ADR-26 |
| `src/SAI.Service.Core/SAI.Service.Core.Application/Abstractions/IAdaptadorConexion.cs` l.4 | «ADR-22 pendiente» | **desactualizado**: ADR-22 está Superado por ADR-27 (Aceptado) |
| `src/SAI.Service.Core/SAI.Service.Core.Application/Equipos/ServicioVerificacion.cs` l.33 | «write path real por NUT está diferido (Incremento B)» | **desactualizado**: el Incremento 4·B implementó el write path NUT |
| `src/SAI.Service.Core/SAI.Service.Core.Application/Monitoreo/ServicioPruebaBateria.cs` l.29, l.63 | «lanzamiento real por NUT está diferido» | **desactualizado**: 4·B agrega `test.battery.start.quick`. Sin credenciales `Sai:Nut:*` sigue sin correr |
| `src/SAI.Service.Core/SAI.Service.Core.Web/Components/Pages/PruebaDeBateria.razor` l.130 | `TEST_NO_DISPONIBLE`: «requiere credenciales de escritura (diferido)» | parcial: lo que falta son las credenciales, no el código |
| `src/SAI.Service.Core/SAI.Service.Core.Web/Components/Pages/PanelVerificaciones/EncabezadoEstadoServicio.razor` l.2 | «Sin acción primaria (P-7 diferido)» | **desactualizado**: el ejercicio guiado P-7 ya está implementado |
| `tests/SAI.Service.Core.Integration.Tests/VerificacionIntegracionTests.cs` l.13 | «El write path por NUT está diferido» | **desactualizado** (ídem 4·B) |

Hay otros usos de «diferido/pendiente» que **no** son pendientes: carga diferida de intervenciones (FA-1), lectura diferida de configuración JWT/cadena, `SecuenciaFisica.PrimeroPendiente`, el chip «Pendiente». Pertenecen a `02_Dominio-Y-Reglas.md` y `04_Aplicacion-Y-Casos-De-Uso.md`.

---

## 7. Desincronizaciones documentales (estado declarado ≠ evidencia)

| Documento | Dice | Evidencia | Fuente de la evidencia |
|---|---|---|---|
| `SDD/Docs/07-Plan-Sprint/README.md` §«Etapa actual» | Etapa actual: **Sprint 0** | Etapa 5 Realizada | `Mini-Plan-v1.0.md` v1.2 |
| `SDD/Docs/07-Plan-Sprint/Mini-Plan-v1.0.md` §3–§7 | Etapas 0–4 «Propuesto», ítems «Pendiente» | cerradas (PR #3–#22) | `git log`, `CHANGELOG.md` |
| `Mini-Plan-v1.0.md` §9 y `07-Plan-Sprint/README.md` | DoD de 08 «pendiente de generación» | existe `SDD/Docs/08-Calidad-Y-Pruebas/Definition-Of-Done-v1.0.md` | `git ls-files` |
| `SDD/Docs/06-Backlog-Tecnico/Product-Backlog-v1.0.md` §3-§4 | 26 US «Borrador», «0 % cerrado» | 24 US cerradas | `CHANGELOG.md` |
| `historias-usuario/US-*.md` (26) | `**Estado:** Borrador` | ídem | ídem |
| `SDD/Docs/08-Calidad-Y-Pruebas/Matriz-Cobertura-Pruebas-v1.0.md` §CU | 12 filas CU en «Pendiente» | hay pruebas por CU en `tests/` | `tests/SAI.Service.Core.Integration.Tests/*IntegracionTests.cs` |
| `SDD/Docs/11-Documentacion/Guia-Despliegue-v1.0.md` l.34 | contenedor «Pendiente (sin Dockerfile)» | `Dockerfile` versionado desde `baa9a54` (PR #29) | `Dockerfile`, `git log` |
| `CHANGELOG.md` «Documentación SDD — categoría 11» | «no hay Dockerfile» | ídem (la entrada es anterior al Dockerfile) | ídem |
| `CHANGELOG.md` «Informe de período» | encabezado «**Etapa 4** · Informe…» y cita «US-11» | el plan lo ubica en Etapa 5 (UF-9) con US-23/US-24 | `Mini-Plan-v1.0.md` §8 |
| `CHANGELOG.md` «Configuración de políticas de apagado versionadas» | encabezado «**Etapa 4** · …» | el plan la ubica en Etapa 2 (UF-2: BT-16, US-06) | `Mini-Plan-v1.0.md` §5.2 |
| `CHANGELOG.md` 3·B2 | «Con esto se completa la **Etapa 3**» (3·C, en cambio, dice «salvo los históricos») | cierra recién con 3·D | `CHANGELOG.md` 3·D |
| `SDD/Docs/Audit/Fase-H-Consolidado-Final-v1.0.md` P1-01 | la Fase D «no fue re-emitido ni recibió addendum» | la Fase D tiene el §8 *Addendum de cierre* | `SDD/Docs/Audit/Fase-D-Backlog-Plan-Sprint-v1.0.md` §8 |
| `SDD/Docs/11-Documentacion/Guia-Contenedor-v1.0.md` l.63; `CHANGELOG.md` «Empaquetado» | el TLS de producción es «ranura de ADR-20» | ADR-20 está Superado por ADR-26 | `Adrs/ADR-20-…-v1.0.md` (`**Estado:**`) |
| `CHANGELOG.md` Unreleased/Notas | «Pendiente: Fase B2 y Fases C a H» | las Fases B2–H se hicieron (PR #2) | `git log` `2be1105`…`109ed97` |
| `SDD/Docs/README.md` §7 | categorías 00–10 «Borrador» v1.0 y 11 «Vigente» | sin cambio de estado tras implementar; además varios docs ya están Vigente (03 de la Fase B2, `Matriz-Sensado-Deriva-v1.0.md`) | encabezados de esos archivos (es estado documental, no un error) |
| `SDD/Docs/05-Arquitectura-Tecnica/README.md` | «22 ADR (18 Aceptado, 4 Propuesto)»; ADR-19/20/22 Propuesto | 29 ADR en `Adrs/`: 24 Aceptado, 1 Propuesto (ADR-21), 4 Superado | `Adrs/*.md` (`**Estado:**`) |
| `SDD/Docs/05-Arquitectura-Tecnica/Arquitectura-Solucion-v1.0.md` §1/§5/§10 | 22 ADR; ADR-19…22 «Propuesto» | ídem | ídem |
| `SDD/Docs/10-Examples/README.md` «Vínculo…» | generación de la 11 pendiente | la 11 está generada y Vigente (commit `4329aef`) | `SDD/Docs/11-Documentacion/README.md` |
| `SDD/Docs/10-Examples/Ejemplo-01-Datos-Seed-v1.0.md`, `Ejemplo-02-Api-Ingesta-v1.0.md` | código en `/samples/01-datos-seed/` y `/samples/02-api-ingesta/` | `/samples/` no existe en el repo | `git ls-files samples` (vacío) |
| `README.md` (raíz), nota final | «Sprint 0 es andamiaje… sin lógica de negocio» | release v1.0 en alcance | `Mini-Plan-v1.0.md` v1.2 §10 |
| `SDD/Docs/README.md` §3 | adaptador de conexión «con dos implementaciones» | tres diseñadas (NUT, directo + add-on, simulada); dos implementadas | `Adrs/ADR-02-…`, `Extensibilidad-v1.0.md` |
| `SDD/Docs/05-Arquitectura-Tecnica/Decisiones-Arquitectura-v1.0.md` v1.5 | «25 Aceptado, 1 Propuesto, 3 Superado» | 24 / 1 / 4 (ADR-23 pasó a Superado en el reajuste, sin fila nueva en el control de cambios) | `Adrs/ADR-23-…-v1.0.md` |
| `Directory.Build.props` y `appsettings.json` `Sello` | `0.1.0-alpha.1`, fecha 2026-07-20, preliminar | código en estado «v1.0 en alcance» | `Mini-Plan-v1.0.md` §10 |

---

## 8. Riesgos del plan aún abiertos

| Riesgo | Descripción | Estado según fuentes | Fuente |
|---|---|---|---|
| R-01 | Trampa de firmware: el SAI podría quedar apagado | depende de la ventana física | `Mini-Plan-v1.0.md` §7.5 |
| R-02 | Presupuesto de 540 s sin medir contra el host real | ídem; vence a los 180 d | ídem |
| R-03 | El ajuste de BIOS puede volverse falso sin aviso | verificación por comportamiento (BT-25) | ídem |
| R-07 | Volumen y retención sin probar | abierto (P-06, P-18) | §6.5, §8.5 |
| R-09 | No hay sensor de temperatura | **estructural**: todo veredicto lleva la reserva | §6.5; `CHANGELOG.md` 3·C |
| R-11 | Sustitución sin escenario de datos | «(no verificado)» | §7.5 |
| R-12 | Apagar sin respaldo | mitigado: arranque forzado en solo aviso hasta verificar | §7.5; `CHANGELOG.md` 4·A |

---

## 9. Qué mirar para actualizar este índice

1. `git -C <repo> log --oneline <ultimo-commit-indexado>..main` y `git branch -a`.
2. Las entradas nuevas de `CHANGELOG.md` `[Unreleased]` o una sección de release nueva.
3. Cambios en la línea `**Estado:**` de `SDD/Docs/05-Arquitectura-Tecnica/Adrs/*.md` (sobre todo ADR-21).
4. `SDD/Docs/07-Plan-Sprint/Mini-Plan-v1.0.md` §10 y §12.
5. Repetir el grep `pendiente|diferid|fuera de alcance|TODO` en `src/` y `tests/`.
