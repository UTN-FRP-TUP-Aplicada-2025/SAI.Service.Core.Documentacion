Tengo todo para el plan. Acá va el plan del Sprint 0 para tu aprobación antes de escribir código.

Bloque A — Cerrar los 4 ADR abiertos
ADR-22 (contrato del puerto del adaptador de conexión) — lo propongo yo (es técnico). El puerto IAdaptadorConexion en Application, con las 4 operaciones del intake, y resultados que se validan por efecto observado (ADR-11):


LeerEstadoAsync() → EstadoSai            (variables NUT + procedencia)
ProbarConectividadAsync() → ResultadoConectividad
OrdenarApagadoConRetornoAsync(retardo) → ResultadoAccion  (efecto observado)
LanzarTestBateriaAsync() → ResultadoAccion
Tres implementaciones previstas (NUT, directo, simulada); en Sprint 0 solo defino el contrato + el esqueleto de la simulada.

ADR-21 (contrato del endpoint de rectificación del 409) — diferido a la Etapa 5 (depende de BT-28, el endpoint de ingesta), tal como fija el Mini-Plan. Queda Propuesto.

ADR-19 y ADR-20 — necesito tu decisión (son infraestructura tuya); te las pregunto abajo.

Bloque B — Andamiaje + panel base (lo genero yo)
Estructura completa del §16, en código .NET real (pero sin compilar acá — validás en tu Dev Container):


.devcontainer/devcontainer.json · .vscode/{launch,tasks}.json
scripts/{build,run,build-all,run-all,dev}.sh
Directory.Build.props (net10.0, nullable, TreatWarningsAsErrors)
src/SAI.Service.Core/{Domain,Application,Infrastructure,Api,Web}/  (5 .csproj, refs hacia el dominio)
tests/{Domain.Tests,Application.Tests,Integration.Tests}/
.github/workflows/ci.yml  (build + test + lint, del pipeline de 09)
SAI.Service.Core.sln
Web: Blazor interactive server + MudBlazor, con el shell base de la maqueta (menú lateral + barra superior + sello de versión) — BT-06.
Compila y corre por run.sh; el panel base navega según la maqueta aprobada. Sin lógica de negocio (eso es Etapa 1+).
Verificación: genero todo, hacés devcontainer up && ./scripts/build-all.sh y validás que compila/corre. Yo no puedo compilar acá (no hay dotnet), lo declaro por transparencia.

Ahora las 2 decisiones que son tuyas:


---
AskUserQuestion

## ADR-19 — ¿Dónde corre NUT (la herramienta de acceso al SAI)?

Seleccione una de las siguientes opciones:

- [x] **En el contenedor (Recomendado)**
  - Un único artefacto desplegable.
  - El contenedor recibe el dispositivo USB anclado por ruta física (ADR-03).
  - Coherente con la simplicidad operativa de un solo host/admin (T-01).

- ☐ **En el host, servicio cliente TCP de `upsd`**
  - El contenedor no necesita el USB; separa el driver del servicio.
  - Requiere credenciales de `upsd` como secreto en runtime.
  - Implica operar y mantener dos piezas.

- ☐ **Other**


## ADR-20 — ¿Cómo se cifra el panel y la API en la LAN?

Seleccione una de las siguientes opciones:

- [x] **Certificado autofirmado en Kestrel (Recomendado)**
  - Sin piezas extra: cifra la cookie y el login en la LAN.
  - Contra: advertencia de certificado en el navegador y rotación manual.
  - Simple para un solo host.

- ☐ **Terminación TLS en reverse proxy del host**
  - Centraliza el certificado y descarga al servicio del TLS.
  - Contra: una pieza más a operar.
  - El WebSocket de Blazor debe pasar bien por el proxy.

- ☐ **Other**


## ¿Arranco el Sprint 0 con este plan (Bloque A + B)?

Seleccione una de las siguientes opciones:

- [x] **Sí, generá el andamiaje**
  - Cierro ADR-19/20/22 con tus decisiones.
  - Genero el esqueleto de los 5 assemblies.
  - Creo el dev container + CI al nivel base.
  - Hago el primer commit listo para el PR.

- ☐ **Ajustar algo primero**
  - Querés cambiar el alcance del Sprint 0 antes de que genere código.

- ☐ **Other**