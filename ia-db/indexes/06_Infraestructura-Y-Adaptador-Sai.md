# 06 — Infraestructura y adaptador al SAI

> **Propósito**: cómo el servicio habla con el SAI, qué corre en segundo plano y qué se configura por
> entorno. Es el índice para extender el acceso al equipo o diagnosticar el sondeo.
> **Fuentes primarias**: [`Infrastructure/Adaptadores/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/Adaptadores/) ·
> [`Infrastructure/Monitoreo/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/Monitoreo/) ·
> [`Infrastructure/DependencyInjection.cs`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Infrastructure/DependencyInjection.cs) ·
> [`SDD/Docs/11-Documentacion/Guia-Extension-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/11-Documentacion/Guia-Extension-v1.0.md) ·
> [`SDD/Docs/11-Documentacion/Guia-Contenedor-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/11-Documentacion/Guia-Contenedor-v1.0.md)

---

## 1. El adaptador de conexión: el único punto de extensión publicado

Todo el acceso al SAI está detrás de dos interfaces de `Application/Abstractions/`: `IAdaptadorConexion`
(operación) e `IDescubridorSai` (descubrimiento). Ninguna entidad ni caso de uso conoce NUT.

| Implementación | Ruta | Cuándo |
| --- | --- | --- |
| `AdaptadorConexionSimulado` | `Adaptadores/AdaptadorConexionSimulado.cs` | Desarrollo y pruebas, sin hardware (**por defecto**) |
| `AdaptadorConexionNut` | `Adaptadores/Nut/AdaptadorConexionNut.cs` | Producción, contra `upsd` de NUT |

Ambas implementan **las dos** interfaces y se registran resolviendo a la **misma instancia**. La selección
es por configuración `Sai:Adaptador` (`Simulado` | `Nut`) en `DependencyInjection.cs`.

**Cómo agregar un backend nuevo**: implementar las dos interfaces en `Infrastructure/Adaptadores/`,
agregar una rama al selector por un valor nuevo de `Sai:Adaptador` y registrar la misma instancia para los
dos puertos. No se toca ningún caso de uso ni entidad.

**Tres límites que un adaptador no puede violar** (o rompe la seguridad operativa):

1. Confirma **por efecto observado**: `Aceptada` solo si el equipo admitió la orden; una excepción de
   transporte se traduce a «no alcanzable», nunca a «apagado exitoso» (ADR-11).
2. **No decide** la modalidad ni el bloqueo por verificación: eso es del dominio (`EvaluadorModalidad`).
3. **Nunca** emite `shutdown.stop`: el ciclo forzado no se cancela (ADR-09).

### Simulado

Devuelve valores fijos y razonables para que el panel funcione sin equipo. Simula una **descarga real**
durante la prueba de batería (la tensión cae ~0,7 V desde 13,2 V y se recupera en una ventana de 12 s),
para que la serie densa muestre una caída verdadera. La bandera `Sai:Simulado:EnBateria` lo hace reportar
`OB` (en batería), lo que permite ejercitar el corte y la verificación de la señal sin hardware.

### NUT

| Pieza | Rol |
| --- | --- |
| `ProtocoloNut.cs` | Parseo **puro** del protocolo textual line-based (RFC 9271), separado del transporte para probarse sin socket: `GET VAR`, `LIST VAR`, `LIST UPS`, entrecomillado y escapes |
| `ClienteNut.cs` / `IClienteNut.cs` | Cliente TCP mínimo: abre una conexión **efímera por operación** (los sondeos son poco frecuentes; sin pool ni keepalive) |
| `AdaptadorConexionNut.cs` | Traduce el estado NUT al modelo (`EstadoSai`) y emite las órdenes de apagado y test |
| `OpcionesNut.cs` | `Host`, `Puerto`, `Ups`, `TimeoutSegundos`, `Usuario`, `Password` |
| `NutException.cs` | Error de transporte/protocolo, traducido a «no alcanzable» hacia arriba |

Las lecturas (`VER`, `LIST UPS`, `LIST VAR`) son **anónimas**; las escrituras (apagado, test de batería)
exigen credenciales. **Sin credenciales configuradas el servicio queda en solo lectura por diseño**: no
apaga a ciegas. El retardo de retorno se emite como `ups.delay.start` y proviene de la política vigente,
no de una constante.

## 2. Servicios en segundo plano

