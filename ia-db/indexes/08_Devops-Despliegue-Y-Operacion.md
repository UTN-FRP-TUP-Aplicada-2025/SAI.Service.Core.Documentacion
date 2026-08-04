# 08 — DevOps, despliegue y operación

> **Propósito**: cómo se compila, versiona, integra, despliega y opera el servicio, y qué hacer cuando
> falla.
> **Fuentes primarias**: [`.github/workflows/ci.yml`](../../../SAI.Service.Core/.github/workflows/ci.yml) ·
> [`scripts/`](../../../SAI.Service.Core/scripts/) · [`.devcontainer/`](../../../SAI.Service.Core/.devcontainer/) ·
> [`SDD/Docs/09-Devops/`](../../../SAI.Service.Core/SDD/Docs/09-Devops/) ·
> [`SDD/Docs/11-Documentacion/Guia-Despliegue-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/11-Documentacion/Guia-Despliegue-v1.0.md) ·
> [`Runbook-Operacion-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/11-Documentacion/Runbook-Operacion-v1.0.md)

---

## 1. Entorno de desarrollo

El **único requisito del host es Docker**: el SDK de .NET 10 vive en el Dev Container
(`mcr.microsoft.com/devcontainers/dotnet:1-10.0`, con docker-in-docker para construir la imagen de
producción desde adentro). Puertos reenviados: 8080 (HTTP) y 8443 (HTTPS).

```bash
devcontainer up --workspace-folder .      # o «Reopen in Container» en VS Code
./scripts/build-all.sh                    # Release, cero warnings
./scripts/run.sh SAI.Service.Core.Web     # equivalente: ./scripts/run-all.sh
curl -sf http://localhost:8080/health     # {"estado":"ok",...}
```

| Script | Qué hace |
| --- | --- |
| `build-all.sh` | `restore` + `build -c Release` de la solución |
| `build.sh <proyecto>` | Compila un proyecto (lo busca bajo `src/` y `tests/`) |
| `run.sh <proyecto>` | Corre un proyecto con `ASPNETCORE_ENVIRONMENT=Development` |
| `run-all.sh` | Atajo a `run.sh SAI.Service.Core.Web` (es un web-monolith: correr «todo» es correr el host) |
| `dev.sh` | Levanta el Dev Container y compila adentro |

**La depuración va por F5** (`.vscode/launch.json`, configuración *coreclr*), nunca por los scripts.

## 2. CI

| | |
| --- | --- |
| Definido en | `.github/workflows/ci.yml` |
| Dispara en | `push` y `pull_request` sobre `main` |
| Target | `ubuntu-latest` · .NET 10 · `linux/amd64` (matriz de un solo eje) |
| Stages implementados | **1–4**: restore · `dotnet format --verify-no-changes` · build Release (cero warnings) · `dotnet test` |

El pipeline **diseñado** en `09-Devops/Pipeline-CI-CD-v1.0.md` tiene 10 stages; los stages 5–10 (e2e con
bUnit/Playwright, SCA, SBOM CycloneDX, build de imagen, firma cosign keyless, publicación al registro
privado) están especificados pero **todavía no implementados** en el workflow — ver
[11_Estado-Y-Pendientes](11_Estado-Y-Pendientes.md).

Disparadores previstos para el pipeline completo: PR → stages 1–8; push a `main` → 1–8; push de tag
`v<X.Y.Z>` → 1–10 (agrega firma y publicación); `schedule` semanal → SCA para detectar CVE nuevas.

## 3. Versionado y ramas

| Aspecto | Regla |
| --- | --- |
| Versionado | **SemVer 2.0.0**; MAJOR por romper `/api/v1/` o el esquema, MINOR por etapa cerrada, PATCH por fix |
| Commits | **Conventional Commits 1.0.0** (`feat`, `fix`, `docs`, `chore`) |
| Herramienta | **MinVer**: la versión se deriva del tag de git más cercano; no se commitea |
| Branching | Trunk-based: `main` siempre desplegable, ramas `etapa/NN-<slug>`, merge solo por PR |
| Canales | SemVer + `latest`; preestreno `-alpha.N` se construye pero **no se publica** |
| Merge y borrado de ramas | Los hace el humano; el agente deja la rama lista y pusheada |

## 4. Ambientes

Dos niveles, **sin staging** (ADR-24: no habría a qué SAI conectarlo).

| Variable | DEV | PROD |
| --- | --- | --- |
| `ASPNETCORE_ENVIRONMENT` | `Development` | `Production` |
| `Sai__Adaptador` | `Simulado` | `Nut` |
| `Sai__Nut__Usuario` / `__Password` | — | usuario NUT de escritura (secreto) |
| `Jwt__ClaveFirma` | clave de desarrollo | secreto ≥ 32 bytes por entorno |
| `DataProtection__RutaLlaves` | vacío (efímero) | carpeta persistente montada (p. ej. `/keys`) |
| `ConnectionStrings__Sai` | `Data Source=sai.db` | `Data Source=/data/sai.db` (volumen) |
| TLS | HTTPS 8443 con `dev-certs` | autofirmado en la LAN (ranura ADR-20/ADR-26) |

**Ningún secreto va en `appsettings.json`.**

## 5. Despliegue

**Estado**: no hay `Dockerfile` ni `docker-compose.yml` en el repositorio. La imagen de producción es una
ranura declarada (ADR-25/ADR-26). El despliegue de referencia hoy es el host publicado.

| Topología | Estado |
| --- | --- |
| Proceso publicado en el host, junto a NUT nativo | Soportada (la de hoy) |
| Contenedor con NUT adentro, USB por `udev` | Pendiente (objetivo de ADR-25) |

