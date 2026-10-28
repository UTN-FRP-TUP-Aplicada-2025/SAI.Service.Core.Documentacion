
# Organización, estilos y patrones de código.

Seguir como referencia  esta esta estructura de solucion, donde Domain , Applicatin, Infraestructure Api y web son un solo proyecto

```
src/
├── MiEmpresa.Reservas.Domain/
├── MiEmpresa.Reservas.Application/
├── MiEmpresa.Reservas.Infrastructure/
├── MiEmpresa.Reservas.Api/       Sdk.Web   → Infrastructure   (controllers o endpoints)
└── MiEmpresa.Reservas.Web/       Sdk.Web   → Api               (arranca el proceso)
````


