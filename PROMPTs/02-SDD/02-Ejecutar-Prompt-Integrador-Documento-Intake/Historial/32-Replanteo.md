
# El problema de fondo

- El puerto IAdaptadorConexion solo le habla al SAI (la UPS) — no al host. Mirá lo que expone IAdaptadorConexion.cs:18-39: leer estado del SAI, probar conectividad, ordenar apagado con retorno, test de batería. Y EstadoSai (AdaptadorConexionSimulado.cs:60-68) tiene tensiones/carga de la UPS, no tiene ningún dato del host (ni "alcanzable", ni uptime).

Eso significa que hoy no existe la infraestructura para lo que elegiste:

 1. "El botón dispara el apagado real y lo cronometra" → no hay una operación que dispare el apagado del SO del host y observe cuándo terminó de bajar. Lo más cercano es OrdenarApagadoConRetornoAsync, pero eso corta la energía (es el supuesto 3 "corte con retorno"), no es el apagado ordenado del SO que este supuesto quiere cronometrar.

 2. "Detectando reinicio del host" → no hay ninguna señal del host para saber que se reinició. Habría que inferirlo (p. ej. ups.load cayendo a ~0 = host abajo) o agregar una observación del host al puerto.

# Lo que implica

Lo que pediste es un incremento nuevo, no un retoque de texto. Toca cuatro capas:

  . Puerto/adaptador: nueva operación "disparar apagado ordenado del host" + una señal de estado del host (abajo/arriba) para cronometrar y detectar el reboot.

  . Simulado: modelar un ciclo de vida del host en el tiempo (trabajando → apagándose → apagado → reiniciando → arriba), respetando el principio del proyecto de que el apagado real nunca se dispara contra un host en desarrollo.

  . Dominio/aplicación: ServicioVerificacion.VerificarPresupuestoAsync pasa de recibir segundos a mano a disparar + poll + registrar el tiempo medido, con un estado transitorio "disparado, esperando reinicio".

  . Panel: gating por ese estado nuevo + la leyenda.

Esto es del tamaño de un Etapa 4·E, con sus pruebas — no encaja como un commit de UX en fix/panel-verificaciones-ux.

Mi recomendación
Partirlo en dos:

Your questions have been answered: "Lo que elegíste (disparar+cronometrar el apagado y detectar el reinicio del host) es un incremento nuevo, no un retoque de texto, y no hay infraestructura de 'host' en el adaptador todavía. ¿Cómo lo encaramos?"="Abrir el incremento completo ahora". You can now continue with these answers in mind.


---

Sí, tenés razón — y ahora tiene más sentido que antes. Cuando te propuse "partirlo en dos", el incremento era enorme (auto-medición + detección de reboot por hardware). Con tu corrección de ADR-25 se achicó: el timing es manual y el reboot se detecta por arranque del servicio. Pero sigue habiendo dos cosas de naturaleza distinta mezcladas:

Texto puro (el reword del punto 2 + afinar Actúa/Qué se prueba): cero backend, cero migración, cero riesgo. Encaja en el PR de UX que ya tenés abierto (fix/panel-verificaciones-ux, 5 commits esperando merge).
Comportamiento (disparar el apagado + marcador persistido PruebaEnCursoDesde + ServicioRearmePruebas de arranque + gating/leyenda + migración + tests): esto es un incremento nuevo con su migración de base y sus pruebas.
Mi recomendación es partirlo así:

Qué entra	Dónde

Parte 1 — ahora	Reword paso 1 ("esta prueba inicia el apagado ordenado") + afinado de textos. Cierra y mergea el PR de UX.	rama actual fix/panel-verificaciones-ux

Parte 2 — incremento	Disparo del apagado, marcador + rearme por reinicio, gating+leyenda, migración, tests, changelog.	rama nueva feature/etapa-4e-... desde main (ya con Parte 1 mergeada)
Ventaja: el PR de UX se cierra limpio y mergeable ya, sin arrastrar una migración de esquema; y el comportamiento nuevo entra revisable por separado, con su changelog y sus pruebas.

---

### Estructura

**¿Cómo encaramos el trabajo: partido en dos o todo junto?**

- [x] **Partido en dos (recomendado)**
  Parte 1 ahora en la rama actual: solo el reword + afinado de textos, y cierro/mergeo el PR de UX. Parte 2 como incremento nuevo (Etapa 4-E) en rama propia desde main: disparo + marcador + rearme + gating + migración + tests + changelog. Cada PR revisable por separado.

- [ ] **Todo junto en el incremento**
  Un solo PR nuevo con todo (texto + comportamiento). Más simple de secuenciar, pero el PR mezcla cosmética con una migración de esquema y lógica nueva, y el PR de UX actual queda sin cerrar o se absorbe acá.

- [ ] **Solo la Parte 1 por ahora**
  Hago únicamente el reword + afinado de textos y cierro el PR de UX. El comportamiento (disparo + rearme) lo planificamos y lo hacemos después, en otra sesión.

- [ ] **Other**