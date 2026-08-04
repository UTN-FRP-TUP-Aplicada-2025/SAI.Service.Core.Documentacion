# 03 — Modelo de datos

> **Propósito**: qué se persiste, cómo se organiza en capas, qué protege la disciplina append-only y cómo
> se agrega o cambia el esquema.
> **Fuentes primarias**: [`SDD/Docs/05-Arquitectura-Tecnica/Modelo-Datos-Logico-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Modelo-Datos-Logico-v1.0.md) ·
> [`Infrastructure/Persistencia/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/Persistencia/) ·
> [`SaiDbContext.cs`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/Persistencia/SaiDbContext.cs)

---

## 1. Motor y disciplina

| Aspecto | Decisión |
| --- | --- |
| Motor | SQLite, archivo único (`sai.db` por defecto) — ADR-18 |
| ORM | EF Core; migraciones versionadas aplicadas **al arranque** (`MigrateAsync` en `Program.cs`) |
| Generación automática de esquema | **No existe**: todo cambio pasa por migración |
| Concurrencia de escritura | 1 (un solo proceso, un solo hilo escritor) |
| Fechas | Persistidas como binario (migración `FechasComoBinario`) para orden y comparación estables |
| Append-only | `InterceptorAppendOnly` bloquea `UPDATE`/`DELETE` sobre las entidades marcadas con `IEntidadHistoria` |

## 2. Las cuatro capas del modelo (ADR-07)

**Catálogo — qué es**: `Fabricante`, `ModeloDispositivo`, `ModeloBateria`, `TipoIntervencion`, `Proveedor`.

**Inventario — cuál es**: `UnidadFisica` con herencia **TPH** (una sola tabla, discriminador `Tipo` ∈
{`Host`, `Dispositivo`, `Bateria`}). Aporta ciclo de vida común (`estado`, `fechaBaja`, `motivoBaja`) y
**baja lógica**: no hay borrado físico; la unidad sigue consultable y no admite operaciones posteriores a
su baja. TPH se eligió sobre TPT/TPC porque las tres especializaciones comparten ciclo de vida, las
consultas recorren el inventario completo y el volumen es de decenas de filas; el costo (columnas
nulables por especialización) se refuerza con `CHECK` por discriminador.

**Vínculos temporales — qué estuvo con qué, cuándo**: `MontajeBateria` (batería↔dispositivo, con posición)
y `CoberturaHost` (dispositivo↔host). Sin solapamiento por clave; a lo sumo uno vigente (`hasta` nulo).

**Historia append-only — qué pasó**: `FuenteDatos`, `SesionSondeo`, `Muestra`, `Agregado`,
`ReglaDerivacion`, `Evento`, `PruebaBateria`, `Politica`/`VersionPolitica`, `Accion`, `Verificacion`,
`Intervencion`, `IntervencionIngerida`, más las puentes `VersionPoliticaVerificacion` e
`IntervencionBateria`.

**Proyección**: `FichaVidaUtil` (cierre de la historia de una batería: días en servicio, cumplimiento de
expectativa, eventos soportados, tendencia de salud y costo por año normalizado).

**Actor persistido**: `AspNetUsers` y tablas de Identity (administrador único, ADR-16).

**Objetos de valor** (no son tablas): `Valor<T>` con `Origen` y `Dinero`, mapeados como *owned types* y
materializados como columnas de la tabla dueña.

## 3. `DbSet` expuestos por `SaiDbContext`

| Propiedad | Entidad | Capa |
| --- | --- | --- |
| `Fabricantes`, `ModelosDispositivo`, `ModelosBateria` | catálogo | Catálogo |
| `Unidades` | `UnidadFisica` (TPH) | Inventario |
| `Montajes`, `Coberturas` | `MontajeBateria`, `CoberturaHost` | Vínculos |
| `Verificaciones`, `SesionesEjercicio` | `Verificacion`, `SesionEjercicio` | Historia |
| `FuentesDatos`, `SesionesSondeo`, `Muestras`, `Agregados`, `Reglas`, `Eventos`, `PruebasBateria` | monitoreo | Historia |
| `Acciones` | `Accion` | Historia |
| `Politicas` | `VersionPolitica` | Historia |
| `Intervenciones`, `Sustituciones`, `IntervencionesIngeridas`, `FichasVidaUtil` | ciclo de vida físico | Historia / Proyección |

Las configuraciones de mapeo viven en `Persistencia/Configuraciones/Modelo*.cs`
(`ModeloEquipos`, `ModeloMonitoreo`, `ModeloIntervenciones`, `ModeloSustituciones`, `ModeloPoliticas`,
`ModeloIngesta`).

## 4. Migraciones

