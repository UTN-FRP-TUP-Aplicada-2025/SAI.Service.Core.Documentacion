# 10 — Glosario

> **Propósito**: vocabulario del dominio y del proyecto, para leer código y documentación sin ambigüedad.
> El dominio y el código están escritos en **español rioplatense**; los términos de abajo son los del
> modelo, no traducciones libres.
> **Fuentes primarias**: [`SDD/Docs/README.md §8`](../../../SAI.Service.Core/SDD/Docs/README.md) ·
> [`SDD/Docs/03-UX-UI-DX/Glosario-UX-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/03-UX-UI-DX/Glosario-UX-v1.0.md) ·
> [`Domain/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Domain/)

---

## Dominio del equipamiento

| Término | Definición |
| --- | --- |
| **SAI** | Sistema de Alimentación Ininterrumpida (UPS): sostiene la alimentación del host cuando falla la red eléctrica |
| **NUT** | *Network UPS Tools*: ecosistema libre que dialoga con el equipo por USB y lo expone por red (`upsd`, puerto 3493) |
| **Host** | El servidor protegido (`i7infra`), modelado como especialización de `UnidadFisica` |
| **Equipos** | Las unidades físicas que administra el sistema: host, SAI y batería |
| **Unidad física** | Supertipo con ciclo de vida y baja lógica común; se especializa en Host, Dispositivo y Batería (TPH) |
| **Dispositivo** | La unidad concreta de SAI (un ejemplar de un `ModeloDispositivo`) |

## Apagado y seguridad operativa

| Término | Definición |
| --- | --- |
| **Apagado ordenado** | Apagar el sistema operativo deteniendo contenedores y sincronizando discos antes de perder alimentación |
| **Reencendido automático** | Que el host arranque solo al restablecerse la energía; exige que el SAI corte su salida aunque el host ya esté apagado |
| **Ciclo forzado** | Modalidad en la que, iniciada la secuencia, el corte del SAI **no se cancela** aunque vuelva la red (nunca `shutdown.stop`) |
| **Modalidad** | Respuesta configurada ante un corte: `SoloAlerta`, `ApagarHostConRetorno`, `ApagarHostLuegoUpsConRetorno`, `CicloForzado` |
| **Modalidad efectiva** | La que realmente se aplica tras el bloqueo por verificación; degrada a `SoloAlerta` si falta algún supuesto |
| **Supuesto** | Una de las cuatro condiciones de seguridad: presupuesto de apagado, señal en batería, reencendido por placa, corte con retorno |
| **Verificación** | Prueba del estado de un supuesto, con evidencia, método y vigencia; estados: `NuncaVerificado`, `Verificado`, `Vencido`, `Refutado`, `PorVencer` (computado) |
| **Bloqueo por verificación** | Regla por la que una acción con un supuesto no vigente queda bloqueada y la modalidad degrada (RN-02) |
| **Techo duro** | Límite absoluto de 540 s para el tiempo reservado de apagado (RN-04, I-10) |
| **Tiempo reservado** | Ventana que el host tiene para apagarse antes de que el SAI corte su salida |
| **Tiempo de retorno** | Retardo con que el SAI repone la salida tras volver la red (`ups.delay.start`); configurable por política |
| **Efecto observado** | Criterio por el que una orden se da por ejecutada solo si el equipo la admitió, nunca por ausencia de error |
| **Política de apagado** | Configuración **versionada e inmutable** de modalidad, umbral, tiempos y supuestos requeridos; se cambia creando una versión nueva |

## Datos, historia y procedencia

