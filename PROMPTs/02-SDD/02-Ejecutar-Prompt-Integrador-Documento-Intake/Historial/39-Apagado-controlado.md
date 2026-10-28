Impacto = ninguno. Verifiqué: nada de lo que escribí (Installer-Guide, CHANGELOG) referencia esos archivos de reglas. El 04-Fase-G.md es un registro histórico del flujo de intake SDD (árbol de PROMPTs/ + tu sistema de reglas en IA/…/Rules), separado del código de SAI.Service.Core y del test de NUT. Y bien que lo editaste vos — por tu indicación permanente yo esa carpeta solo la leo. Si hay una conexión puntual que se me escapa, decímela; si no, no toca nada de lo nuestro.

Vamos con B — pero necesito cerrar dos cosas que solo vos sabés
B es sudo upsmon -c fsd → apaga limpio todo el host. Antes de disparar:

¿Hay una impresión 3D en curso? (print3d-server). Si la hay, no disparamos hasta que termine o la pares vos.
¿Estás físicamente en la máquina, por si el power-back de la BIOS no la levanta?
El plan, una vez que confirmes
Yo bajo los contenedores con estado, empezando limpio por la VM de Windows y dándole tiempo: docker stop windows → después print3d-server my-ai-api my-ai-video my-ai-webui runner-github discord-bot-moderador portainer. Dejo solo sai-web-e1. (Aunque B hace apagado limpio del SO, prefiero bajar Windows a mano para que tenga su tiempo de cierre ACPI.)
Vos disparás — necesita sudo, no lo puedo correr yo:

sudo upsmon -c fsd
Eso hace: upsmon (primary) → shutdown -h (SO cierra servicios) → flag /etc/killpower → el SAI corta la salida → a los 180 s la repone → power-back BIOS → boot → sai-web-e1 vuelve solo.
Yo pierdo el shell mientras esté abajo (~4-5 min). Cuando vuelva, verifico el ciclo con el boot_id que ya guardé (5a67e751…): que cambió (hubo reboot), que los contenedores volvieron, que upsmon relevantó, y que la app quedó sana.
Confirmame los dos puntos (impresión 3D / presencia física) y le doy para adelante con el paso 1. No bajo nada ni disparás hasta ese OK.

---

Confirmado. Arranco el paso 1 — bajo los contenedores con estado, empezando limpio por la VM de Windows con tiempo generoso para su cierre ACPI (hasta 3 min), y después el resto. sai-web-e1 no lo toco.

```
cd /home/fernando/workspaces/workspace-dev
echo "=== 1) Windows (cierre ACPI, hasta 180s) ==="
time docker stop -t 180 windows
echo "windows → $(docker ps -a --filter name=^windows$ --format '{{.Status}}')"
echo "=== 2) el resto (hasta 30s c/u) ==="
docker stop -t 30 print3d-server my-ai-api my-ai-video my-ai-webui runner-github discord-bot-moderador portainer
echo "=== estado final: qué queda corriendo ==="
docker ps --format '{{.Names}}\t{{.Status}}'
```


```
=== 1) Windows (cierre ACPI, hasta 180s) ===
windows

real	0m4,212s
user	0m0,022s
sys	0m0,004s
windows → Exited (143) Less than a second ago
=== 2) el resto (hasta 30s c/u) ===
print3d-server
my-ai-api
my-ai-video
my-ai-webui
runner-github
discord-bot-moderador
portainer
=== estado final: qué queda corriendo ===
sai-web-e1	Up 15 hours
```

---

Observación: windows salió con Exited (143) en 4 s (SIGTERM). Puede que no haya alcanzado a cerrar el guest del todo, pero el contenedor está detenido, que es lo que importa para el test. Si querés, después chequeás la VM.

Ahora disparás vos — sudo upsmon -c fsd

sudo upsmon -c fsd
Qué hace: fuerza el FSD → upsmon (primary) ejecuta shutdown -h → el SO cierra servicios limpio → al final, por el flag /etc/killpower, le ordena al SAI cortar la salida.

Una honestidad más, antes de que aprietes
El apagado del SO va a ser limpio, eso sí. Pero el auto-retorno no está garantizado con fsd: el corte de killpower asume un escenario de "volvé cuando regrese la red", y acá la red nunca se fue (forzamos el fsd con la luz presente). Según cómo el driver arme ese corte, puede pasar que:

el SAI corte y reponga a los ~180 s → power-back BIOS → arranca solo ✅, o
el SAI corte y quede esperando un ciclo de red que no va a ocurrir → el host queda apagado ❌ → lo prendés a mano (por eso te pedí estar presente).
Cualquiera de los dos es un resultado válido del test — justamente estamos averiguando empíricamente cómo se comporta tu SAI. Con vos ahí, si queda apagado, botón de encendido y listo.

Después
Yo pierdo el shell cuando baje. Avisame cuando el host esté de vuelta (arrancó solo o lo prendiste) y verifico todo: que hubo reboot (boot_id cambió), que sai-web-e1 relevantó, que upsmon volvió sano, y que la app quedó bien. Ojo: los contenedores que bajé (windows, IA, etc.) no vuelven solos —los paré a mano— así que después te ayudo a levantarlos.

Cuando quieras, apretá. Quedo esperando tu aviso de que volvió.