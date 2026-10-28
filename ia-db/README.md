# ia-db — SAI.Service.Core

> **Instrucción para IA:** este archivo es el **punto de entrada único** a la base de conocimiento de `SAI.Service.Core`. Leé primero el resumen ejecutivo; después ubicá tu tema en la tabla de navegación y cargá **sólo 1–2 índices** de `indexes/`. Abrí las fuentes del repo (cada afirmación de los índices cita su ruta) únicamente cuando el índice no alcance. No recorras el repositorio entero si la respuesta está indexada.

---

## Navegación

| Necesito saber… | Leo este índice |
| --- | --- |
| De qué trata, stack, decisiones clave, qué índice cubre cada tema | [`00_MASTER-INDEX.md`](indexes/00_MASTER-INDEX.md) |
| Capas, grafo de proyectos, `Program.cs`, flujo sondeo → apagado, las 29 ADR | [`01_Arquitectura.md`](indexes/01_Arquitectura.md) |
| Entidades, supuestos, modalidades, vigencias, procedencia, RN e invariantes | [`02_Dominio-Y-Reglas.md`](indexes/02_Dominio-Y-Reglas.md) |
| Tablas, mapeo EF Core, migraciones, append-only, reglas conceptuales RC | [`03_Modelo-De-Datos.md`](indexes/03_Modelo-De-Datos.md) |
| Qué servicio implementa cada caso de uso, puertos, trazabilidad CU → US → BT | [`04_Aplicacion-Y-Casos-De-Uso.md`](indexes/04_Aplicacion-Y-Casos-De-Uso.md) |
| Páginas del panel, autenticación (cookie/JWT), endpoints REST, guía UX y maqueta | [`05_Interfaces-Panel-Y-Api.md`](indexes/05_Interfaces-Panel-Y-Api.md) |
| Adaptador NUT y simulado, protocolo NUT, hosted services, configuración `Sai:*` | [`06_Infraestructura-Y-Adaptador-Sai.md`](indexes/06_Infraestructura-Y-Adaptador-Sai.md) |
| Suites de prueba, compilación estricta, cobertura y gates del SDD, auditorías | [`07_Calidad-Y-Pruebas.md`](indexes/07_Calidad-Y-Pruebas.md) |
| Scripts, Dev Container, Dockerfile, CI, versionado, despliegue y runbook | [`08_Devops-Despliegue-Y-Operacion.md`](indexes/08_Devops-Despliegue-Y-Operacion.md) |
| Dónde está cada documento del árbol `SDD/`, convención de nombres de archivo, `AGENTS.md` | [`09_Documentacion-Sdd.md`](indexes/09_Documentacion-Sdd.md) |
| Qué significa un término, enum, variable NUT o prefijo de ID | [`10_Glosario.md`](indexes/10_Glosario.md) |
| Versión, ramas, PR, qué falta, qué documentos quedaron desactualizados | [`11_Estado-Y-Pendientes.md`](indexes/11_Estado-Y-Pendientes.md) |

---

## Resumen ejecutivo

| Dato | Valor |
| --- | --- |
| Proyecto | `SAI.Service.Core` |
| Tipo | `web-monolith` .NET: un proceso (`SAI.Service.Core.Web`), un contenedor, un archivo SQLite |
| Stack | .NET 10 (`net10.0`) · Blazor Server + MudBlazor 8.5.0 · ASP.NET Core Identity + JWT Bearer 10.0.0 · EF Core 10.0.0 sobre SQLite · NUT (`upsd`, TCP 3493) · xUnit 2.9.2 + FluentAssertions 6.12.2 · Docker `aspnet:10.0` · GitHub Actions |
| Repo | GitHub `UTN-FRP-TUP-Aplicada-2025/SAI.Service.Core` |
| Versión | Build `0.1.0-alpha.1` (`Directory.Build.props`); última release del `CHANGELOG.md`: `[0.1.0]` 2026-07-20 (sólo documental); todo el código en `[Unreleased]`; sin tags |
| Rama y commit indexados | `main` · `6fd23d5` *Merge pull request #29 from UTN-FRP-TUP-Aplicada-2025/feature/politica-retardo-retorno* (2026-08-04) |
| Estado | Mini-Plan cumplido (Etapa 5 Realizada, CU-01…CU-12 cerrados), pero sin release cortada |

