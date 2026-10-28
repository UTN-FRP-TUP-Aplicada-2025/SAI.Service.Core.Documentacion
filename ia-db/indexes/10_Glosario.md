# 10 — Glosario

> **Propósito:** vocabulario canónico de SAI.Service.Core: términos del dominio eléctrico/operativo, términos propios del modelo (procedencia, vigencia, supuestos, modalidades), nombres de enums y tipos del `Domain`, siglas técnicas y familias de identificadores del SDD. Cada entrada es breve y lleva su fuente; el detalle (reglas, invariantes, flujos) vive en los otros índices.
>
> **Fuentes primarias:** `SDD/Intake/SOLUTION-INTAKE-Sai-Service-Core-v1.0.md` §12 (glosario del dominio, el más extenso), `SDD/Docs/00-Contexto/Vision-Producto-v1.0.md` §9, `SDD/Docs/02-Especificacion-Funcional/Modelo-Datos/Modelo-Conceptual-v1.0.md` §6, `SDD/Docs/03-UX-UI-DX/Glosario-UX-v1.0.md` y los enums de `src/SAI.Service.Core/SAI.Service.Core.Domain/`.
>
> Rutas relativas a la raíz del repo `SAI.Service.Core`. Abreviatura en tablas: `Domain/…` = `src/SAI.Service.Core/SAI.Service.Core.Domain/…`; `Application/…`, `Infrastructure/…`, `Web/…` análogo.

---

## 0. Glosarios fuente y su jerarquía

| Glosario | Ruta | Alcance | Nota |
|---|---|---|---|
| Glosario del dominio del cliente (§12) | `SDD/Intake/SOLUTION-INTAKE-Sai-Service-Core-v1.0.md` | 24 términos del dominio, con valores reales del equipo relevado | Fuente canónica; los demás lo reusan por referencia |
| Glosario del dominio (§9) | `SDD/Docs/00-Contexto/Vision-Producto-v1.0.md` | 10 términos, versión resumida del §12 | — |
| Glosario (§6) | `SDD/Docs/02-Especificacion-Funcional/Modelo-Datos/Modelo-Conceptual-v1.0.md` | 8 términos del modelo; remite al intake §12 | — |
| Glosario rápido (§8) | `SDD/Docs/README.md` | 16 términos de una línea; suma «Equipos», «Adaptador de conexión», «Política de apagado» | Remite al intake §12 y a la categoría 03 |
| Glosario UX | `SDD/Docs/03-UX-UI-DX/Glosario-UX-v1.0.md` (v1.1) | Superficie, estado, configuración por esquema, aprovisionamiento, sello de versión | No redefine términos del dominio |
| Comentarios XML de enums y tipos | `Domain/**/*.cs` | Nombres canónicos en código | Ver §3 y §4 |

```text
Intake §12 ──► Visión §9 (resumen)
     ├──► SDD/Docs/README.md §8 (glosario rápido)
     │
     ├──► Modelo-Conceptual §6 (términos del modelo)
     └──► Glosario-UX (sólo vocabulario de experiencia; referencia, no duplica)
```

---

## 1. Términos del dominio eléctrico y operativo

