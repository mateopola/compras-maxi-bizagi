# Compras MAXI — MVP / Prototipo funcional (Bizagi 11.2.5)

> **Propósito del MVP (42 horas):** Automatizar y **orquestar en un BPM** las tareas manuales del proceso de compras para **conectar áreas**, **reducir tiempos operativos**, **disminuir costos**, **evitar errores humanos** y mantener el **stock actualizado** en la base de datos.  
> Este MVP libera tiempo al equipo para enfocarse en **tareas de mayor valor** (análisis y decisión) en lugar de actividades repetitivas.

---

## 0) TL;DR (Resumen ejecutivo)

- **Qué entregamos:** Un **proceso ejecutable en Bizagi** (BPMN 2.0) con formularios reutilizables, catálogos parametrizados, lógica de negocio esencial, **SLA** con temporizador, **generación automática de número de orden (radicado) vía SOAP**, y base para notificaciones y reportes.  
- **Qué resuelve:** Disminuye reprocesos, acelera el flujo, traza decisiones, reduce errores y **visibiliza el estado** de cada solicitud.  
- **Qué no es todavía:** No integra ERP / correo entrante / dashboards; se dejan **diseñados en el roadmap**.  
- **Por qué MVP:** El ejercicio exigía 42 horas. Priorizamos valor y **time-to-demo** con base sólida para crecer.

---

## 1) Alcance y principios de diseño

### 1.1 Objetivos del MVP
- **Orquestación** del proceso end-to-end (Compras → Inventario → Adquisiciones → Cierre).
- **SLA**: caducidad a los **3 días** en Inventario (parametrizable para demo).
- **Decisiones críticas** modeladas (stock sí/no, carpeta completa).
- **Radicado único** de orden de compra vía **Web Service SOAP**.
- **Parametrización de catálogos** (Productos, Proveedores, Sucursales).
- **Trazabilidad** de estados y resultados (éxito, negada por stock, vencida).

### 1.2 Beneficios esperados
- **Menos tiempo de ciclo** y **menos costos operativos**.
- **Menos errores humanos** (envíos, numeración, duplicidad).
- **Stock consistente** (base de datos alimentada y preparada para ERP).
- **Visibilidad** del estado y decisiones (quién, cuándo, por qué).

### 1.3 Alcance fuera del MVP (planificado)
- Integración **ERP** (inventarios y maestros) + **webhooks** para disparar el proceso.
- **Monitoreo de correo** para iniciar casos desde e-mails.
- **RPA + IA** para diligenciamiento inteligente de formularios.
- **Dashboards/Indicadores** (Power BI / Bizagi) y **Process Mining**.
- Endurecimiento de **seguridad**, **auditoría**, **idempotencia** y **reintentos**.

---

## 2) Arquitectura funcional (BPMN 2.0)

**Lanes:** Gestión de Compras / Inventario / Adquisiciones.  
**Rutas:**
1. **SÍ stock** → Rechazo y notificación (no se compra).
2. **NO stock** → Generar N° OC (SOAP) → Consolidar propuesta → Validar carpeta → Fin.
3. **Vencimiento** (SLA 3 días en Inventario) → Cancelación y notificación.

**Eventos/Tareas clave:**
- **Boundary Timer** (Inventario): 3 días (para demo se puede configurar a 2 minutos).
- **Service Task**: generación de **Número de Orden Único (radicado)** vía SOAP.
- **User Tasks**: registrar solicitud, validar existencia, consolidar, validar carpeta.
- **Send/Notification**: avisos por e-mail (definidos para habilitar).

> El diagrama y capturas se encuentran en `docs/`.

---

## 3) Modelo de datos y catálogos

### 3.1 Entidades
- **Proceso (Master):** `ProcesodeComprasMAXI`  
  Atributos: `FechaSolicitud`, `NombreGestorCompras`, `fkSucursal`, `DetalleProductos (colección)`, `StockSuficiente`, `NumeroOrdenUnico`, `fkProveedorSeleccionado`, `MontoTotalAprobado`, `ResultadoFinal`.
- **Detalle (Master):** `DetalleProducto`  
  Atributos: `fkProducto`, `CantidadRequerida`, `Descripcion`.