**Función principal.** Monitorea por NUT un SAI que respalda un único servidor Linux sin respaldo, guarda la historia de mediciones, eventos e intervenciones, calcula la salud de la batería como tendencia propia (el equipo no la informa) y, ante un corte sostenido, ordena el apagado del SAI **con retorno** (`shutdown.return`) para garantizar que el host vuelva a encender solo. Expone un panel para el administrador único y una API de ingesta idempotente para un GMAO externo.

**Arquitectura en una línea.** Clean Architecture en cinco assemblies: `Web` → {`Api`, `Infrastructure`} → `Application` → `Domain` (framework-free); un `BackgroundService` sondea cada 5 s, el dominio deriva eventos con funciones puras y `ServicioApagadoOrdenado` decide según la política vigente.

**Principios que explican sus reglas.**

| Principio | Consecuencia | ADR |
| --- | --- | --- |
| Estado base seguro | Arranca en `SoloAlerta`; si alguno de los 4 supuestos no está verificado y vigente, la acción queda `Bloqueada` | ADR-10 |
| Retorno garantizado | Iniciado el apagado, el corte del SAI no se cancela; nunca `shutdown.stop` | ADR-09 |
| Efecto observado | Ninguna acción se da por hecha sin ver su efecto; 3 sondeos perdidos ⇒ `DesconexionUsb` | ADR-11 |
| Disparo sin `LB` | Se decide por tiempo en batería y `battery.voltage`, nunca por `LB` ni `battery.charge` | ADR-12 |
| Historia append-only | Las tablas de hechos no admiten `UPDATE`/`DELETE` (interceptor de `SaveChanges`) | ADR-04 |
| Procedencia obligatoria | Todo valor declara su `Origen` (medido, derivado, declarado, imputado, …) | ADR-06 |
| Tiempo como dato | Vínculos con vigencia `[Desde, Hasta)`; catálogo ≠ inventario ≠ historia | ADR-05, ADR-07 |
| Un único acceso al equipo | Todo pasa por el puerto `IAdaptadorConexion` | ADR-02, ADR-27 |

Detalle y fuentes: [`00_MASTER-INDEX.md`](indexes/00_MASTER-INDEX.md).

---

## Estructura del repositorio (archivos versionados, primer nivel)

```
SAI.Service.Core/
├── AGENTS.md               ← contrato para agentes (build, tests, límites); deriva de SDD/Docs/11-Documentacion/Contrato-Agentes-v1.0.md
├── README.md               ← presentación del repo y onboarding
├── CHANGELOG.md            ← Keep a Changelog; fuente autoritativa del avance
├── SAI.Service.Core.sln    ← 8 proyectos (src + tests)
├── Directory.Build.props   ← net10.0, cero warnings, analizadores, versión 0.1.0-alpha.1
├── Dockerfile              ← multi-stage sdk:10.0 → aspnet:10.0, volúmenes /app/data y /keys
├── .dockerignore
├── .gitignore              ← incluye .publish-e1/
├── .devcontainer/          ← Dev Container .NET 10 + docker-in-docker
├── .github/workflows/      ← ci.yml: restore, format, build Release, test
├── scripts/                ← build.sh, build-all.sh, run.sh, run-all.sh, dev.sh
├── src/SAI.Service.Core/   ← 5 assemblies: Domain, Application, Infrastructure, Api, Web (205 archivos)
├── tests/                  ← Domain.Tests, Application.Tests, Integration.Tests (50 archivos)
└── SDD/                    ← 195 archivos
    ├── Intake/             ← SOLUTION-INTAKE + MANIFEST
    ├── Docs/               ← categorías 00–03 y 05–11 (sin 04-Prompts-AI) + Audit/ (fases A–H)
    └── Maquetas/           ← maqueta estática Bootstrap de la Fase B2
```

