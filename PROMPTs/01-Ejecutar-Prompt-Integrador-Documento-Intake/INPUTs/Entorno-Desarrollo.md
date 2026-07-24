## Entorno de desarrollo (constraint)

- El desarrollo se realiza DENTRO de un Dev Container (spec containers.dev),
  sin SDK de .NET en el host. Único requisito del host: Docker.
- El SDK vive dentro del contenedor → dentro del contenedor el SDK es "local".
  Por lo tanto los scripts de build/run son AGNÓSTICOS al entorno y asumen
  `dotnet` en el PATH. NO generar un set de scripts duplicado por entorno:
  no existe una "versión dev-container" distinta de build/run.
- La orquestación del contenedor se define de forma declarativa en
  `.devcontainer/devcontainer.json` (+ Dockerfile/compose si aplica) y se
  levanta con VS Code ("Reopen in Container") o el CLI `devcontainer up`.
  No usar scripts que hagan `docker run` a mano.
- La depuración se resuelve con `.vscode/launch.json` (coreclr) + F5, NO con
  los scripts de build/run. Los scripts solo compilan/ejecutan.
- La imagen de dev (devcontainer: SDK + debugger) es distinta de la imagen de
  producción (runtime-only, construida por otro pipeline). El devcontainer NO
  define la imagen de prod.

### Artefactos a generar
- `.devcontainer/devcontainer.json`
- `.vscode/launch.json` y `.vscode/tasks.json`
- Un único set `build.sh` / `run.sh` parametrizado por proyecto
  (ej. `build.sh <proyecto>`), más `build-all.sh` / `run-all.sh` que iteran.
- (Opcional) `dev.sh` de entrada headless:
  `devcontainer up && devcontainer exec ./scripts/build-all.sh`.