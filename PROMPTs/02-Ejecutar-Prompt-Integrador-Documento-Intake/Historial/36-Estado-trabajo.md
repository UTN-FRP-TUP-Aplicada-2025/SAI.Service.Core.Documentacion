✅ PR #18 abierto
https://github.com/UTN-FRP-TUP-Aplicada-2025/SAI.Service.Core/pull/18

Base fix/panel-verificaciones-ux (se re-apunta a main cuando mergees el PR de UX). Los 6 commits:

Commit	Qué
55f4933	Rediseño UX Fases 1+2 (P-1…P-6, PorVencer, evidencia comparada, microcopy)
b163a11	Ejercicio guiado (P-7)
b3bcdfc	Fix: cultura nombrada rompía el render
d60a59f	Fix: llave suelta
(msg)	Fix: mensaje del write path en lenguaje de operador
d56f2bf	Indicador de estado del SAI en vivo
Estado del trabajo
Suite: 210 pasados, 0 fallidos, 0 warnings.
Contenedor: sai-web-e1 arriba, contra tu SAI real, con credenciales de escritura — los pasos 2 y 3 están armados para apagar de verdad.
Validado en vivo: indicador de batería, mensajes de operador, ciclo persistido de reinicio del host.
Lo que hicimos, en perspectiva
Empezó como "ejecutar una SPEC de UX" y terminó siendo un ciclo completo: evaluación → implementación → validación en hardware real. Esa última parte fue la que más valió — los tres fixes (b3bcdfc, d60a59f, el del mensaje) salieron de tu prueba en vivo, no de la suite. Ninguno lo agarraron 210 tests verdes, porque eran de render y de lenguaje al usuario. Buen recordatorio de que "verde" no es "funciona".

Pendientes (tuyos, cuando quieras)
Mergear #18, y el PR de UX (fix → main) que sigue abierto y arrastra todo.
El test end-to-end real de los pasos 2/3: apretar el botón, ver el host apagarse y volver por el power-back de la BIOS. Con --restart unless-stopped, el contenedor debería volver solo.
Deuda anotada: saimon/saimon con permiso de apagado es débil para algo parecido a producción.
¿Te dejo acá, o querés que te acompañe el test end-to-end de apagado real?