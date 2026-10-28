Tengo la especificación de la Etapa 1 — Persistencia, alta de administrador y sesión (EP-03). Es una etapa sustancial (24 SP, 4 BT + 2 US). Acá va el plan para tu aprobación antes de codear.

# Objetivo

Servicio navegable en la LAN donde el administrador único puede darse de alta, iniciar sesión, cerrar sesión y cambiar contraseña, sobre persistencia versionada (migraciones al arranque) y con procedencia obligatoria en todo valor.

# Qué voy a construir

1. Persistencia real (BT-07) — Infrastructure

SaiDbContext pasa a IdentityDbContext (esquema de Identity para el admin).
Primera migración EF Core + aplicación al arranque (Migrate() en Program.cs).
Por ahora solo el esquema de acceso; las tablas del parque llegan en la Etapa 2+.
2. Fundaciones del dominio (BT-08, BT-09) — Domain (framework-free)

Valor<T> con Origen obligatorio (invariante I-7): ningún valor sin procedencia. Propongo el enum Origen { Medido, Derivado, Declarado, Imputado, EstimadoPorDriver } (del intake).
Disciplina append-only (I para la historia): un interceptor de SaveChanges que rechaza Update/Delete sobre entidades marcadas append-only (todavía sin entidades de historia, pero el mecanismo queda listo y testeado).
3. Autenticación real (BT-10, US-01, US-02) — Web + Application

ASP.NET Core Identity (PBKDF2 por defecto, cookie de sesión — ADR-16).
US-01 — Alta inicial (primer arranque): si no hay admin, la app rutea a /alta-inicial (superficie sin chrome, según Design-Rules-Primer-Arranque); creado el admin, esa ruta se cierra y se exige login.
US-02 — Login / Cerrar sesión / Cambiar contraseña: las superficies Acceso-Login y las acciones de la barra (hoy deshabilitadas) pasan a ser reales.
4. Pruebas (mitigación R-10: los invariantes se escriben como tests)

Unitarias: I-7 (Valor<T> exige Origen), rechazo append-only.
Integración: alta del admin, login, logout, cambio de contraseña (WebApplicationFactory).
Cómo lo verifico
Igual que el Sprint 0: build + test + format + corro la app en contenedor .NET 10 y confirmo que arranca, migra y que los flujos responden. Vos validás en el navegador (alta → login → cambio de contraseña → logout). Refresco el sai-web para que lo veas.

# Un par de decisiones que quiero confirmarte
Integración Identity + Blazor: propongo el patrón estándar de Blazor Web App con Identity (componentes de Identity + cookie auth), no los endpoints de API de Identity. Es lo que encaja con el panel interactivo. ¿OK?
Alcance del esquema en Etapa 1: solo Identity (acceso). Las tablas del parque (Muestra, Agregado, etc.) las modelo cuando se construya su feature (Etapa 2+), no ahora. ¿De acuerdo, o querés que ya genere el esquema completo de las 23 tablas?
¿Apruebo y arranco a codear la Etapa 1 con esto (y esas 2 decisiones), o ajustás algo?

---

aprobado.
para las apis implementá el flujo de autenticado ROPC, con bearer token jwt 
