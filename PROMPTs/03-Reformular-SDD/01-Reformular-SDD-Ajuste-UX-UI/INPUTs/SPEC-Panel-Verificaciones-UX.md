# SPEC — Rediseño UX/UI del Panel de verificaciones

**Proyecto:** Sai-Service-Core
**Módulo:** Panel de verificaciones
**Versión del spec:** 1.0
**Estado:** propuesta para evaluación
**Stack destino:** .NET 10 · Blazor · Razor components

---

## 0. Instrucciones para el agente

Este documento es una **propuesta**, no una orden. Antes de aplicar:

1. Leé el código actual del módulo y contrastá cada hallazgo (sección 2) contra la implementación real. Si un hallazgo ya está resuelto o no aplica, marcalo como `NO APLICA` y explicá por qué.
2. Evaluá cada propuesta (sección 3) contra el costo real de implementación en el código existente. Cada propuesta tiene una prioridad sugerida; podés discutirla, pero justificá.
3. **No implementes la fase 3 (asistente guiado) sin confirmación explícita.** Es la única que cambia el modelo de interacción, no solo la presentación.
4. Respetá el modelo de dominio de la sección 4: la lógica de estado va en el dominio/servicio, no en el markup Razor.
5. Si algo del spec entra en conflicto con una decisión de arquitectura ya tomada en la solución, prevalece la solución existente — reportá el conflicto en vez de romperla.
6. Al terminar, entregá: lista de cambios aplicados, lista de propuestas descartadas con motivo, y las preguntas abiertas (sección 9) que no pudiste resolver solo.

---

## 1. Contexto y objetivo

### 1.1 Qué hace la pantalla

El Panel de verificaciones valida los **cuatro supuestos** que el servicio necesita dar por ciertos antes de habilitar el apagado automático del host ante un corte de energía. Mientras los cuatro no estén verificados, el servicio permanece en modo **"solo aviso"**.

Los cuatro supuestos, en el orden en que ocurren físicamente:

| # | Supuesto | Quién actúa | Quién opera |
|---|---|---|---|
| 1 | Señal en batería (OB) | el SAI avisa | el operador corta la red |
| 2 | Tiempo de apagado ordenado del host | el host se apaga | el operador cronometra |
| 3 | Corte con retorno | el SAI corta y repone su salida | el operador ordena y observa |
| 4 | Encendido automático del host por presencia de energía | el host / BIOS arranca solo | el operador observa |

Los cuatro forman **un solo ejercicio físico encadenado, con presencia en sitio**, aunque cada uno se puede re-verificar por separado cuando vence.

### 1.2 Objetivo del rediseño

> Que un usuario **fuera de contexto** —un técnico municipal que abre la pantalla por primera vez, o el mismo desarrollador seis meses después— entienda en menos de 30 segundos: **qué se está probando, en qué orden, qué implica el estado actual del servicio, y cuál es la próxima acción.**

Objetivo secundario: reducir la carga visual de la pantalla sin perder información auditable.

### 1.3 Fuera de alcance

- Cambios en la lógica de verificación, en los protocolos de comunicación con el SAI, o en el cálculo de la ventana de apagado.
- Cambios en el resto de los módulos del menú lateral (salvo tokens de color compartidos, ver 5.1).
- Persistencia, esquema de base de datos, o el formato del informe de período.
- Internacionalización.

---

## 2. Diagnóstico del estado actual

Cada hallazgo tiene id (`H-n`), severidad, y el criterio con el que se considera resuelto.

### H-1 · El orden de las tarjetas contradice la secuencia física — **alta**

**Síntoma.** El banner explicativo describe una cadena causal (cortás la red → el SAI avisa OB → el host se apaga → el SAI corta salida → vuelve la energía → el host arranca). La grilla 2×2 presenta las tarjetas en un orden distinto: apagado del host, señal en batería, encendido automático, corte con retorno.

**Impacto.** El usuario lee la secuencia, mira las tarjetas, y no puede mapear una cosa con la otra. Es el obstáculo más grande para el objetivo de 1.2.

**Resuelto cuando.** Las tarjetas aparecen numeradas 1–4 en el orden físico real, y ese número es visible sin interacción.

### H-2 · El estado del servicio está enterrado en un banner de alerta — **alta**

**Síntoma.** El dato más importante de la pantalla ("2 de 4 verificados, el servicio está en solo aviso") está en un banner ámbar, inmediatamente seguido de un segundo banner ámbar con el texto explicativo del ejercicio. Dos banners consecutivos del mismo color producen ceguera a banners: el usuario saltea los dos.

Además, **"solo aviso" es jerga interna que no comunica la consecuencia.** El usuario fuera de contexto no sabe si eso es bueno, malo, o qué le falta hacer.

**Resuelto cuando.** El estado del servicio es un bloque de encabezado (no una alerta), con: chip de estado, progreso 2/4, y una frase en lenguaje llano que diga la consecuencia operativa —"el sistema todavía no apaga el host automáticamente"— y qué falta para cambiarla.

### H-3 · No hay divulgación progresiva — **alta**

**Síntoma.** Las cuatro tarjetas muestran el procedimiento completo permanentemente, incluidas las dos ya verificadas. Las verificadas ocupan aproximadamente la mitad del alto útil de la pantalla con instrucciones que nadie va a volver a leer en ese contexto.

