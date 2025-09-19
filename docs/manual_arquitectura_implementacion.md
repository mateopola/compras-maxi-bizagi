# Manual de Arquitectura e Implementación: Compras MAXI (Bizagi 11.2.5)

> **Propósito:** Este documento detalla la arquitectura técnica, el diseño de implementación y la configuración de la solución de automatización para el proceso de compras de Supermercados MAXI. Sirve como una guía exhaustiva para desarrolladores, administradores de Bizagi y arquitectos de soluciones.

---

## 1. Resumen y Alcance del MVP

### 1.1. Propósito del MVP (42 horas)
Automatizar y **orquestar en un BPM** las tareas manuales del proceso de compras para **conectar áreas**, **reducir tiempos operativos**, **disminuir costos**, **evitar errores humanos** y mantener el **stock actualizado** en la base de datos. Este MVP libera tiempo al equipo para enfocarse en **tareas de mayor valor** (análisis y decisión) en lugar de actividades repetitivas.

### 1.2. Objetivos del MVP
- **Orquestación** del proceso end-to-end (Compras → Inventario → Adquisiciones → Cierre).
- **SLA**: caducidad a los **3 días** en Inventario (parametrizable para demo).
- **Decisiones críticas** modeladas (stock sí/no, carpeta completa).
- **Radicado único** de orden de compra vía **Web Service SOAP**.
- **Parametrización de catálogos** (Productos, Proveedores, Sucursales).
- **Trazabilidad** de estados y resultados (éxito, negada por stock, vencida).

### 1.3. Alcance fuera del MVP (Roadmap)
- Integración **ERP** (inventarios y maestros) + **webhooks** para disparar el proceso.
- **Monitoreo de correo** para iniciar casos desde e-mails.
- **RPA + IA** para diligenciamiento inteligente de formularios.
- **Dashboards/Indicadores** (Power BI / Bizagi) y **Process Mining**.
- Endurecimiento de **seguridad**, **auditoría**, **idempotencia** y **reintentos**.

---

## 2. Arquitectura Funcional (BPMN 2.0)

El proceso fue rediseñado para eliminar ineficiencias y puntos de fallo. El siguiente modelo BPMN 2.0 es el plano ejecutable de la solución.

*(Nota: El diagrama BPMN original no está disponible. Se recomienda generarlo desde el BEX del proyecto en `export/`)*

### 2.1. Análisis Detallado de la Arquitectura del Proceso
El diagrama está estructurado para reflejar con precisión las responsabilidades y la lógica de negocio.

- **Piscina (Pool) y Carriles (Lanes):** Se utiliza una única piscina (`Proceso de Compras MAXI`) con tres carriles que delimitan las responsabilidades funcionales:
    - **Gestión de Compras:** Responsable del inicio y cierre del ciclo de vida de la solicitud.
    - **Inventario:** Responsable exclusivo de la validación de existencias.
    - **Adquisiciones:** Responsable de la gestión de la orden de compra y la selección de proveedores.

- **Rutas de Proceso:**
    1. **SÍ stock** → Rechazo y notificación (no se compra).
    2. **NO stock** → Generar N° OC (SOAP) → Consolidar propuesta → Validar carpeta → Fin.
    3. **Vencimiento** (SLA 3 días en Inventario) → Cancelación y notificación.

- **Tipos de Tareas y Eventos Utilizados:**
    - **Tareas de Usuario (User Tasks):** `Registrar Solicitud`, `Validar Existencia`, `Consolidar Propuesta`, `Validar Carpeta`. Puntos de interacción humana.
    - **Tarea de Servicio (Service Task):** `Generar Número de Orden de Compra`. Integración automática con el servicio web SOAP.
    - **Tarea de Script (Script Task):** `Actualizar Estado a Cancelado`. Ejecuta una acción automática interna.
    - **Tarea de Envío (Send Task):** `Notificar...`. Modela el envío automático de notificaciones.
    - **Evento de Límite de Temporizador (Boundary Timer Event):** Adjunto a `Validar Existencia en Inventario`, hace cumplir el SLA de 3 días.
    - **Evento Intermedio de Temporizador (Intermediate Timer Event):** `Esperar 3 días por Propuestas`. Pausa deliberada del proceso.

---