- **Catálogos (Parameter):** `Cat_Producto`, `Cat_Proveedor`, `Cat_Sucursal`.

### 3.2 Catálogos para demo
- Se proveen archivos **oficiales de Bizagi** con hojas `Data / Configuration / List` para facilitar la carga.  
- **Recomendación:** dejar **solo 5 productos habilitados** (`Deshabilitada = FALSE`) y filtrar en formularios por `Deshabilitada = false` para performance y claridad.

---

## 4) Interfaces (formularios)

**Patrón “Formulario Maestro”**: un formulario base clonado y ajustado por etapa (editable/solo lectura).

- **Registrar Solicitud:** datos generales + tabla `DetalleProductos` (agregar filas inline).
- **Validar Existencia en Inventario:** vista de solo lectura + `StockSuficiente` (Sí/No).
- **Consolidar y Seleccionar Mejor Propuesta:** solo lectura + `fkProveedorSeleccionado`, `MontoTotalAprobado`.
- **Validar Carpeta de Compra Completa:** resumen de todo + decisión final (Sí/No).

**Sugerencias UX del MVP:**
- Columna **Producto** como control **Search** con filtro `Deshabilitada = false`.
- Validaciones mínimas (ver sección 6).

---

## 5) Integración: SOAP para número de orden (radicado)

- **WSDL:** `https://legacy.bizagi.com/OfficeSupplyWS/OfficeService.asmx?WSDL`  
- **Binding:** `OfficeServiceSoap12`  
- **Operación:** `PurchaseOrderNumber`  
- **Output mapping:** respuesta → `ProcesodeComprasMAXI.NumeroOrdenUnico`.

**Manejo de errores (MVP):**
- Si la invocación falla, se muestra mensaje y se **permite reintento** en la misma tarea.
- Validación: no avanzar si `NumeroOrdenUnico` está vacío.

---

## 6) Reglas de negocio y validaciones (MVP)

- **Registrar (On Enter):** set `FechaSolicitud = Now()`, `NombreGestorCompras = Me.Case.Creator.FullName`.
- **Validar rutas por stock:** 
  - **SÍ** → Rechazo + notificación.
  - **NO** → Adquisiciones.
- **Vencimiento (On Enter de notificación):** set `ResultadoFinal = "Vencida por Vencimiento"`.
- **Validar Carpeta (On Exit):** set `ResultadoFinal = "Compra Exitosa"` cuando decisión final = Sí.

**Validaciones de datos:**
- **Registrar:** al menos **1** ítem en `DetalleProductos`; `CantidadRequerida > 0`.
- **Consolidar:** `fkProveedorSeleccionado` requerido; `MontoTotalAprobado > 0`.
- **Visibilidad:** solo lectura en tareas de revisión; edición solo donde corresponde.

---

## 7) Notificaciones (para habilitar)

| Evento | Destinatarios | Resumen |
|---|---|---|
| Rechazo por stock | Solicitante | Id del caso, sucursal, resumen productos. |
| Vencimiento Inventario | Solicitante + Sup. Compras | Caso cancelado por SLA (3 días). |
| OC generada | Adquisiciones | `NumeroOrdenUnico`, sucursal, total estimado. |

> Requiere SMTP configurado (ver sección 10.2).

---

## 8) Reportes y trazabilidad (MVP + siguientes pasos)

**MVP:** Búsquedas guardadas en el portal:
- Compras por proveedor (últimos 30 días).
- Solicitudes negadas por stock.
- Solicitudes vencidas.

**Evolución:**
- **Dashboards** (Power BI / Bizagi): SLA, rechazo por stock, volumen por sucursal/proveedor, ahorro estimado.
- **Process Mining:** análisis de variantes, cuellos de botella, tiempos de espera.

---

## 9) Seguridad y gobierno

- **Roles por lane:** Compras, Inventario, Adquisiciones + Supervisores.
- **Permisos:** cada rol **ve/edita** únicamente su etapa; vistas de solo lectura para validaciones.
- **Trazabilidad:** estados (`ResultadoFinal`) y `NumeroOrdenUnico` para auditoría básica.
- **Futuro:** auditoría fina de cambios, logs centralizados, segregación de funciones.