**Impacto.** Empuja las acciones pendientes —lo único accionable— fuera del pliegue.

**Resuelto cuando.** Una verificación vigente se muestra colapsada en una fila compacta (título, evidencia resumida, vencimiento relativo, acción de re-verificar) y el procedimiento completo es expandible.

### H-4 · La jerarquía visual de las acciones está invertida respecto del riesgo — **alta**

**Síntoma.**
- "Confirmar señal (lee el equipo)" — acción de **solo lectura, riesgo cero** — se presenta como botón verde sólido, el elemento de mayor peso visual del panel.
- "Ejecutar corte con retorno" — acción **destructiva: corta la corriente real del host** — se presenta en marrón/ámbar, indistinguible del banner informativo que está arriba.

**Impacto.** Riesgo operativo real, no solo estético.

**Resuelto cuando.** La acción destructiva es la única en rojo/danger del panel, requiere confirmación explícita en modal, y la acción de lectura pasa a estilo secundario.

### H-5 · Tres familias cromáticas sin relación semántica — **media**

**Síntoma.** Sidebar verde oscuro, acentos teal, banners marrón/ámbar, botón primario verde. El color no codifica nada consistente: verde aparece tanto en "verificado" (estado) como en "confirmar señal" (acción).

**Resuelto cuando.** Verde / ámbar / rojo quedan **reservados exclusivamente para estado y riesgo**. Las acciones usan neutro + un único acento. Ver 5.1.

### H-6 · Fechas ISO crudas y falta de estado "por vencer" — **media**

**Síntoma.** `vigente hasta 2027-01-18` obliga al usuario a calcular mentalmente cuánto falta. Y el modelo de estado solo contempla dos valores (verificado / nunca verificado), cuando el estado que efectivamente genera trabajo es "está por vencer".

**Resuelto cuando.** Fecha localizada + tiempo relativo, y el modelo de estado tiene cuatro valores (ver 4.1) con umbral configurable.

### H-7 · La evidencia es absoluta en vez de comparada — **media**

**Síntoma.** `apagado cronometrado en 1 s bajo carga con contenedores detenidos`. Un valor solo no dice si el resultado es bueno. El significado de la prueba es *"el tiempo medido tiene que ser holgadamente menor a la ventana reservada"* — pero la ventana no se muestra.

**Resuelto cuando.** La evidencia se muestra como comparación: `1 s medidos vs. 120 s reservados`, con indicación visual de si el margen es suficiente.

### H-8 · Estructura de contenido correcta pero mal renderizada — **media**

**Síntoma.** El patrón `Actúa: … · vos … / Qué se prueba: … / Esperás ver: … / Evidencia: …` es excelente y hay que conservarlo. Pero se renderiza como párrafos con etiqueta en negrita corrida, lo que lo vuelve un muro de texto y desperdicia la estructura.

Además: los títulos tienen largos muy dispares (`Encendido por presencia de energía en la alimentación del host` envuelve a dos líneas y rompe la alineación de la grilla).

**Resuelto cuando.** `Actúa` se representa como chips con ícono; los demás campos conservan la etiqueta pero con tratamiento tipográfico diferenciado; los títulos se acortan y el texto largo pasa a subtítulo.

### H-9 · Las verificaciones vigentes son callejones sin salida — **baja**

**Síntoma.** Una tarjeta verificada no ofrece ninguna acción: ni re-verificar, ni ver historial, ni saltar al registro de intervenciones donde está el detalle.

**Resuelto cuando.** Toda tarjeta, en cualquier estado, tiene al menos una acción disponible.

### H-10 · Terminología interna sin definir — **baja**

**Síntoma.** "Supuestos" es vocabulario de diseño del sistema, no del operador. También falta indicar cuáles pruebas requieren presencia física y cuáles se pueden disparar remotamente.

**Resuelto cuando.** Se usa un término neutro ("condiciones" o "pruebas") y cada tarjeta declara si requiere presencia y si corta corriente real.

---

## 3. Propuestas

### P-1 · Encabezado de estado del servicio (resuelve H-2)

Reemplaza el primer banner ámbar por un bloque de encabezado con:

- **Chip de estado del servicio**: `Solo aviso` (ámbar) | `Apagado automático activo` (verde).
- **Contador**: `2 de 4 pruebas verificadas`.
- **Barra de progreso** de 4 segmentos, uno por prueba, coloreados por estado.
- **Frase de consecuencia en lenguaje llano**, en 16px, no en tono de alerta:
  > El sistema todavía **no apaga el host automáticamente**. Faltan 2 pruebas para habilitar el apagado ante corte de energía.
- **Acción primaria única** de la pantalla: `Iniciar ejercicio guiado` (ver P-7; si la fase 3 no se implementa, esta acción no existe y el encabezado no lleva botón).

**Criterio de aceptación.** Ningún otro elemento de la pantalla tiene más peso visual que este bloque. El bloque no usa estilo de banner de alerta.

### P-2 · Reordenar y numerar las pruebas por secuencia física (resuelve H-1)

Layout de **una columna vertical**, no grilla 2×2. Orden fijo:

1. Señal en batería
2. Apagado ordenado del host
3. Corte con retorno
4. Arranque automático del host

Cada tarjeta lleva un indicador numérico circular a la izquierda del título, que además codifica el estado por color y por ícono (check para verificada, número para pendiente).

