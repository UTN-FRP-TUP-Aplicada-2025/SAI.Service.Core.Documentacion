# Changelog

Todos los cambios notables de la documentación de este repositorio se registran en este archivo.

El formato se basa en [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/).

## [Sin publicar] - 2026-08-04

### Añadido

- `ia-db/`: base de conocimiento indexada de `SAI.Service.Core`, pensada para que un agente cargue solo el índice que necesita en vez de recorrer el repositorio completo.
  - `ia-db/README.md`: punto de entrada único, con la tabla de navegación «necesito saber… → leo este índice» y el resumen ejecutivo del sistema.
  - `ia-db/indexes/`: doce índices temáticos, de `00_MASTER-INDEX` a `11_Estado-Y-Pendientes`, que cubren arquitectura, dominio y reglas, modelo de datos, aplicación y casos de uso, interfaces del panel y la API, infraestructura y adaptador del SAI, calidad y pruebas, devops y operación, documentación SDD, glosario y estado del proyecto.
  - Indexado hecho sobre la rama `feature/politica-retardo-retorno` del repositorio de código (último commit `087ad71`).
- `PROMPTs/01-Crear-Indexado/`: los dos prompts que gobiernan esa base — `Crear-Indexado.md` la genera desde cero y `Actualizar-Indexado.md` la mantiene al día cuando el código o el SDD cambian.

### Cambiado

- Renumeración de `PROMPTs/` para que el indexado ocupe la posición `01`, que es la que se ejecuta primero:
  - `01-Ejecutar-Prompt-Integrador-Documento-Intake/` → `02-Ejecutar-Prompt-Integrador-Documento-Intake/`
  - `02-Ejecutar-Prompt-Orquestador/` → `03-Ejecutar-Prompt-Orquestador/`
  - `03-Reformular-SDD/` → `04-Reformular-SDD/`
- Las rutas de invocación y las referencias a `INPUTs/` dentro de los prompts movidos se actualizaron al número nuevo, para que las instrucciones sigan resolviendo.

## [Sin publicar] - 2026-07-29

### Añadido

- `Guides/Installer-Guide.md`: guía para quien instala y administra SAI.Service.Core en un host con un SAI/UPS real. Documento vivo, con:
  - Panorama de la comunicación con el SAI: el servicio es un cliente NUT por TCP (`127.0.0.1:3493`) contra `upsd`, y no pasa por `upsmon`.
  - Las tres piezas de NUT (`upsd`, driver, `upsmon`) y quién toca el USB.
  - Separación de responsabilidades: comandar el SAI vs. lograr que el host se apague limpio ante un corte.
  - Credenciales NUT (`upsd.users`), configuración del servicio, `upsmon` en detalle y el apagado ordenado con retorno.
  - Diagnóstico y comandos útiles, y un registro de problemas resueltos en formato síntoma → causa → solución.
- `PROMPTs/03-Reformular-SDD/02-Reajuste-SDD-Refactorizacion/`: prompt para reajustar la solución de `SAI.Service.Core` tras los cambios del Framework SDD (renumeración de la categoría 10 a 11 y nuevas reglas de generación de documentación final).
- `PROMPTs/01-Ejecutar-Prompt-Integrador-Documento-Intake/Historial/`: entradas `36` a `39` del historial, con el estado del PR de UX del panel de verificaciones, el esquema de testing contra el SAI real, la corrección de la cadena de comunicación con NUT y el análisis del apagado controlado (corte con retorno vs. FSD de `upsmon`).

### Corregido

- `Historial/04-Fase-G.md`: referencia al documento de reglas de ejemplos actualizada a `Rules-Examples` (sin el prefijo numérico obsoleto).

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