---

## 10) Instalación y configuración

### 10.1 Importar solución
1. En Bizagi Studio **11.2.5**, importar **BEX** y **BDEX** desde `export/`.  
2. Publicar/ejecutar en ambiente de desarrollo.  
3. Validar roles, asignaciones y licenciamiento.

### 10.2 Configurar e-mail (SMTP) — opcional
- **Configuration → Environment → Development → Mail Server**: host, puerto (587/465), TLS/SSL, usuario/clave, From.  
- Probar con una **plantilla de notificación** (o con un PowerShell de smoke test).

### 10.3 Cargar catálogos
- En **Work Portal → Admin → Entities → Manage**:
  - Cargar **Cat_Producto**, **Cat_Proveedor**, **Cat_Sucursal** usando los **Excel oficiales** adjuntos (hojas `Data/Configuration/List`).  
  - Para demo, habilitar **solo 5** productos y filtrar en formulario por `Deshabilitada = false`.

### 10.4 Integración SOAP
- Verificar WSDL y binding en el **Web Service Connector**.  
- Confirmar **Output mapping** a `NumeroOrdenUnico`.  
- Probar la **tarea de servicio**.

---

## 11) Cómo probar (E2E)

### 11.1 Escenarios
1. **Happy path (NO stock)**: Registrar → Inventario (NO) → Generar N° OC → Consolidar → Validar carpeta = Sí → **Compra Exitosa**.  
2. **SÍ stock**: Registrar → Inventario (SÍ) → **Negada por Stock** (+ notificación).  
3. **Vencimiento**: Registrar → no atender Inventario → a los **3 días** (o **2 min** en demo) → **Vencida por Vencimiento** (+ notificación).

### 11.2 Tips de demo
- Timer del **boundary** a 2 minutos (solo demo).  
- Control **Search** para Producto + filtro `Deshabilitada = false`.

---

## 12) Solución de problemas (FAQ)

**La lista de productos se queda cargando y expira sesión**  
- Causa: demasiados registros habilitados en `Cat_Producto` y sin filtro.  
- Solución: poner **`Deshabilitada = TRUE`** a los que no uses (se adjunta Excel de ejemplo) y en el control aplicar filtro `Deshabilitada = false` (usar **Search**).

**El SOAP no devuelve número**  
- Revisar conectividad a WSDL y binding **OfficeServiceSoap12**.  
- Ver traza del **faultstring** y permitir **reintento** en la tarea de servicio.  
- No avanzar si `NumeroOrdenUnico` vacío.

**Correos no llegan**  
- Validar **SMTP** (puerto, TLS/SSL, credenciales).  
- Probar envío desde un script PowerShell.  
- Revisar bandeja de salida / logs del servidor de correo.

---

## 13) Roadmap (próximas iteraciones)

- **ERP + Webhooks**: inventario en línea y disparo del proceso por evento.  
- **Correo entrante**: iniciar casos leyendo una casilla de compras.  
- **RPA + IA**: captura inteligente en formularios (reglas + base de conocimiento).  
- **Dashboards**: SLA, costo por OC, tasa de rechazo, proveedores top/bajo desempeño.  
- **Process Mining**: mejora continua apoyada en datos reales.  
- **Resiliencia**: colas, reintentos, circuit breaker, trazabilidad avanzada.

---

## 14) Entregables del ejercicio

- **Documentación del flujo** (`docs/`): BPMN + explicación.  
- **BEX del proyecto** (`export/`).  
- **BDEX del proyecto** (`export/`).  
- **README** (este documento).  
- **ZIP único** con todo el repo.  
- **Repositorio público en GitHub** (incluir URL en comentarios del ejercicio).

---

## 15) Estructura del repositorio

├─ README.md
├─ .gitignore
├─ docs/ # BPMN, manual de arquitectura/implementación
├─ export/ # BEX, BDEX
├─ imgs/ # Evidencias (pantallas del caso, notificaciones, BPMN)
├─ scripts/ # Notas/reglas auxiliares si aplica
└─ data/
└─ catalogos/