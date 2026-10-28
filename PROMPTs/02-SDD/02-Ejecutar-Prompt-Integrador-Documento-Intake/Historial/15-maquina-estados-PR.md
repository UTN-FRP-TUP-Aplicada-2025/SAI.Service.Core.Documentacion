Abrí el PR
https://github.com/UTN-FRP-TUP-Aplicada-2025/SAI.Service.Core/pull/new/feature/etapa-2a-dominio-equipos

Base: main ← Compare: feature/etapa-2a-dominio-equipos

Título:


Etapa 2·A — Modelo del ciclo de vida de equipos (BT-11, BT-12)
Descripción:


## Resumen
Primer incremento de la Etapa 2: modelo de dominio **framework-free** del ciclo de vida de equipos, con los invariantes aplicables corriendo como pruebas (mitigación del riesgo R-10). Sin persistencia ni UI (llegan en los incrementos siguientes).

### Tres capas (ADR-07)
- **Catálogo**: `Fabricante`, `ModeloDispositivo` (potencia nominal con `Origen`), `ModeloBateria` (I-21).
- **Inventario**: `UnidadFisica` (base) → `Host` / `Dispositivo` / `Bateria`; baja lógica (estado + fecha + motivo, sin borrado físico) y máquina de estados.
- **Vínculos temporales**: `Vigencia` (intervalo semiabierto `[desde, hasta)`, ADR-05); `MontajeBateria`, `CoberturaHost`; `Vigencias.AdmiteNuevo` (no-solape por clave); `ResolutorTemporal` (unidad vigente en un instante, RC-07).

### Invariantes como pruebas
I-1, I-2, I-3, I-4 (vigencia y sucesión sin hueco), I-5, I-6 (baja consultable y máquina de estados), I-21 (vida de flotación exige temperatura). **35 pruebas de dominio nuevas · suite 65/65 verde** · build 0/0 · `dotnet format` limpio.

### Notas
- `TipoIntervencion` / `Proveedor` (catálogo) se difieren a Etapa 4, donde se usan (Intervención).
- EF mapping, migración y panel de alta: incrementos C de esta etapa.
Un punto a confirmar (I-6 — máquina de estados)
El diagrama de transiciones (§7.8) que citan los casos de prueba no está en SDD/Docs (vive en el intake externo). Implementé este conjunto, derivado de CU-09 + TC-06 + ADR-07:


EnStock     → EnServicio, DadoDeBaja
EnServicio  → EnReparacion, DadoDeBaja
EnReparacion→ EnServicio, DadoDeBaja
DadoDeBaja  → (terminal)
Si tu diagrama del intake tiene otras aristas (p. ej. EnStock → EnReparacion, o baja solo desde ciertos estados), decímelo y lo ajusto — es un cambio chico. Si te sirve así, seguimos.

Cuando mergees y borres la rama, avisame con un "mergeado" y arranco el Incremento B (adaptador NUT real contra tu SAI en 127.0.0.1:3493).