| Término | Definición breve | Sinónimos / notas | Fuente |
|---|---|---|---|
| **SAI** | Sistema de Alimentación Ininterrumpida: sostiene la alimentación del host cuando falla la red. El relevado es *offline / line interactive* | UPS | Intake §12; Visión §9 |
| **Host** (servidor protegido) | Máquina cuya continuidad resguarda el servicio; su cobertura por un SAI es un vínculo temporal | — | `Domain/Inventario/Host.cs` |
| **Equipos** | Las unidades físicas que administra la solución: host protegido, SAI y batería (término que reemplazó a «parque» en Glosario-UX v1.1) | — | `SDD/Docs/README.md` §8; Glosario-UX §7 |
| **Apagado ordenado** | Apagar el SO deteniendo contenedores y sincronizando discos **antes** de perder la alimentación | — | Intake §12 |
| **Reencendido automático** | Que el host arranque solo al volver la energía; requiere una **transición** ausencia→presencia, por eso el SAI debe cortar su salida aunque el host ya esté apagado | Autoencendido; BIOS «Restore on AC Power Loss» | Intake §12; `Application/Abstractions/IAdaptadorConexion.cs` |
| **Ciclo forzado** | Modalidad en la que, iniciada la secuencia, el corte del SAI **no se cancela** aunque vuelva la red (nunca se emite `shutdown.stop`) | «Mandatory Power Cycle» (CyberPower) | Intake §12 y CL-01; `Domain/Verificaciones/Modalidad.cs`; `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-09-Modalidad-Ciclo-Forzado-De-Apagado-v1.0.md` |
| **Apagado con retorno** | Orden al SAI de cortar su salida tras un retardo (ventana para que baje el host) y restaurarla tras otro retardo una vez vuelta la red | `shutdown.return` (NUT) | `Application/Abstractions/IAdaptadorConexion.cs` (`OrdenarApagadoConRetornoAsync`); `Infrastructure/Adaptadores/Nut/AdaptadorConexionNut.cs` |
| **Corte de suministro** | OL→OB→OL con duración ≥ 60 s | `CorteSuministro` | Intake §12; `Domain/Monitoreo/TipoEvento.cs` |
| **Microcorte** | Parpadeo breve: OL→OB→OL en < 60 s (regla v2; la v1 usaba 30 s, no mezclar sin normalizar) | `Microcorte`; parámetro `microcorteMaxSeg` (= 60 en la siembra de reglas) | Intake §12; `Domain/Monitoreo/DerivadorEventos.cs`; `Web/Program.cs` |
| **Retorno de la red** | El equipo vuelve a línea (OB→OL) | `RetornoRed` | `Domain/Monitoreo/TipoEvento.cs` |
| **Flotación** | Carga permanente con que el cargador mantiene la batería; medida 13,41 V (2,235 V/celda). Informa sobre el cargador, no la batería | *Float* | Intake §12 |
| **Vida de flotación** | Vida útil declarada de un modelo de batería; sólo vale con temperatura de referencia (I-21) | `VIDA_FLOTACION_SIN_TEMPERATURA` (código de `Application/Equipos/ServicioAltaEquipos.cs`) | `Domain/Catalogo/ModeloBateria.cs`; `SDD/Docs/02-Especificacion-Funcional/Reglas-De-Negocio/RN-13-Vida-De-Flotacion-Con-Temperatura-De-Referencia-v1.0.md` |
| **Autonomía** | Tiempo que el SAI sostiene la carga. **No se mide** en este equipo (`battery.runtime` no existe → `noCalculable`); no sirve de umbral | *Runtime* | Intake §12; `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-12-Disparo-Sin-Dependencia-Del-Flag-Lb-v1.0.md` |
| **Flag LB** | *Low battery* del firmware; nunca observado en el equipo: el disparo **no** depende de él | — | ADR-12; `Domain/Monitoreo/DerivadorEventos.cs` |
| **OL / OB** | Estados de `ups.status`: *on line* (en línea) / *on battery* (en batería); codificados 0 / 1 en las lecturas | `EstadoUps.EnLinea/EnBateria` | `Domain/Monitoreo/Variables.cs`; `Application/Abstractions/EstadoUps.cs` |
| **Disparo del apagado** | Se cumple la condición de apagado: tiempo en OB (`umbralObSeg`) + `battery.voltage`; nunca LB ni `battery.charge` | `DisparoApagado`; `regla-disparo` | ADR-12; `Domain/Monitoreo/DerivadorEventos.cs`; `Domain/Monitoreo/TipoEvento.cs` |
| **Techo duro (540 s)** | Tope máximo del tiempo reservado para apagar; no se puede construir una versión de política que lo supere (I-10) | `TechoDuroApagadoSeg`; `TIEMPO_APAGADO_EXCEDE_TECHO` (código de `Application/Politicas/ServicioPoliticas.cs`) | `Domain/Acciones/Accion.cs`; `Domain/Politicas/VersionPolitica.cs`; RN-04 |
| **Salud de batería** | Degradación relativa al estado nuevo; **no** es porcentaje ni SoH: tendencia de la caída de tensión en el autotest a carga igualada | SoH evitado a propósito | Intake §12; `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-13-Metodo-De-Salud-Por-Caida-De-Tension-v1.0.md` |
| **Línea base** | Primera medición de referencia de la prueba de batería (−0,47 V, mín. 12,94 V, recuperación 13,24 V en ~35 s, carga 13 %, 2026-07-19) | *Baseline* | Intake §12 |
| **Carga igualada / comparable** | Condición para que una prueba entre en la tendencia: carga concurrente comparable a la línea base (I-16) | `PruebaBateria.Comparable` | `Domain/Monitoreo/PruebaBateria.cs`; RN-06 |
| **Reserva de temperatura** | Advertencia fija del veredicto: «sin corrección por temperatura (sin sensor)» | `ReservaTemperatura` | `Domain/Monitoreo/CalculadorSaludBateria.cs` |
| **AGM / VRLA** | Plomo-ácido con electrolito absorbido, regulada por válvula; enum cerrado del modelo: `AGM`, `GEL`, `Inundada` | En código no hay enum: `ModeloBateria.Tecnologia` es texto libre (p. ej. "VRLA-AGM") | Intake §12; `Domain/Catalogo/ModeloBateria.cs` |
| **Capacidad C20** | Capacidad a régimen de descarga de 20 h; magnitud comparable entre modelos | — | Intake §12 |
| **Días sin protección** | Días en que el host no tuvo SAI: hueco entre dos `CoberturaHost`; el informe lo reporta como `diasSinProteccion` | — | Intake §12; `Application/Intervenciones/CalculadorProteccion.cs` |
| **Ventana de mantenimiento** | Ejercicio destructivo con presencia física (UF-8 / CU-10) donde se verifican los supuestos cortando la energía | — | Intake §6; `SDD/Docs/02-Especificacion-Funcional/Casos-De-Uso/CU-10-Ventana-De-Mantenimiento-Y-Verificacion-v1.0.md` |
| **Ejercicio guiado** | Sesión persistida de la ventana de mantenimiento que sobrevive al reinicio del host; el paso se deriva de las verificaciones | `SesionEjercicio` | `Domain/Verificaciones/SesionEjercicio.cs`; `Application/Equipos/ServicioEjercicioGuiado.cs` |
| **Secuencia física** | Orden real de los supuestos en el ejercicio: señal en batería → presupuesto de apagado → corte con retorno → reencendido por placa (≠ orden de declaración del enum) | `SecuenciaFisica` | `Domain/Verificaciones/SecuenciaFisica.cs` |
| **Disposición final** | A dónde fue y quién recibió la batería/equipo retirado (trazabilidad ambiental) | `DisposicionFinal` | `Domain/Intervenciones/DisposicionFinal.cs` |
| **Ficha de vida útil** | Proyección que cierra la historia de una batería: duración real, eventos, cumplimiento de expectativa y costo por año de servicio | `FichaVidaUtil` | Intake §12; `Domain/Intervenciones/FichaVidaUtil.cs` |
| **Costo por año normalizado** | Costo por año de servicio en moneda original + equivalente USD marcado `Derivado` con fuente de cotización | — | `Domain/Intervenciones/FichaVidaUtil.cs`; RN-07 |
| **GMAO** | Sistema externo de gestión de mantenimiento que empuja intervenciones por la API de ingesta, con confianza media | Sistema externo; fuente `fd-gmao-externo` | `Domain/Intervenciones/IntervencionIngerida.cs`; `SDD/Docs/README.md` §9 |

