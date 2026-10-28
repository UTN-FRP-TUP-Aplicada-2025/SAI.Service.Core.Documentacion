# 00 — Índice maestro

> **Propósito:** visión general de SAI.Service.Core en una lectura: qué problema resuelve, qué es, con qué stack exacto está construido, qué decisiones lo estructuran y en qué índice está el detalle de cada tema. No desarrolla ningún dominio: para eso están los índices `01`–`11`.
> **Fuente primaria:** `SDD/Docs/00-Contexto/Vision-Producto-v1.0.md`, `AGENTS.md`, `Directory.Build.props`, `src/SAI.Service.Core/*/*.csproj`, `tests/*/*.csproj`, `Dockerfile`, `.devcontainer/devcontainer.json`, `.github/workflows/ci.yml` y `SDD/Docs/05-Arquitectura-Tecnica/Decisiones-Arquitectura-v1.0.md`.
> Rutas relativas a la raíz del repo `SAI.Service.Core`. Corte: `main` en `6fd23d5` (2026-08-04).

---

## 1. Qué es, en una tabla

| Aspecto | Valor | Fuente |
| --- | --- | --- |
| Producto | Servicio que administra el ciclo de vida y el monitoreo de un SAI (UPS) que respalda un host Linux, y ordena su apagado con retorno garantizado | `AGENTS.md` («Qué es») |
| Tipo de solución | `web-monolith`: un proceso, un contenedor, un archivo SQLite | `SDD/Docs/05-Arquitectura-Tecnica/Arquitectura-Solucion-v1.0.md` §2 |
| Superficies | Panel Blazor Server (MudBlazor) para el administrador + API REST `/api/v1` para la ingesta de terceros | `AGENTS.md`; detalle en `05_Interfaces-Panel-Y-Api.md` |
| Acceso al equipo | Sólo por NUT (`nutdrv_qx` + `upsd`), a través del puerto `IAdaptadorConexion` | ADR-01, ADR-27; detalle en `06_Infraestructura-Y-Adaptador-Sai.md` |
| Alcance deliberado | Un servidor, un SAI activo, un administrador; el crecimiento previsto es en profundidad de histórico, no en cantidad de equipos ni usuarios | `SDD/Docs/00-Contexto/Vision-Producto-v1.0.md` §4 |
| Usuarios | Administrador único (dueño, implementador y beneficiario: una sola persona) + un sistema GMAO externo que empuja intervenciones | `Vision-Producto-v1.0.md` §2 |
| Proceso de construcción | SDD v3.0: Intake → categorías 00–11 → auditorías A–H → código por etapas (Sprint 0 + Etapas 1–5) | `SDD/Docs/README.md`; detalle en `09_Documentacion-Sdd.md` |

---

## 2. Problema que resuelve

El monitoreo básico y el apagado ante un corte ya los resuelven herramientas existentes; el proyecto **no los reconstruye**. Ataca cuatro carencias y un problema crítico (`Vision-Producto-v1.0.md` §1):

| # | Carencia | Respuesta del servicio |
| --- | --- | --- |
| 1 | No hay histórico de salud de la batería: el equipo no expone indicador y la bandera «reemplazar batería» (`LB`) nunca se enciende | Salud propia como tendencia relativa de la caída de tensión en el autotest, con confianza explícita (ADR-13) |
| 2 | No hay modelo de ciclo de vida de los equipos (altas, recambios, reparaciones, sustituciones) | Catálogo + inventario + vínculos temporales + historia append-only (ADR-04, ADR-05, ADR-07) |
| 3 | No hay verificación viva de los supuestos (p. ej. que el host vuelva a encender solo) | Cuatro supuestos verificados con vigencia; sin ellos el apagado queda bloqueado (ADR-10, ADR-14) |
| 4 | No hay panel remoto ni administración; el software del fabricante se descartó por una RCE sin parche | Panel Blazor + API con autenticación (ADR-16, ADR-28) |
| ★ | **Garantizar el reencendido:** el host sólo arranca ante una transición sin energía → con energía, así que el SAI debe cortar su salida aunque la red vuelva durante la cuenta regresiva | Modalidad `CicloForzado` / `shutdown.return`, nunca `shutdown.stop` (ADR-09) |

Promesa central (`Vision-Producto-v1.0.md` §3): (1) que el servidor vuelva a encenderse solo, **negándose a apagarlo mientras no pueda probarlo**; (2) construir el histórico de salud, ciclo de vida y costos que ninguna herramienta construye para este equipo.