## 3. Modelo de Datos (Arquitectura de Datos)

Un proceso es tan inteligente como los datos que lo sustentan. El modelo fue diseñado para capturar la trazabilidad completa y permitir la generación de KPIs.

*(Nota: El Diagrama Entidad-Relación original no está disponible. Se puede visualizar en Bizagi Studio.)*

### 3.1. Desglose Detallado de las Entidades

| Entidad | Atributo | Tipo de Dato (Bizagi) | Descripción Detallada |
| :--- | :--- | :--- | :--- |
| **ProcesodeComprasMAXI** (Master) | Fecha Solicitud | `DateTime` | Fecha y hora de creación de la solicitud (automático). |
| | Monto Total Aprobado | `Currency` | Valor final de la compra. |
| | Nombre Gestor Compras | `String` | Nombre del empleado que inicia el caso (automático). |
| | Numero Orden Unico | `String` | Número único devuelto por el servicio web SOAP. |
| | Resultado Final | `String` | Estado final del caso (Ej: "Compra Exitosa", "Negada por Stock"). |
| | StockSuficiente | `Boolean` | Decisión (Sí/No) del área de Inventario. |
| | fkProveedorSeleccionado | Related Entity (`Cat_Proveedor`) | Vínculo con el proveedor ganador. |
| | fkSucursal | Related Entity (`Cat_Sucursal`) | Vínculo con la sucursal de origen. |
| | DetalleProductos | Collection (`DetalleProducto`) | Colección (Uno a Muchos) con la lista de productos. |
| **DetalleProducto** (Master) | CantidadRequerida | `Integer` | Unidades solicitadas para un producto. |
| | descripcion | `String` | Notas o especificaciones adicionales por producto. |
| | fkProducto | Related Entity (`Cat_Producto`) | Vínculo con un producto del catálogo. |
| **Cat_Sucursal** (Parameter) | NombreSucursal | `String` | Nombre de la sucursal. |
| **Cat_Proveedor** (Parameter) | NombreProveedor | `String` | Nombre del proveedor. |
| **Cat_Producto** (Parameter) | NombreProducto | `String` | Nombre del producto. |

---

## 4. Interfaces de Usuario (Formularios)

Se adopta una estrategia de **"Formulario Maestro"** para garantizar consistencia y acelerar el desarrollo. Un formulario base se clona y se adapta para cada tarea, ajustando la propiedad `Editable` de los controles.

- **Formulario 1: Registrar Solicitud de Compra**
  - **Editables:** `Nombre Gestor Compras`, `Fecha Solicitud`, `fkSucursal`, y la tabla `DetalleProductos` (con `Inline add` habilitado).
- **Formulario 2: Validar Existencia en Inventario**
  - **Solo Lectura:** Información general y tabla de productos.
  - **Editable:** `StockSuficiente` (Radio Button Sí/No).
- **Formulario 3: Consolidar y Seleccionar Mejor Propuesta**
  - **Solo Lectura:** Información general y tabla de productos.
  - **Editables:** `fkProveedorSeleccionado`, `MontoTotalAprobado`.
- **Formulario 4: Validar Carpeta de Compra Completa**
  - **Solo Lectura:** Todos los controles, a modo de resumen final.

**Sugerencia UX:** Usar un control **Search** para la columna `Producto` en la tabla, con un filtro `Deshabilitada = false` para mejorar el rendimiento y la claridad.

---

## 5. Lógica de Negocio e Integraciones

### 5.1. Reglas de Negocio y Validaciones (MVP)
Se utilizan Expresiones en los eventos `On Enter` y `On Exit` de las actividades.

- **Registrar (On Enter):**
  ```javascript
  <ProcesodeComprasMAXI.FechaSolicitud> = DateTime.Now;
  <ProcesodeComprasMAXI.NombreGestorCompras> = Me.Case.Creator.FullName;
  ```
- **Validar rutas por stock:**
  - **SÍ** → Ruta hacia `Notificar Rechazo por Existencia`.
  - **NO** → Ruta hacia `Adquisiciones`.
- **Notificar Rechazo (On Enter):**
  ```javascript
  <ProcesodeComprasMAXI.ResultadoFinal> = "Negada por Stock";
  ```
