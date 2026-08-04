

## Sprint 0 completo y commiteado (42157fe, en feature/sprint-0-arranque).

Lo que hay ahora

- **Bloque A** — decisiones cerradas: ADR-25 (NUT en contenedor), ADR-26 (TLS Kestrel autofirmado), ADR-27 (contrato IAdaptadorConexion); ADR-19/20/22 superados; índice a 27 ADR.
- **Bloque B** — andamiaje: solución .NET 10 de 5 assemblies (Clean Architecture) + 3 proyectos de test, Dev Container, scripts, CI inicial, y el panel base en Blazor+MudBlazor (menú lateral + barra superior + sello de versión según la maqueta, con "Cambiar Contraseña"/"Cerrar Sesión" y las 9 superficies de trabajo). Stubs para el adaptador simulado, el DbContext y la ranura de Identity. Sin lógica de negocio.

## Importante: el código está sin compilar

No tengo dotnet en este entorno, así que no puedo afirmar que compila — lo generé con cuidado, pero la verdad de "compila y corre" la da tu Dev Container. El subagente anticipó estos riesgos reales del primer build:

1. TreatWarningsAsErrors + analizadores (los pide el intake): pueden escalar diagnósticos a error en el primer build. El más probable.
2. MudBlazor 8.5.0 sobre .NET 10: si esa versión no soporta net10, hay que bumpear a la que sí.
3. dotnet format --verify-no-changes: el código está escrito a mano; conviene correr dotnet format una vez para normalizar.
4. HTTPS 8443 depende del dev-cert confiado (dotnet dev-certs https --trust la primera vez).

## Validación de cierre del Sprint 0 (tu parte)
En tu Dev Container:

```
devcontainer up --workspace-folder .
dotnet format SAI.Service.Core.sln          # normaliza antes del build
./scripts/build-all.sh                       # restore + build Release
./scripts/run.sh SAI.Service.Core.Web        # → http://localhost:8080  (salud en /health)
```

Y verificá que el panel base coincide con la maqueta en el navegador.

Cómo seguimos
Si falla el build, pegame los errores y los corrijo acá mismo en la rama (para eso están los riesgos anticipados). Iteramos hasta que compile.
Si compila y corre y el panel te cierra, pusheo feature/sprint-0-arranque y te armo el PR del Sprint 0 — el mismo flujo de siempre (mergeás, borrás la rama, retomo).
No pusheé todavía (está local). ¿Lo probás en el Dev Container y me contás cómo salió el build?

---

> Respuesta 

. lanza vos el devcontainer y analiza los errores y hace los ajustes así puedo validarte que la maqueta se refleje en el diseño

> agregar que utilice los scripts