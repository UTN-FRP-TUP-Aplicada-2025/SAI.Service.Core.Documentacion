tengo esta consulta del agente IA 
```
# Etapa 4·E — Prueba de tiempo de apagado activa y rearme por reinicio del host

## Context

En el panel de verificaciones, el supuesto **"Prueba de tiempo de apagado del host"** (`PresupuestoDeApagado`)
hoy es una **carga manual**: el usuario apaga el host por su cuenta, lo cronometra a mano y escribe los
segundos en una caja de texto (`VerificarPresupuestoAsync(int segundos)`).

En la última ronda de feedback (Historial/30.md) el usuario decidió cambiar el modelo:

- El **botón dispara el apagado ordenado del host y lo cronometra automáticamente** (confirmado: sin caja
  de texto; el tiempo lo mide el sistema).
- Al presionar, el botón **se bloquea** con la leyenda *"Se activará cuando reinicie el equipo para iniciar
  una nueva prueba"*, para que el usuario no re-dispare el apagado "a lo loco".
- El botón **se re-habilita al detectar que el host reinició**.
- La misma mecánica de *bloqueo + leyenda + rearme por reinicio* se aplica al botón
  **"Encendido por presencia de energía en la alimentación del host"** (`ReencendidoPorPlaca`), que **no**
  dispara apagado (registra la observación arrancó/no-arrancó del operador).
- Reword del paso 1: *"esta prueba inicia el apagado ordenado"* en vez de *"vos iniciás"*.

Esto excede un retoque de texto: hoy `IAdaptadorConexion` solo le habla a la UPS y `EstadoSai` no tiene
ningún dato del host, así que no hay forma de disparar el apagado del host ni de detectar su reinicio. Se
construye como incremento nuevo, **validado contra el adaptador simulado** (principio del proyecto: el
apagado real nunca se dispara contra un host durante el desarrollo — el simulado es el banco de pruebas).

**Rama:** parte de `fix/panel-verificaciones-ux` (que ya tiene los ajustes de UX + el rename ya commiteado),
no de `main` — para no perder ese trabajo. La feature entra como su propio commit/PR.

## Diseño

**Señal del host sin agente en el host.** El único dato observable es la UPS. Cuando el host se apaga, su
carga sobre la salida del SAI cae a ~0 (`EstadoSai.CargaSalidaPorcentaje` / `ups.load`); cuando reinicia, la
carga vuelve. Se usa un umbral configurable (`Sai:Verificacion:CargaHostAbajoPct`, def. 5%) como proxy
"host abajo/arriba" (efecto observado, ADR-11).

**Estado transitorio.** Se agrega a `Verificacion` un marcador de prueba en curso (nullable, persistido),
ortogonal al `Estado`:
- `PruebaEnCursoDesde` (DateTimeOffset?) — cuándo se disparó / arrancó la prueba.
- `HostVistoAbajoEn` (DateTimeOffset?) — primer instante en que se observó el host abajo (para el elapsed).
- Derivado `EsperandoReinicio(ahora)` = `PruebaEnCursoDesde` no nulo y el host todavía no volvió a subir.

Ciclo de `PresupuestoDeApagado`: `NuncaVerificado`/`Vencido` → **[disparar]** ordena apagado del host +
`IniciarPrueba(ahora)` → el sondeo observa el host abajo → captura `HostVistoAbajoEn` y `Verificar(...)` con
evidencia *"apagado cronometrado en N s (medido)"* y vigencia 180 d → el botón sigue bloqueado
(`EsperandoReinicio`) → el sondeo observa el host arriba → `RearmarPorReinicio(ahora)` limpia el marcador →
el botón vuelve a habilitarse.

`ReencendidoPorPlaca`: **[Arrancó solo]** verifica + `IniciarPrueba` (gate hasta reinicio); **[No arrancó]**
refuta (bloqueo permanente, sin cambios). No dispara apagado.

## Cambios por capa

**Dominio** (`SAI.Service.Core.Domain/Verificaciones/`)
- `Verificacion.cs`: campos `PruebaEnCursoDesde`, `HostVistoAbajoEn`; métodos `IniciarPrueba(ahora)`,
  `RegistrarHostAbajo(ahora)`, `RearmarPorReinicio(ahora)`, y `EsperandoReinicio(ahora)`. Reusar el patrón
  de transición/guards ya existente (`Verificar`, `Refutar`, `EstadoEfectivo`).

**Aplicación** (`SAI.Service.Core.Application/`)
- `Abstractions/IAdaptadorConexion.cs`: nueva op `Task<ResultadoAccion> OrdenarApagadoOrdenadoHostAsync(CancellationToken ct)`
  (ordena el apagado ordenado del host para cronometrar su ventana; distinta del corte con retorno).
- `Equipos/ServicioVerificacion.cs`: `DispararPruebaPresupuestoAsync` (ordena apagado + `IniciarPrueba`,
  ya no recibe segundos); en `RegistrarReencendidoAsync(true)` marca `IniciarPrueba`. Nuevo
  `ObservarHostParaVerificacionesAsync(EstadoSai, ct)` que, dado el estado leído, avanza los marcadores
  (host abajo → captura elapsed y `Verificar`; host arriba → `RearmarPorReinicio`) usando el umbral.
- `Monitoreo/ServicioMonitoreo.cs`: en `SondearAsync`, tras leer `EstadoSai`, invocar
  `ObservarHostParaVerificacionesAsync` (misma ronda/alcance) para el rearme automático.
- Opciones: `OpcionesVerificacion { CargaHostAbajoPct }` leída como las demás en `DependencyInjection`.

**Infraestructura** (`SAI.Service.Core.Infrastructure/`)
- `Adaptadores/AdaptadorConexionSimulado.cs`: modelar el ciclo de vida del host en el tiempo tras
  `OrdenarApagadoOrdenadoHost` (trabajando → apagándose: `CargaSalidaPorcentaje` baja a 0 en ~N s →
  abajo → reiniciando → carga vuelve a 35%), replicando el patrón temporal ya usado para la descarga de
  batería (`_descargaInicio`/`DuracionDescarga`). Config de duraciones para la demo.
- `Adaptadores/Nut/AdaptadorConexionNut.cs`: implementar la op mapeándola al write path autenticado ya
  existente (shutdown ordenado); en desarrollo el default sigue siendo el simulado.
- `Persistencia/Configuraciones/ModeloEquipos.cs` (`ConfigurarVerificaciones`): mapear las dos columnas
  nuevas.
- Nueva migración `EsquemaPruebaApagado` (patrón de las migraciones existentes en `Persistencia/Migraciones/`).

**Web** (`SAI.Service.Core.Web/Components/Pages/PanelDeVerificaciones.razor`)
- `PresupuestoDeApagado`: quitar `MudNumericField`; el botón pasa a "Ejecutar prueba de apagado" y llama a
  `DispararPruebaPresupuestoAsync`. Si `EsperandoReinicio` → botón deshabilitado + leyenda.
- `ReencendidoPorPlaca`: tras presionar, si `EsperandoReinicio` → botones deshabilitados + misma leyenda.
- Reword del paso 1 (`PasosSupuesto`) → *"Con el host trabajando, esta prueba inicia su apagado ordenado
  (el sistema operativo cierra todos los servicios)."*
- Ajustar `QueSePrueba`/`Actúa` de `PresupuestoDeApagado` para reflejar que la prueba dispara y cronometra.

**Tests**
- Dominio (`tests/SAI.Service.Core.Domain.Tests`): transiciones de `Verificacion` (IniciarPrueba →
  EsperandoReinicio; RegistrarHostAbajo captura elapsed; RearmarPorReinicio limpia; refutado no arranca prueba).
- Integración (`tests/SAI.Service.Core.Integration.Tests`): disparar → simulado host abajo → `Verificado`
  con segundos medidos; sondeo detecta reinicio → botón rearmado; gating impide re-disparo mientras
  `EsperandoReinicio`.

**Changelog**: entrada *Etapa 4·E* en `CHANGELOG.md` (formato de las etapas previas).

## Verificación end-to-end

 1. `dotnet build` y `dotnet test` (los tres proyectos de test) en verde.
 
 2. Correr la app con el simulado (contenedor `sai-web-e1`, ver memoria de despliegue local) y en
   `/panel-de-verificaciones`:
   - Presionar "Ejecutar prueba de apagado" → botón se bloquea con la leyenda.
   - Ver la carga caer (simulado) → la tarjeta pasa a *Verificado* con los segundos medidos.
   - Al recuperarse la carga (reinicio simulado) → el botón se re-habilita.
   - Repetir la mecánica de bloqueo/leyenda en "Encendido por presencia de energía".
 3. Confirmar que con el adaptador **simulado** ningún apagado real se dispara.

## Fuera de alcance

- Validar el write path NUT real contra un host físico (sigue el principio de solo-simulado en desarrollo).
- Cambios en los otros dos supuestos (`SenalEnBateria`, `CorteConRetorno`).

```
el proyecto es `/DEV/SAI.Service.Core/SDD/Docs/README.md`