**Criterio de aceptación.** El orden es fijo y no depende del estado. Una prueba vencida no se reordena al principio de la lista: rompería la correspondencia con la secuencia física, que es justamente lo que se quiere enseñar. La atención se dirige por color, no por posición.

### P-3 · Tira de secuencia (resuelve H-1, complementa P-2)

Sustituye el segundo banner ámbar por una tira horizontal de nodos encadenados, en texto plano y con flechas:

`Cortás la red → El SAI avisa OB → El host se apaga → El SAI corta salida → Vuelve la luz, arranca solo`

Precedida de una línea: *"Es un solo ejercicio físico, con presencia. Cada paso registra un tramo."*

**Criterio de aceptación.** No usa estilo de alerta. Cada nodo de la tira tiene una correspondencia visual identificable con su tarjeta (mismo número o mismo vocabulario).

### P-4 · Divulgación progresiva por estado (resuelve H-3, H-9)

**Estado `Vigente`** → fila colapsada, una línea:
`✓  2 · Apagado ordenado del host  ·  1 s medidos vs. 120 s reservados  ·  vigente hasta el 18 ene 2027 (faltan 6 meses)` + acción `Re-verificar` + enlace `Ver procedimiento` (expande).

**Estado `PorVencer`** → misma fila colapsada, con chip ámbar `Vence en N días` y la acción `Re-verificar` promovida.

**Estado `NuncaVerificada` o `Vencida`** → tarjeta expandida completa: chips de actor, `Qué se prueba`, pasos numerados, `Esperás ver`, aviso de riesgo si corresponde, acción.

**Criterio de aceptación.** Con dos pruebas vigentes y dos pendientes, ambas acciones pendientes son visibles sin scroll en una pantalla de 900px de alto. El procedimiento completo de una prueba vigente sigue siendo accesible en un clic, sin recargar la página.

### P-5 · Jerarquía de acciones por riesgo (resuelve H-4)

| Acción | Riesgo | Tratamiento |
|---|---|---|
| Confirmar señal (lee el equipo) | ninguno (solo lectura) | botón secundario, outline, ícono de ojo |
| Re-verificar | ninguno hasta que se ejecuta | botón terciario / ghost |
| Iniciar ejercicio guiado | medio | única acción primaria de la pantalla |
| Ejecutar corte con retorno | **alto — corta corriente real** | botón danger + modal de confirmación |

El modal de confirmación del corte con retorno debe:
- Enunciar la consecuencia en primera línea, sin eufemismos: *"Esto corta la corriente que alimenta al host."*
- Listar las precondiciones que el operador debe haber verificado (arranque automático en BIOS, presencia en sitio).
- Requerir una acción deliberada, no un simple "Aceptar" — por ejemplo, tipear el nombre del equipo.
- Botón de confirmación en rojo, botón de cancelar como acción por defecto del foco.

**Criterio de aceptación.** El rojo/danger aparece exactamente en un lugar del panel. Ninguna acción destructiva se ejecuta con un solo clic.

### P-6 · Chips de contexto por tarjeta (resuelve H-8, H-10)

Reemplaza la línea `Actúa: el SAI (avisa) · vos cortás la red` por chips con ícono:

- `Actúa el SAI` / `Actúa el host` (quién ejecuta)
- `Vos cortás la red` / `Vos cronometrás` / `Vos observás` (qué hace el operador)
- `Presencial` — cuando requiere presencia física
- `Corta corriente real` — chip danger, solo en corte con retorno

**Criterio de aceptación.** Los chips son legibles sin leer el cuerpo de la tarjeta. Un operador puede filtrar visualmente qué puede hacer desde el escritorio y qué requiere ir al sitio.

### P-7 · Asistente de ejercicio completo — **fase 3, requiere confirmación** (resuelve H-1 de raíz)

Dado que las cuatro pruebas son un solo ejercicio físico, el modo primario de verificación debería ser un asistente que recorra los cuatro pasos en una sesión, registrando cada tramo a medida que ocurre. La re-verificación individual queda como caso secundario, para cuando vence una prueba suelta.

Requisitos mínimos si se implementa:
- Sesión de ejercicio persistida, no estado en memoria del circuito Blazor.
- **Recuperación ante interrupción.** Es el requisito crítico: el escenario de fallo esperable es justamente que el host no vuelva a arrancar, momento en el cual el operador se queda sin UI. La sesión debe poder retomarse desde otro dispositivo o después de un reinicio, con el progreso parcial intacto.
- Cada paso confirmado se persiste inmediatamente, no al final.
- Salida del asistente en cualquier punto sin perder lo ya verificado.

**No implementar sin confirmación explícita.** Si no se implementa, P-1 a P-6 siguen siendo válidos y entregan la mayor parte de la mejora.

---

## 4. Modelo de dominio y contratos

### 4.1 Estado de verificación

La lógica de estado va en el dominio o en el servicio, **no en el markup Razor**. Un único cálculo alimenta el badge, el color, la variante de la tarjeta, el orden de las acciones y el contador del encabezado.

```
enum EstadoVerificacion
{
    NuncaVerificada,
    Vigente,
    PorVencer,
    Vencida
}
```

Derivación a partir de `FechaVencimiento` (nullable) y un umbral configurable `DiasPreavisoVencimiento` (valor sugerido: 30):