```bash
git fetch origin && git checkout <tag-o-main>
./scripts/build-all.sh
dotnet publish src/SAI.Service.Core/SAI.Service.Core.Web --configuration Release -o ./.publish
export ASPNETCORE_ENVIRONMENT=Production Sai__Adaptador=Nut \
       Jwt__ClaveFirma="<secreto>" DataProtection__RutaLlaves=/keys \
       ConnectionStrings__Sai="Data Source=/data/sai.db" \
       Sai__Nut__Usuario="<usuario>" Sai__Nut__Password="<secreto>"
dotnet ./.publish/SAI.Service.Core.Web.dll
```

Al arrancar aplica migraciones y siembra (rol, reglas de derivación, política inicial en solo aviso,
fuente `fd-gmao-externo`). La primera vez desvía a `/alta-inicial`.

**Verificación post-despliegue**: `/health` en `ok`; `upsc sai@localhost ups.status` sin error; el panel
muestra valores **reales** (no los fijos del simulado); los cuatro supuestos aparecen en «no verificado» y
la modalidad efectiva es solo aviso — eso es lo **correcto**, no un defecto.

**Rollback**: volver al tag anterior, republicar y arrancar. **Punto de no retorno: las migraciones.**
EF Core migra hacia adelante al arrancar; antes de desplegar una versión con migración nueva hay que
respaldar `sai.db` (copia del archivo con el servicio detenido). El rollback de código sin rollback de
esquema solo es seguro si la migración fue aditiva — el caso habitual en un modelo append-only.

**Contenedor** (contrato que la imagen deberá cumplir): puertos 8080/8443; volúmenes para el keyring y
para la base SQLite; el USB **no** lo toma el servicio sino el driver de NUT; healthcheck
`curl -sf http://localhost:8080/health` cada 30 s con 3 reintentos; 256–512 MB y 0,5 vCPU alcanzan.

## 6. Operación

Tres señales resumen la salud: `/health` responde · el sondeo escribe muestras · el adaptador alcanza al
SAI.

| Métrica | Atención | Alarma | Acción |
| --- | --- | --- | --- |
| Antigüedad del último sondeo | > 15 s | > 3 sondeos perdidos | OPS-01 |
| Supuestos verificados | < 4 | 0 con política ≠ solo aviso | Ventana de mantenimiento (CU-10) |
| Días sin protección | > 0 | sin cobertura vigente | Revisar sustitución del SAI (CU-09) |
| Estado de la última `Accion` | `EfectoNoConfirmado` | repetido | OPS-02 |

Logs a stdout, nivel `Information` (`Microsoft.AspNetCore` en `Warning`). Patrones útiles: `accion` /
`apagado`, `nut` / `alcanzable`, `Applying migration`.

## 7. Incidentes conocidos

| ID | Síntoma | Causa habitual | Resolución |
| --- | --- | --- | --- |
| OPS-01 | El panel muestra el SAI «no alcanzable»; muestras de calidad perdida | Problema de NUT, no del servicio | `upsc sai@localhost ups.status`; reiniciar `upsd`/driver; verificar la ruta física del USB. El servicio se recupera solo en la ronda siguiente |
| OPS-02 | `Accion` en `EfectoNoConfirmado` | Faltan credenciales NUT de **escritura** | Configurar `Sai__Nut__Usuario`/`Password` con un usuario `upsmon master` con permiso `shutdown.return`. Sin ellas queda en solo lectura **por diseño** |
| OPS-03 | Tras reiniciar se pierden sesiones y los formularios dan HTTP 400 | Keyring de DataProtection efímero | Montar y configurar `DataProtection__RutaLlaves` (ADR-29) |
| OPS-04 | Ante un corte el host no se apaga aunque el servicio registró el disparo | El apagado del host lo ejecuta la cadena NUT | `MONITOR sai@localhost 1 <usuario> <clave> master` en `upsmon.conf` y usuario `upsmon master` en `upsd.users`; recargar NUT. Evidencia de ciclo limpio: `last -x` muestra «shutdown», no «crash» |

**Escalamiento**: al ser un servicio de administrador único, se escala hacia la documentación. Un
incidente no catalogado se registra en la bitácora de eventualidades con síntoma, causa e **intentos
descartados**, y si es un patrón nuevo se suma como `OPS-XX`. Volver a solo aviso siempre es una posición
segura.

## 8. Eventualidades ya vividas (lo que no funcionó)

| ID | Situación | Lección |
| --- | --- | --- |
| EVE-01 | `upsmon` no disparaba el apagado | Faltaba `MONITOR` y el usuario era `slave`, no `master` |
| EVE-02 | Sin `dotnet` en el entorno de generación | Usar contenedor SDK efímero; `--startup-project` a Infrastructure |
| EVE-03 | `CultureNotFoundException` en runtime | `InvariantGlobalization=true`: **no hay culturas nombradas**; nada de `CultureInfo.GetCultureInfo("es-AR")` |
| EVE-04 | Formularios SSR con HTTP 400 antiforgery | Un solo token (el de `EditForm`) y `UseAntiforgery()` después de `UseAuthentication()` |
| EVE-05 | Sesiones perdidas al reiniciar | Persistir el keyring con `SetApplicationName` + `PersistKeysToFileSystem` |
| EVE-06 | Conflicto de merge con dos migraciones paralelas | **Regenerar** la migración de la segunda rama sobre el modelo fusionado |
| EVE-07 | Jerga de NUT filtrándose a la UI | Dos audiencias: operador a la UI, detalle técnico al log |
