Tu cadena está mal — la real es esta
Vos escribiste:


sai.service.core → so → upsmon → usb → sai → batería      ❌
La real:


```
                          ┌─ upsmon ──► apaga el SO           (cliente aparte)
                          │
sai.service.core ──3493──► upsd ──► driver ──USB──► SAI ──► batería   ✅
     (cliente TCP)                (nut-driver@sai)
```

upsmon no está en serie con nada tuyo. Es un cliente hermano, no un intermediario.


# Qué es cada pieza (y quién toca el USB)

  - NUT = Network UPS Tools. Es la suite entera, el paraguas. No es un servicio; es el conjunto: driver + upsd + upsmon + herramientas.
  - driver (nut-driver@sai) = el único que toca el USB. Es dueño exclusivo del dispositivo. Nadie más habla con el SAI directo — ni la app, ni upsmon, ni vos.
  - upsd (nut-server) = el servidor/hub de red (puerto 3493). Todos los clientes le hablan a él, y él le pide al driver. Es el punto único.
  - upsmon (nut-monitor) = un cliente más, con una sola tarea: vigilar el estado y, si toca, apagar el SO con shutdown -h now. No "gestiona el UPS". No lee para la app. Solo apaga el host.

# Respondiendo tus preguntas puntuales

**¿Nos comunicamos directo al SAI sin upsmon?**
Sí, y ya lo hacemos. sai.service.core es un cliente TCP que le habla a upsd (127.0.0.1:3493). upsmon no interviene. Lo más "directo" que se puede llegar al SAI es a través del driver (dueño del USB) vía upsd — eso vale para la app, para upsc, para todos.

¿Estamos testeando en realidad que upsmon funcione?
No. Para las verificaciones de la app (señal en batería, corte, apagado), upsmon es irrelevante. Eso ya lo probamos: leímos OB y mandamos SET — todo por el camino app→upsd→driver→SAI, con upsmon caído. Funcionó igual, justamente porque upsmon no está en el medio.

Tus dos puntos, reordenados
Estás mezclando dos cosas independientes:

Qué verifica	Camino	Estado hoy
1. Que NUT llegue al SAI	driver ve el USB, upsd responde, la app lee/comanda	app → upsd → driver → USB → SAI	✅ verificado (leímos OB, mandamos SET; falta el INSTCMD, que confirma el test de batería)
2. Que el host se apague limpio ante un corte	upsmon reacciona y ejecuta shutdown -h now	upsd(estado) → upsmon → SO	❌ roto (upsmon fallado)
El punto 1 no necesita upsmon. El punto 2 es upsmon, y es un concern del sistema operativo, no de la app.

Entonces, ¿qué diagnosticamos?
Dos cosas distintas, y conviene no confundirlas:

Para que la app haga su trabajo (verificar los supuestos, ordenar el corte): solo importa el camino 1. Está casi todo validado; el test de batería lo cierra, sin apagar nada.
Para que un corte real no te deje el host sucio: importa upsmon (camino 2). Está roto — y eso es un problema real de tu despliegue, no del test: si mañana se corta la luz de verdad, este host no se apaga limpio.
Mi propuesta, ahora que está claro
Son dos trabajos separados, hagámoslos separados:

Cerrar el camino 1 (la app ↔ SAI) con el test de batería — seguro, es un INSTCMD real, no apaga nada. Con eso, todo lo que la app necesita del SAI queda probado end-to-end.
Arreglar upsmon como tarea aparte — averiguar por qué nut-monitor falló y levantarlo. Eso te da el apagado limpio real, y recién ahí un test de shutdown.return sería un apagado ordenado y no un corte duro.
¿Arranco por el test de batería (cierra el punto 1 ya, sin riesgo), y después miramos por qué upsmon está caído?

---

Lo que sí puedo afirmar
upsmon muere a los 10 ms de arrancar, con exit 1. Eso es un error de configuración al leer su archivo, no un problema de hardware — y lo confirma el hecho de que hace 2 minutos comandamos el SAI sin problema (driver, upsd y USB están sanos). Así que descartá:

❌ que NUT no vea el SAI → lo ve (leímos, escribimos, mandamos INSTCMD).
❌ problema de USB → el driver nut-driver@sai corre y responde.
✅ el problema es solo de upsmon.conf — casi seguro le falta (o tiene mal) la línea MONITOR, que es lo que le dice a qué UPS vigilar y con qué credenciales. Un upsmon.conf sin MONITOR válido sale con "insufficient power configured" y muere al instante, exactamente como este.
Para confirmarlo — corré estos dos (con sudo) y pegame la salida

sudo journalctl -u nut-monitor -b | tail -20
sudo grep -viE '^\s*#|^\s*$' /etc/nut/upsmon.conf
El primero da el mensaje fatal exacto; el segundo muestra qué tiene hoy (si no aparece ninguna línea MONITOR, ese es el problema).

# El arreglo, casi seguro

Vas a necesitar una línea MONITOR que apunte al SAI con un usuario que exista en upsd.users. Ya tenés a saimon con rol upsmon slave, así que probablemente sea agregar/corregir en `/etc/nut/upsmon.conf`:


```
MONITOR sai@localhost 1 saimon saimon master
Formato: MONITOR <ups>@<host> <nº fuentes> <usuario> <clave> <rol>.
```

Un matiz de rol: este host es el que tiene el SAI enchufado por USB, así que le corresponde master (el que da la orden final de corte al UPS al terminar de apagarse). Pero en upsd.users saimon figura como upsmon slave. Para que no haya desajuste, o ponés slave en ambos lados (funciona: apaga el host pero no comanda el killpower final), o master en ambos. Con lo que me pases de los dos comandos te digo la línea exacta y en qué archivo tocar.

Después: sudo systemctl restart nut-monitor y verificamos que quede active.

 > Che, un apunte de encuadre: esto ya es arreglar el despliegue NUT de tu host, que es valioso pero es harina de otro costal que el código de SAI.Service.Core (que quedó cerrado y pusheado en el PR #18). Lo del upsmon te sirve para que un corte real te apague limpio — vale la pena — pero decime si querés que sigamos metidos en esto o lo dejás anotado como tarea de infra y cerramos acá. Yo te acompaño en lo que elijas.