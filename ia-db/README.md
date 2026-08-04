# ia-db — SAI.Service.Core

> **Instrucción para IA**: este archivo es el **punto de entrada único** de la base de conocimiento de
> `SAI.Service.Core`. Leelo completo antes de explorar el repositorio. Después, elegí en la tabla de
> navegación el índice del tema que necesitás y cargá **solo ese** (a lo sumo dos). Recién si el índice
> resulta insuficiente, abrí las fuentes que él referencia. **Nunca recorras el repositorio completo ni
> cargues toda la documentación SDD cuando hay información indexada.**

---

## 1. Navegación — «Necesito saber… → leo este índice»

| Necesito saber… | Leo este índice |
| --- | --- |
| De qué trata el sistema, con qué stack, qué decisiones lo gobiernan | [00_MASTER-INDEX](indexes/00_MASTER-INDEX.md) |
| Cómo está construido: capas, dependencias, flujo del apagado, ADRs | [01_Arquitectura](indexes/01_Arquitectura.md) |
| Las entidades, invariantes y reglas de negocio del dominio | [02_Dominio-Y-Reglas](indexes/02_Dominio-Y-Reglas.md) |
| Tablas, tipos, índices, migraciones, append-only | [03_Modelo-De-Datos](indexes/03_Modelo-De-Datos.md) |
| Qué caso de uso resuelve cada servicio de aplicación y su puerto | [04_Aplicacion-Y-Casos-De-Uso](indexes/04_Aplicacion-Y-Casos-De-Uso.md) |
| Las superficies: páginas del panel, endpoints REST, autenticación | [05_Interfaces-Panel-Y-Api](indexes/05_Interfaces-Panel-Y-Api.md) |
| Cómo se habla con el SAI (NUT), los adaptadores y los servicios de fondo | [06_Infraestructura-Y-Adaptador-Sai](indexes/06_Infraestructura-Y-Adaptador-Sai.md) |
| Cómo se prueba: pirámide, cobertura, suites reales, prueba imposible | [07_Calidad-Y-Pruebas](indexes/07_Calidad-Y-Pruebas.md) |
| Cómo se compila, versiona, despliega y opera; incidentes conocidos | [08_Devops-Despliegue-Y-Operacion](indexes/08_Devops-Despliegue-Y-Operacion.md) |
| Dónde vive cada documento del árbol SDD y qué contiene | [09_Documentacion-Sdd](indexes/09_Documentacion-Sdd.md) |
| Qué significa un término del dominio (SAI, procedencia, ciclo forzado…) | [10_Glosario](indexes/10_Glosario.md) |
| En qué estado está el proyecto, qué falta y qué está declarado pendiente | [11_Estado-Y-Pendientes](indexes/11_Estado-Y-Pendientes.md) |

---

## 2. Resumen ejecutivo

| Campo | Valor |
| --- | --- |
| Proyecto indexado | `SAI.Service.Core` (repositorio `DEV/SAI.Service.Core`) |
| Tipo | `web-monolith` — un solo proceso desplegable |
| Stack | .NET 10 · C# · Blazor Server (interactive server) + MudBlazor · EF Core + SQLite · NUT 2.8 |
| Documentación de origen | Árbol SDD dentro del propio repo (`SDD/Docs/`, categorías 00–11) |
| Versión del código | `0.1.0` publicada + `[Unreleased]` con las etapas 1 a 5 integradas |
| Rama de trabajo al indexar | `feature/politica-retardo-retorno` (último commit `087ad71`) |
| Idioma | Español rioplatense en dominio, código y documentación |

**Qué hace.** Administra el ciclo de vida y el monitoreo de un SAI (UPS) que respalda al host Linux
`i7infra`, cubriendo lo que NUT no modela: inventario con historia trazable, salud de batería, apagado
ordenado gobernado por verificación e informes de período. Sondea el estado del SAI, deriva eventos con
reglas versionadas y, ante un corte sostenido, decide un apagado ordenado — pero **solo si los cuatro
supuestos de seguridad están verificados y vigentes**; si no, degrada a «solo aviso» y no apaga nada.

