# Manual de Arquitectura e Implementación: Hiperautomatización del Proceso de Compras MAXI

## Preámbulo: La Visión del Experto en Hiperautomatización

Este documento detalla la arquitectura y la implementación de la solución de automatización para el proceso de compras de Supermercados MAXI. El objetivo trasciende la simple digitalización de un flujo de trabajo; representa una reingeniería estratégica diseñada para transformar un proceso manual y aislado en un activo empresarial orquestado, inteligente y resiliente.

Nuestra filosofía se basa en que los procesos no son silos, sino un ecosistema interconectado. Por ello, cada decisión de diseño, desde el modelo de proceso hasta la última regla de negocio, se ha tomado con el fin de alinear los objetivos estratégicos de la empresa (eficiencia, control de costos, agilidad) con la ejecución operativa, mitigando riesgos y generando una visibilidad de 360 grados sobre la operación.

---

## 1. Fase I: Arquitectura del Proceso "To-Be" (El Plano Maestro)

El primer paso fue rediseñar el flujo de trabajo para eliminar ineficiencias y puntos de fallo inherentes al proceso manual. El resultado es el siguiente modelo BPMN 2.0, que sirve como el plano ejecutable de nuestra solución.

### 1.1. Diagrama de Proceso Optimizado (BPMN 2.0)

