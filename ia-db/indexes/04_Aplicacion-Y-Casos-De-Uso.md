# 04 — Aplicación y casos de uso

> **Propósito**: qué servicio de aplicación resuelve cada caso de uso, qué puerto usa y dónde está su
> implementación. Es el índice para responder «¿dónde toco para cambiar el comportamiento X?».
> **Fuentes primarias**: [`Application/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Application/) ·
> [`Infrastructure/DependencyInjection.cs`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/DependencyInjection.cs) ·
> [`SDD/Docs/02-Especificacion-Funcional/Casos-De-Uso/`](../../../SAI.Service.Core/SDD/Docs/02-Especificacion-Funcional/Casos-De-Uso/) ·
> [`CHANGELOG.md`](../../../SAI.Service.Core/CHANGELOG.md)

---

## 1. Organización

`Application/` tiene **un subdirectorio por flujo**; cada uno contiene su servicio de caso de uso, sus
DTO de entrada/salida y el **puerto** (interfaz de repositorio) que Infrastructure implementa.
`Application/Abstractions/` guarda los puertos transversales del adaptador al SAI.

## 2. Servicios por flujo

| Flujo | Servicio | Puerto propio | CU / US |
| --- | --- | --- | --- |
| Equipos | `ServicioAltaEquipos` — alta de catálogo, inventario y vínculos temporales; siembra las verificaciones en `NuncaVerificado` | `IRepositorioEquipos` | CU-02, US-03/US-04/US-05 |
| Equipos | `ServicioVerificacion` — estado de los cuatro supuestos, renovación por evidencia, preaviso de vencimiento | `IRepositorioEquipos` | CU-10, US-15/US-17 |
| Equipos | `ServicioEjercicioGuiado` — ventana de mantenimiento guiada (secuencia física) | `IRepositorioEquipos` | CU-10, US-16 |
| Monitoreo | `ServicioMonitoreo` — persiste la muestra de cada ronda y deriva eventos con reglas versionadas | `IRepositorioMonitoreo` | CU-04, US-08/US-09 |
| Monitoreo | `ServicioPanelEnVivo` — arma el estado en vivo del panel con procedencia visible | `IRepositorioMonitoreo` | CU-04, US-07/US-10 |
| Monitoreo | `ServicioHistoricos` — series, agregados con cobertura y marcas de eventos | `IRepositorioMonitoreo` | CU-06, US-11 |
| Monitoreo | `ServicioPruebaBateria` — prueba programada/manual a 1 Hz y veredicto de salud con confianza | `IRepositorioMonitoreo` | CU-07, US-12/US-13 |
| Acciones | `ServicioApagadoOrdenado` — lee la política vigente, deriva la modalidad efectiva y ordena el apagado por el adaptador | `IRepositorioPoliticas` + `IAdaptadorConexion` | CU-05, US-14/US-15 |
| Políticas | `ServicioPoliticas` — crea **versiones nuevas** (nunca edita la vigente), valida techo duro y parámetros, previsualiza la modalidad efectiva | `IRepositorioPoliticas` | CU-03, US-06 |
| Intervenciones | `ServicioRecambioBateria` — cierra/abre el montaje y arma la ficha de vida útil con costo por año | `IRepositorioIntervenciones` | CU-08, US-18/US-19 |
| Intervenciones | `ServicioSustitucionSai` — reparación/sustitución con cobertura suplente y días sin protección | `IRepositorioSustituciones` | CU-09, US-20 |
| Informes | `ServicioInformePeriodo` — informe de período y comparación de marcas (solo lectura sobre la historia) | `IRepositorioInformes` | CU-12, US-23/US-24 |
| Ingesta | `ServicioIngesta` — ingesta idempotente por clave con los cuatro caminos 201/200/409/422 | `IRepositorioIngesta` | CU-11, US-21/US-22 |

Todos se registran como **scoped** en `AddInfrastructure`, salvo las opciones (singleton) y los hosted
services.

## 3. Puertos del adaptador al SAI (`Application/Abstractions/`)

```csharp
// IAdaptadorConexion.cs — operación (contrato cerrado, ADR-27)
Task<EstadoSai>             LeerEstadoAsync(CancellationToken ct);
Task<ResultadoConectividad> ProbarConectividadAsync(CancellationToken ct);
Task<ResultadoAccion>       OrdenarApagadoConRetornoAsync(TimeSpan retardo, /* retardoRetorno */ …, CancellationToken ct);
Task<ResultadoAccion>       LanzarTestBateriaAsync(CancellationToken ct);

// IDescubridorSai.cs — descubrimiento
Task<IReadOnlyList<DispositivoDescubierto>> DescubrirAsync(CancellationToken ct);
```

Tipos de apoyo en la misma carpeta: `EstadoSai`, `EstadoUps`, `ResultadoAccion`,
`ResultadoConectividad`, `DispositivoDescubierto`, `IPlanificador`.

> `OrdenarApagadoConRetornoAsync` recibe el **retardo de retorno** como argumento desde la política
> vigente (antes estaba fijo en 180 s dentro del adaptador NUT). El adaptador NUT lo emite como
> `ups.delay.start`.

## 4. Reglas de diseño de la capa

1. **La UI propone, el humano confirma, el sistema valida.** `ServicioPoliticas.PrevisualizarAsync`
   deriva la modalidad efectiva y avisa si degradaría, **sin ejecutar nada**; `CrearVersionAsync` valida
   antes de persistir.
2. **Postcondición de fallo**: si la validación falla, **nada se registra** (aplica a políticas y a
   ingesta).
3. **Versionar en vez de editar**: una política se cambia creando la versión siguiente; la vigente es la
   de mayor número, y las anteriores quedan intactas.
4. **El servicio no habla con NUT**: siempre por `IAdaptadorConexion`.
5. **La decisión de modalidad es del dominio**, no de la aplicación ni del adaptador
   (`EvaluadorModalidad`).

## 5. Flujos de usuario (UF) y su estado

| UF | Flujo | Etapa | Estado |
| --- | --- | --- | --- |
| UF-1 | Alta de equipos y puesta en marcha | Etapa 2 | Implementado |
| UF-2 | Configuración de políticas | Etapa 4·D | Implementado (versionado en base) |
| UF-3 | Monitoreo en vivo | Etapa 3 | Implementado |
| UF-4 | Históricos y gráficas | Etapa 3 | Implementado |
| UF-5 | Prueba de batería y salud | Etapa 3 | Implementado |
| UF-6 | Recambio de batería y ficha de vida útil | Etapa 4·C | Implementado |
| UF-7 | Reparación/sustitución del SAI | Etapa 4 | Implementado |
| UF-8 | Verificación y apagado ordenado | Etapa 4·B/E | Implementado en software; el ciclo físico requiere ventana de mantenimiento |
| UF-9 | Informe de período y comparación de marcas | Etapa 4 | Implementado |
| UF-10 | Ingesta automatizada de intervenciones | Etapa 5 | Implementado (cierra la Etapa 5) |

Diferido a v2 y **no implementado**: US-25 (adaptador de conexión directo) y US-26 (add-ons de dialecto de
protocolo) — ambos son *Could*, diseñados pero fuera del compromiso de v1.

## 6. Decisiones de implementación que conviene conocer

| Decisión | Motivo |
| --- | --- |
| Los costos del informe se muestran **en su moneda y fecha original**; el equivalente USD solo aparece en la comparación de marcas | Ahí el valor derivado sí está registrado con su fuente de cotización; fabricar una conversión sería inventar (RN-07) |
| Un período sin actividad devuelve `PERIODO_SIN_DATOS` | Un informe vacío pero bien formateado sería engañoso |
| Con menos de dos modelos con ficha cerrada, la comparación avisa **confianza baja** | Comparar una marca contra nada no es comparar |
| El host es **implícito** en los informes | El sistema es mono-host: la barra de parámetros solo pide el período |
| La ingesta usa la propia historia como almacén de idempotencia | El índice único de la clave más la huella sha256 distinguen reintento (200) de conflicto (409) |
