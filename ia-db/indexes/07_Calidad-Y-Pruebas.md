# 07 — Calidad y pruebas

> **Propósito**: cómo se prueba el sistema, qué suites existen, qué pisos de cobertura son bloqueantes y
> qué se declara imposible de automatizar.
> **Fuentes primarias**: [`tests/`](../../../SAI.Service.Core/tests/) ·
> [`SDD/Docs/08-Calidad-Y-Pruebas/Estrategia-Testing-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/08-Calidad-Y-Pruebas/Estrategia-Testing-v1.0.md) ·
> [`Definition-Of-Done-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/08-Calidad-Y-Pruebas/Definition-Of-Done-v1.0.md) ·
> [`Matriz-Cobertura-Pruebas-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/08-Calidad-Y-Pruebas/Matriz-Cobertura-Pruebas-v1.0.md)

---

## 1. Suites reales

| Suite | Ruta | Hechos/teorías (al indexar) | Qué cubre |
| --- | --- | --- | --- |
| `SAI.Service.Core.Domain.Tests` | `tests/SAI.Service.Core.Domain.Tests/` | ~124 | Invariantes, vigencias e intersección, resolutor temporal, derivador de eventos, salud de batería, `Dinero`, verificaciones, secuencia física, versión de política, sustituciones, `Accion` |
| `SAI.Service.Core.Application.Tests` | `tests/SAI.Service.Core.Application.Tests/` | ~31 | Servicios de caso de uso: informe de período, ingesta, políticas, calculador de protección |
| `SAI.Service.Core.Integration.Tests` | `tests/SAI.Service.Core.Integration.Tests/` | ~115 | EF Core + SQLite físico, `WebApplicationFactory`, adaptador NUT y protocolo, acceso, alta, apagado, monitoreo, históricos, informes, ingesta, interceptor append-only, componentes del panel |

Herramientas: **xUnit** + **FluentAssertions**; `FabricaSai.cs` centraliza las fixtures de integración.

**Pruebas contra SAI real**: las marcadas con `[HechoNutVivo]` se **omiten por defecto** y solo corren con
`SAI_NUT_LIVE=1` y red al `upsd` del host (p. ej. `--network host`). En CI no hay `upsd`, así que quedan
skipped — es esperado, no un fallo.

## 2. Pirámide y cobertura

Distribución objetivo: **70 % unitarias / 25 % integración / 5 % end-to-end** (reconciliada a favor del
intake frente al 70/20/10 de referencia para `web-monolith`). Justificación: el núcleo de valor es lógica
pura, y con un solo desarrollador la suite e2e se acota a lo que no se puede cubrir de otro modo.

Pisos **bloqueantes** por capa (fuente única de la categoría 08):

| Capa | Líneas | Ramas |
| --- | --- | --- |
| `Domain` | 90 % | 85 % |
| `Application` | 80 % | 70 % |
| `Api` (frontera con contrato crítico) | 80 % | 70 % |
| `Infrastructure` | 70 % | 60 % |
| `Web` (presentación) | 60 % | 50 % |
| **Conjunto de la solución** | **80 %** | **70 %** |

La cobertura se reporta **por assembly**, nunca como un único número global: un 80 % agregado escondería
100 % en getters y 40 % en la lógica de apagado. Los pisos se pueden subir sin ADR; bajarlos exige ADR.
Mutation testing **no** se adopta en v1.

## 3. BDD sin gherkin

Los criterios de aceptación de cada CU/US se redactan en Given-When-Then y se materializan como **pruebas
de integración**, no como archivos `.feature`: con un solo desarrollador, mantener step definitions cuesta
más de lo que aporta. Cada test nombra en su título y documentación el CU o RN que cubre y su cláusula
Given-When-Then.

## 4. Aislamiento y datos

- **Adaptador simulado**: pieza central del aislamiento. Permite guionar `OL`/`OB`, caídas de tensión,
  muestras `perdida` y comandos sin efecto, y ejercitar la decisión de apagado sin hardware ni riesgo.
- **SQLite en archivo temporal efímero** por test o por clase, con migraciones aplicadas al inicio. Ningún
  test depende del orden ni comparte estado.
- **Datos sintéticos** derivados de los ocho escenarios E-1..E-8 del intake, versionados con el código.
  No hay datos de producción. Su modificación pasa por PR: no hay regeneración automática que pueda
  «arreglar» un test cambiando su expectativa.
- **Series temporales guionadas** para los casos límite: microcorte de una sola muestra, muestra parcial y
  perdida con nulos, serie que nunca enciende `LB` y aun así dispara, pérdida de muestras durante la
  prueba a 1 Hz.
- **Sin catch silenciosos**: los tests fallan rápido ante un invariante violado; cada test tiene al menos
  una aserción explícita.

## 5. La prueba imposible (declarada)

El flujo **F-3** —ciclo completo de apagado y reencendido físico, correspondiente a CU-05— **no se puede
automatizar**. El adaptador simulado cubre la lógica de decisión; el comportamiento real del firmware
(`ver-shutdown-return`) y de la BIOS (`ver-bios-autoencendido`) se verifica en la **ventana de
mantenimiento** (CU-10) y su resultado se registra como evidencia de una `Verificacion` con su vigencia,
**no como un test verde**. Hasta que esa ventana ocurra, el servicio permanece forzado en `SoloAlerta`.

## 6. Definition of Done (lo que hay que cumplir para cerrar)

Cuatro capas acumulativas (US → BT → etapa → release). Los criterios mecánicos más citados:

- Compila en Release con **cero warnings** (los warnings son errores).
- Cada criterio Given-When-Then de la US tiene una prueba verde; ningún test huérfano ni criterio sin test.
- La cobertura por capa de lo tocado no baja de su piso.
- Ningún valor de dominio se persiste sin `Origen` (I-7).
- Análisis estático sin diagnósticos de severidad *error* y sin diferencias de formato.
- Si se tocó el panel, la pantalla se validó **en el navegador** contra la línea base visual.
- Si se cerró un defecto, existe un test de regresión que lo previene.
- Si la tarea implementa un invariante I-1..I-21, su prueba existe y está verde.

## 7. Comandos de validación

```bash
./scripts/build-all.sh                                       # 0 warnings
dotnet test SAI.Service.Core.sln --configuration Release     # 0 failed
dotnet test tests/SAI.Service.Core.Domain.Tests              # subconjunto rápido

# si se tocó el modelo de datos:
dotnet ef migrations has-pending-model-changes \
  --project src/SAI.Service.Core/SAI.Service.Core.Infrastructure \
  --startup-project src/SAI.Service.Core/SAI.Service.Core.Infrastructure   # esperado: sin cambios
```

Sin `dotnet` en el host, todo lo anterior corre en el Dev Container o en un contenedor SDK efímero
`mcr.microsoft.com/dotnet/sdk:10.0` montando el repositorio (EVE-02).

## 8. Otros documentos de la categoría 08

| Documento | Contenido |
| --- | --- |
| `Estrategia-Calidad-v1.0.md` | Los 10 quality gates y su carácter bloqueante |
| `Criterios-Validacion-v1.0.md` | Cómo se valida cada NFR; marca los PENDIENTE (N-03, N-20, N-25) |
| `Matriz-Cobertura-Pruebas-v1.0.md` | Trazabilidad CU/RN ↔ test |
| `Casos-Prueba-Referenciales-v1.0.md` | Casos de prueba de referencia por CU |
| `Matriz-Sensado-Deriva-v1.0.md` | Variables sensadas, su procedencia y su deriva |
| `Guia-Testing-Extensibilidad-v1.0.md` | Cómo probar un adaptador nuevo |
| `Plan-Pruebas-v1.0.md` · `Definition-Of-Done-v1.0.md` | Plan y criterio de cierre |