| Término | Definición |
| --- | --- |
| **Procedencia (`Origen`)** | Origen declarado de cada valor: `Medido`, `Derivado`, `Declarado`, `Imputado`, `EstimadoPorDriver`, `NoCalculable` |
| **Append-only** | Disciplina de escritura: se insertan hechos, nunca se actualizan ni se borran; corregir es agregar un hecho nuevo |
| **Baja lógica** | Marcar una unidad como retirada sin borrarla; el borrado físico no existe en este dominio |
| **Vínculo temporal** | Relación «qué estuvo con qué, cuándo», modelada como entidad con intervalo (`MontajeBateria`, `CoberturaHost`) |
| **Vigencia** | Intervalo semiabierto `[desde, hasta)`; `hasta` nulo = vigente |
| **Resolución temporal** | Resolver qué batería estaba montada en un instante, en vez de guardar la batería en la métrica |
| **Muestra** | Lectura del estado del equipo en un instante, con su calidad (`completa`, `parcial`, `perdida`) |
| **Agregado** | Resumen de una ventana de tiempo con su función, cantidad de muestras y **cobertura obligatoria**; entidad distinta de `Muestra` |
| **Cobertura** | Proporción de la ventana efectivamente muestreada; sin ella un promedio no es interpretable |
| **Evento** | Hecho derivado de las muestras por una **regla versionada**: corte, retorno, microcorte, desconexión, tensión fuera de rango, disparo |
| **Microcorte** | Parpadeo breve: OL→OB seguido de OB→OL en menos de 60 s (regla v2) |
| **Regla de derivación** | Regla versionada que produce eventos a partir de muestras; la versión es parte de la identidad del evento |
| **Sesión de sondeo** | Período de sondeo bajo un mismo driver y dialecto; guarda **una sola vez** el mapa variable→procedencia |
| **Acción** | Registro append-only de una decisión de apagado: modalidad solicitada, modalidad efectiva y resultado observado |

## Ciclo de vida y costos

| Término | Definición |
| --- | --- |
| **Salud de batería** | Tendencia **relativa** derivada de la caída de tensión durante el autotest a carga igualada; no es un porcentaje ni un SoH |
| **Confianza del veredicto** | Grado declarado del veredicto de salud; con menos de 4 pruebas comparables es `baja` |
| **Comparabilidad** | Condición para comparar dos pruebas de batería: carga equivalente dentro de tolerancia |
| **Vida de flotación** | Vida esperada del modelo de batería; inválida sin su temperatura de referencia (RN-13) |
| **Intervención** | Servicio técnico registrado con costos, hallazgos, disposición final y dos tiempos (válido/registrado) |
| **Ficha de vida útil** | Proyección de cierre de una batería: días en servicio, cumplimiento de expectativa, costo por año normalizado |
| **Costo por año de servicio** | Métrica normalizada (a USD, marcada como derivada con su fuente de cotización) para comparar marcas |
| **Cobertura suplente** | Dispositivo que cubre al host mientras el titular está en reparación |
| **Días sin protección** | Días del período en que el host no tuvo ningún dispositivo cubriéndolo |
| **Disposición final** | Destino y receptor de una batería retirada (trazabilidad ambiental) |

## Integración

| Término | Definición |
| --- | --- |
| **Ingesta idempotente** | Ingesta por API en la que reenviar el mismo hecho no lo duplica, gracias a `X-Idempotency-Key` |
| **GMAO** | Sistema externo de gestión de mantenimiento; empuja intervenciones por la API (`fd-gmao-externo`) |
| **Fuente de datos** | Origen declarado de un conjunto de datos con su **confianza base** (el sondeo local es `alta`; la ingesta externa, `media`) |
| **Conflicto de idempotencia** | Misma clave con cuerpo distinto ⇒ HTTP 409; nunca 200 |
| **problem+json** | Formato de error RFC 7807 usado por la API |

## Proceso y framework

| Término | Definición |
| --- | --- |
| **SDD** | Framework de documentación por categorías (00–11) con el que se generó `SDD/Docs/` |
| **ADR** | *Architecture Decision Record*: decisión registrada e **inmutable**; si evoluciona, se supera con una nueva |
| **CU / RN / RC / NB / US / BT / NFR** | Caso de uso · Regla de negocio · Regla conceptual de modelo · Necesidad de negocio · Historia de usuario · Tarea técnica · Requisito no funcional |
| **UF** | Flujo de usuario (`UF-1`..`UF-10`), unidad de secuenciación del plan |
| **EVE-XX / OPS-XX** | Entrada de la bitácora de eventualidades · Incidente conocido del runbook |
| **VER-XX** | Contrato de verificación de un ejemplo de la categoría 10 |
| **`web-monolith`** | Tipo de proyecto del framework: un solo proceso desplegable con UI y API |