Fuente: `git ls-files` sobre `main`. `.vscode/` está versionado pero excluido del indexado.

---

## Restricciones para IA

- **No modificar el proyecto desde esta base.** La ia-db describe; los cambios al código o al SDD se hacen en el repo `SAI.Service.Core`, siguiendo su propio flujo.
- **Respetar `AGENTS.md`** (y el contrato del que deriva, `SDD/Docs/11-Documentacion/Contrato-Agentes-v1.0.md`): el framework SDD (`IA/IA.SDD`, externo) y toda carpeta `PROMPTs/` sólo lectura; no relajar append-only, procedencia ni el estado base seguro; acceso al SAI sólo por `IAdaptadorConexion`; el merge de PR y el borrado de ramas son humanos.
- **Ante contradicción, prevalece el repositorio** (el código de `main` por encima del SDD, y ambos por encima de esta base). Si encontrás una diferencia, corregí el índice afectado.
- **No inventar.** Lo que un índice marca «(no verificado)» sigue sin verificar; no completar por inferencia.
- **Actualizar de forma incremental:** sólo los índices afectados por cambios reales desde la `Base` del manifiesto, con el prompt de actualización; no regenerar todo si alcanza una actualización parcial.
- **No duplicar** contenido entre índices: referenciar por nombre de archivo.
- La base no se indexa a sí misma.

---

## Manifiesto de generación

- Generado por : `/IA/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md` (en disco: `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md`; Profile `/IA/PROMPTs/IA.Prompts/PromptFramework/Profiles/Knowledge-Indexing.md`) (invocado desde `/DEV/SAI-Service/SAI.Service.Core.Documentacion/PROMPTs/01-Indexado/Crear-Indexado.md`, vía `/Repos-Hosts/Infra/Host.Infra.Documentacion/PROMPTs/Indexado/Crear-Indexado.md`)
- Alcance      : `/DEV/SAI-Service/SAI.Service.Core` — modo proyecto, ia-db alojada en `/DEV/SAI-Service/SAI.Service.Core.Documentacion/ia-db`
- Fuentes      : `src/`, `tests/`, `SDD/` (`Intake/`, `Docs/` 00–11 + `Audit/`, `Maquetas/`), `scripts/`, `.devcontainer/`, `.github/workflows/`, archivos raíz (`README.md`, `AGENTS.md`, `CHANGELOG.md`, `SAI.Service.Core.sln`, `Directory.Build.props`, `Dockerfile`, `.dockerignore`) e historial git de `main`
- Exclusiones  : `.git`, `.vs`, `.vscode`, `.idea`, `bin`, `obj`, `dist`, `build`, `out`, `publish`, `node_modules`, `packages`, binarios e imágenes, bases de datos, `ia-db`, `.publish-e1` y lo ignorado por `.gitignore`
- Base         : `6fd23d5`
- Generado     : 2026-10-26 · Versión: 2.0 (regeneración completa; reemplaza la 1.0 del 2026-08-04, commit `7541f3e` de `SAI.Service.Core.Documentacion`)
- Actualizar   : `/DEV/SAI-Service/SAI.Service.Core.Documentacion/PROMPTs/01-Indexado/Actualizar-Indexado.md` → `/IA/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md` (en disco: `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md`)
- Detectar cambios: `git -C /DEV/SAI-Service/SAI.Service.Core log --oneline 6fd23d5..main` + entradas nuevas de `CHANGELOG.md`; qué revisar por índice: `indexes/11_Estado-Y-Pendientes.md` §9
- Rutas         : todas las rutas de los índices son relativas a la raíz del repo `SAI.Service.Core`; las de este manifiesto, a la raíz del workspace