---

## 2. Términos propios del modelo

| Término | Definición breve | Fuente |
|---|---|---|
| **Procedencia** | Origen declarado de todo valor almacenado; responde «¿lo midió el aparato o lo calculó el software?». Seis valores (ver §3) | Intake §12; `Domain/Valores/Origen.cs`; `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-06-Procedencia-Obligatoria-En-Todo-Valor-v1.0.md` |
| **Valor con origen** | Objeto de valor `Valor<T>` que ata contenido + `Origen`; no se construye sin procedencia (I-7) | `Domain/Valores/Valor.cs` |
| **Procedencia canónica** | Mapa variable→origen guardado una vez por sesión de sondeo (no por muestra) | `Domain/Monitoreo/Variables.cs`; `Domain/Monitoreo/SesionSondeo.cs` |
| **Dinero** | Importe que siempre viaja con moneda y fecha (I-18) | `Domain/Valores/Dinero.cs`; RN-07 |
| **Cuadre de costos** | Total = repuestos + mano de obra, misma moneda; si no, `COSTOS_NO_CUADRAN` | `Domain/Intervenciones/Costos.cs`; RN-08 |
| **Vínculo temporal** | Relación «qué estuvo con qué y cuándo» modelada como entidad con intervalo (`MontajeBateria`, `CoberturaHost`) | Intake §12; `Domain/Vinculos/IVinculoTemporal.cs`; ADR-05 |
| **Vigencia** | Intervalo semiabierto `[Desde, Hasta)`; `Hasta` nulo = **vigente**, nunca «desconocido» | `Domain/Vinculos/Vigencia.cs`; RC-02 |
| **Sucesión sin hueco** | El instante de cierre pertenece al vínculo siguiente: dos vínculos que se tocan no solapan | `Domain/Vinculos/Vigencia.cs`; RC-03 |
| **Clave de exclusividad** | Clave por la que dos vínculos no pueden solapar: `(dispositivo, posición)` o `host` | `Domain/Vinculos/Vigencias.cs`; `Domain/Vinculos/MontajeBateria.cs`; `Domain/Vinculos/CoberturaHost.cs` |
| **Resolución temporal** | Resolver qué vínculo regía en un instante; la historia densa guarda dispositivo+instante, nunca la batería | `Domain/Vinculos/ResolutorTemporal.cs`; RC-07 |
| **Montaje congelado** | La prueba de batería resuelve el montaje una sola vez y no lo recalcula (I-15) | `Domain/Monitoreo/PruebaBateria.cs` |
| **Catálogo / Inventario / Historia** | «Qué es» (tipos: `Fabricante`, `ModeloDispositivo`, `ModeloBateria`) / «cuál es» (`UnidadFisica`: `Host`, `Dispositivo`, `Bateria`) / hechos registrados | `Domain/Catalogo/`, `Domain/Inventario/`; ADR-07 |
| **Unidad física** | Supertipo del inventario con máquina de estados y baja lógica | `Domain/Inventario/UnidadFisica.cs` |
| **Baja lógica** | Retiro con `estado` + `fechaBaja` + `motivoBaja`, sin borrado; sigue consultable (I-5). El borrado físico no existe | Intake §12; RC-08; RN-12 |
| **Historia append-only** | Entidades que se insertan y nunca se modifican ni borran; corregir = hecho nuevo. Marcador `IEntidadHistoria`, guardado por `InterceptorAppendOnly` | `Domain/Historia/IEntidadHistoria.cs`; `Infrastructure/Persistencia/InterceptorAppendOnly.cs`; ADR-04; RC-06 |
| **Bitemporalidad** | Dos tiempos por hecho: válido (cuándo ocurrió) y registrado (cuándo se supo) | Intake §12; `Domain/Intervenciones/IntervencionIngerida.cs` (`TiempoValido`/`TiempoRegistrado`); `Domain/Intervenciones/Intervencion.cs` (`InstanteOcurrido`/`InstanteRegistrado`) |
| **Idempotencia de ingesta** | Reenviar un hecho no lo duplica, por clave del emisor (`X-Idempotency-Key`); la huella sha256 del cuerpo distingue reintento de conflicto | Intake §12; `Domain/Intervenciones/IntervencionIngerida.cs`; RN-09; ADR-17 |
| **Sesión de sondeo** | Contexto de una serie de muestras: driver, dialecto, intervalo y mapa de procedencia | `Domain/Monitoreo/SesionSondeo.cs` |
| **Muestra** | Lectura puntual del SAI por el sondeo, con su calidad (§3) | `Domain/Monitoreo/Muestra.cs` |
| **Agregado** | Resumen de una ventana (`PT1H`, `P1D`) con mín/máx/promedio y **cobertura**; tipo distinto de `Muestra` (no hereda) | Intake §12; `Domain/Monitoreo/Agregado.cs`; ADR-08; RC-04 |
| **Cobertura (de agregado)** | Fracción de muestras con dato en la ventana; < 0,95 lleva advertencia (I-20) | `Domain/Monitoreo/CalculadorAgregado.cs`; RN-10 |
| **Regla de derivación** | Regla versionada que deriva eventos; la versión es parte de la identidad y cada evento graba (código, versión) | `Domain/Monitoreo/ReglaDerivacion.cs`; RC-09 |
| **Evento** | Hecho derivado del sondeo (§3, `TipoEvento`), con incertidumbre estructural de duración si es más breve que el sondeo | `Domain/Monitoreo/Evento.cs` |
| **Fuente de datos** | De dónde provienen las lecturas o hechos, con confianza base | `Domain/Monitoreo/FuenteDatos.cs`; cabecera `X-Fuente-Datos` (ADR-17) |
| **Política / versión de política** | La política se configura creando una versión inmutable nueva; la vigente es la de mayor número; toda acción refiere a una versión (I-13) | `Domain/Politicas/VersionPolitica.cs`; RN-11; RC-05 |
| **Acción** | Registro append-only de lo hecho ante un disparo: modalidad solicitada, efectiva y estado por efecto observado | `Domain/Acciones/Accion.cs` |
| **Modalidad solicitada / efectiva** | La pedida en la política vs. la que rige tras el bloqueo por verificación | `Domain/Verificaciones/EvaluadorModalidad.cs`; Intake §12 |
| **Supuesto** | Afirmación de la que depende la política de apagado (p. ej. «la BIOS reenciende el host»); cuatro, ver §3 | `Domain/Verificaciones/Supuesto.cs`; ADR-10 |
| **Verificación** | Estado vivo (no append-only) de un supuesto con evidencia, método, fecha y vigencia; se siembra en `NuncaVerificado` al dar de alta | `Domain/Verificaciones/Verificacion.cs`; `Application/Equipos/ServicioAltaEquipos.cs` |
| **Bloqueo por verificación** | Mientras los cuatro supuestos no estén verificados **y vigentes**, la modalidad efectiva degrada a `SoloAlerta` (I-11) | `Domain/Verificaciones/EvaluadorModalidad.cs`; RN-02; ADR-10 |
| **Arranque seguro** | El servicio arranca siempre en solo aviso | RN-01; ADR-10 |
| **Validación por efecto observado** | Una acción sólo se da por hecha si se observa su efecto; nunca por ausencia de error | ADR-11; RN-03; `Domain/Acciones/EstadoAccion.cs` |
| **Verificación por comportamiento** | El ajuste de BIOS no se lee: se prueba cortando energía y observando el arranque; se renueva por evidencia | ADR-14 |
| **Preaviso / por vencer** | Verificación vigente que vence dentro del umbral de preaviso: estado efectivo computado, no persistido | `Domain/Verificaciones/EstadoVerificacion.cs`; `Domain/Verificaciones/Verificacion.cs` |
| **Puesta en marcha (estado)** | Modalidad efectiva, supuestos verificados/totales (4) y si hay equipos; `Degradado` = efectiva `SoloAlerta` | `Application/Equipos/EstadoPuestaEnMarcha.cs` |
| **Sucesión de coberturas** | Línea de tiempo del host con tramos cubiertos y huecos «sin protección» | `Application/Intervenciones/CalculadorProteccion.cs` |
| **Cobertura suplente** | SAI que cubre al host mientras el titular está en reparación o tras su reemplazo | `Domain/Intervenciones/SustitucionSai.cs`; CU-09 |
| **Único proceso escritor** | Un solo proceso escribe la base; aplica la regla de exclusividad antes de abrir vínculos | `Domain/Vinculos/Vigencias.cs`; ADR-04; ADR-18 |