![Diagrama BPMN del Proceso de Compras MAXI](https://i.imgur.com/83uX6gX.png)

### 1.2. Análisis Detallado de la Arquitectura del Proceso

El diagrama está estructurado para reflejar con precisión las responsabilidades y la lógica de negocio.

-   **Piscina (Pool) y Carriles (Lanes):** Se utiliza una única piscina (`Proceso de Compras MAXI`) para representar el alcance total del proceso bajo el control de la organización. Dentro de esta, se definen tres carriles que delimitan las responsabilidades funcionales:
    -   **Gestión de Compras:** Responsable del inicio y cierre del ciclo de vida de la solicitud.
    -   **Inventario:** Responsable exclusivo de la validación de existencias.
    -   **Adquisiciones:** Responsable de la gestión de la orden de compra y la selección de proveedores.

-   **Decisión Estratégica Clave - Hiperautomatización del Envío:** Se tomó la decisión estratégica de eliminar la tarea manual `Preparar y Enviar Orden a Proveedores`. En el modelo optimizado, la Tarea de Servicio `Generar Número de Orden de Compra` no solo obtiene el número, sino que actúa como un disparador para que el sistema pueda orquestar el envío automático de la orden a los proveedores. Este cambio es fundamental, ya que:
    -   **Mitiga Riesgos:** Elimina el error humano asociado al envío manual de correos.
    -   **Aumenta la Eficiencia:** Reduce el tiempo de ciclo al eliminar un cuello de botella manual.
    -   **Libera Recursos:** Permite al personal de Adquisiciones enfocarse en el análisis de propuestas, una tarea de mayor valor.

-   **Tipos de Tareas y Eventos Utilizados:**
    -   **Tareas de Usuario (User Tasks):** `Registrar Solicitud`, `Validar Existencia`, `Consolidar Propuesta`, `Validar Carpeta`. Son los puntos de interacción humana donde se requiere juicio y toma de decisiones.
    -   **Tarea de Servicio (Service Task):** `Generar Número de Orden de Compra`. Representa una integración automática con un sistema externo (el servicio web), eliminando la dependencia del Excel manual.
    -   **Tarea de Script (Script Task):** `Actualizar Estado a Cancelado`. Ejecuta una acción automática interna del proceso sin intervención humana.
    -   **Tarea de Envío (Send Task):** `Notificar...`. Se utiliza para modelar el envío automático de notificaciones.
    -   **Evento de Límite de Temporizador (Boundary Timer Event):** Adjunto a `Validar Existencia en Inventario`, hace cumplir el SLA de 3 días, cancelando automáticamente la solicitud si no se atiende a tiempo.
    -   **Evento Intermedio de Temporizador (Intermediate Timer Event):** `Esperar 3 días por Propuestas`. Pausa el proceso deliberadamente para crear la ventana de tiempo para la recepción de propuestas de proveedores.

---

## 2. Fase II: Arquitectura de Datos (La Fundación de la Inteligencia)

Un proceso es tan inteligente como los datos que lo sustentan. El siguiente modelo de datos fue diseñado no solo para almacenar información, sino para capturar la trazabilidad completa y permitir la generación de KPIs significativos.

### 2.1. Diagrama Entidad-Relación Final

![Diagrama Entidad-Relación de la solución](https://i.imgur.com/2Xgq8lP.png)

### 2.2. Desglose Detallado de las Entidades

Bizagi organiza las entidades en carpetas según su propósito. En nuestro proyecto, la estructura es la siguiente:

![Estructura de Entidades en Bizagi Studio](https://i.imgur.com/g8Vw10N.png)

-   **Entidades Master:** Almacenan los datos transaccionales de cada caso.
-   **Entidades Parameter:** Almacenan los catálogos o listas de opciones.

A continuación, el detalle de cada entidad y sus atributos:

| Entidad | Atributo | Tipo de Dato (en Bizagi) | Descripción Detallada |
| :--- | :--- | :--- | :--- |
| **ProcesodeComprasMAXI** (Master) | Fecha Solicitud | `DateTime` | Almacena la fecha y hora exactas de creación de la solicitud. Se llena automáticamente. |
| | Monto Total Aprobado | `Currency` | Almacena el valor final de la compra, registrado por el Supervisor de Adquisiciones. |
| | Nombre Gestor Compras | `String` | Nombre del empleado que inicia la solicitud. Se llena automáticamente. |
| | Numero Orden Unico | `String` | Almacena el número único devuelto por el servicio web. |
| | Resultado Final | `String` | Guarda el estado final del caso (Ej: "Compra Exitosa", "Negada por Stock"). |
| | StockSuficiente | `Boolean` | Almacena la decisión (Sí/No) del área de Inventario. |
| | fkProveedorSeleccionado | Related Entity (`Cat_Proveedor`) | Relación que vincula la solicitud con el proveedor ganador. |
| | fkSucursal | Related Entity (`Cat_Sucursal`) | Relación que vincula la solicitud con la sucursal que la originó. |
| | DetalleProductos | Collection (`DetalleProducto`) | Relación de Colección (Uno a Muchos) que contiene la lista de productos. |
| **DetalleProducto** (Master) | CantidadRequerida | `Integer` | Almacena el número de unidades solicitadas para un producto específico. |
| | descripcion | `String` | Campo de texto para notas o especificaciones adicionales por producto. |
| | fkProducto | Related Entity (`Cat_Producto`) | Relación que vincula esta línea de detalle con un producto del catálogo. |
| | ProcesodeComprasMAXI | Related Entity | Relación inversa creada automáticamente por Bizagi para la colección. |
| **Cat_Sucursal** (Parameter) | NombreSucursal | `String` | Nombre de la sucursal (Ej: "Sucursal Centro"). |
| **Cat_Proveedor** (Parameter) | NombreProveedor | `String` | Nombre del proveedor (Ej: "Proveedor A"). |
| **Cat_Producto** (Parameter) | NombreProducto | `String` | Nombre del producto (Ej: "Lápiz HB #2"). |

---

## 3. Fase III: Diseño de la Experiencia de Usuario (Interfaces Inteligentes)

Adoptamos una estrategia de reutilización de formularios para garantizar consistencia, acelerar el desarrollo y mitigar errores. Construimos un único "Formulario Maestro" y lo adaptamos para cada tarea.

### 3.1. La Estrategia del "Formulario Maestro"

1.  **Creación:** Se diseña un formulario completo en la primera tarea (`Registrar Solicitud de Compra`).
2.  **Reutilización:** Para las tareas subsecuentes, en lugar de crear un formulario nuevo, se utiliza la función `Copy from` en la cinta de opciones del Diseñador de Formularios para clonar el maestro.
3.  **Adaptación:** En cada formulario clonado, se ajustan las propiedades de los controles (principalmente la propiedad `Editable`) para mostrar u ocultar campos, o hacerlos de solo lectura, según el contexto de la tarea.

### 3.2. Diseño Detallado de los Formularios

-   **Formulario 1: Registrar Solicitud de Compra**
    -   **Controles Editables:** `Nombre Gestor Compras`, `Fecha Solicitud`, `fkSucursal`, y la tabla `DetalleProductos`.
    -   **Funcionalidad Clave:** La colección `DetalleProductos` se arrastra al formulario, creando un control de Tabla (Table). Se configuran sus columnas (`fkProducto`, `CantidadRequerida`, `descripcion`) y se habilita la propiedad `Inline add` para una experiencia de usuario fluida al añadir productos.

-   **Formulario 2: Validar Existencia en Inventario**
    -   **Reutilización:** Se clona el formulario maestro.
    -   **Controles de Solo Lectura:** Todos los campos y la tabla del formulario maestro se configuran con la propiedad `Editable = No`.
    -   **Control Editable:** Se añade el atributo `StockSuficiente` (de tipo Booleano), que se renderiza como un control de Radio Button (Sí/No). Este es el único punto de interacción.

-   **Formulario 3: Consolidar y Seleccionar Mejor Propuesta**
    -   **Reutilización:** Se clona el formulario maestro.
    -   **Controles de Solo Lectura:** La información general y la tabla de productos se configuran como no editables.
    -   **Controles Editables:** Se añade un nuevo grupo "Decisión de Adquisición" que contiene los campos editables `fkProveedorSeleccionado` y `MontoTotalAprobado`.

-   **Formulario 4: Validar Carpeta de Compra Completa**
    -   **Reutilización:** Se clona el formulario maestro.
    -   **Funcionalidad:** Es un formulario de resumen total. Se añaden todos los campos relevantes del proceso (`StockSuficiente`, `fkProveedorSeleccionado`, etc.).
    -   **Controles de Solo Lectura:** Todos los controles del formulario se configuran con `Editable = No`, creando una vista final segura para la aprobación.

---

## 4. Fase IV: Lógica de Negocio e Integraciones (El Cerebro del Proceso)

Esta fase dota al proceso de la inteligencia para ejecutar tareas, tomar decisiones y comunicarse con sistemas externos.

### 4.1. Automatización de Tareas Administrativas

Se utilizan Expresiones en los eventos `On Enter` y `On Exit` de las actividades.

Para insertar una expresión: En el `Expression Manager`, se hace clic derecho en la línea de flujo y se selecciona `Expression`. Se hace doble clic en el nuevo bloque para abrir el editor de código.

![Editor de Expresiones de Bizagi](https://i.imgur.com/00lqXgX.png)

**Reglas Implementadas:**

-   **Actividad:** `Registrar Solicitud de Compra`
    -   **Evento:** `On Enter`
    -   **Código:**
        ```javascript
        // Asigna la fecha y hora actual del sistema
        <ProcesodeComprasMAXI.FechaSolicitud> = DateTime.Now;
        // Asigna el nombre completo del usuario que creó el caso
        <ProcesodeComprasMAXI.NombreGestorCompras> = Me.Case.Creator.FullName;
        ```

-   **Actividad:** `Notificar Rechazo por Existencia`
    -   **Evento:** `On Enter`
    -   **Código:** `<ProcesodeComprasMAXI.ResultadoFinal> = "Negada por Stock";`

-   **Actividad:** `Notificar Cancelación por Vencimiento`
    -   **Evento:** `On Enter`
    -   **Código:** `<ProcesodeComprasMAXI.ResultadoFinal> = "Vencida por Vencimiento";`

-   **Actividad:** `Validar Carpeta de Compra Completa`
    -   **Evento:** `On Exit`
    -   **Código:** `<ProcesodeComprasMAXI.ResultadoFinal> = "Compra Exitosa";`

### 4.2. Integración del Servicio Web SOAP

Este es un pilar de la hiperautomatización, reemplazando el Excel manual.

1.  **Localización del Asistente:** Se accede al asistente correcto a través del menú superior de Bizagi Studio: `Tools -> Web Service Connector`.
2.  **Configuración de la Conexión:**
    -   En el asistente, se selecciona el tipo **SOAP**.
    -   Se ingresa la URL del WSDL: `https://legacy.bizagi.com/OfficeSupplyWS/OfficeService.asmx?WSDL`
    -   Se hace clic en **Go**. Bizagi analiza el servicio y muestra los métodos disponibles.
        ![Asistente de Conector de Servicio Web en Bizagi](https://i.imgur.com/2Y4406B.png)
    -   Se selecciona el método `PurchaseOrderNumber` y se asigna un nombre descriptivo a la interfaz, como `ObtenerNumeroOrdenCompra`.
    -   Se finaliza el asistente para guardar la conexión.
3.  **Invocación desde el Proceso:**
    -   En el paso de "Business Rules", se selecciona la Tarea de Servicio `Generar Número de Orden de Compra`.
    -   Se configura la invocación de la interfaz, seleccionando el conector recién creado.
    -   En la pestaña `Output Mapping`, se arrastra el campo de respuesta del servicio (`ResPurchaseOrder`) y se suelta sobre el campo de nuestro modelo de datos `NumeroOrdenUnico` para crear el vínculo.

---

## 5. Fase V: Configuración Final y Ejecución

Para que la aplicación sea funcional y demostrable, se realizaron los siguientes pasos finales:

-   **Poblar Catálogos:** Se identificó que las listas desplegables en los formularios aparecían vacías. La solución fue:
    1.  Ejecutar el proyecto con el botón **Run**.
    2.  En el Portal de Trabajo que se abre en el navegador, navegar a **Admin -> Entities**.
    3.  Seleccionar cada entidad paramétrica (`Cat_Producto`, `Cat_Sucursal`, `Cat_Proveedor`) y usar el botón de añadir (`+`) para ingresar los datos de ejemplo necesarios para las pruebas (contenidos en los archivos `.csv` de la carpeta `/data/catalogos`).

Este documento representa el estado completo y detallado del proyecto de hiperautomatización. Cada componente ha sido diseñado con un propósito claro, contribuyendo a una solución orquestada que entrega un valor estratégico medible a Supermercados MAXI.
