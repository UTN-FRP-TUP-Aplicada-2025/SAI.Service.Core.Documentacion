# 02 — Dominio y reglas de negocio

> **Propósito**: el modelo de dominio y las reglas que lo gobiernan — qué entidades existen, qué
> enumeraciones cierran el vocabulario y qué regla de negocio respalda cada comportamiento.
> **Fuentes primarias**: [`src/SAI.Service.Core/SAI.Service.Core.Domain/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Domain/) ·
> [`SDD/Docs/02-Especificacion-Funcional/Reglas-De-Negocio/`](../../../SAI.Service.Core/SDD/Docs/02-Especificacion-Funcional/Reglas-De-Negocio/) ·
> [`Modelo-Datos/Modelo-Conceptual-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/02-Especificacion-Funcional/Modelo-Datos/Modelo-Conceptual-v1.0.md) ·
> [`reglas-conceptuales-de-modelo/`](../../../SAI.Service.Core/SDD/Docs/02-Especificacion-Funcional/Modelo-Datos/reglas-conceptuales-de-modelo/)

---

## 1. Áreas del dominio

| Área | Carpeta | Piezas principales |
| --- | --- | --- |
| Catálogo | `Domain/Catalogo/` | `Fabricante`, `ModeloDispositivo`, `ModeloBateria` |
| Inventario | `Domain/Inventario/` | `UnidadFisica` (base), `Host`, `Dispositivo`, `Bateria`, `EstadoUnidad` |
| Vínculos temporales | `Domain/Vinculos/` | `Vigencia`, `Vigencias`, `IVinculoTemporal`, `MontajeBateria`, `CoberturaHost`, `ResolutorTemporal` |
| Monitoreo | `Domain/Monitoreo/` | `Muestra`, `Evento`, `Agregado`, `SesionSondeo`, `ReglaDerivacion`, `DerivadorEventos`, `CalculadorAgregado`, `PruebaBateria`, `CalculadorSaludBateria`, `VeredictoSalud`, `Variables` |
| Verificaciones | `Domain/Verificaciones/` | `Supuesto`, `Verificacion`, `EstadoVerificacion`, `Modalidad`, `EvaluadorModalidad`, `SecuenciaFisica`, `SesionEjercicio`, `VigenciasSupuesto` |
| Acciones | `Domain/Acciones/` | `Accion`, `EstadoAccion` |
| Políticas | `Domain/Politicas/` | `VersionPolitica` |
| Intervenciones | `Domain/Intervenciones/` | `Intervencion`, `TipoIntervencionSai`, `IntervencionIngerida`, `SustitucionSai`, `Costos`, `DisposicionFinal`, `FichaVidaUtil` |
| Valores | `Domain/Valores/` | `Valor<T>`, `Origen`, `Dinero` |
| Historia | `Domain/Historia/` | `IEntidadHistoria` (marcador del interceptor append-only) |

## 2. Enumeraciones que cierran el vocabulario

Todas arrancan en **1** a propósito: el `default` (0) no es un valor válido, así que una entidad mal
construida es rechazable.

**`Origen`** — procedencia de todo valor (ADR-06, RN-05, invariante I-7):

| Valor | Significado |
| --- | --- |
| `Medido` | Medición directa del equipo o sensor |
| `Derivado` | Calculado a partir de otros valores |
| `Declarado` | Ingresado a mano por el administrador |
| `Imputado` | Relleno ante ausencia de dato, con criterio explícito |
| `EstimadoPorDriver` | El equipo no lo expone; lo interpola el driver (caso `battery.charge`) |
| `NoCalculable` | El valor queda nulo **pero con procedencia declarada** — nunca un número inventado |

Solo `Medido` y `Declarado` son aptos para tendencia de salud (RN-06).

**`Modalidad`** — respuesta ante un corte (ADR-09, ADR-10):

| Valor | Comportamiento |
| --- | --- |
| `SoloAlerta` | Estado base seguro: avisa, no apaga nada. Es también el degradado de RN-02 |
| `ApagarHostConRetorno` | Apaga el host de forma ordenada, con retorno al volver la energía |
| `ApagarHostLuegoUpsConRetorno` | Apaga el host y luego el propio SAI, ambos con retorno |
| `CicloForzado` | Iniciada la secuencia, el corte **no se cancela** aunque vuelva la red (nunca `shutdown.stop`) |

