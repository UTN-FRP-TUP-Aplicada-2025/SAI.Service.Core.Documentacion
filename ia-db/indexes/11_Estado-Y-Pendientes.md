# 11 — Estado del proyecto y pendientes

> **Propósito**: qué está construido, qué está declarado pendiente y qué riesgos siguen abiertos. Es el
> índice que evita proponer trabajo ya hecho o dar por hecho lo que no existe.
> **Fuentes primarias**: [`CHANGELOG.md`](../../../SAI.Service.Core/CHANGELOG.md) ·
> [`SDD/Docs/07-Plan-Sprint/Mini-Plan-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/07-Plan-Sprint/Mini-Plan-v1.0.md) ·
> [`SDD/Docs/00-Contexto/Roadmap-Producto-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/00-Contexto/Roadmap-Producto-v1.0.md) ·
> [`Decisiones-Arquitectura-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Decisiones-Arquitectura-v1.0.md)
>
> **Vigencia**: instantánea del **2026-08-04**. El `CHANGELOG.md` es la fuente viva; si difiere, manda él.

---

## 1. Estado por etapa

| Etapa | Alcance | Estado |
| --- | --- | --- |
| Sprint 0 — Arranque | Andamiaje .NET 10, scripts, Dev Container, CI, panel base; cierre de 3 de las 4 decisiones abiertas (ADR-25/26/27) | **Completa** |
| Etapa 1 — Persistencia, alta de administrador y sesión | EF Core + SQLite, Identity, JWT ROPC, keyring de DataProtection | **Completa** |
| Etapa 2 — Alta de equipos y adaptador NUT | Dominio del ciclo de vida, adaptador NUT real + descubrimiento, wizard de alta | **Completa** |
| Etapa 3 — Monitoreo, salud e históricos | Sondeo y muestras, derivación de eventos, panel en vivo, prueba de batería y veredicto, históricos | **Completa** |
| Etapa 4 — Verificación y ciclo de vida | Bloqueo por verificación, apagado ordenado, recambio de batería, sustitución del SAI, políticas versionadas, informe de período | **Completa** |
| Etapa 5 — Integración | Ingesta idempotente `POST /api/v1/intervenciones` (cierra UF-10) | **Completa** |

Los tres módulos que eran *placeholder* en Sprint 0 (EP-04 políticas, EP-06 sustitución, EP-07 informe)
están implementados. Último incremento integrado: **tiempo de retorno del SAI configurable por política**
(antes fijo en 180 s dentro del adaptador NUT).

Versión publicada: `0.1.0` (2026-07-20, documentación e intake). Todo el software posterior está en
`[Unreleased]`. Rama de trabajo al indexar: `feature/politica-retardo-retorno`.

## 2. Pendientes declarados

### Decisiones abiertas

| Ítem | Estado | Qué falta |
| --- | --- | --- |
| **ADR-21** — contrato del endpoint de rectificación del 409 | **Propuesto** | El 409 devuelve `accionSugerida` como texto; el endpoint que la ejecuta no existe ni tiene contrato definido |

Las otras tres decisiones abiertas de Sprint 0 se cerraron (ADR-19→25, ADR-20→26, ADR-22→27).

### Infraestructura no materializada

| Ítem | Estado |
| --- | --- |
| `Dockerfile` / imagen de producción runtime-only | **No existe** en el repositorio; contrato de la imagen ya especificado en `Guia-Contenedor` |
| TLS de producción (certificado autofirmado en la LAN) | Ranura declarada en `appsettings.json`; no implementada (ADR-26 aceptada, sin materializar) |
| Anclaje del USB por regla `udev` | Es configuración de despliegue: **no vive en el repositorio** |

### Pipeline

Implementados en `ci.yml`: **stages 1–4** (restore, formato, build Release, tests).
Especificados pero **no implementados**: stage 5 (e2e con bUnit/Playwright), 6 (SCA), 7 (SBOM CycloneDX),
8 (build de imagen + smoke test), 9 (firma cosign keyless), 10 (publicación al registro privado).
Consecuencia práctica: los gates 5–10 de la Definition of Done **no se están ejerciendo automáticamente**.

### NFR sin respaldo numérico

| NFR | Qué falta |
| --- | --- |
| N-03 | Tiempo real del resto del apagado del sistema operativo — se cronometra en la ventana de mantenimiento |
| N-20 | Tamaño máximo del archivo SQLite tras la agregación — se valida antes de producción (riesgo R-07) |
| N-25 | SLO de disponibilidad del propio servicio — propuesto («rondas completadas / esperadas ≥ 0,99 mensual»), sin comprometer |

### Alcance diferido a v2

`US-25` (adaptador de conexión **directo**) y `US-26` (capa de add-ons de dialecto de protocolo): ambas
*Could*, **diseñadas pero no implementadas**, fuera del compromiso de v1.

## 3. La verificación física, que es lo que gobierna todo

Mientras los cuatro supuestos no se verifiquen en una **ventana de mantenimiento real** sobre el host, el
servicio permanece en `SoloAlerta` y **no apaga nada**. Eso incluye:

- `ver-shutdown-return` (el ciclo apagado→reencendido del firmware) — sin caducidad una vez verificado.
- `ver-bios-autoencendido` y `ver-flag-ob` — vigencia de 365 días.
- `ver-presupuesto-apagado` — vigencia de 180 días, y **su medición todavía no se hizo** (N-03).

No es un defecto ni un pendiente de código: es el diseño (ADR-10). Un panel que muestre los cuatro
supuestos en «no verificado» tras un despliegue limpio está funcionando correctamente.

## 4. Riesgos abiertos que condicionan decisiones

| ID | Riesgo | Severidad |
| --- | --- | --- |
| R-12 | Decidir apagar un servidor sin backups; si falla, falla de noche y sin testigos | Crítico |
| R-01 | La trampa de firmware podría dejar el SAI apagado indefinidamente | Crítico |
| R-02 | El presupuesto de 540 s no está medido contra el apagado real | Alto |
| R-09 | Sin sensor de temperatura, el confusor de la tendencia de salud no tiene solución con este equipo | Alto (para salud de batería) |
| R-13 | Presentar `battery.charge` (interpolado por el driver) como medido | Alto — mitigado por procedencia obligatoria |
| R-07 | Retención de métricas no probada contra el volumen real (~6,3 M filas/año) | Bajo |
| R-11 | La sustitución de SAI no tenía escenario de datos que la ejercitara | Bajo |

## 5. Al retomar el trabajo, verificar primero

1. `git log --oneline -5` y el bloque `[Unreleased]` del `CHANGELOG.md`: es la fuente de estado real y se
   mantiene con detalle por incremento.
2. `./scripts/build-all.sh` y `dotnet test SAI.Service.Core.sln -c Release` en verde.
3. `dotnet ef migrations has-pending-model-changes` sin cambios (si se tocó el modelo).
4. Si algo de este índice contradice al repositorio, **manda el repositorio**: corregir el índice
   afectado de forma incremental y actualizar la fecha del manifiesto en el [README](../README.md).
