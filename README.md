# Compras MAXI – Bizagi 11.2.5: Ejercicio de Hiperautomatización

## 1. Descripción del Proyecto

Este repositorio contiene los artefactos para un ejercicio práctico de hiperautomatización del proceso de compras "MAXI", implementado en **Bizagi 11.2.5**. El objetivo es transformar el proceso manual de solicitud de compras de Supermercados MAXI en un flujo de trabajo automatizado, inteligente y resiliente.

La solución optimiza la colaboración entre las áreas de Compras, Inventario y Adquisiciones, aplicando reglas de negocio, SLAs (Acuerdos de Nivel de Servicio) y la integración con servicios externos para eliminar cuellos de botella y tareas manuales.

## 2. Arquitectura de la Solución

La solución se basa en tres pilares fundamentales, detallados en el manual de implementación:

-   **Proceso de Negocio (BPMN):** Un flujo de proceso optimizado que orquesta las tareas humanas y automáticas, gestionando las aprobaciones, temporizadores para SLAs y notificaciones.
-   **Modelo de Datos:** Una estructura de datos relacional que garantiza la integridad y trazabilidad de la información, permitiendo la generación de reportes y análisis.
-   **Integraciones y Reglas:** Automatización de tareas mediante reglas de negocio y la integración con un servicio web SOAP para la generación de números de orden de compra únicos, eliminando el uso de controles manuales en Excel.

> **Documentación Completa**
> Para un desglose técnico y funcional detallado, incluyendo diagramas BPMN, el modelo de datos, diseño de formularios y la configuración de las integraciones, consulte el [**Manual de Arquitectura e Implementación**](./docs/manual_arquitectura_implementacion.md).

## 3. Estructura del Repositorio

-   **/docs**: Contiene la documentación funcional y técnica del proceso.
    -   `manual_arquitectura_implementacion.md`: El documento principal que detalla toda la solución.
-   **/export**: Destinado a almacenar los archivos de exportación de Bizagi (`.bex` para el modelo de proceso y `.bdex` para los datos parametrizados). **Nota:** Estos archivos deben ser generados desde Bizagi Studio.
-   **/imgs**: Guarda imágenes, logos o capturas de pantalla relevantes para el proyecto.
-   **/scripts**: Incluye scripts de apoyo (como los de inicialización de este repositorio).
-   **/data/catalogos**: Contiene los datos maestros en formato CSV para ser cargados en las entidades de Bizagi.

## 4. Guía de Implementación y Configuración

Siga estos pasos para desplegar y ejecutar el proyecto en su entorno local de Bizagi.

**Prerrequisitos:**
*   Bizagi Studio v11.2.5 o compatible.
*   Git.

**Pasos:**
1.  **Clonar el Repositorio:**
    ```bash
    git clone <URL-del-repositorio>
    cd compras-maxi-bizagi
    ```
2.  **Generar y Ubicar Artefactos Bizagi:**
    *   Dentro de Bizagi Studio, construya el proceso siguiendo el [manual de arquitectura](./docs/manual_arquitectura_implementacion.md).
    *   Exporte el modelo de proceso y los datos parametrizados.
    *   Guarde los archivos resultantes (`.bex` y `.bdex`) en la carpeta `/export` de este repositorio.
3.  **Importar el Proceso en un Nuevo Entorno (si aplica):**
    *   En Bizagi Studio, importe el archivo `.bex` para desplegar el modelo del proceso.
4.  **Cargar Datos Maestros:**
    *   Ejecute el proyecto (`Run`).
    *   En el Portal de Trabajo, vaya a **Admin → Entidades**.
    *   Cargue los archivos `.csv` de la carpeta `/data/catalogos` en las entidades paramétricas correspondientes (`Cat_Producto`, `Cat_Proveedor`, `Cat_Sucursal`).
5.  **Verificar Configuraciones:**
    *   Asegúrese de que la integración con el servicio web SOAP esté configurada como se indica en la documentación.
    *   **WSDL URL**: `https://legacy.bizagi.com/OfficeSupplyWS/OfficeService.asmx?WSDL`
6.  **Ejecutar Casos de Prueba:**
    *   Cree nuevos casos desde el Portal de Trabajo para validar el flujo completo.

## 5. Entregables del Ejercicio

-   [ ] Documentación del flujo (`/docs/manual_arquitectura_implementacion.md`).
-   [ ] Archivo de exportación del modelo (`/export/compras-maxi.bex`).
-   [ ] Archivo de exportación de datos parametrizados (`/export/compras-maxi-data.bdex`).
-   [ ] `README.md` actualizado y completo.
-   [ ] Repositorio público en GitHub con todos los artefactos.
-   [ ] Archivo `.zip` único con el contenido del repositorio.

## 6. Datos de Prueba

Los archivos CSV en `/data/catalogos` contienen datos de ejemplo para las entidades maestras:
-   `productos.csv`: Lista de suministros de oficina disponibles.
-   `proveedores.csv`: Lista de proveedores ficticios.
-   `sucursales.csv`: Lista de sucursales de la empresa.