---

## 3. Enums del Domain (valores canónicos)

Todos arrancan en 1: el `default` (0) no es válido (lo comenta cada archivo salvo `Supuesto.cs` y `SesionEjercicio.cs`, donde sólo surge de los valores).

| Enum | Valores | Significado clave | Ruta |
|---|---|---|---|
| `Origen` (procedencia) | `Medido`, `Derivado`, `Declarado`, `Imputado`, `EstimadoPorDriver`, `NoCalculable` | `NoCalculable`/`Imputado` dejan valor nulo con origen declarado («nunca un número inventado»). Persistido como `medido \| derivado \| estimadoPorDriver \| declarado \| imputado \| noCalculable` | `Domain/Valores/Origen.cs`; `SDD/Docs/05-Arquitectura-Tecnica/Modelo-Datos-Logico-v1.0.md` |
| `Modalidad` | `SoloAlerta`, `ApagarHostConRetorno`, `ApagarHostLuegoUpsConRetorno`, `CicloForzado` | `SoloAlerta` = solo aviso, estado base y degradado seguro | `Domain/Verificaciones/Modalidad.cs` |
| `Supuesto` | `PresupuestoDeApagado`, `SenalEnBateria`, `ReencendidoPorPlaca`, `CorteConRetorno` | Los cuatro supuestos de seguridad operativa | `Domain/Verificaciones/Supuesto.cs` |
| `EstadoVerificacion` | `NuncaVerificado` (persistido `sinVerificar`), `Verificado`, `Vencido`, `Refutado`, `PorVencer` (efectivo) | `Refutado` bloquea permanente; `Vencido` sólo pide repetir | `Domain/Verificaciones/EstadoVerificacion.cs` |
| `EstadoSesionEjercicio` | `EnCurso`, `Completada`, `Abandonada` | Completada = los cuatro supuestos vigentes | `Domain/Verificaciones/SesionEjercicio.cs` |
| `EstadoAccion` | `BloqueadaPorVerificacion`, `SoloAviso`, `Ejecutada`, `EfectoNoConfirmado` | Resultado de la acción por efecto observado | `Domain/Acciones/EstadoAccion.cs` |
| `TipoEvento` | `CorteSuministro`, `RetornoRed`, `Microcorte`, `DesconexionUsb`, `TensionFueraDeRango`, `DisparoApagado` | `DesconexionUsb` = sondeos perdidos consecutivos (pérdida de comunicación) | `Domain/Monitoreo/TipoEvento.cs` |
| `CalidadMuestra` | `Completa`, `Parcial`, `Perdida` | Huecos por variable, no sólo por muestra | `Domain/Monitoreo/CalidadMuestra.cs` |
| `VeredictoSalud` | `SinDegradacionDetectable`, `Degradada`, `DegradacionSevera` | Tendencia relativa, nunca SoH %, Ah ni autonomía | `Domain/Monitoreo/VeredictoSalud.cs` |
| `ConfianzaVeredicto` | `Baja`, `Media`, `Alta` | Arranca baja; con ≥ 4 comparables sube a `Media` (el calculador nunca asigna `Alta`) | `Domain/Monitoreo/ConfianzaVeredicto.cs`; `Domain/Monitoreo/CalculadorSaludBateria.cs` |
| `ConfianzaFuente` | `Alta`, `Media`, `Baja` | Confianza base de una fuente de datos (ojo: orden inverso a `ConfianzaVeredicto`) | `Domain/Monitoreo/ConfianzaFuente.cs` |
| `EstadoUnidad` | `EnStock`, `EnServicio`, `EnReparacion`, `DadoDeBaja` | Ciclo de vida; no salta pasos (I-6); `DadoDeBaja` terminal | `Domain/Inventario/EstadoUnidad.cs` |
| `TipoIntervencionSai` | `Reparacion`, `Reemplazo` | Deja al saliente en reparación o dado de baja | `Domain/Intervenciones/TipoIntervencionSai.cs` |
| `EstadoUps` (Application) | `EnLinea`, `EnBateria` | Abstracción de `ups.status` OL/OB | `Application/Abstractions/EstadoUps.cs` |