---

## 3. Stack con versiones exactas

| Pieza | Versión | Fuente |
| --- | --- | --- |
| Marco | `net10.0`, `LangVersion latest`, `Nullable` y `ImplicitUsings` activos | `Directory.Build.props` |
| Rigor de compilación | `TreatWarningsAsErrors`, `EnforceCodeStyleInBuild`, `AnalysisLevel latest-recommended`, `EnableNETAnalyzers`, `InvariantGlobalization` | `Directory.Build.props` |
| Versión del producto | `0.1.0-alpha.1` (fija; MinVer previsto, no incorporado) | `Directory.Build.props`; ver `11_Estado-Y-Pendientes.md` |
| UI | Blazor Server (interactive server) + `MudBlazor` 8.5.0 | `src/SAI.Service.Core/SAI.Service.Core.Web/SAI.Service.Core.Web.csproj` |
| Autenticación | ASP.NET Core Identity (`Microsoft.AspNetCore.Identity.EntityFrameworkCore` 10.0.0) + `Microsoft.AspNetCore.Authentication.JwtBearer` 10.0.0 | `.../Web/SAI.Service.Core.Web.csproj`, `.../Infrastructure/SAI.Service.Core.Infrastructure.csproj` |
| Persistencia | SQLite con `Microsoft.EntityFrameworkCore.Sqlite` 10.0.0 (+ `Design` 10.0.0) | `.../Infrastructure/SAI.Service.Core.Infrastructure.csproj` |
| Hosting de fondo | `Microsoft.Extensions.Hosting.Abstractions` 10.0.0 | ídem |
| Pines de seguridad | `SQLitePCLRaw.bundle_e_sqlite3` 2.1.12, `System.Security.Cryptography.Xml` 10.0.10 | ídem |
| Protocolo del SAI | NUT por TCP a `upsd:3493` (cliente propio `ClienteNut`) | `.../Infrastructure/Adaptadores/Nut/` |
| Pruebas | xunit 2.9.2, xunit.runner.visualstudio 2.8.2, FluentAssertions 6.12.2, Microsoft.NET.Test.Sdk 17.12.0, coverlet.collector 6.0.2, Microsoft.AspNetCore.Mvc.Testing 10.0.0 | `tests/*/*.csproj` |
| Imagen | `mcr.microsoft.com/dotnet/sdk:10.0` (build) → `mcr.microsoft.com/dotnet/aspnet:10.0` (runtime), no-root uid 1001, `EXPOSE 8080` | `Dockerfile` |
| Entorno de desarrollo | Dev Container `mcr.microsoft.com/devcontainers/dotnet:1-10.0` + `docker-in-docker:2`; no hay `dotnet` en el host | `.devcontainer/devcontainer.json`, `AGENTS.md` |
| CI | GitHub Actions, un workflow (`ci.yml`), .NET `10.0.x` en `ubuntu-latest` | `.github/workflows/ci.yml` |

---

## 4. Mapa de capas en una línea

`Web` (host, panel, Identity/JWT, composition root) → {`Api` (`/health`, `/api/v1`), `Infrastructure` (EF/SQLite, adaptadores NUT/simulado, hosted services)} → `Application` (casos de uso + puertos) → `Domain` (framework-free). Diamante, no cadena lineal como dice ADR-15: ver `01_Arquitectura.md` §3.

```
sondeo 5 s ─► lectura del SAI ─► Muestra ─► DerivadorEventos ─► DisparoApagado
            ─► política vigente + EvaluadorModalidad (4 supuestos) ─► Accion (SoloAviso | Bloqueada | Ejecutada | EfectoNoConfirmado)
```

Recorrido completo: `01_Arquitectura.md` §5.

---

## 5. Principios y decisiones estructurantes

Las 29 ADR viven en `SDD/Docs/05-Arquitectura-Tecnica/Adrs/` (24 Aceptado · 1 Propuesto · 4 Superado, según el `Estado` de cada archivo; catálogo completo en `01_Arquitectura.md` §7). Las que explican las reglas del sistema:

