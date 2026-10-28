Plan propuesto — Etapa 2 (EP-04, 58 SP)
Objetivo: el administrador registra el SAI y su batería desde el panel (con vínculos temporales y baja lógica), prueba la conexión, y publica políticas de apagado versionadas; el servicio arranca forzado en solo aviso con los cuatro supuestos en no verificado.

Propongo dividirla en 4 incrementos (cada uno compila, testea y se valida en navegador; PR por incremento):

#	Incremento	Ítems	Foco
A	Dominio del ciclo de vida	BT-11, BT-12	Catálogo/inventario/historia con baja lógica; Vigencia como intervalo + ResolutorTemporal; invariantes como tests (I-1..I-21). Sin hardware.
B	Adaptador de conexión + descubrimiento	BT-13, BT-14, BT-15, US-03	Anclaje USB por ruta física; contrato del puerto; adaptador NUT real; descubrimiento y prueba de conexión.
C	Alta desde el panel	US-04, US-05	Alta de catálogo+inventario con vigencias y baja lógica (MudBlazor interactivo, según maqueta); siembra de verificaciones + arranque en solo aviso.
D	Políticas (UF-2)	BT-16, US-06	Políticas de apagado versionadas (nueva versión sin editar la vigente), techo duro de 540 s; pantalla de configuración.
Empezaría por A (dominio puro, base de todo lo demás), y las páginas nuevas irían en MudBlazor interactivo (formularios ricos, no SSR estático como las de acceso — acá no hay restricción de cookie).

Dos decisiones antes de arrancar:

## ¿Cómo entregamos la Etapa 2 (58 SP)? Es la más grande hasta ahora.

- [x ] **4 incrementos / 4 PRs (recomendado)**
  Dominio → Adaptador+descubrimiento → Alta en panel → Políticas. PRs chicos, revisables y validables uno por uno. Mismo ritmo que veníamos.

- [ ] **2 PRs por flujo**
  UF-1 (alta de equipos completa: dominio+adaptador+panel) en un PR, UF-2 (políticas) en otro. Menos PRs, cada uno más grande.

- [ ] **1 PR único**
  Toda la Etapa 2 en un solo PR. Máximo tamaño, difícil de revisar y validar por partes.

- [ ] **Other**


## El adaptador NUT real (BT-15) habla con un SAI vía NUT. ¿Hay un SAI/servidor NUT accesible en tu entorno de desarrollo para validarlo en vivo?

- [ ] **No hay hardware ahora (recomendado)**
  Implemento el adaptador NUT real detrás del puerto con tests, pero el simulado sigue de default en DEV. Se valida contra un SAI real cuando lo tengas conectado.

- [ x] **Sí, hay NUT/SAI accesible**
  Me pasás host/puerto (o cómo llegar al NUT) y configuro+valido el adaptador real contra ese servidor NUT en esta etapa.

- [ ] **Diferir el NUT real**
  Esta etapa queda solo con el adaptador simulado; muevo BT-15 a cuando haya hardware. Avanzamos alta/políticas/panel con el simulado.

- [ ] **Other**


Respuesta 

Marcá "Sí, hay NUT/SAI accesible" y pasale estos datos:


Servidor NUT: 127.0.0.1  (localhost)
Puerto:       3493  (TCP, protocolo NUT estándar)
UPS name:     sai
Usuario RO:   saimon  /  password: saimon   (rol upsmon slave — solo lectura)
Lectura anónima de variables: permitida (GET VAR no requiere login)
SAI real:     INNO TECH / Voltronic (Cypress 0665:5161), driver nutdrv_qx (Voltronic-QS-Hex)
Estado vivo:  ups.status=OL, battery.charge=100, input.voltage=228.7, ups.load=12
Prueba CLI:   upsc sai        (lista completa)
              upsc sai ups.status