**Arquitectura en una línea.** Clean Architecture en cinco assemblies (`Domain ← Application ←
Infrastructure/Api ← Web`) dentro de un único proceso, con todo el acceso al SAI aislado detrás del puerto
`IAdaptadorConexion` (implementaciones NUT y simulada) y toda la historia escrita en modo append-only.

**Los tres principios que explican casi todas sus reglas** — si una decisión del código parece rara, casi
siempre se explica por uno de estos:

1. **Estado base seguro** — arranca en `SoloAlerta`; no apaga un servidor sin haber probado antes que
   vuelve a encenderse (ADR-10, RN-01, RN-02).
2. **Procedencia obligatoria** — todo valor almacenado declara su origen (medido, derivado,
   estimadoPorDriver, declarado, imputado, noCalculable); nada se presenta como medido si no lo es (ADR-06).
3. **Validación por efecto observado** — una orden se da por ejecutada solo si el equipo la admitió, nunca
   por ausencia de excepción (ADR-11, RN-03).

---

## 3. Estructura del proyecto indexado

```text
DEV/SAI.Service.Core/
├── AGENTS.md                  # contrato de contexto para agentes (derivado de SDD 11)
├── CHANGELOG.md               # bitácora funcional detallada por etapa (fuente de estado real)
├── SAI.Service.Core.sln       # 5 proyectos src + 3 de tests
├── Directory.Build.props      # net10.0, Nullable, cero-warnings, InvariantGlobalization
├── scripts/                   # build-all.sh, build.sh, run.sh, run-all.sh, dev.sh
├── .devcontainer/             # el SDK de .NET 10 vive acá; el host solo necesita Docker
├── .github/workflows/ci.yml   # CI real: stages 1–4 (restore, formato, build, tests)
├── SDD/                       # documentación SDD: Docs/ (00–11), Intake/, Maquetas/, Audit/
├── src/SAI.Service.Core/      # las cinco capas (ver 01_Arquitectura)
└── tests/                     # Domain.Tests · Application.Tests · Integration.Tests
```

---

## 4. Restricciones para un agente que consuma esta base

- **No modificar el proyecto indexado desde esta base.** Esta ia-db describe; no autoriza cambios.
- **No tocar `SDD/` ni ninguna carpeta `PROMPTs/`**: son de solo lectura por contrato (`AGENTS.md`).
- **No relajar** la disciplina append-only, la procedencia obligatoria ni el estado base seguro: son
  invariantes del producto, no detalles de implementación.
- **Todo acceso al SAI pasa por `IAdaptadorConexion`**; nunca hablar NUT directamente desde dominio,
  aplicación o UI.
- **No dar por vigente lo que esta base afirma sin contrastarlo** cuando la tarea depende de un detalle
  fino: cada índice referencia su fuente exacta; ante contradicción, **prevalece el repositorio** y este
  índice debe corregirse.
- **No inventar**: si un dato no está acá ni en la fuente referenciada, decirlo en lugar de inferirlo.
- Antes de cerrar una tarea que agregue documentos, cambie la estructura o modifique el modelo,
  **actualizar los índices afectados** de forma incremental (no regenerar la base entera).

---

## 5. Manifiesto de generación

- Generado por : `/IA/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md`
  (invocado desde `/DEV/SAI.Service.Core.Documentacion/PROMPTs/01-Crear-Indexado/Crear-Indexado.md`)
- Alcance      : `DEV/SAI.Service.Core` — modo proyecto (un único proyecto), ia-db alojada en el
  repositorio de documentación `DEV/SAI.Service.Core.Documentacion`
- Fuentes      : `README.md`, `AGENTS.md`, `CHANGELOG.md`, `Directory.Build.props`,
  `SAI.Service.Core.sln`, `scripts/`, `.devcontainer/`, `.github/workflows/`,
  `SDD/Docs/` (00–11, `Audit/`), `SDD/Intake/`, `SDD/Maquetas/`,
  `src/SAI.Service.Core/` (5 capas), `tests/` (3 suites)
- Exclusiones  : `.git`, `.vs`, `.vscode`, `bin`, `obj`, `.publish-e1`, binarios y lo ignorado por
  `.gitignore`
- Generado     : 2026-08-04 · Versión: 1.0
- Actualizar   : `/IA/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md`
