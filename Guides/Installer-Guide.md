# Guía del instalador — SAI.Service.Core

> Documento vivo. Reúne los conceptos que necesita quien **instala y administra** SAI.Service.Core en un
> host con un SAI/UPS real: cómo el servicio se comunica con el equipo, qué piezas intervienen, cómo se
> configura y cómo se diagnostica. Se irá ampliando a medida que surjan preguntas.

## Índice

1. [Panorama: cómo habla SAI.Service.Core con el SAI](#1-panorama-cómo-habla-saiservicecore-con-el-sai)
2. [Las tres piezas de NUT](#2-las-tres-piezas-de-nut)
3. [Dos responsabilidades que conviene no mezclar](#3-dos-responsabilidades-que-conviene-no-mezclar)
4. [Credenciales NUT (upsd.users)](#4-credenciales-nut-upsdusers)
5. [Configuración de SAI.Service.Core](#5-configuración-de-saiservicecore)
6. [upsmon: el apagado limpio del host](#6-upsmon-el-apagado-limpio-del-host)
7. [El apagado ordenado con retorno](#7-el-apagado-ordenado-con-retorno)
8. [Diagnóstico y comandos útiles](#8-diagnóstico-y-comandos-útiles)
9. [Problemas resueltos (registro)](#9-problemas-resueltos-registro)

---

## 1. Panorama: cómo habla SAI.Service.Core con el SAI

SAI.Service.Core **no habla directamente con el hardware del SAI**. Habla con **NUT** (Network UPS Tools),
que es la capa estándar que gestiona el equipo. Concretamente, el servicio es un **cliente de red** que se
conecta al servidor de NUT (`upsd`) por TCP, normalmente en `127.0.0.1:3493`.

```
                               ┌─ upsmon ──► apaga el sistema operativo   (cliente aparte)
                               │
SAI.Service.Core ──TCP 3493──► upsd ──► driver ──USB──► SAI ──► batería
   (cliente NUT)                        (nut-driver@<ups>)
```

Punto clave que evita confusiones: **el servicio no pasa por `upsmon`.** `upsmon` es otro cliente de NUT,
con una tarea muy específica (apagar el sistema operativo); no es un intermediario del servicio.

---

## 2. Las tres piezas de NUT

**NUT** es la *suite* completa (el paraguas), no un único proceso. Se compone de tres piezas con roles
distintos:

| Pieza | Servicio (systemd) | Qué hace | ¿Toca el USB? |
|---|---|---|---|
| **driver** | `nut-driver@<ups>` | Habla con el hardware del SAI y traduce su protocolo. Es el **dueño exclusivo** del dispositivo. | **Sí, solo él** |
| **upsd** (servidor) | `nut-server` | Expone el SAI en la red (puerto 3493). Relaya el estado y **acepta los comandos** (`INSTCMD`, `SET`). Es el punto único de acceso. | No |
| **upsmon** (monitor) | `nut-monitor` | Vigila el estado y, cuando corresponde, **apaga el sistema operativo** (`shutdown -h now`). Nada más. | No |

Consecuencias prácticas:

- **Nadie habla con el SAI salteando el driver.** Ni el servicio, ni `upsmon`, ni las herramientas de línea
  de comandos (`upsc`, `upscmd`): todos pasan por `upsd`, y `upsd` le pide al driver. No hay “conexión
  directa al dispositivo” por fuera de esto.
- **`upsmon` no gestiona el SAI.** El que lo gestiona/posee es el **driver**; el que lo sirve en red es
  **`upsd`**. `upsmon` solo apaga el host.

---

## 3. Dos responsabilidades que conviene no mezclar

Son dos caminos **independientes**. Confundirlos lleva a diagnosticar mal.

| | Qué resuelve | Camino | Depende de |
|---|---|---|---|
| **A. Comandar el SAI** | Leer estado y enviar órdenes (probar batería, ordenar apagado) — lo que el servicio necesita | servicio → `upsd` → driver → USB → SAI | driver + `upsd` |
| **B. Apagar el host limpio** | Que, ante un corte, el sistema operativo cierre servicios antes de perder la energía | `upsd` (estado) → `upsmon` → SO | `upsmon` |

- Para que **el servicio haga su trabajo** (verificar los supuestos, ordenar el corte con retorno) solo
  importa el **camino A**. `upsmon` es irrelevante ahí.
- Para que **un corte real no deje el host sucio** importa el **camino B** (`upsmon`). Es una
  responsabilidad del **sistema operativo**, no del servicio.

Se pueden verificar por separado, y conviene hacerlo así.

---

## 4. Credenciales NUT (upsd.users)

Los usuarios que `upsd` acepta se definen en **`/etc/nut/upsd.users`**. Hay dos niveles que importan:

- **Solo lectura**: alcanza para leer el estado (tensiones, `ups.status`, carga). Un usuario con rol
  `upsmon slave`/`master` ya puede leer.
- **Escritura / operación**: hace falta para **ordenar acciones** sobre el equipo (probar batería, apagado
  con retorno). Requiere permisos explícitos:
  - `actions = SET` → permite fijar variables (`SET VAR`, p. ej. los retardos de apagado).
  - `instcmds = <comando>` → permite ejecutar comandos instantáneos (`INSTCMD`). Se puede listar varios,
    una línea por comando.

Ejemplo de un usuario de operación con **mínimo privilegio** (solo lo que el servicio usa):

```ini
[saiop]
    password = una-clave-fuerte
    actions = SET
    instcmds = shutdown.return
    instcmds = test.battery.start.quick
```

> Evitá `instcmds = ALL`: habilitaría también comandos peligrosos como `load.off` o `shutdown.stayoff`.

Tras editar `upsd.users`, recargá el servidor: `sudo systemctl reload nut-server` (o `sudo upsd -c reload`).

**Seguridad:** el usuario de operación puede **apagar el host**. Usá una clave fuerte y no reutilices
credenciales de ejemplo (`usuario/usuario`) en nada que se parezca a producción.

---

## 5. Configuración de SAI.Service.Core

El servicio elige el adaptador y los datos de conexión por configuración. Claves (sección `Sai`):

| Clave | Ejemplo | Para qué |
|---|---|---|
| `Sai:Adaptador` | `Nut` | `Nut` = SAI real por NUT; cualquier otro valor (o ausente) = adaptador **simulado** (sin hardware) |
| `Sai:Nut:Host` | `127.0.0.1` | host de `upsd` |
| `Sai:Nut:Puerto` | `3493` | puerto de `upsd` (default 3493) |
| `Sai:Nut:Ups` | `sai` | nombre del UPS en `ups.conf` |
| `Sai:Nut:Usuario` | `saiop` | usuario de operación (para escribir/comandar) |
| `Sai:Nut:Password` | `…` | clave del usuario |

En variables de entorno (contenedor), los `:` se escriben como `__`:
`Sai__Adaptador=Nut`, `Sai__Nut__Host=127.0.0.1`, `Sai__Nut__Ups=sai`, `Sai__Nut__Usuario=…`,
`Sai__Nut__Password=…`.

**Sin credenciales de escritura**, el servicio arranca igual y **lee** el estado, pero **no puede
comandar**: las acciones de apagado responden que no tiene permiso (y el detalle técnico queda en el log del
servicio, no en la pantalla del operador). Es el modo seguro por defecto.

El **adaptador simulado** es útil para instalar/probar la interfaz sin hardware ni riesgo: acepta las
órdenes de forma inerte (no apaga nada).

---

## 6. upsmon: el apagado limpio del host

`upsmon` es la pieza que, ante un corte prolongado, **le dice al sistema operativo que se apague limpio**
antes de que el SAI le quite la energía. Es el equivalente a la señal del botón de encendido: hay un evento,
el SO lo “ve” y arranca el apagado ordenado.

Se configura en **`/etc/nut/upsmon.conf`**. La directiva central es `MONITOR`:

```
MONITOR <ups>@<host> <nº de fuentes> <usuario> <clave> <rol>
```

Ejemplo para un host con el SAI enchufado localmente por USB:

```
MONITOR sai@localhost 1 saimon saimon master
```

- **`master`** (o `primary`): este host es el que tiene el SAI conectado; es responsable de la orden final
  de corte al equipo cuando termina de apagarse.
- **`slave`** (o `secondary`): monitorea por red y apaga su propio SO, pero no comanda el corte final.
- El usuario/clave de la línea `MONITOR` deben existir en `upsd.users` con el rol `upsmon` correspondiente.

Si `upsmon.conf` no tiene una línea `MONITOR` válida, el servicio **muere al instante** al arrancar
(exit 1, “insufficient power configured”). Síntoma típico: `nut-monitor` en estado `failed` mientras
`nut-server` y `nut-driver@…` están `active`.

Tras editar: `sudo systemctl restart nut-monitor` y verificá con `systemctl is-active nut-monitor`.

> **Importante:** que `upsmon` esté caído **no rompe** al servicio (el servicio usa el camino A). Pero
> significa que **un corte real no apagaría el host de forma limpia**. Es un problema de despliegue a
> resolver aparte del servicio.

### Cuándo apaga `upsmon` (y cómo verificar que reacciona, sin apagar el host)

`upsmon` **no apaga ante cualquier corte**. Dispara el `SHUTDOWNCMD` solo cuando la situación es terminal:

- el SAI está **en batería y con batería baja** (`OB` + `LB`), o
- se declara un **apagado forzado** (`FSD`), o
- la cantidad de fuentes de energía sanas cae por debajo de `MINSUPPLIES`.

Un corte breve, restaurado **antes** de que la batería llegue a baja, **no** provoca apagado: `upsmon` solo
lo registra. Eso permite verificar que está vigilando **sin arriesgar el host**:

1. Cortá la energía de red unos segundos y volvé a conectarla (igual que la prueba de “señal en batería”).
2. Revisá su registro:
   ```bash
   sudo journalctl -u nut-monitor -e
   ```
   Tenés que ver que detectó el paso a batería y el retorno (mensajes tipo *“UPS … on battery”* y luego
   *“UPS … on line power”*). Si aparecen, el camino B está armado y reaccionando.

> Probar el **apagado limpio completo** (que efectivamente ejecute el `shutdown`) implica llegar a batería
> baja o forzar `FSD` (`upsmon -c fsd`), lo que **sí apaga el host de verdad**. No lo hagas en un host con
> servicios en producción sin prepararlo.

**Test end-to-end del apagado limpio (apaga el host).** Con el host preparado (servicios con estado
bajados, presencia física por si el arranque automático falla):

```bash
sudo upsmon -c fsd
```

Secuencia esperada: `upsmon` (primary) ejecuta el `SHUTDOWNCMD` → el SO cierra servicios → por el flag
`/etc/killpower` el SAI corta su salida → la repone tras `ups.delay.start` → si la BIOS tiene
encendido automático por presencia de energía, el host arranca solo.

**Cómo confirmar que el apagado fue limpio (y no un corte duro)**, cuando el host vuelve:

```bash
last -x -n 5 reboot shutdown
```

- **`shutdown system down`** en el registro → el SO se apagó **ordenado** (camino B correcto).
- **`crash`** en el registro → el host se cortó **en seco** (sin apagado del SO). Es lo que deja el botón
  `shutdown.return` de la app: ordena al SAI cortar, pero **no** apaga el SO (ver sección 7).

Este contraste `shutdown` vs `crash` es la forma más directa de saber si el camino B está realmente
funcionando, más allá de que el host haya vuelto a encender.

---

## 7. El apagado ordenado con retorno

Cuando el servicio ordena el “corte con retorno”, envía a `upsd` el comando **`shutdown.return`** junto con
dos retardos (`ups.delay.shutdown` y `ups.delay.start`). Físicamente:

1. El **SAI** espera `ups.delay.shutdown` segundos y **corta la energía de su salida**.
2. La salida queda sin energía durante `ups.delay.start` segundos.
3. Al reponer la energía, el host vuelve a arrancar **si su BIOS tiene activado el encendido automático por
   presencia de energía** (“power-back” / “restore on AC power loss”).

Dos advertencias que el instalador debe tener presentes:

- **`shutdown.return` es una orden al *hardware* del SAI**: le dice al equipo que corte su salida. **No le
  dice al sistema operativo que se apague.** El apagado limpio del SO es responsabilidad de `upsmon`
  (sección 6). Si `upsmon` no está activo/configurado, la orden equivale a **un corte de energía en seco**
  del host, con todo lo que estuviera corriendo.
- **No hay garantía de retorno**: si la BIOS no tiene el encendido automático, o el firmware del SAI no
  repone la salida, el host queda apagado. Por eso el servicio marca esta prueba como de riesgo y pide
  confirmación deliberada.

---

## 8. Diagnóstico y comandos útiles

Todos apuntan a `<ups>@<host>` (p. ej. `sai@127.0.0.1`). Las lecturas no alteran nada.

**Estado del SAI (solo lectura):**
```bash
upsc sai@127.0.0.1 ups.status          # OL = en línea, OB = en batería
upsc sai@127.0.0.1                      # todas las variables
upscmd -l sai@127.0.0.1                 # comandos que el SAI soporta
upsrw sai@127.0.0.1                     # variables escribibles (p. ej. los retardos)
```

**Validar credenciales de escritura SIN apagar nada** (hace `LOGIN` + `SET` de un retardo a su valor
actual → prueba el permiso sin efecto):
```bash
read -rsp "Clave: " PW; echo
upsrw -s ups.delay.start=$(upsc sai@127.0.0.1 ups.delay.start) -u <usuario> -p "$PW" sai@127.0.0.1
```

**Confirmar el permiso de `INSTCMD` sin apagar el host** (test de batería: `INSTCMD` real, solo una descarga
breve; **no** corta la salida):
```bash
upscmd -u <usuario> -p "$PW" sai@127.0.0.1 test.battery.start.quick
```

**Estado de las piezas de NUT:**
```bash
systemctl is-active nut-server nut-driver@sai nut-monitor
```
Interpretación rápida: si podés **leer/comandar** el SAI pero `nut-monitor` está `failed`, el problema es
**solo de `upsmon.conf`** (camino B), no del driver ni del USB (camino A sano).

---

## 9. Problemas resueltos (registro)

Registro de problemas concretos y su solución, en formato **síntoma → causa → solución**. Se agrega cada
caso que se resuelve.

### `nut-monitor` (upsmon) queda en `failed` mientras el resto de NUT anda

- **Síntoma:** `systemctl is-active nut-monitor` → `failed`; no hay proceso `upsmon`; pero `nut-server` y
  `nut-driver@sai` están `active` y se puede leer y comandar el SAI (`upsc`, `upscmd` funcionan).
- **Causa:** falta —o está mal— la directiva `MONITOR` en `/etc/nut/upsmon.conf`. `upsmon` sale con exit 1 a
  los milisegundos de arrancar (“insufficient power configured”). No es un problema de USB ni del driver: si
  se puede comandar el SAI, el camino A está sano; esto es puramente del camino B.
- **Solución:** agregar en `upsmon.conf` una línea `MONITOR` que apunte al UPS con un usuario que exista en
  `upsd.users` con rol `upsmon`, y reiniciar el servicio:
  ```
  MONITOR sai@localhost 1 saimon saimon master
  ```
  ```bash
  sudo systemctl restart nut-monitor
  systemctl is-active nut-monitor      # debe quedar 'active'
  ```

### `upsmon` arranca como primary pero le niegan privilegios (`ERR ACCESS-DENIED`)

- **Síntoma:** `nut-monitor` está `active` y **monitorea bien** (registra *on battery* / *on line power*),
  pero en su log de arranque aparece:
  ```
  Primary managerial privileges unavailable on UPS [sai@localhost]
  Response: [ERR ACCESS-DENIED]
  ```
- **Causa:** desajuste de rol entre `upsmon.conf` y `upsd.users`. La línea `MONITOR` declara `master`
  (correcto: este host tiene el SAI por USB), pero el usuario en `upsd.users` figura como `upsmon slave`
  —solo privilegios de secundario—, así que `upsd` le niega los de primary.
- **Impacto:** el monitoreo funciona, pero en un **apagado real** el primary no puede setear el flag `FSD`
  ni dar la orden final de corte al SAI (killpower) cuando el SO termina de apagarse: el apagado limpio
  queda **a medias**. No se nota mirando solo *on battery*/*on line*; hay que revisar el log de arranque.
- **Solución:** en `upsd.users`, poner el usuario como `upsmon master`, recargar `upsd` y reiniciar
  `upsmon`:
  ```ini
  [saimon]
      ...
      upsmon master        # era 'slave'
  ```
  ```bash
  sudo systemctl reload nut-server
  sudo systemctl restart nut-monitor
  sudo journalctl -u nut-monitor -e   # ya no debe aparecer ERR ACCESS-DENIED
  ```
- **Ruido benigno:** en el arranque de `upsmon` es normal ver `fopen /run/nut/upsmon.pid: No such file`,
  `Could not find PID file…` y las líneas `upsnotify: … will not spam` — no son errores.

### El panel dice “El sistema no tiene permiso para enviarle órdenes al SAI”

- **Síntoma:** al ejecutar *corte con retorno* o *prueba de apagado*, el operador ve ese mensaje. (El detalle
  técnico —comando, claves— queda en el **log del servicio**, no en la pantalla.)
- **Causa:** el usuario NUT que usa el servicio no tiene permisos de escritura.
- **Solución:** en `upsd.users`, darle `actions = SET` e `instcmds = …` (sección 4), recargar `upsd`
  (`sudo systemctl reload nut-server`), y configurar `Sai:Nut:Usuario`/`Password` con ese usuario.

### Al cortar la energía, el panel no refleja el estado del equipo

- **Síntoma:** el SAI está en batería (verificable con `upsc sai@127.0.0.1 ups.status` → `OB`), pero el panel
  no lo muestra y la verificación de señal no observa el corte.
- **Causa habitual:** el servicio está con el **adaptador simulado**, que no mira el hardware real.
- **Solución:** `Sai:Adaptador=Nut` (y los datos de `Sai:Nut:*`). Con el simulado, ninguna lectura proviene
  del SAI real.

---

*Última actualización: documento en construcción; se completa a partir de las consultas del instalador.*