| Hosted service | Qué hace |
| --- | --- |
| `ServicioSondeo` | Ronda cada `Sai:Sondeo:IntervaloSeg` (5 s) con `PeriodicTimer`. **Cada ronda abre su propio alcance de DI** (un `DbContext` por ronda, no uno de vida larga) y persiste una muestra con su calidad. Es resiliente: una ronda que falla no detiene el planificador. No conserva estado crítico — el contador de fallidos es reconstruible desde la historia |
| `ServicioRearmePruebas` | Corre una sola vez al arrancar: rearma las pruebas de verificación que quedaron esperando un reinicio del host. Bajo ADR-25 (NUT en el contenedor), que el servicio vuelva a la vida **es** la señal honesta de que el host cicló. Un fallo del rearme no impide el arranque |

## 3. Repositorios (implementación de los puertos)

`RepositorioEquipos`, `RepositorioMonitoreo`, `RepositorioIntervenciones`, `RepositorioSustituciones`,
`RepositorioPoliticas`, `RepositorioIngesta`, `RepositorioInformes` — todos scoped sobre `SaiDbContext`.
`SaiDbContextFactory` existe para el tiempo de diseño (`dotnet ef`).

## 4. Configuración

Todas las claves admiten override por variable de entorno con `__` como separador de sección.

| Clave | Default | Efecto |
| --- | --- | --- |
| `ConnectionStrings:Sai` | `Data Source=sai.db` | Cadena SQLite (en contenedor, dentro del volumen de datos) |
| `Sai:Adaptador` | `Simulado` | `Simulado` \| `Nut` |
| `Sai:Nut:Host` / `Puerto` / `Ups` / `TimeoutSegundos` | `127.0.0.1` / `3493` / `sai` / `5` | Conexión al `upsd` |
| `Sai:Nut:Usuario` / `Password` | vacío | Credenciales de **escritura**; vacío ⇒ solo lectura |
| `Sai:Sondeo:IntervaloSeg` / `Habilitado` | `5` / `true` | Cadencia del sondeo y apagado del poller |
| `Sai:Simulado:EnBateria` | `false` | El simulado reporta `OB` |
| `Sai:Apagado:ModalidadSolicitada` / `TiempoReservadoSeg` / `TiempoRetornoSeg` | `SoloAlerta` / `120` / `180` | **Semilla** de la política inicial; la versión vigente en base es la que manda |
| `Sai:Verificacion:DiasPreavisoVencimiento` | — | Umbral del estado computado `PorVencer` |
| `Sai:Prueba:*` | — | `NumeroMuestras`, `IntervaloMuestraMs`, `FlotacionMinimaSeg`, `ToleranciaCargaPct` |
| `Jwt:ClaveFirma` | vacío | **Obligatoria en prod**, ≥ 32 bytes o el arranque falla |
| `Jwt:Emisor` / `Audiencia` / `MinutosVigencia` | `sai-service-core` / `sai-service-core-api` / `60` | Parámetros del token |
| `DataProtection:RutaLlaves` | vacío | **Obligatoria en prod**: keyring persistente; vacío ⇒ efímero |
| `Kestrel:Endpoints:Http:Url` | `http://0.0.0.0:8080` | Endpoint HTTP (HTTPS 8443 solo en Development) |
| `Sello:*` | — | Versión legible y fecha mostradas en la barra superior |

> El binder de configuración **no** se usa en `Infrastructure` (no referencia el paquete): las opciones se
> leen a mano en `DependencyInjection.cs`. Al agregar una opción hay que sumar su lectura ahí.
> El tiempo reservado se acota con `Math.Clamp(…, 0, 540)` por si la configuración excede el techo duro.

## 5. Relación con NUT en el host

El servicio **no toma el USB**: lo hace el driver de NUT. El nodo se ancla por **ruta física de puerto**
con una regla `udev` (el equipo no expone `iSerial` y `hidraw` es volátil) — ADR-03. Efecto lateral
buscado: «el SAI que esté enchufado ahí», de modo que reemplazar el equipo no rompe el binding.

**El apagado del host lo ejecuta la cadena NUT** (`upsmon`/`upssched`), no este servicio: el servicio
decide y ordena `shutdown.return`. Por eso `upsmon.conf` necesita `MONITOR sai@localhost 1 <usuario>
<clave> master` y el usuario debe ser `upsmon master` en `upsd.users` — con `slave` da `ERR ACCESS-DENIED`
(EVE-01, OPS-04).

Verificación rápida del entorno NUT: `upsc sai@localhost ups.status` debe devolver un estado (`OL`, `OB`…)
sin error de conexión.
