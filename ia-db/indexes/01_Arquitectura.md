# 01 — Arquitectura

> **Propósito**: cómo está construido el sistema — capas, dependencias, ubicación en el repositorio,
> flujo del camino crítico y el índice de las decisiones registradas (ADR).
> **Fuentes primarias**: [`SDD/Docs/05-Arquitectura-Tecnica/Arquitectura-Solucion-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Arquitectura-Solucion-v1.0.md) ·
> [`Decisiones-Arquitectura-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Decisiones-Arquitectura-v1.0.md) ·
> [`Adrs/`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Adrs/) ·
> [`SDD/Docs/11-Documentacion/Recorrido-Codigo-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/11-Documentacion/Recorrido-Codigo-v1.0.md)

---

## 1. Estilo y su justificación

**Clean Architecture en cinco assemblies** (ADR-15), un único proceso y un único despliegue. Elegida
frente a dos alternativas explícitamente descartadas:

| Alternativa descartada | Por qué |
| --- | --- |
| Capas tradicionales con acceso a datos desde la UI | Acopla la única lógica irreversible (decidir el apagado) al ORM y al framework web; encarece probar los invariantes |
| Event-driven con event store y CQRS | Rechazada por la fuente: la inmutabilidad buscada es **disciplina de escritura**, no infraestructura; un usuario, un SAI, un host no la justifican |

Sobre ese esqueleto, tres decisiones estructurales propias del dominio: **adaptador de conexión**
(ADR-02/ADR-27), **planificador como hosted service** (ADR-09..ADR-12) e **historia append-only** (ADR-04).

## 2. Los cinco assemblies

| Assembly | Responsabilidad | Depende de |
| --- | --- | --- |
| `SAI.Service.Core.Domain` | Entidades, objetos de valor (`Valor<T>` con `Origen`, `Dinero`), invariantes I-1..I-21, resolutor temporal, evaluador de modalidad, cálculo de salud | **nada** (framework-free) |
| `SAI.Service.Core.Application` | Casos de uso y **puertos** (interfaces de repositorio y de adaptador) | Domain |
| `SAI.Service.Core.Infrastructure` | EF Core + SQLite, interceptor append-only, adaptadores NUT/simulado, hosted services, cableado DI | Application, Domain |
| `SAI.Service.Core.Api` | Endpoints REST `/api/v1` (salud, informativo, ingesta) | Application |
| `SAI.Service.Core.Web` | **Composition root** y único proceso: panel Blazor, Identity, JWT, monta la API | Api, Infrastructure |

Regla verificable: **ninguna flecha apunta hacia afuera del dominio**. Si un archivo de `Domain` necesita
algo de `Infrastructure`, el diseño está mal: se resuelve con un puerto en `Application/Abstractions/`.

## 3. Vista de procesos

Un solo proceso, un único hilo escritor de la historia. Dentro conviven:

| Elemento | Comportamiento | Referencia |
| --- | --- | --- |
| Planificador de sondeo (hosted service) | Ronda cada `IntervaloSeg` (5 s por defecto; 1 Hz durante prueba de batería). Cada ronda: lee estado, persiste `Muestra` con calidad, deriva eventos, empuja al panel | N-06, N-07, N-08 |
| Temporizadores con cancelación | La condición «en batería» debe sostenerse `umbralDisparoSegundos` (300 s de partida) antes de generar la `Accion`; se cancela si cesa antes | N-05, CU-05 FA-1 |
| Circuito Blazor Server | Un WebSocket por sesión; el servidor empuja estado sin polling. Estado en memoria efímero y reconstruible — la verdad vive en SQLite | — |
| Detección de pérdida de comunicación | 3 sondeos consecutivos sin respuesta ⇒ evento `DesconexionUsb` + alerta | N-09, ADR-11 |

Latencia objetivo de decisión por ronda: **< 1 s** (para no desplazar la siguiente ronda de 5 s).

## 4. Mapa arquitectura → repositorio

Todo bajo [`src/SAI.Service.Core/`](../../../SAI.Service.Core/src/SAI.Service.Core/).

| Componente | Ruta | ADR |
| --- | --- | --- |
| Inventario (Host/Dispositivo/Batería, TPH) | `Domain/Inventario/` | ADR-07 |
| Vínculos temporales (`MontajeBateria`, `CoberturaHost`, `Vigencia`, `ResolutorTemporal`) | `Domain/Vinculos/` | ADR-05 |
| Monitoreo (`Muestra`, `Evento`, `Agregado`, `DerivadorEventos`, `CalculadorSaludBateria`) | `Domain/Monitoreo/` | ADR-08, ADR-12, ADR-13 |
| Verificaciones (los 4 supuestos, `EvaluadorModalidad`, `SecuenciaFisica`, `SesionEjercicio`) | `Domain/Verificaciones/` | ADR-10, ADR-14 |
| Acciones de apagado (`Accion`, techo duro 540 s) | `Domain/Acciones/` | RN-04 |
| Intervenciones (recambio, sustitución, ingesta, ficha de vida útil) | `Domain/Intervenciones/` | ADR-04 |
| Políticas (`VersionPolitica`) | `Domain/Politicas/` | CU-03 |
| Valores con procedencia (`Valor`, `Origen`, `Dinero`) e historia (`IEntidadHistoria`) | `Domain/Valores/`, `Domain/Historia/` | ADR-06, ADR-04 |
| Puertos del adaptador (`IAdaptadorConexion`, `IDescubridorSai`, `EstadoSai`, `ResultadoAccion`) | `Application/Abstractions/` | ADR-27 |
| Servicios de caso de uso (un subdirectorio por flujo) | `Application/{Equipos,Monitoreo,Acciones,Intervenciones,Politicas,Informes,Ingesta}/` | — |
| Persistencia (`SaiDbContext`, repos, `Configuraciones/`, `Migraciones/`, `InterceptorAppendOnly`) | `Infrastructure/Persistencia/` | ADR-18, ADR-04 |
| Adaptadores (`AdaptadorConexionSimulado`, `Nut/`) | `Infrastructure/Adaptadores/` | ADR-02, ADR-27 |
| Hosted services (`ServicioSondeo`, `ServicioRearmePruebas`) | `Infrastructure/Monitoreo/` | ADR-25 |
| Cableado DI (`AddInfrastructure`) | `Infrastructure/DependencyInjection.cs` | — |
| Endpoints REST (`EndpointsSalud`, `EndpointsApiV1`) | `Api/Endpoints/` | ADR-17, ADR-28 |
| Composition root, páginas, autenticación | `Web/Program.cs`, `Web/Components/Pages/`, `Web/Autenticacion/`, `Web/Endpoints/` | ADR-16, ADR-29 |

**Dónde vive cada cosa** (preguntas frecuentes):

| Pregunta | Ruta |
| --- | --- |
| Esquema de datos | `Infrastructure/Persistencia/Configuraciones/Modelo*.cs` + `SaiDbContext.cs` |
| Append-only | `Infrastructure/Persistencia/InterceptorAppendOnly.cs` (marcador `Domain/Historia/IEntidadHistoria`) |
| Selección del adaptador | `Infrastructure/DependencyInjection.cs`, clave `Sai:Adaptador` |
| Bloqueo por verificación | `Domain/Verificaciones/EvaluadorModalidad.cs` |
| Semilla del estado inicial | `Web/Program.cs`, bloque posterior a `MigrateAsync()` |
| Páginas del panel | `Web/Components/Pages/*.razor` |

**Convención para agregar una funcionalidad** (de adentro hacia afuera):
`Domain → Application (servicio + puerto) → Infrastructure (repo + Modelo*.Configurar + migración) →
Api/Web → seed → tests → docs`.

## 5. ADRs

29 decisiones registradas en [`Adrs/`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Adrs/),
inmutables: una decisión que evoluciona no se edita, se supera con una ADR nueva.
Estado al indexar: **25 Aceptado · 1 Propuesto (ADR-21) · 3 Superado (ADR-19, ADR-20, ADR-22) · 1 Superado
por reajuste del framework (ADR-23)**.

| ADR | Título | Estado |
| --- | --- | --- |
| 01 | Adopción de NUT como acceso al SAI | Aceptado |
| 02 | Adaptador de conexión con tres implementaciones | Aceptado |
| 03 | Anclaje del USB por ruta física de puerto | Aceptado |
| 04 | Historia append-only sin event store ni CQRS | Aceptado |
| 05 | Vigencia como entidad con intervalo | Aceptado |
| 06 | Procedencia obligatoria en todo valor | Aceptado |
| 07 | Separación de catálogo, inventario e historia | Aceptado |
| 08 | `Agregado` no hereda de `Muestra` | Aceptado |
| 09 | Modalidad ciclo forzado: no cancelar el corte | Aceptado |
| 10 | Arranque seguro y bloqueo por verificación | Aceptado |
| 11 | Validación por efecto observado | Aceptado |
| 12 | Disparo sin dependencia del flag `LB` | Aceptado |
| 13 | Salud por tendencia de caída de tensión | Aceptado |
| 14 | Verificación de BIOS por comportamiento | Aceptado |
| 15 | Clean Architecture en cinco assemblies | Aceptado |
| 16 | Autenticación de administrador único | Aceptado |
| 17 | Manejo de errores de la API de ingesta | Aceptado |
| 18 | SQLite con EF Core y migraciones | Aceptado |
| 19 | Ubicación de NUT: contenedor u host | Superado por ADR-25 |
| 20 | TLS del panel y la API en la LAN | Superado por ADR-26 |
| 21 | Contrato del endpoint de rectificación del 409 | **Propuesto** (diferido) |
| 22 | Forma del contrato del adaptador | Superado por ADR-27 |
| 23 | Omisión de Developer Guide dedicada | Superado (reajuste SDD v3.0) |
| 24 | Modelo de ambientes DEV/PROD sin staging | Aceptado |
| 25 | NUT en el contenedor | Aceptado |
| 26 | TLS autofirmado en Kestrel | Aceptado |
| 27 | Contrato del puerto del adaptador de conexión | Aceptado |
| 28 | Autenticación de la API con Bearer JWT vía ROPC | Aceptado |
| 29 | Persistencia del keyring de DataProtection | Aceptado |

## 6. Camino crítico — disparo del apagado

```mermaid
sequenceDiagram
    participant S as ServicioSondeo (hosted)
    participant A as IAdaptadorConexion
    participant M as ServicioMonitoreo
    participant D as DerivadorEventos
    participant P as ServicioApagadoOrdenado
    participant E as EvaluadorModalidad
    S->>A: LeerEstadoAsync (cada 5 s)
    A-->>S: EstadoSai (OL / OB, tensiones, carga)
    S->>M: persistir Muestra (con calidad y procedencia)
    M->>D: evaluar ventana contra reglas versionadas
    D-->>P: Evento DisparoApagado (tiempo en OB + tensión; nunca por LB)
    P->>E: derivar modalidad efectiva (4 supuestos vigentes?)
    E-->>P: modalidad solicitada · o degradación a SoloAlerta
    P->>A: OrdenarApagadoConRetornoAsync(retardo, retorno)
    A-->>P: ResultadoAccion (Aceptada / EfectoNoConfirmado / no alcanzable)
    P->>P: registrar Accion append-only por efecto observado
```

Puntos que no se pueden alterar sin romper la seguridad operativa:

1. El disparo **no** depende del flag `LB` ni de `battery.runtime` (ADR-12): se observó que el equipo
   nunca los emite de forma confiable.
2. La modalidad efectiva **siempre** pasa por `EvaluadorModalidad`; el adaptador no decide nada.
3. Iniciada la secuencia en ciclo forzado, **nunca** se emite `shutdown.stop` (ADR-09): la BIOS necesita
   la transición ausencia→presencia de energía para autoencender el host.
4. La `Accion` se registra por **efecto observado**, incluidos los casos bloqueado y solo aviso.

## 7. Cross-cutting

| Concern | Decisión |
| --- | --- |
| Logging | Estructurado y local (stdout). Se loguea cada acción sobre el equipo con su resultado observado, cada sondeo fallido, cada cambio de estado de `Verificacion` y cada versión de política. Sin tracing distribuido ni exportación externa |
| Errores de API | problem+json (RFC 7807) con códigos estables 201/200/409/422 (ADR-17) |
| Errores del camino crítico | `EFECTO_NO_CONFIRMADO` mantiene el estado seguro; no se reporta como ejecutado lo no observado |
| Configuración | Variables de entorno (separador `__`); secretos nunca en el repositorio |
| Seguridad operativa | Arranque en `SoloAlerta` + bloqueo por verificación (ADR-10) |
| Autenticación | Identity con cookie para el panel (ADR-16); Bearer JWT vía ROPC para la API (ADR-28) |
| Jerga técnica | Patrón de dos audiencias: lenguaje llano a la UI, detalle NUT solo al log (EVE-07) |

## 8. Atributos de calidad con objetivo numérico

25 NFR (N-01..N-25) en la fuente. Los que condicionan el código a diario:

| NFR | Objetivo |
| --- | --- |
| N-01 | Retardo de corte del SAI ≤ **540 s** (techo duro; el formulario rechaza más) |
| N-04 | Retardo de reencendido: 180 s por defecto (hoy configurable por política) |
| N-05 | Umbral de disparo: 300 s de partida, ajustable por versión de política |
| N-06 | Latencia de decisión por ronda < 1 s |
| N-07 / N-08 | Sondeo a 5 s; 1 Hz durante prueba de batería |
| N-09 | 3 sondeos sin respuesta ⇒ `DesconexionUsb` |
| N-10 | `input.voltage` en [198, 242] V; fuera 30 s sostenidos ⇒ evento |
| N-11 | Microcorte: < 60 s entre OL→OB y OB→OL (regla v2) |
| N-12 | `battery.voltage` < 13,3 V ⇒ celda en corto; > 14,5 V ⇒ celda abierta |
| N-13..N-15 | Vigencias: presupuesto de apagado 180 d; BIOS y flag OB 365 d; shutdown-return sin caducidad |
| N-16 | Tendencia de salud: ≥ 4 pruebas comparables, si no, confianza `baja` |
| N-18 | Volumen ≈ 6,3 M filas/año a 5 s |
| N-21 / N-22 | Cobertura ≥ 80/70 global; ≥ 90/85 en `Domain` |
| N-23 / N-24 | Idempotencia 100 %; cero valores sin `Origen` |

Pendientes de medición declarados: N-03 (resto del apagado del SO), N-20 (tamaño del archivo SQLite),
N-25 (SLO de disponibilidad). Ver [11_Estado-Y-Pendientes](11_Estado-Y-Pendientes.md).

## 9. Riesgos arquitectónicos

14 riesgos (R-01..R-14) en la fuente. Los de severidad crítica:

| ID | Riesgo | Mitigación anclada |
| --- | --- | --- |
| R-01 | El ciclo apagado→reencendido no está verificado; la trampa de firmware podría dejar el SAI apagado para siempre | Prueba física en ventana de mantenimiento antes de habilitar cualquier modalidad ≠ `SoloAlerta` |
| R-12 | **Riesgo principal**: el servicio decide apagar un servidor sin backups; si falla, falla de noche y sin testigos | Bloqueo por verificación; arranque forzado en `SoloAlerta` |
| R-02 | El presupuesto de 540 s no está medido contra el apagado real | Medición cronometrada en CU-10; verificación con vigencia de 180 días |
| R-09 | Sin sensor de temperatura, el confusor de la tendencia de salud no tiene solución con este equipo | Declarado; toda conclusión de salud lleva reserva explícita |
| R-13 | Guardar `battery.charge` sin marcar que es interpolación del driver | Procedencia obligatoria; no apto para tendencia de salud |