quisiera saber  como va a medir el tiempo de apagado del host?, si el boton manda manda el shutown al host, el host va a bajar todos los servicios, incluido el del SAI, no tiene manera 

- no modifiques nada, solo resolve mi cuestión


---

# Lo que sí es coherente construir
  1. Toda la mecánica de UX + rearme (bloqueo del botón, leyenda "se activará cuando reinicie", EsperandoReinicio, RearmarPorReinicio por reinicio). Esto funciona de verdad porque los marcadores se persisten en SQLite: al reiniciar, el contenedor lee el marcador, ve la carga de vuelta y rearma. Vale para PresupuestoDeApagado y para ReencendidoPorPlaca.
Los rewords de texto (paso 1, QueSePrueba, Actúa). Sin problema.
  2. El disparo del apagado ordenado (OrdenarApagadoOrdenadoHostAsync en el puerto + NUT + simulado). Sirve — solo que su efecto termina matando al proceso, que es justamente el punto.
Lo que NO es coherente
  3. Cronometrar la ventana de apagado desde el sondeo (el sondeo observa el host abajo → captura HostVistoAbajoEn → "apagado cronometrado en N s (medido)"). El observador muere durante la medición. Sacá esto del alcance tal como está escrito.

# Para el número de tiempo, la salida honesta (elegí uno)

  - Recomendado — no auto-medirlo: dejá PresupuestoDeApagado como carga manual / medición en la ventana de mantenimiento (CU-10), que es exactamente donde el "resto del apagado del SO" ya figura como PENDIENTE en los NFR. Es lo alineado con la arquitectura y no inventa un observador que no existe. La feature entonces aporta UX + rearme, y el segundo lo sigue poniendo el operador o CU-10.

  - Si querés un número automático: medí una cota inferior — sellar en persistencia en el handler de SIGTERM ("servicios abajo en Tx") y restar PruebaEnCursoDesde. Pero etiquetalo como lo que es ("host tardó ≥ N s en tumbar el SAI"), nunca como el presupuesto de apagado completo.

  - Mejor pero dependiente de hardware: al reiniciar, preguntarle a la UPS/NUT el timestamp del último corte de salida (si el firmware lo expone) y derivar la ventana. Requiere verificar qué da el equipo real → no lo comprometas hasta validarlo.

# El mensaje para devolverle al agente

 > La premisa "el sondeo cronometra el apagado" es incompatible con ADR-25 + Pipeline C1: el sondeo corre en el contenedor que el propio apagado detiene. Construí la UX + el rearme por reinicio (eso sí es viable con marcadores persistidos), pero el tiempo de apagado no es auto-medible desde el host: dejalo manual/CU-10, o medí y rotulá explícitamente una cota inferior. No lo reportes como "medido".

En una línea: el sujeto que mide y el objeto medido son el mismo proceso; quedate con el rearme —que sí sobrevive al reinicio— y sacá el cronometraje automático del alcance.