### 3.1 Supuestos: nombre en código, código SDD y vigencia

| `Supuesto` (código) | Código en SDD | Qué afirma | Vigencia | Orden físico |
|---|---|---|---|---|
| `SenalEnBateria` | `ver-flag-ob` | El equipo reporta observablemente que pasó a batería | 365 d | 1 |
| `PresupuestoDeApagado` | `ver-presupuesto-apagado` | El tiempo reservado alcanza para apagar el host | 180 d | 2 |
| `CorteConRetorno` | `ver-shutdown-return` | El apagado con retorno repone la salida y reenciende | sin caducidad | 3 |
| `ReencendidoPorPlaca` | `ver-bios-autoencendido` | La placa arranca sola al volver la energía | 365 d | 4 |

Fuentes: `Domain/Verificaciones/Supuesto.cs`, `Domain/Verificaciones/VigenciasSupuesto.cs`, `Domain/Verificaciones/SecuenciaFisica.cs`, ADR-14, `SDD/Docs/08-Calidad-Y-Pruebas/Casos-Prueba-Referenciales-v1.0.md`, `SDD/Docs/08-Calidad-Y-Pruebas/Matriz-Cobertura-Pruebas-v1.0.md` §3 (N-13..N-15). La correspondencia nombre-de-código ↔ código SDD es por nombre y vigencia coincidentes (no verificado como tabla explícita en una fuente única). En código, el código real de la verificación sembrada es `ver-{dispositivo}-{supuesto}` (`Application/Equipos/ServicioAltaEquipos.cs`).