- `FechaVencimiento` nula → `NuncaVerificada`
- `FechaVencimiento` en el pasado → `Vencida`
- faltan ≤ `DiasPreavisoVencimiento` días → `PorVencer`
- en otro caso → `Vigente`

El umbral es configuración, no constante en código.

### 4.2 Estado del servicio

```
enum ModoServicio
{
    SoloAviso,
    ApagadoAutomatico
}
```

Regla: `ApagadoAutomatico` requiere que las cuatro verificaciones estén en `Vigente` o `PorVencer`. Cualquier verificación en `NuncaVerificada` o `Vencida` fuerza `SoloAviso`.

**Decisión a validar (ver 9.1):** si una verificación vence, ¿el servicio degrada automáticamente a `SoloAviso`? El spec asume que sí, y que la pantalla debe comunicarlo con claridad cuando ocurra.

### 4.3 View model de tarjeta

Campos que la vista necesita, para que el componente Razor sea presentación pura:

- `Numero` (1–4, orden físico fijo)
- `Titulo` (corto) y `Subtitulo` (descripción técnica)
- `Estado` (`EstadoVerificacion`)
- `Actor` (quién ejecuta: SAI / host)
- `AccionOperador` (qué hace la persona)
- `RequierePresencia` (bool)
- `CortaCorrienteReal` (bool)
- `QueSePrueba`, `EsperasVer` (texto)
- `Pasos` (colección ordenada)
- `Evidencia`: valor medido, valor de referencia, unidad — no una cadena preformateada (ver H-7)
- `FechaVencimiento`, `DiasRestantes`

### 4.4 Descomposición de componentes sugerida

- `EncabezadoEstadoServicio` — P-1
- `TiraSecuencia` — P-3
- `TarjetaVerificacion` — despacha a `TarjetaColapsada` / `TarjetaExpandida` según `Estado`
- `ChipEstado` — mapea `EstadoVerificacion` a etiqueta + token de color, en un solo lugar
- `DialogoConfirmacionRiesgo` — reutilizable, parametrizado; el corte con retorno es su primer consumidor

---

## 5. Sistema visual

### 5.1 Uso del color

Regla única: **el color codifica estado y riesgo, nunca acción ni decoración.**

| Rol | Uso permitido | Uso prohibido |
|---|---|---|
| Verde / success | verificación vigente, servicio en apagado automático | botones de acción |
| Ámbar / warning | por vencer, servicio en solo aviso, prueba pendiente | texto informativo genérico |
| Rojo / danger | corte con retorno, verificación vencida, avisos de riesgo real | cualquier otra cosa |
| Neutro | toda la cronología, chips de contexto, acciones secundarias | — |
| Acento (uno solo) | acción primaria de la pantalla, enlaces | estados |

Si la paleta actual del sidebar (verde oscuro) entra en conflicto con el verde de estado, **gana el estado**: el sidebar debe moverse a un neutro oscuro o a un tono claramente distinto del verde de "verificado".

### 5.2 Tipografía y densidad

- Título de tarjeta: 16px, peso medio. Subtítulo: 13px, secundario.
- Cuerpo: 15–16px, interlineado 1.6. El contenido es procedimental y se lee en sitio, a veces en condiciones malas: no bajar de 15px.
- Etiquetas de campo (`Qué se prueba`, `Esperás ver`) en color secundario, en línea con el valor, no en negrita corrida.
- Tarjetas colapsadas: alto máximo de una fila de ~56px.

### 5.3 Formato de fechas y tiempos (resuelve H-6)

- Fecha absoluta localizada + relativa entre paréntesis: `vigente hasta el 18 ene 2027 (faltan 6 meses)`.
- Nunca ISO crudo en la UI. El ISO puede quedar en el `title`/tooltip para copiar.
- Relativa granular cuando falta poco: días si faltan menos de 60, meses si más.

---

## 6. Microcopy

| Ubicación | Antes | Después |
|---|---|---|
| Encabezado | 2 de 4 supuestos verificados. El servicio permanece en solo aviso hasta verificar los cuatro. | Solo aviso · 2 de 4 pruebas verificadas — El sistema todavía no apaga el host automáticamente. Faltan 2 pruebas para habilitar el apagado ante corte de energía. |
| Título prueba 2 | Prueba de tiempo de apagado del host | Apagado ordenado del host |
| Título prueba 4 | Encendido por presencia de energía en la alimentación del host | Arranque automático del host<br>*(subtítulo: por presencia de energía en su fuente — se configura en la BIOS)* |
| Badge | Nunca verificado | Pendiente |
| Acción prueba 1 | CONFIRMAR SEÑAL (LEE EL EQUIPO) | Confirmar señal (solo lectura) |
| Evidencia prueba 2 | apagado cronometrado en 1 s bajo carga con contenedores detenidos | 1 s medidos vs. 120 s reservados · bajo carga, contenedores detenidos |
| Término general | supuestos | pruebas / condiciones |

Convenciones: sentence case en botones y badges (no mayúsculas sostenidas), voseo consistente con el resto de la aplicación, sin signos de exclamación en copy de sistema.

---

## 7. Accesibilidad

