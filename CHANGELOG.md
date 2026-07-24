# Changelog

Todos los cambios notables de la documentación de este repositorio se registran en este archivo.

El formato se basa en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/).

## [Sin publicar] - 2026-07-24

### Cambiado

- Reorganización de la carpeta `PROMPTs/` con nomenclatura numerada por etapa:
  - `PROMPTs/Generar-SDD/` → `PROMPTs/01-Ejecutar-Prompt-Integrador-Documento-Intake/`, con el historial ampliado de `00` a `35` (fases, PRs, máquina de estados, investigación de NUT y replanteos) y los `INPUTs/` e `Imgs/` reubicados.
  - `PROMPTs/02-Ejecutar-Prompt-Orquestador/`: prompt del agente orquestador de documentación.

### Añadido

- `PROMPTs/03-Reformular-SDD/`: prompts de reformulación del SDD.
  - `01-Reformular-SDD-Ajuste-UX-UI/`: ajuste de UX/UI del panel de verificaciones, con `INPUTs/` (maqueta HTML y SPEC de UX).

### Corregido

- `PROMPTs/02-Ejecutar-Prompt-Orquestador/`: el prompt del orquestador quedaba anidado dentro de un directorio mal nombrado (`Ejecutar-Prompt-Orquestador.md/`); ahora es un archivo `Ejecutar-Prompt-Orquestador.md` directo.

## [Sin publicar] - 2026-07-21

### Añadido

- `PROMPTs/Generar-SDD/Historial/`: registro histórico del proceso de generación del SDD (Fases A–H) por el agente orquestador de documentación, incluyendo:
  - Fases E, F, G y H (00 a 06), con el cierre del SDD y el handoff a codificación.
  - Solicitud de request (07) y primer build / verificación de la maqueta en Dev Container (08–10).
  - Capturas de evidencia: `Vista-DevContainer.png` y `Vista-Maqueta.png`.