**`Supuesto`** — los cuatro supuestos de seguridad operativa (ADR-10):

| Supuesto | Qué afirma |
| --- | --- |
| `PresupuestoDeApagado` | El tiempo reservado alcanza para apagar el host antes de agotar la batería |
| `SenalEnBateria` | El equipo reporta de forma observable que pasó a batería |
| `ReencendidoPorPlaca` | La placa del host se enciende sola al restaurarse la energía (BIOS) |
| `CorteConRetorno` | El apagado ordenado con retorno reenciende el host tras volver la energía |

**`EstadoVerificacion`**: `NuncaVerificado` (sembrado en el alta) → `Verificado` (solo por evidencia) ·
`Vencido` (vuelve a exigir verificación) · `Refutado` (**bloqueo permanente**) · `PorVencer` (estado
**computado**, nunca persistido: sigue contando como vigente pero genera trabajo de renovación).

**`TipoEvento`**: `CorteSuministro` (OL→OB) · `RetornoRed` (OB→OL) · `Microcorte` · `DesconexionUsb`
(sondeos perdidos) · `TensionFueraDeRango` · `DisparoApagado`.

## 3. Reglas de negocio (RN)

Una por archivo en [`Reglas-De-Negocio/`](../../../SAI.Service.Core/SDD/Docs/02-Especificacion-Funcional/Reglas-De-Negocio/).

| RN | Regla | Dónde se materializa |
| --- | --- | --- |
| RN-01 | Arranque seguro en solo alerta | `VersionPolitica` inicial sembrada en `Web/Program.cs`; `Modalidad.SoloAlerta` |
| RN-02 | Bloqueo por verificación | `Domain/Verificaciones/EvaluadorModalidad.cs`; `Accion.ModalidadEfectiva` |
| RN-03 | Validación por efecto observado | `ResultadoAccion` del adaptador; `EstadoAccion` |
| RN-04 | Techo duro del tiempo de apagado (**540 s**) | `Accion.TechoDuroApagadoSeg`; validación en `ServicioPoliticas` + CHECK en base |
| RN-05 | Procedencia obligatoria de todo valor | `Valor<T>` con `Origen`; mapa de procedencia en `SesionSondeo` |
| RN-06 | Aptitud de datos para tendencia de salud | `CalculadorSaludBateria`; excluye `Derivado`/`EstimadoPorDriver`/`Imputado` |
| RN-07 | Todo importe con moneda y fecha | `Dinero` (moneda y fecha obligatorias) |
| RN-08 | Cuadre de costos de intervención | `Costos.Cuadra()`: total = repuestos + mano de obra |
| RN-09 | Idempotencia de la ingesta | `IntervencionIngerida` con índice único de clave + huella sha256 |
| RN-10 | Agregado con cobertura y advertencia | `Agregado` (cobertura obligatoria); `ServicioHistoricos` |
| RN-11 | Acción referida a versión de política | FK de `Accion` a `VersionPolitica`, nunca a `Politica` |
| RN-12 | Baja lógica y coherencia temporal | `UnidadFisica.AdmiteOperacionEn`; sin borrado físico |
| RN-13 | Vida de flotación con temperatura de referencia | `ModeloBateria` (la vida es inválida sin su temperatura) |

## 4. Reglas conceptuales del modelo (RC)

| RC | Regla |
| --- | --- |
| RC-01 | Procedencia por valor |
| RC-02 | Vigencia como entidad con intervalo |
| RC-03 | Sucesión de vínculos sin hueco |
| RC-04 | `Agregado` no hereda de `Muestra` |
| RC-05 | Acción referida a versión de política |
| RC-06 | Historia append-only |
| RC-07 | Resolución temporal de la batería |
| RC-08 | Baja lógica y coherencia temporal |
| RC-09 | Evento referido a regla versionada |

## 5. Los patrones de modelado que hay que entender