- El estado **nunca** se comunica solo por color: siempre acompañado de texto (`Pendiente`, `Vigente`) y de ícono (check / número / alerta).
- Contraste mínimo AA (4.5:1) para texto sobre fondos teñidos de estado. Verificar especialmente ámbar sobre blanco, que es donde suele fallar.
- La tarjeta colapsada/expandida usa un control expandible real con `aria-expanded`, no un div con `onclick`.
- El modal de confirmación del corte atrapa el foco, el foco inicial va a Cancelar, y `Esc` cierra sin ejecutar.
- Orden de tabulación que siga el orden visual: encabezado → prueba 1 → 2 → 3 → 4.

---

## 8. Fases de implementación

### Fase 1 — Alta relación valor/costo, sin cambios de dominio

`P-1`, `P-2`, `P-3`, formato de fechas de `5.3`, microcopy de la sección 6.
Solo layout y texto. Entrega la mayor parte de la mejora de comprensión.

### Fase 2 — Requiere el modelo de estado de 4.1

`P-4` (divulgación progresiva), `P-5` (jerarquía de riesgo + modal), `P-6` (chips), evidencia comparada de `H-7`, paleta de `5.1`.

### Fase 3 — Cambia el modelo de interacción, requiere confirmación

`P-7` (asistente de ejercicio guiado con sesión persistida y recuperación).

---

## 9. Preguntas abiertas

Resolver antes o durante la implementación. Si el agente no puede resolverlas por su cuenta, debe reportarlas.

1. **Degradación automática.** Cuando una verificación vence, ¿el servicio pasa solo a `SoloAviso`, o queda en apagado automático hasta que alguien intervenga? Cambia la semántica del encabezado y probablemente requiera una notificación.
2. **Umbral de preaviso.** ¿30 días es razonable para el ciclo de mantenimiento real, o los períodos de vigencia (que en los datos observados son de 6 y 12 meses) sugieren otro valor? ¿Es por prueba o global?
3. **Ventana reservada.** ¿De dónde sale el valor de referencia para la comparación de evidencia de `H-7` — de configuración de políticas, o se calcula? Sin ese dato, P-4 muestra el valor medido solo.
4. **Multi-equipo.** ¿El panel es por equipo SAI o global? Si el módulo "Alta de equipos" admite varios, el encabezado de estado necesita un selector y el contador 2/4 se vuelve ambiguo.
5. **Permisos.** ¿Cualquier usuario autenticado puede ejecutar el corte con retorno, o requiere un rol? El tratamiento de P-5 asume que quien ve el botón puede ejecutarlo.

---

## 10. Riesgos y trade-offs asumidos

| Decisión | Costo aceptado | Mitigación |
|---|---|---|
| Colapsar verificaciones vigentes | Un auditor pierde un clic para leer el procedimiento completo | Enlace "Ver procedimiento" en la fila colapsada; el detalle completo vive en el informe de período |
| Orden fijo por secuencia física | Una prueba vencida no salta al principio de la lista | La atención se dirige por color y por el contador del encabezado, no por posición |
| Asistente guiado (P-7) | Costo de implementación alto, sobre todo la recuperación ante interrupción | Aislado en fase 3; las fases 1 y 2 son independientes y no lo presuponen |
| Reservar verde exclusivamente para estado | Puede forzar retoque de la paleta del sidebar, que afecta otros módulos | Cambiar el sidebar a neutro oscuro es un cambio de un token |
| Una sola columna en vez de grilla 2×2 | Más scroll en pantallas anchas | Las tarjetas colapsadas compensan de sobra el alto ganado |

---

## 11. Checklist de verificación final

- [ ] Un usuario que nunca vio la pantalla identifica en menos de 30 s: qué se prueba, en qué orden, y qué implica el estado actual
- [ ] Las cuatro pruebas están numeradas en el orden físico real
- [ ] La consecuencia de "solo aviso" está escrita en lenguaje llano, sin jerga
- [ ] Existe exactamente una acción primaria en la pantalla
- [ ] El rojo/danger aparece exactamente en un lugar
- [ ] Ninguna acción destructiva se ejecuta con un solo clic
- [ ] No hay fechas en formato ISO visibles en la UI
- [ ] Ningún estado se comunica únicamente por color
- [ ] Con 2 pendientes y 2 vigentes, ambas acciones pendientes son visibles sin scroll a 900px de alto
- [ ] El cálculo de `EstadoVerificacion` existe en un solo lugar del código
- [ ] Toda tarjeta, en cualquier estado, ofrece al menos una acción

---

## Anexo A — Maqueta HTML de referencia

Archivo autónomo: `/DEV/SAI.Service.Core.Documentacion/PROMPTs/Ejercucion-SDD/Ejecucion-SDD-Ajuste-UX-UI/INPUTs/Maqueta-Panel-Verificaciones.html` (se abre en el navegador sin dependencias, sin CDN, con modo claro y oscuro).

### A.1 Qué es normativo y qué no

| Es normativo | No es normativo |
|---|---|
| La estructura y el orden de los bloques | Los valores hexadecimales concretos de los tokens |
| La jerarquia visual de las acciones (P-5) | La familia tipográfica |
| El estado colapsado vs. expandido (P-4) | El markup exacto: en Blazor esto se descompone en los componentes de 4.4 |
| Los chips de contexto y el chip de riesgo (P-6) | Los íconos elegidos (son de relleno, usar el set del proyecto) |
| El copy de la sección 6 | El ancho de 860px del contenedor |

La maqueta es HTML plano y estático a propósito: no tiene lógica, no tiene datos, y no debe copiarse tal cual a un componente Razor. Sirve para acordar el resultado visual antes de tocar el código.

