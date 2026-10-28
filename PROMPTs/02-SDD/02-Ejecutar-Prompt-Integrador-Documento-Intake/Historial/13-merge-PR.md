

Abrí el PR acá
https://github.com/UTN-FRP-TUP-Aplicada-2025/SAI.Service.Core/pull/new/feature/etapa-1-persistencia-acceso

Base: main ← Compare: feature/etapa-1-persistencia-acceso

Título:


Etapa 1 — Persistencia, autenticación del administrador y API con Bearer JWT (ROPC)
Descripción (copiá/pegá):


## Resumen
Primera etapa con lógica real sobre el andamiaje del Sprint 0.

- **Persistencia**: EF Core + SQLite con migración inicial y `DbContext` (`IdentityDbContext`) con interceptor append-only (ADR-04, ADR-18). La migración se aplica al arranque y siembra el rol único `administrador` idempotentemente.
- **Autenticación del administrador único** (ADR-16): ASP.NET Core Identity, sesión por cookie. Alta inicial, login, cambio de contraseña y cierre de sesión como formularios SSR.
- **Guarda de primer arranque en tres capas** (ruteo / superficie / acción): middleware que desvía a `/alta-inicial` mientras no exista administrador.
- **API REST con Bearer JWT vía ROPC** (**ADR-28**): `POST /api/v1/token` emite el token; `/api/v1/*` lo exige por la policy `Api`. Esquema dual con la cookie del panel.
- **Tema visual de la maqueta** aplicado al panel MudBlazor.

## Robustez detectada al validar en navegador
- Cadena de conexión y clave JWT leídas de forma **diferida** (aísla pruebas; firmar y validar comparten el secreto).
- **Keyring de DataProtection persistido en volumen** con nombre de app estable (**ADR-29**) → sesión y antiforgery sobreviven a reinicios.
- Formularios SSR con **un único** token antiforgery (se quita el `<AntiforgeryToken/>` duplicado que daba HTTP 400).
- **`UseAntiforgery` después de `UseAuthentication`** → los formularios autenticados validan el token contra el usuario correcto.

## Verificación
- Suite **30/30** (Domain 15, Application 2, Integration 13), con guards de antiforgery y flujo autenticado de cambio de contraseña. Build 0/0, `dotnet format` limpio.
- Validado en contenedor: alta → login → panel → cambiar contraseña → cerrar sesión; API token ROPC → `/api/v1/ping` 200.
- ADRs nuevos: **ADR-28** y **ADR-29**. Índice de decisiones a v1.5 (29 ADR). CHANGELOG actualizado.
Después
Igual que las etapas anteriores: mergeá el PR y borrá la rama remota, avisame con un "mergeado" y yo retomo — sincronizo main local, borro la rama local y arrancamos la Etapa 2.

El contenedor de validación (sai-web-e1) sigue levantado en :8080 por si querés seguir probando; lo bajo cuando me digas o al arrancar la próxima etapa.

---

bajalo ya, y segui con la proxima etapa
ya hice el PR y borre la rama