- **Notificar Vencimiento (On Enter):**
  ```javascript
  <ProcesodeComprasMAXI.ResultadoFinal> = "Vencida por Vencimiento";
  ```
- **Validar Carpeta (On Exit):**
  - Si la decisión es "Sí", se establece el resultado final.
  ```javascript
  <ProcesodeComprasMAXI.ResultadoFinal> = "Compra Exitosa";
  ```

**Validaciones de Datos:**
- **Registrar:** Al menos **1** ítem en `DetalleProductos`; `CantidadRequerida > 0`.
- **Consolidar:** `fkProveedorSeleccionado` requerido; `MontoTotalAprobado > 0`.

### 5.2. Integración: SOAP para Número de Orden
- **WSDL:** `https://legacy.bizagi.com/OfficeSupplyWS/OfficeService.asmx?WSDL`
- **Binding:** `OfficeServiceSoap12`
- **Operación:** `PurchaseOrderNumber`
- **Implementación:**
  1. Usar el **Web Service Connector** de Bizagi (`Tools -> Web Service Connector`).
  2. Configurar la conexión SOAP con la URL del WSDL.
  3. Seleccionar la operación `PurchaseOrderNumber`.
  4. En la Tarea de Servicio `Generar Número de Orden de Compra`, invocar el conector.
  5. En `Output Mapping`, mapear la respuesta del servicio al atributo `ProcesodeComprasMAXI.NumeroOrdenUnico`.
- **Manejo de errores (MVP):** Si la invocación falla, se muestra un mensaje y se permite el reintento en la misma tarea. No se debe avanzar si `NumeroOrdenUnico` está vacío.

---

## 6. Instalación y Configuración

### 6.1. Importar Solución
1. En Bizagi Studio **11.2.5**, importar **BEX** y **BDEX** desde la carpeta `export/`.
2. Publicar/ejecutar en el ambiente de desarrollo.
3. Validar roles, asignaciones y licenciamiento.

### 6.2. Cargar Catálogos
1. En el **Work Portal**, navegar a **Admin → Entities → Manage Entities**.
2. Seleccionar `Cat_Producto`, `Cat_Proveedor` y `Cat_Sucursal`.
3. Usar la opción de importación de Excel, utilizando los archivos CSV de la carpeta `data/catalogos` como guía para crear los archivos de carga masiva con las hojas `Data`, `Configuration` y `List`.
4. **Recomendación para Demo:** Habilitar solo 5 productos (`Deshabilitada = FALSE`) y filtrar en el formulario por `Deshabilitada = false`.

### 6.3. Configurar E-mail (SMTP) — Opcional
- **Ubicación:** `Configuration → Environment → Development → Mail Server`.
- **Parámetros:** Host, puerto (587/465), TLS/SSL, usuario/clave, y dirección "From".
- **Validación:** Probar con una plantilla de notificación.

---

## 7. Cómo Probar (Escenarios E2E)

1. **Happy Path (NO stock):** Registrar → Inventario (NO) → Generar N° OC → Consolidar → Validar carpeta = Sí → **Compra Exitosa**.
2. **Rechazo por Stock (SÍ stock):** Registrar → Inventario (SÍ) → **Negada por Stock** (+ notificación).
3. **Vencimiento de Tarea:** Registrar → No atender la tarea de Inventario → A los 3 días (o 2 min en demo) → **Vencida por Vencimiento** (+ notificación).

---

## 8. Solución de Problemas (FAQ)

- **La lista de productos se queda cargando y expira sesión:**
  - **Causa:** Demasiados registros habilitados en `Cat_Producto` y sin filtro en el formulario.
  - **Solución:** Poner `Deshabilitada = TRUE` a los productos que no se usan y aplicar un filtro `Deshabilitada = false` en el control del formulario (preferiblemente un control **Search**).
- **El SOAP no devuelve número de orden:**
  - **Solución:** Revisar la conectividad al WSDL y que el binding sea `OfficeServiceSoap12`. Verificar la traza del error (`faultstring`) y asegurar que la tarea de servicio permita reintentos.
- **Los correos de notificación no llegan:**
  - **Solución:** Validar la configuración del servidor SMTP (puerto, TLS/SSL, credenciales). Probar el envío con un script externo (ej. PowerShell) para aislar el problema.