# 05 — Interfaces: panel y API

> **Propósito**: las superficies del sistema — páginas del panel Blazor, endpoints HTTP, autenticación y
> el contrato REST hacia terceros.
> **Fuentes primarias**: [`Web/Components/Pages/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Web/Components/Pages/) ·
> [`Web/Program.cs`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Web/Program.cs) ·
> [`Api/Endpoints/`](../../../SAI.Service.Core/src/SAI.Service.Core/SAI.Service.Core.Api/Endpoints/) ·
> [`SDD/Docs/05-Arquitectura-Tecnica/Contratos-REST-v1.0.md`](../../../SAI.Service.Core/SDD/Docs/05-Arquitectura-Tecnica/Contratos-REST-v1.0.md) ·
> [`SDD/Docs/03-UX-UI-DX/`](../../../SAI.Service.Core/SDD/Docs/03-UX-UI-DX/)

---

## 1. Páginas del panel (Blazor interactive server + MudBlazor)

| Ruta | Componente | Qué hace |
| --- | --- | --- |
| `/` y `/panel-estado-en-vivo` | `PanelEstadoEnVivo.razor` | Estado en vivo del SAI con procedencia visible por valor |
| `/acceso` | `Acceso/Acceso.razor` | Login del administrador único |
| `/alta-inicial` | `Acceso/AltaInicial.razor` | Alta del administrador en el primer arranque |
| `/cuenta/cambiar-contrasena` | `Acceso/CambiarContrasena.razor` | Cambio de contraseña |
| `/alta-de-equipos` | `AltaDeEquipos.razor` | Catálogo, inventario y vínculos temporales; descubrimiento del dispositivo |
| `/configuracion-de-politicas` | `ConfiguracionDePoliticas.razor` | Crea versiones de política; presets, «En palabras», historial |
| `/panel-de-verificaciones` | `PanelDeVerificaciones.razor` + `PanelVerificaciones/` | Los cuatro supuestos, su vigencia y el ejercicio guiado |
| `/panel-de-apagado` | `PanelDeApagado.razor` | Historial de acciones con modalidad solicitada vs efectiva |
| `/prueba-de-bateria` | `PruebaDeBateria.razor` | Prueba manual/programada y veredicto de salud con confianza |
| `/historicos-y-graficas` | `HistoricosYGraficas.razor` | Series, agregados con cobertura y marcas de eventos |
| `/registro-de-intervenciones` | `RegistroDeIntervenciones.razor` | Recambios de batería y ficha de vida útil |
| `/sustitucion-del-sai` | `SustitucionDelSai.razor` | Reparación/sustitución con cobertura suplente |
| `/informe-de-periodo` | `InformeDePeriodo.razor` | Informe de período y comparación de marcas |

Subcomponentes reutilizables del panel de verificaciones (`Components/Pages/PanelVerificaciones/`):
`TarjetaVerificacion`, `ChipEstado`, `TiraSecuencia`, `BannerEjercicio`, `IndicadorEnVivo`,
`EncabezadoEstadoServicio`, `DialogoConfirmacionRiesgo`, más los apoyos `DescriptorEnVivo`,
`FormatoRelativo` y `VistaPanelVerificaciones`.

Layout y tema: `Components/Layout/` (`MainLayout`, `LayoutAcceso`, `TemaSai`), con el sello de versión en
la barra superior (`SelloVersion.cs`, sección `Sello` de `appsettings.json`).

## 2. Endpoints HTTP

| Método | Ruta | Auth | Qué devuelve |
| --- | --- | --- | --- |
| GET | `/health` | anónimo | `{"estado":"ok","servicio":"SAI.Service.Core","utc":…}` |
| GET | `/api/v1/` | anónimo | Descriptor informativo de la API |
| GET | `/api/v1/ping` | Bearer (policy `Api`) | Prueba de punta a punta del flujo máquina-a-máquina |
| POST | `/api/v1/token` | anónimo (credenciales) | Token JWT por ROPC (ADR-28) — definido en `Web/Endpoints/EndpointsAcceso.cs` |
| POST | `/api/v1/intervenciones` | Bearer (policy `Api`) | **Ingesta idempotente** — el único contrato formal hacia terceros |
| POST | `/cuenta/cerrar-sesion` | cookie | Cierre de sesión del panel |

## 3. Contrato de ingesta — `POST /api/v1/intervenciones`

Cabeceras **obligatorias**: `X-Idempotency-Key` (clave del emisor) y `X-Fuente-Datos` (fuente registrada,
que aporta la confianza base — el GMAO externo `fd-gmao-externo` entra con confianza **media**).

| Código | Condición | Cuerpo |
| --- | --- | --- |
| **201** | Cuerpo válido y clave nueva | `{id, creado:true, confianza:"media", tiempoValido, tiempoRegistrado}` |
| **200** | Misma clave, **mismo** cuerpo | `{id, creado:false}` — el mismo id, sin duplicar |
| **409** `conflicto_idempotencia` | Misma clave, cuerpo **distinto** | problem+json con `sha256Original`, `sha256Recibido`, `accionSugerida`. **Nunca 200**: «devolver 200 sería peor que duplicar» |
| **422** | Invariante roto | problem+json con `campo` e `invariante`: `validacion` (cuadre de costos RN-08, o `Dinero` sin moneda/fecha RN-07) o `coherencia_temporal` (hecho fechado después de la baja de la unidad, RN-12) |
| **400** `cabeceras_faltantes` | Falta alguna cabecera obligatoria | problem+json |
| **401** | Sin token Bearer válido | — |

Detalle de implementación: el endpoint lee el **cuerpo crudo**, calcula la huella sha256 y deserializa de
ese mismo texto, de modo que un reintento byte a byte idéntico produce la misma huella (200) y cualquier
diferencia produce otra (409).

Formato de errores: **problem+json (RFC 7807)**, habilitado con `AddProblemDetails()` (ADR-17). Los `type`
`conflicto_idempotencia`, `validacion` y `coherencia_temporal` son **estables**: no se renombran sin
cambio de versión.

Versionado: la versión va en la ruta (`/api/v1/`). Cambios aditivos no rompen versión; quitar o renombrar
un campo obligatorio, cambiar la semántica de un código o un `type` reservado abre `/api/v2/`.

**Pendiente declarado**: el endpoint de *rectificación* que sugiere el 409 no existe — es ADR-21, en
estado Propuesto. Hoy el 409 devuelve la acción sugerida como texto.

## 4. Autenticación y sesión

| Superficie | Esquema | Detalle |
| --- | --- | --- |
| Panel | Cookie de Identity (`sai.auth`), esquema por defecto | ASP.NET Core Identity sobre `SaiDbContext`; administrador único con rol `administrador` (ADR-16) |
| API `/api/v1` | Bearer JWT, policy `Api` | Token emitido por `POST /api/v1/token` (ROPC, ADR-28); HMAC con `Jwt:ClaveFirma` (≥ 32 bytes o el arranque **falla**) |

Política de contraseña: ≥ 12 caracteres, con minúsculas y dígitos (no exige mayúsculas ni símbolos).

Detalles del pipeline que **no** se pueden reordenar sin romper cosas:

1. Los parámetros de validación del Bearer se resuelven **de forma diferida** desde `IOptions<OpcionesJwt>`
   — el mismo origen que usa el generador para firmar. Leerlos *eager* disocia la clave de validación de
   la de firma y produce 401 en las pruebas de integración.
2. `UseAntiforgery()` va **después** de `UseAuthentication()`: los tokens de formularios SSR autenticados
   se atan al usuario por su claim; al revés, el POST falla con HTTP 400 (EVE-04).
3. El **keyring de DataProtection** debe persistirse en un volumen (`DataProtection:RutaLlaves`), o cada
   reinicio invalida sesiones y rompe los formularios antiforgery (ADR-29, EVE-05).

## 5. Guarda de primer arranque

Un middleware previo a la autenticación desvía **todas** las superficies del panel a `/alta-inicial`
mientras no exista ningún administrador. Quedan exentas: `/alta-inicial`, `/acceso`, `/cuenta`, `/health`,
`/api`, `/_blazor`, `/_framework`, `/_content` y todo path cuyo último segmento tenga extensión (activos
estáticos). Una vez creado el administrador, la comprobación se apaga con una bandera cacheada para no
consultar la base en cada solicitud. Es la capa de **ruteo**; las superficies de acceso y la acción de
alta conservan además sus propias guardas.

## 6. UX y maqueta

La maqueta navegable aprobada vive en
[`SDD/Maquetas/Sai-Service-Core/`](../../../SAI.Service.Core/SDD/Maquetas/Sai-Service-Core/) (HTML/CSS/JS
estático, una página por superficie). Los wireframes y el contrato de datos de la maqueta están en
[`SDD/Docs/03-UX-UI-DX/`](../../../SAI.Service.Core/SDD/Docs/03-UX-UI-DX/), junto con la línea base visual
y el glosario UX.

Principio de redacción de la interfaz (EVE-07): **dos audiencias**. A la UI va lenguaje de operador; el
detalle técnico de NUT (`shutdown.return`, claves de configuración) va solo al log. Hay un test anti-jerga
que lo defiende.