### 3.2 Divergencia de nombres de modalidad (doc vs código)

| Documentación SDD / intake | Código (`Modalidad.cs`) |
|---|---|
| `SoloAlerta` | `SoloAlerta` |
| `SoloHost` | `ApagarHostConRetorno` |
| `HostLuegoUpsConRetorno` | `ApagarHostLuegoUpsConRetorno` |
| `CicloForzado` | `CicloForzado` |

Fuentes doc: Intake §12, ADR-09, `SDD/Docs/03-UX-UI-DX/Wireframes-Configuracion-De-Politicas-v1.0.md`, US-06. El `Contrato-Datos-Maqueta-v1.0.md` exige «enum en su forma canónica de dominio». Que la equivalencia `SoloHost` ≡ `ApagarHostConRetorno` sea semánticamente exacta (el código agrega «con retorno») queda **(no verificado)**. «Solo aviso» y «solo alerta» se usan como sinónimos (`EstadoAccion.SoloAviso` vs `Modalidad.SoloAlerta`).

---

## 4. Variables sensadas (nombres NUT) y su procedencia

| Constante | Variable NUT | Unidad | Procedencia canónica | Fuente |
|---|---|---|---|---|
| `TensionEntrada` | `input.voltage` | V | Medido | `Domain/Monitoreo/Variables.cs` |
| `TensionSalida` | `output.voltage` | V | Medido | ídem |
| `CargaSalida` | `ups.load` | % | Medido | ídem |
| `CargaBateria` | `battery.charge` | % | **Derivado** (interpolación del driver, nunca medido) | ídem |
| `EstadoUps` | `ups.status` | 0 = OL / 1 = OB | Medido | ídem |
| `TensionBateria` | `battery.voltage` | V | Medido; base del disparo | ídem |
| — | `battery.runtime` | — | No existe en el equipo → `noCalculable` | Intake §12 |

Códigos de regla de derivación: `regla-corte`, `regla-tension`, `regla-desconexion`, `regla-disparo` (`Domain/Monitoreo/DerivadorEventos.cs`). Constantes de salud: `MuestrasMinimas = 3`, `ComparablesParaSubirConfianza = 4`, `UmbralDegradacion = 1.2` (`Domain/Monitoreo/CalculadorSaludBateria.cs`). El significado exacto de `UmbralDegradacion` está en `02_Dominio-Y-Reglas.md`.

---

## 5. Términos técnicos propios

| Término | Definición breve | Fuente |
|---|---|---|
| **NUT** | *Network UPS Tools*: ecosistema libre que dialoga con el equipo; componentes `upsmon`, `upssched`, `upsc`, `upsd`, `nutdrv_qx` | Intake §12; ADR-01 |
| **NUT en el contenedor** | NUT (`nutdrv_qx` + `upsd`) corre dentro del contenedor del servicio, único consumidor del USB | `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-25-Nut-En-El-Contenedor-v1.0.md` |
| **Dialecto de protocolo** | Variante concreta que habla un equipo; aquí `Voltronic-QS-Hex 0.10` (familia Megatec/Qx, serie-sobre-HID). «El firmware manda» | Intake §12; ADR-01 |
| **Add-on de dialecto** | Capa de extensión para dialectos de protocolo | `SDD/Docs/06-Backlog-Tecnico/historias-usuario/US-26-Capa-De-Add-Ons-De-Dialecto-De-Protocolo-v1.0.md` |
| **Adaptador de conexión** | Puerto `IAdaptadorConexion` que aísla el dominio del «cómo» se habla con el equipo; tres implementaciones: NUT, directo (diseñado, no implementado), simulado | `Application/Abstractions/IAdaptadorConexion.cs`; ADR-02; ADR-22; ADR-27 |
| **Adaptador simulado** | Implementación sin hardware para probar políticas; puede forzar OB (`Sai:Simulado:EnBateria`) | `Infrastructure/Adaptadores/AdaptadorConexionSimulado.cs` |
| **Adaptador directo** | Implementación sin NUT, planificada | US-25 |
| **Anclaje del USB por ruta física** | Regla `udev` sobre `ID_PATH` del puerto (no por serial) compartida con el contenedor | `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-03-Anclaje-Del-Usb-Por-Ruta-Fisica-De-Puerto-v1.0.md` |
| **Rearme de pruebas** | Al arrancar el servicio, limpia las verificaciones que esperaban un reinicio del host | `Infrastructure/Monitoreo/ServicioRearmePruebas.cs` |
| **Planificador** | Planificador interno de sondeo (rondas por intervalo, evaluación de políticas, temporizadores con cancelación); hoy es sólo un contrato marcador sin implementación | `Application/Abstractions/IPlanificador.cs`; ADR-15 |
| **Rectificación** | Endpoint para corregir hechos ingeridos; contrato **pendiente** (P-05) | `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-21-Contrato-Del-Endpoint-De-Rectificacion-v1.0.md` |
| **problem+json** | Formato de error RFC 7807 de la API de ingesta | ADR-17 |
| **ROPC / Bearer JWT** | La API se autentica con JWT obtenido por usuario+contraseña en `POST /api/v1/token`; la cookie sirve sólo al panel | `SDD/Docs/05-Arquitectura-Tecnica/Adrs/ADR-28-Autenticacion-De-La-Api-Con-Bearer-Jwt-Via-Ropc-v1.0.md` |
| **Administrador único** | Única identidad del sistema (propietario, implementador y beneficiario) | ADR-16; `SDD/Docs/00-Contexto/README.md` |
| **TLS autofirmado en Kestrel** | HTTPS servido por el propio proceso, sin reverse proxy | ADR-20; ADR-26 |
| **Keyring de DataProtection** | Claves de ASP.NET persistidas para sobrevivir reinicios | ADR-29 |
| **Clean Architecture en cinco assemblies** | `Domain`, `Application`, `Infrastructure`, `Api`, `Web` como capas de un único proyecto | ADR-15; `Domain/DominioMarcador.cs` |
| **Sello de versión** | Versión legible al pie de la superficie; la calcula MinVer (`Directory.Build.props`) | `Web/SelloVersion.cs`; Glosario-UX §5 |