### A.2 Mapeo de tokens

La maqueta define sus tokens en `:root`. Al integrarla, mapear contra el sistema de estilos existente del proyecto — si no existe, adoptar estos nombres:

| Token | Rol | Dónde se usa |
|---|---|---|
| `--surface-0` | fondo de página | `body` |
| `--surface-1` | superficie sutil | tira de secuencia, chips neutros, hover |
| `--surface-2` | superficie de tarjeta | todas las tarjetas |
| `--border` / `--border-strong` | hairline / borde de control | tarjetas, botones |
| `--text-primary` / `--text-secondary` | texto / texto de apoyo | cuerpo, subtítulos, etiquetas de campo |
| `--bg-success` / `--text-success` / `--fill-success` | estado vigente | número con check, segmentos de progreso |
| `--bg-warning` / `--text-warning` | pendiente, por vencer, solo aviso | chip de estado del servicio, número pendiente |
| `--bg-danger` / `--text-danger` / `--border-danger` | riesgo real y vencido | tarjeta de corte con retorno, aviso, botón peligro |
| `--accent` | única acción primaria | botón de ejercicio guiado |

Regla de 5.1 aplicada en la maqueta: **ningún botón usa verde ni ámbar.** El verde y el ámbar aparecen solo en indicadores de estado; el rojo aparece exactamente una vez.

### A.3 Correspondencia con las propuestas

Los comentarios HTML de la maqueta marcan qué propuesta implementa cada bloque:

- `P-1` → `<section class="card">` inicial, encabezado de estado con progreso de 4 segmentos
- `P-3` → `<section>` de la tira de secuencia (reemplaza el segundo banner ámbar)
- `P-2` → orden de los `<article>`: 1 señal en batería, 2 apagado, 3 corte con retorno, 4 arranque
- `P-4` → `.card` (expandida, pasos 1 y 3) vs. `.card--fila` (colapsada, pasos 2 y 4)
- `P-5` → clases `.primaria`, `.peligro`, `.terciaria` y el botón por defecto
- `P-6` → `.chips` con `.chip--neutral` y el `.chip--danger` de corte de corriente

### A.4 Diferencias deliberadas respecto de la pantalla actual

1. Una sola columna en vez de grilla 2×2.
2. Numeración visible 1–4 en orden físico.
3. Los dos banners ámbar desaparecen: uno se convierte en encabezado de estado, el otro en tira de secuencia neutra.
4. Los pasos 2 y 4 pasan de tarjeta completa a fila de una línea.
5. `Confirmar señal` deja de ser el botón de mayor peso visual y pasa a secundario.
6. `Ejecutar corte con retorno` es lo único en rojo. En la maqueta el botón no abre el modal — el modal de P-5 queda por especificar en implementación.
7. Fechas localizadas con tiempo relativo.
8. Evidencia comparada (`1 s medidos vs. 120 s reservados`) en vez de valor suelto.

### A.5 Código completo