15 migraciones aplicadas, en orden cronológico
([`Persistencia/Migraciones/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/Persistencia/Migraciones/)):

| # | Migración | Qué introduce |
| --- | --- | --- |
| 1 | `EsquemaInicialAcceso` | Identity y acceso del administrador único |
| 2 | `EsquemaEquipos` | Catálogo, inventario TPH, vínculos temporales, verificaciones |
| 3 | `EsquemaMonitoreo` | `FuenteDatos`, `SesionSondeo`, `Muestra`, `Agregado` |
| 4 | `EsquemaEventos` | `ReglaDerivacion` y `Evento` |
| 5 | `FechasComoBinario` | Cambia el almacenamiento de fechas a binario |
| 6 | `EsquemaSaludBateria` | `PruebaBateria` y veredicto de salud |
| 7 | `EsquemaAcciones` | `Accion` (apagado ordenado) |
| 8 | `EsquemaIntervenciones` | `Intervencion`, costos, `FichaVidaUtil` |
| 9 | `EsquemaPruebaApagado` | Prueba de apagado en ventana de mantenimiento |
| 10 | `EsquemaMedicionApagado` | Medición del presupuesto de apagado |
| 11 | `EsquemaSesionEjercicio` | `SesionEjercicio` (ejercicio guiado de verificación) |
| 12 | `EsquemaSustituciones` | `SustitucionSai` con cobertura suplente |
| 13 | `EsquemaPoliticas` | `VersionPolitica` (política versionada e inmutable) |
| 14 | `EsquemaIngesta` | `IntervencionIngerida` (idempotencia por clave) |
| 15 | `PoliticaTiempoRetorno` | Campo `TiempoRetornoSeg`; las políticas existentes toman 180 s |

**Cómo agregar una migración** (sin `dotnet` en el host — el SDK vive en el Dev Container o en un
contenedor SDK efímero `mcr.microsoft.com/dotnet/sdk:10.0`):

```bash
dotnet ef migrations add <Nombre> \
  --project    src/SAI.Service.Core/SAI.Service.Core.Infrastructure \
  --startup-project src/SAI.Service.Core/SAI.Service.Core.Infrastructure
# validación obligatoria antes de cerrar el cambio:
dotnet ef migrations has-pending-model-changes ...   # esperado: sin cambios
```

El `--startup-project` apunta a **Infrastructure** (es quien referencia `EntityFrameworkCore.Design` y
tiene `SaiDbContextFactory`); con `Web` falla. Ver EVE-02 en la bitácora de eventualidades.

> **Trampa conocida (EVE-06)**: al integrar dos ramas que agregaron una migración cada una, hay que
> **regenerar** la migración de la segunda sobre el modelo ya fusionado. Editar el `ModelSnapshot` a mano
> o aceptar el auto-merge de git deja el `Designer` mintiendo sobre el modelo.

## 5. Restricciones y patrones físicos

| Tipo | Ejemplos |
| --- | --- |
| CHECK | `tiempoReservadoApagadoSeg ≤ 540` (RN-04/I-10); coherencia de columnas por discriminador TPH |
| Únicas | Clave de idempotencia de la ingesta (RN-09/I-19); a lo sumo un vínculo temporal vigente por clave |
| FK notables | `Accion → VersionPolitica` (nunca a `Politica`, RC-05); `Evento → ReglaDerivacion` + versión (RC-09) |
| FK deliberadamente **ausente** | `IntervencionIngerida` **no** referencia `UnidadFisica`: registra un hecho externo de baja confianza que puede citar un dispositivo aún no dado de alta; la coherencia temporal se valida sobre las unidades conocidas |

## 6. Retención y volumen

| Dato | Resolución completa | Después | Retención total |
| --- | --- | --- | --- |
| `Muestra` | `P30D` | se agrega a `PT1H` y se descarta | 30 días |
| `Agregado` | `PT1H` | — | `P10Y` |
| `Evento` | — | — | indefinido |

Volumen dimensionado: **≈ 6,3 millones de filas/año** con sondeo a 5 s (720 muestras/hora). El tamaño
máximo del archivo tras la agregación está **pendiente de medición** (N-20, riesgo R-07). Multi-tenant no
aplica: un administrador, un host, un SAI activo.

## 7. Semilla al arranque

`Web/Program.cs`, tras `MigrateAsync()`, siembra de forma **idempotente**:

1. El rol `administrador` de Identity (el usuario se crea en `/alta-inicial`, primer arranque).
2. Las cuatro **reglas de derivación** v1: corte/microcorte (`microcorteMaxSeg=60`), tensión
   (`min=198`, `max=242`, `sostenidoSeg=30`), desconexión (`sondeosPerdidos=3`) y disparo
   (`umbralObSeg=300`, `batVoltMin=13.3`, `batVoltMax=14.5`).
3. La **política de apagado inicial** (v1) tomando `Sai:Apagado` como semilla — a partir de ahí la versión
   vigente en base es la fuente de verdad.
4. La **fuente de datos** `fd-gmao-externo` con confianza `Media` (para el header `X-Fuente-Datos`).