Para el detalle de estos componentes ver `06_Infraestructura-Y-Adaptador-Sai.md`, `05_Interfaces-Panel-Y-Api.md` y `01_Arquitectura.md`.

---

## 6. Vocabulario UX (resumen)

Fuente única: `SDD/Docs/03-UX-UI-DX/Glosario-UX-v1.0.md`. Detalle de superficies en `05_Interfaces-Panel-Y-Api.md`.

| Grupo | Términos |
|---|---|
| Superficie y navegación | superficie, nombre canónico, vista, panel (≠ «panel de control» = producto), shell partido (acceso / trabajo), barra de identidad |
| Estado y feedback | estados vacío/cargando/con datos/error, skeleton, toast, banda de resultado, **banner de bloqueo** (política degradada a solo aviso), chip de estado, **alerta de conectividad** (3 sondeos sin respuesta), rechazo indiferenciado |
| Configuración por esquema | descriptor de parámetro, ayuda contextual, divulgación progresiva, preset, modo simulación, `PropuestaDeConfiguracion`, ranura del asistente, configuración de aplicación vs. de entorno |
| Primer arranque | predicado de aprovisionamiento (existe un administrador), guard en tres capas, orientación posterior |
| Identidad de versión | sello de versión, distintivo de artefacto preliminar, marcador de origen indeterminado |

Unificación v1.1 de términos: ver §8.

---

## 7. Familias de identificadores del SDD