**Vínculo temporal en vez de FK directa (RC-07, ADR-05)** — una `Muestra` guarda `dispositivoId` e
`instante`, **no** la batería. La batería se resuelve por `MontajeBateria` con `ResolutorTemporal`.
Consecuencia práctica: corregir la fecha de un recambio **reatribuye todo el histórico** sin migrar datos.

**Intervalos semiabiertos** — `Vigencia` modela `[desde, hasta)` con `hasta` nulo = vigente. Sin
solapamiento por clave y a lo sumo uno vigente. `Vigencia.Intersecar` y `DiasEnPeriodo` recortan un
intervalo a un período (base de los informes), contando el fin abierto hasta el corte del informe.

**Procedencia declarada una sola vez** — la `SesionSondeo` lleva el mapa `variable → origen`; las muestras
lo heredan en vez de repetir la procedencia por columna.

**Historia append-only (ADR-04)** — una entidad de historia implementa `IEntidadHistoria` y queda
protegida por `InterceptorAppendOnly`: se insertan filas, nunca se actualizan ni se borran. Corregir un
dato es **agregar un hecho nuevo**.

**Modalidad efectiva ≠ modalidad solicitada** — la política vigente pide una modalidad; el
`EvaluadorModalidad` la degrada a `SoloAlerta` si los cuatro supuestos no están verificados y vigentes.
La `Accion` guarda las dos, de modo que el historial muestra qué se quiso hacer y qué se hizo.

**Veredicto de salud con confianza declarada (ADR-13)** — la salud es una **tendencia relativa** derivada
de la caída de tensión a carga igualada, no un porcentaje ni un SoH. Con menos de 4 pruebas comparables,
la confianza es `baja`. El lenguaje admitido es «se comporta peor que antes», nunca un número absoluto.

## 6. Invariantes

El dominio declara **21 invariantes (I-1..I-21)** en el modelo conceptual; se escriben como pruebas
unitarias antes de codificar (mitigación del riesgo R-10). Los citados con más frecuencia en el código:

| Invariante | Enunciado |
| --- | --- |
| I-7 | Procedencia obligatoria: no existe valor de dominio sin `Origen` |
| I-10 | El tiempo reservado de apagado nunca supera 540 s |
| I-18 | Todo `Dinero` lleva moneda y fecha |
| I-19 | Clave de idempotencia única en la ingesta |
| I-20 | Para `input.voltage`, el agregado conserva mínimo y máximo además del promedio |

La lista completa está en el modelo conceptual de la categoría 02; la cobertura de cada uno, en
[07_Calidad-Y-Pruebas](07_Calidad-Y-Pruebas.md).

## 7. Casos de uso y su cobertura de reglas

| CU | Título | RN aplicables |
| --- | --- | --- |
| CU-01 | Autenticación del administrador | RN-01 |
| CU-02 | Alta de equipos y puesta en marcha | RN-01, RN-05, RN-12, RN-13 |
| CU-03 | Configuración de políticas de apagado | RN-04, RN-11 |
| CU-04 | Monitoreo en vivo del estado del SAI | RN-03, RN-05 |
| CU-05 | Ejecución del apagado ordenado ante corte | RN-02, RN-03, RN-04, RN-11 |
| CU-06 | Históricos y gráficas de evolución | RN-10 |
| CU-07 | Prueba de batería y veredicto de salud | RN-06, RN-13 |
| CU-08 | Recambio de batería y ficha de vida útil | RN-07, RN-08, RN-12 |
| CU-09 | Reparación y sustitución del SAI | RN-12 |
| CU-10 | Ventana de mantenimiento y verificación | RN-01, RN-02, RN-03 |
| CU-11 | Ingesta automatizada de intervenciones | RN-07, RN-08, RN-09, RN-12 |
| CU-12 | Informe de período y comparación de marcas | RN-06, RN-07, RN-10 |

Cuerpo de cada CU en [`02-Especificacion-Funcional/Casos-De-Uso/`](../../../SAI.Service.Core/SDD/Docs/02-Especificacion-Funcional/Casos-De-Uso/);
qué servicio lo implementa, en [04_Aplicacion-Y-Casos-De-Uso](04_Aplicacion-Y-Casos-De-Uso.md).