```html
<!DOCTYPE html>
<html lang="es-AR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Maqueta — Panel de verificaciones (Sai-Service-Core)</title>
<style>
  :root {
    --surface-0: #f5f4ef;
    --surface-1: #efeee8;
    --surface-2: #ffffff;
    --border: rgba(0,0,0,.10);
    --border-strong: rgba(0,0,0,.18);
    --text-primary: #26262a;
    --text-secondary: #5f5e5a;
    --radius: 8px;

    --bg-success: #eaf3de;  --text-success: #3b6d11;  --fill-success: #639922;
    --bg-warning: #faeeda;  --text-warning: #854f0b;  --fill-warning: #ba7517;
    --bg-danger:  #fcebeb;  --text-danger:  #a32d2d;  --border-danger: #f09595;
    --accent: #185fa5;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --surface-0: #1a1a19;
      --surface-1: #232322;
      --surface-2: #2c2c2a;
      --border: rgba(255,255,255,.12);
      --border-strong: rgba(255,255,255,.22);
      --text-primary: #ecebe6;
      --text-secondary: #a3a29c;

      --bg-success: #27500a;  --text-success: #c0dd97;  --fill-success: #639922;
      --bg-warning: #633806;  --text-warning: #fac775;  --fill-warning: #ba7517;
      --bg-danger:  #791f1f;  --text-danger:  #f7c1c1;  --border-danger: #a32d2d;
      --accent: #85b7eb;
    }
  }

  * { box-sizing: border-box; }
  body {
    margin: 0; padding: 32px 24px;
    background: var(--surface-0); color: var(--text-primary);
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    font-size: 16px; line-height: 1.6;
  }
  main { max-width: 860px; margin: 0 auto; }
  h1 { font-size: 28px; font-weight: 400; margin: 0 0 24px; }
  p { margin: 0; }
  .card {
    background: var(--surface-2); border: 1px solid var(--border);
    border-radius: 12px; padding: 20px 24px; margin-bottom: 12px;
  }
  .card--riesgo { border-color: var(--border-danger); }
  .card--fila { padding: 14px 24px; }
  .row { display: flex; align-items: center; gap: 12px; }
  .row--top { align-items: flex-start; }
  .grow { flex: 1; min-width: 0; }
  .titulo { font-size: 17px; font-weight: 500; }
  .sub { font-size: 13px; color: var(--text-secondary); }

  .chip {
    display: inline-flex; align-items: center; gap: 6px;
    font-size: 13px; padding: 4px 12px; border-radius: var(--radius); white-space: nowrap;
  }
  .chip--warning { background: var(--bg-warning); color: var(--text-warning); }
  .chip--success { background: var(--bg-success); color: var(--text-success); }
  .chip--danger  { background: var(--bg-danger);  color: var(--text-danger); }
  .chip--neutral { background: var(--surface-1);  color: var(--text-secondary); font-size: 12px; padding: 3px 10px; }

  .num {
    width: 26px; height: 26px; border-radius: 50%; flex-shrink: 0;
    display: flex; align-items: center; justify-content: center; font-size: 13px;
  }
  .num--pend { background: var(--bg-warning); color: var(--text-warning); }
  .num--ok   { background: var(--bg-success); color: var(--text-success); }

  .progreso { display: flex; gap: 4px; margin-top: 16px; }
  .progreso span { flex: 1; height: 5px; border-radius: 3px; background: var(--border-strong); }
  .progreso span.on { background: var(--fill-success); }

  button {
    font: inherit; font-size: 14px; cursor: pointer;
    background: transparent; color: var(--text-primary);
    border: 1px solid var(--border-strong); border-radius: var(--radius);
    padding: 8px 14px; display: inline-flex; align-items: center; gap: 7px;
  }
  button:hover { background: var(--surface-1); }
  button.primaria { background: var(--accent); border-color: var(--accent); color: #fff; }
  button.peligro  { border-color: var(--border-danger); color: var(--text-danger); }
  button.terciaria { border-color: transparent; color: var(--text-secondary); }
  button.terciaria:hover { border-color: var(--border); }

  .secuencia { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; font-size: 13px; color: var(--text-secondary); }
  .secuencia b { font-weight: 400; background: var(--surface-2); border: 1px solid var(--border); border-radius: var(--radius); padding: 5px 10px; }
  .chips { display: flex; gap: 6px; flex-wrap: wrap; margin: 12px 0; }
  .campo { font-size: 15px; margin-bottom: 6px; }
  .campo em { font-style: normal; color: var(--text-secondary); }
  ol { margin: 12px 0; padding-left: 22px; font-size: 15px; }
  li { margin-bottom: 4px; }
  .aviso { background: var(--bg-danger); border-radius: var(--radius); padding: 12px 14px; margin: 14px 0; }
  .aviso p { font-size: 14px; color: var(--text-danger); }
  .icon { width: 16px; height: 16px; flex-shrink: 0; }
  .sr { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); }
</style>
</head>
<body>

<svg class="sr" aria-hidden="true"><defs>
  <symbol id="i-check" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M5 12l5 5L20 7"/></symbol>
  <symbol id="i-bell" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M10 5a2 2 0 1 1 4 0a7 7 0 0 1 4 6v3a4 4 0 0 0 2 3H4a4 4 0 0 0 2-3v-3a7 7 0 0 1 4-6"/><path d="M9 17v1a3 3 0 0 0 6 0v-1"/></symbol>
  <symbol id="i-play" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M7 4v16l13-8z"/></symbol>
  <symbol id="i-eye" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="2"/><path d="M22 12c-2.7 4.7-6 7-10 7s-7.3-2.3-10-7c2.7-4.7 6-7 10-7s7.3 2.3 10 7"/></symbol>
  <symbol id="i-plug" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M9 2v6M15 2v6M6 8h12v3a6 6 0 0 1-12 0z"/><path d="M12 17v5"/></symbol>
  <symbol id="i-alert" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M12 3L2 20h20z"/><path d="M12 9v5M12 17.5v.01"/></symbol>
  <symbol id="i-server" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="18" height="7" rx="2"/><rect x="3" y="13" width="18" height="7" rx="2"/><path d="M7 7.5v.01M7 16.5v.01"/></symbol>
  <symbol id="i-user" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="8" r="4"/><path d="M4 21a8 8 0 0 1 16 0"/></symbol>
  <symbol id="i-pin" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round"><path d="M12 21s7-6.2 7-11a7 7 0 1 0-14 0c0 4.8 7 11 7 11z"/><circle cx="12" cy="10" r="2.5"/></symbol>
  <symbol id="i-arrow" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M5 12h13M13 6l6 6-6 6"/></symbol>
</defs></svg>

<main>
<h1>Panel de verificaciones</h1>

<!-- P-1 · Encabezado de estado del servicio -->
<section class="card">
  <div class="row row--top" style="gap:16px;flex-wrap:wrap">
    <div class="grow">
      <p class="sub" style="margin-bottom:6px">Estado del servicio</p>
      <div class="row" style="gap:10px;margin-bottom:8px">
        <span class="chip chip--warning"><svg class="icon"><use href="#i-bell"/></svg>Solo aviso</span>
        <span class="sub">2 de 4 pruebas verificadas</span>
      </div>
      <p>El sistema todavía <strong style="font-weight:500">no apaga el host automáticamente</strong>. Faltan 2 pruebas para habilitar el apagado ante corte de energía.</p>
    </div>
    <button class="primaria"><svg class="icon"><use href="#i-play"/></svg>Iniciar ejercicio guiado</button>
  </div>
  <div class="progreso" role="img" aria-label="2 de 4 pruebas verificadas">
    <span class="on"></span><span class="on"></span><span></span><span></span>
  </div>
</section>

<!-- P-3 · Tira de secuencia -->
<section style="background:var(--surface-1);border-radius:12px;padding:16px 20px;margin-bottom:20px">
  <p class="sub" style="margin-bottom:10px">Es un solo ejercicio físico, con presencia. Cada paso registra un tramo:</p>
  <div class="secuencia">
    <b>Cortás la red</b><svg class="icon"><use href="#i-arrow"/></svg>
    <b>El SAI avisa OB</b><svg class="icon"><use href="#i-arrow"/></svg>
    <b>El host se apaga</b><svg class="icon"><use href="#i-arrow"/></svg>
    <b>El SAI corta salida</b><svg class="icon"><use href="#i-arrow"/></svg>
    <b>Vuelve la luz, arranca solo</b>
  </div>
</section>

<!-- Paso 1 · Pendiente → tarjeta expandida (P-4) -->
<article class="card">
  <div class="row" style="margin-bottom:10px">
    <span class="num num--pend">1</span>
    <div class="grow">
      <p class="titulo">Señal en batería</p>
      <p class="sub">El SAI avisa que pasó a batería (OB)</p>
    </div>
    <span class="chip chip--warning">Pendiente</span>
  </div>
  <div class="chips">
    <span class="chip chip--neutral"><svg class="icon"><use href="#i-server"/></svg>Actúa el SAI</span>
    <span class="chip chip--neutral"><svg class="icon"><use href="#i-user"/></svg>Vos cortás la red</span>
    <span class="chip chip--neutral"><svg class="icon"><use href="#i-pin"/></svg>Presencial</span>
  </div>
  <p class="campo"><em>Qué se prueba —</em> que al quedarse sin energía de red, el SAI avise que está funcionando con su batería. Ese aviso es lo que dispara el apagado.</p>
  <ol>
    <li>Cortás la energía de red que entra al SAI.</li>
    <li>El SAI queda alimentando con su batería y lo informa.</li>
    <li>El sistema lee ese aviso y confirma el estado «en batería» (no manda ninguna orden al equipo).</li>
  </ol>
  <p class="campo" style="margin-bottom:14px"><em>Esperás ver —</em> el estado del equipo cambia a «en batería».</p>
  <button><svg class="icon"><use href="#i-eye"/></svg>Confirmar señal (solo lectura)</button>
</article>

<!-- Paso 2 · Vigente → fila colapsada (P-4) -->
<article class="card card--fila">
  <div class="row">
    <span class="num num--ok"><svg class="icon"><use href="#i-check"/></svg></span>
    <div class="grow">
      <p class="titulo">2 · Apagado ordenado del host</p>
      <p class="sub">1 s medidos vs. 120 s reservados · vigente hasta el 18 ene 2027 (faltan 6 meses)</p>
    </div>
    <button class="terciaria" aria-expanded="false">Ver procedimiento</button>
    <button class="terciaria">Re-verificar</button>
  </div>
</article>

<!-- Paso 3 · Pendiente + riesgo alto (P-4, P-5) -->
<article class="card card--riesgo">
  <div class="row" style="margin-bottom:10px">
    <span class="num num--pend">3</span>
    <div class="grow">
      <p class="titulo">Corte con retorno</p>
      <p class="sub">El SAI corta su salida y la repone</p>
    </div>
    <span class="chip chip--danger">Corta corriente real</span>
  </div>
  <div class="chips">
    <span class="chip chip--neutral"><svg class="icon"><use href="#i-server"/></svg>Actúa el SAI</span>
    <span class="chip chip--neutral"><svg class="icon"><use href="#i-pin"/></svg>Presencial</span>
  </div>
  <p class="campo"><em>Qué se prueba —</em> que el SAI pueda cortar la corriente que alimenta al host y volver a dársela por sí solo. Es el mecanismo del apagado.</p>
  <ol>
    <li>Se le ordena al SAI apagar la salida que alimenta al host y volver a activarla.</li>
    <li>El SAI espera el tiempo configurado —para que el host termine de apagarse— y recién ahí corta la corriente de la salida.</li>
    <li>Cuando regresa la energía de red, el SAI se activa tras una breve espera de estabilización y vuelve a alimentar la salida del host.</li>
  </ol>
  <p class="campo"><em>Esperás ver —</em> el SAI corta la corriente del host y, al volver la energía de red, la reactiva.</p>
  <div class="aviso">
    <p><svg class="icon" style="vertical-align:-3px;margin-right:5px"><use href="#i-alert"/></svg>No hay garantía de que el SAI reponga su salida ni de que el host vuelva a encender. Verificá antes el arranque automático en la BIOS del host.</p>
  </div>
  <button class="peligro"><svg class="icon"><use href="#i-plug"/></svg>Ejecutar corte con retorno</button>
</article>

<!-- Paso 4 · Vigente → fila colapsada (P-4) -->
<article class="card card--fila">
  <div class="row">
    <span class="num num--ok"><svg class="icon"><use href="#i-check"/></svg></span>
    <div class="grow">
      <p class="titulo">4 · Arranque automático del host</p>
      <p class="sub">Arrancó solo al restaurarse la energía · vigente hasta el 22 jul 2027 (falta 1 año)</p>
    </div>
    <button class="terciaria" aria-expanded="false">Ver procedimiento</button>
    <button class="terciaria">Re-verificar</button>
  </div>
</article>

</main>
</body>
</html>

```