| ADR | Decisión en una línea |
| --- | --- |
| 01 | Acceso al SAI sólo por NUT; sin traductor de protocolo propio |
| 02 / 27 | Puerto `IAdaptadorConexion` en Application, implementaciones en Infrastructure (NUT, simulado; directo diseñado); resultados por efecto observado |
| 04 | Historia append-only como disciplina, sin event store ni CQRS |
| 05 | Vigencia como entidad con intervalo semiabierto `[Desde, Hasta)` |
| 06 | Procedencia obligatoria: todo valor lleva su `Origen` (`Valor<T>`) |
| 07 | Catálogo, inventario, vínculos temporales e historia separados |
| 09 | `CicloForzado`: iniciado el apagado, el corte no se cancela; nunca `shutdown.stop` |
| 10 | Arranque seguro en `SoloAlerta`; sin los supuestos verificados la acción se bloquea |
| 11 | Toda acción se valida por efecto observado; 3 sondeos perdidos ⇒ `DesconexionUsb` |
| 12 | El disparo se decide por tiempo en batería y `battery.voltage`, nunca por `LB` ni `battery.charge` |
| 13 | Salud de batería = tendencia relativa de la caída de tensión, con confianza explícita |
| 15 | Clean Architecture en cinco assemblies; un solo desplegable |
| 16 / 28 / 29 | Administrador único con Identity y cookie; API con Bearer JWT vía ROPC; keyring de DataProtection persistido |
| 18 | SQLite + EF Core con migraciones aplicadas al arranque; un archivo, un escritor |
| 24 / 25 / 26 | Dos ambientes (DEV, PROD en `<host de producción>`); NUT dentro del contenedor; TLS autofirmado en Kestrel |

Principios no negociables que fija `AGENTS.md` («Límites»): append-only, procedencia obligatoria, estado base seguro (arranque en solo aviso; no apagar sin los cuatro supuestos) y acceso al SAI sólo por `IAdaptadorConexion`.

---

## 6. Números de referencia

| Dato | Valor | Detalle en |
| --- | --- | --- |
| Proyectos de la solución | 8 (5 de `src/` + 3 de `tests/`) | `01_Arquitectura.md` |
| Archivos `.cs` versionados en `src/` | 170 | `git ls-files src` |
| Casos de uso / reglas de negocio / invariantes | 12 CU · 13 RN · 21 I | `04_…`, `02_…` |
| Tablas de dominio / migraciones | 21 (+ 7 de Identity) / 15 | `03_Modelo-De-Datos.md` |
| Páginas del panel / rutas / endpoints HTTP | 13 páginas con `@page` (14 rutas: `/` y `/panel-estado-en-vivo` son la misma) / 6 | `05_Interfaces-Panel-Y-Api.md` |
| Métodos de prueba (grep) | ~270 (Domain 124 · Application 31 · Integration 115) | `07_Calidad-Y-Pruebas.md` |
| Backlog | 26 US (161 SP) · 30 BT · 7 épicas | `04_…`, `11_…` |
| Estado del plan | Etapa 5 Realizada; sin release ni tags | `11_Estado-Y-Pendientes.md` |

---

## 7. Mapa «tema → índice»

| Tema | Índice |
| --- | --- |
| Capas, grafo de `.csproj`, `Program.cs`, recorrido de punta a punta, catálogo de ADR | `01_Arquitectura.md` |
| Entidades, valores, vigencias, supuestos, modalidades, RN e invariantes | `02_Dominio-Y-Reglas.md` |
| `SaiDbContext`, tablas, migraciones, append-only, RC-01…RC-09 | `03_Modelo-De-Datos.md` |
| Servicios de Application, puertos, CU → servicio → US → BT, códigos de resultado | `04_Aplicacion-Y-Casos-De-Uso.md` |
| Panel Blazor, autenticación, endpoints REST, guía UX y maqueta | `05_Interfaces-Panel-Y-Api.md` |
| Adaptadores NUT y simulado, protocolo NUT, hosted services, claves `Sai:*` | `06_Infraestructura-Y-Adaptador-Sai.md` |
| Suites de prueba, compilación estricta, estrategia SDD 08, auditorías | `07_Calidad-Y-Pruebas.md` |
| Scripts, Dev Container, Dockerfile, CI, versionado, despliegue, runbook | `08_Devops-Despliegue-Y-Operacion.md` |
| Árbol `SDD/`, convención de nombres de archivo y estados, contrato de `AGENTS.md` | `09_Documentacion-Sdd.md` |
| Vocabulario, enums, variables NUT, familias de IDs | `10_Glosario.md` |
| Versión, ramas, PR, pendientes, desincronizaciones documentales | `11_Estado-Y-Pendientes.md` |

Cada índice registra sus propias **divergencias** entre documentación y código; ante contradicción, prevalece el código de `main`.