| Prefijo | Significa | Rango / ubicación |
|---|---|---|
| `NB-` | Necesidad de negocio | NB-01..08 (NB-09 sólo como propuesta en Audit Fase B), `SDD/Docs/01-Necesidades-Negocio/Necesidades-De-Negocio/` |
| `CU-` | Caso de uso | CU-01..12 (CU-13..19 sólo en la granularidad previa, subsumidos: `Especificacion-Funcional-v1.0.md` §6), `SDD/Docs/02-Especificacion-Funcional/Casos-De-Uso/` |
| `RN-` | Regla de negocio | RN-01..13, `.../Reglas-De-Negocio/` |
| `RC-` | Regla conceptual de modelo | RC-01..09, `.../Modelo-Datos/reglas-conceptuales-de-modelo/` |
| `I-` | Invariante de dominio | I-1..I-21, definidos en Intake §21.B.2; cruzados con pruebas en `SDD/Docs/08-Calidad-Y-Pruebas/Matriz-Cobertura-Pruebas-v1.0.md` §4.2 |
| `ADR-` | Decisión de arquitectura | ADR-01..29, `SDD/Docs/05-Arquitectura-Tecnica/Adrs/` |
| `EP-` | Épica | EP-01..07, `SDD/Docs/06-Backlog-Tecnico/Product-Backlog-v1.0.md` |
| `US-` / `BT-` | Historia de usuario / ítem de backlog técnico | US-01..26 (`historias-usuario/`) / BT-01..30 (inline en `Backlog-Tecnico-v1.0.md`); `SDD/Docs/06-Backlog-Tecnico/` |
| `TC-` | Caso de prueba referencial | TC-01..40, `SDD/Docs/08-Calidad-Y-Pruebas/Casos-Prueba-Referenciales-v1.0.md` |
| `SD-` | Comprobación de deriva: una fila por cada `SUP`/`CMP`/`EST`/`NAV`/`DM` de la línea base (umbrales `U-SUP`…`U-DM`) | `SDD/Docs/08-Calidad-Y-Pruebas/Matriz-Sensado-Deriva-v1.0.md` |
| `SUP-` / `CMP-` / `EST-` / `NAV-` / `DM-` / `EV-` | Superficie (11) / componente (19) / estado (79) / ruta (9) / campo del contrato de datos (24) / evidencia | `SDD/Docs/03-UX-UI-DX/Linea-Base-Visual-v1.0.md`, `Contrato-Datos-Maqueta-v1.0.md` |
| `CL-` | Caso límite («qué pasa si») | CL-01..28, Intake §7 |
| `UF-` | Flujo típico | UF-1..10, Intake §6 |
| `E-n` / `F-n` | Escenario con ejemplo completo (E-1..E-8; E-9 propuesto en `Plan-Pruebas-v1.0.md`) / flujo end-to-end (F-1..F-7) | Intake §20 (Anexo A) / §21.B.3 (Anexo B) |
| `F-NN` | Capacidad funcional MoSCoW, F-01..F-33 — ojo: colisiona con los flujos `F-n` | Intake §4 |
| `C-` / `F0..F5` | Capacidad del roadmap (C-01..16) / fase del producto | `SDD/Docs/00-Contexto/Roadmap-Producto-v1.0.md` §2 |
| `E-0n` | Exclusión («lo que NO es esta solución»), E-01..E-09 — ojo: colisiona con los escenarios `E-n` | Intake §9 |
| `N-` | Requisito no funcional (NFR), N-01..N-25 | `SDD/Docs/05-Arquitectura-Tecnica/Arquitectura-Solucion-v1.0.md` §8 (origen: Intake §17.P.10); cruzados en `Matriz-Cobertura-Pruebas-v1.0.md` §3 |
| `EVE-` | Eventualidad registrada (síntoma, causa, resolución) | EVE-01..07, `SDD/Docs/11-Documentacion/Bitacora-Eventualidades-v1.0.md` |
| `R-` | Riesgo, R-01..R-14 (alias S-1..S-11 de la fuente en los once primeros) | Intake §11; Visión §8; `Arquitectura-Solucion-v1.0.md` §9; `Plan-Pruebas-v1.0.md` |
| `M-` | Métrica de éxito | Intake §8 |
| `P-` | Pendiente / decisión abierta | P-01..09, Intake; `SDD/Docs/00-Contexto/README.md` |
| `PA-` | Principio/decisión de arquitectura del intake | Intake §17 |
| `H-` · `P0..P3` · `D1..D8` | Hallazgo de auditoría · severidad · criterio de estilo de la matriz de auditoría | `SDD/Docs/Audit/*.md` |
| `STAGE-` | Stage del pipeline especificado, STAGE-01..10 | `SDD/Docs/09-Devops/Pipeline-CI-CD-v1.0.md` |
| `VER-` / `EXT-` | Contrato de verificación de un sample (VER-01..02) / punto de extensión (EXT-01) | `SDD/Docs/10-Examples/Ejemplo-01-Datos-Seed-v1.0.md` y `Ejemplo-02-Api-Ingesta-v1.0.md` §9 / `SDD/Docs/11-Documentacion/Guia-Extension-v1.0.md` |
| `AG-` | Subagente autor de la categoría (AG-00..AG-11, AG-03M, AG-ROOT) | cabecera «Autor» de cada doc del SDD |
| `FA-` | Flujo alternativo de un CU | Casos de uso (p. ej. CU-08 FA-1, CU-09 FA-2) |

Rangos obtenidos por `grep` de los patrones en `SDD/` (no verificado que cada ID tenga una única definición; p. ej. `E-07` aparece con cero inicial y `F-2`/`F-3` sin él).

El mapa completo del árbol documental está en `09_Documentacion-Sdd.md`; los pendientes P-xx abiertos, en `11_Estado-Y-Pendientes.md`.

---

## 8. Términos evitados o con trampa

| Evitar / cuidar | Por qué | Fuente |
|---|---|---|
| «Estado de salud», SoH, % de salud | El veredicto es tendencia relativa; magnitudes cuantitativas prohibidas | Intake §12; ADR-13; `Domain/Monitoreo/VeredictoSalud.cs` |
| Usar `battery.charge` como umbral | Es derivado por el driver | ADR-12; `Domain/Monitoreo/Variables.cs` |
| Autonomía / runtime | No existe en el equipo | Intake §12 |
| Confundir `Refutado` con `Vencido` | El primero bloquea permanente; el segundo sólo pide repetir | Intake §12; `Domain/Verificaciones/EstadoVerificacion.cs` |
| Servir un agregado como muestra | El promedio horario borra microcortes | ADR-08; `Domain/Monitoreo/Agregado.cs` |
| `Hasta` nulo como «desconocido» | Significa vigente | `Domain/Vinculos/Vigencia.cs` |
| Mezclar microcortes de reglas v1 (30 s) y v2 (60 s) | Corrompe la serie; por eso la regla está versionada | Intake §12; `Domain/Monitoreo/ReglaDerivacion.cs` |
| «Panel» vs «panel de control» | Superficie vs. producto entero | Glosario-UX §1 |
| Jerga de NUT en la UI (`shutdown.return`, claves) | Dos audiencias: lenguaje de operador a la UI, detalle técnico al log | `SDD/Docs/11-Documentacion/Bitacora-Eventualidades-v1.0.md` EVE-07 |
| «Parque», «secreto» | Unificados a «equipos» y «contraseña» | Glosario-UX §7 (v1.1) |
