# Tool-Prompt — Prompt Integrador - Integrar documentos y generar Documento Intake

> **Invocación**:
> - `Lee y ejecuta /DEV/SAI.Service.Core.Documentacion/PROMPTs/02-Ejecutar-Prompt-Integrador-Documento-Intake/Ejecutar-Prompt-Integrador-Documento-Intake.md`
> Overview: Integrar los documentos de entrada y especificación de desarrollo en el documento Documento Intake

---

# Contexto 

  1. El como se implementa una solución nueva con el `Framework SDD` se trata en: `/IA/IA.SDD/SDD/Guides/SDD-Getting-Started-Guide.md`.  Este framework se aplica sobre el  `Repositorio destino` dado más abajo. Leer  `Framework SDD`, aquí están los pasos y métología SDD.
  `Template Intake a construir` es `/IA/IA.SDD/SDD/Devs/Intake/SOLUTION-INTAKE-template.md`

  2. `Repositorio destino` es `/DEV/SAI.Service.Core`, es donde se generará la documentación de especificación SDD, y en etapas posteriores se ejectuará la codificación mediante la invocación de un prompt orquestador.

  3. Sobre el contexto necesario para construir el documento intake demadando por `Framework SDD`. Leer y poner en contexto en que consiste la solución a especificar en función de lo que requiera el `Template Intake a construir`:

    3.1 Leer `/DEV/SAI.Service.Core.Documentacion/PROMPTs/02-Ejecutar-Prompt-Integrador-Documento-Intake/INPUTs/Planteo-Analisis-Unificado-Antecedente-SAI-Service.md`, es el documento de mayor interés porque describe las prestaciones que el servicio debe cumplir y en que consiste.


4. **Stack a utilizar:**

  4.1. Servicio Web .NET 10, Blazor con páginas itenactive server ,entity framework.

  4.2. Librerias MudBlazor.

  4.3. Base de datos sqlite. 

  4.4. Arquitectura del servicio: monolotica (Front , API Rest, etc todos corriendo en un mismo servicio), front , api rest, backend en un solo servicio.

  4.5. Arquitectura de la solución: Leer `/DEV/SAI.Service.Core.Documentacion/PROMPTs/02-Ejecutar-Prompt-Integrador-Documento-Intake/INPUTs/Topologia-Proyecto-Solucion.md`.
 

5. **Datos de la solución:**

  **Nombre de la solución: SAI.Service.Core**

6. **Etapas de desarrollo relativas a partir de las fases de codificación**

  Los sprint relativos a la codificación deben asegurar puntos de validación al agente humano verificando estructuras, validaciones con entregables tangibles. Las primeras etapas deben consistir en:

  6.1. **Etapa 1**

    6.1.a. Crear el scaffolding de la solución y script bat para las tareas del run/build local.
    
    6.1.b. Las solución debe compilar y correr mediante los script but sitados.
    
    6.1.c. Luego, el orquestador debe solicitar validar visualmente la estructura de la solución por el agente humano.

  6.2. **Etapa 2**

    6.2.a. Crear el front, menú lateral, barra superior. 

    6.2.b. El servicio debe compilar correctamente y debe ser lanzado.
    
    6.2.c. Se orquestador debe solicitar validar visualmente en el navegador el panel de cotrol web para que se 
    cumple con el diseño definido en la maqueta dada en la etapa de espcificación UX-UI.

  6.3. **Etapa 3**

    6.3.a. Integrar sqlite, entidades necesarias para la autentificación y autorización.

    6.3.b. Idem Crear la primera intefaz de usuario que solicita usuario y contraseña para dar de alta el administrador. Luego redirecciona a la página principal

    6.3.c. Idem al punto 6.2.c. de la etapa 2.

  6.4. **Etapa 4**

    6.4.a. Integrar las interfaces para login, y para cambio de contraseña, las acciones desde la barra superior del admin como cerrar sesión, y cambio de contraseña.

    6.4.b. Idem al punto 6.2.c. de la etapa 2.


  6.4. Las Siguientes etapas se deben estructurar según todas los flujos de usuarios previstos:


    UF1["UF-1 · Alta del parque<br/>y puesta en marcha"]
    UF2["UF-2 · Configuración<br/>de políticas"]
    UF3["UF-3 · Monitoreo<br/>en vivo"]
    UF4["UF-4 · Históricos<br/>y gráficas"]
    UF5["UF-5 · Prueba de batería<br/>y salud"]
    UF6["UF-6 · Servicio técnico:<br/>recambio de batería"]
    UF7["UF-7 · Reparación o<br/>sustitución del SAI"]
    UF8["UF-8 · Ventana de<br/>mantenimiento"]
    UF9["UF-9 · Informe de período<br/>y comparación de marcas"]
    UF10["UF-10 · Ingesta<br/>automatizada"]


 En cada etapa implementar todo lo necesario y en cada una se debe verificar en el navegador que las pantallas funcionan. 

7. **Entorno de desarrollo**

  7.1. Entorno de ejecución destino:  el entorno final será docker en linux , pero esto queda fuera del alcance de este proyecto.

  7.2. Entorno de ejecución durante el desarrollo: seguir `/DEV/SAI.Service.Core.Documentacion/PROMPTs/02-Ejecutar-Prompt-Integrador-Documento-Intake/INPUTs/Entorno-Desarrollo.md`

8. **Jerarquí de usuarios**

  - Un solo usuario administrador.
  - El sistema cuando inicia debe pedir el nombre de usuario y contraseña para la administración.


# Objetivos

  Aplicar el `Framework SDD` sobre el contexto dado y el `Repositorio destino` .

---

# Solicitudes

  Aplicar los objetivos propuestos cumpliendo con estos puntos.

  1. Crear la jerarquía de carpetas y archivos indicadas por  `Framework SDD` en repositorio destino: `Repositorio destino`

  2. Construir el documento intake como índica:  `Framework SDD`  en `Repositorio destino`: `Template Intake a construir`. Utilizar los documentos e información dada en la sección `Contexto` para generar el nuevo documento intake a partir de lo que demande las reglas de los agentes para tal fin y el mismo template del documento intake.

  Los demas puntos que van desde la ejecución del prompt orquestador de `Framework SDD` en repositorio destino: `Repositorio destino` no aplican en este prompt.

---

# Restricciones

  - No realizar commit, push ni pull request.
  - No inventar información.
  - Toda afirmación deberá estar respaldada por evidencias verificables.
  - No introducir modificaciones en  `Framework SDD` 

