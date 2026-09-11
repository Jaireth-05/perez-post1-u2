# perez-post1-u2
Post-contenido — Exportación de reportes académicos con patrones creacionales justificados

## Decisiones de diseño

### Decisión 1: Patrón creacional para la exportación de reportes Parte 1

**Patrón elegido:** Abstract Factory.

**Análisis del problema:**

El sistema debe exportar actas de calificaciones en tres formatos (PDF, Excel, HTML), donde cada exportación produce dos piezas relacionadas —el cuerpo (tabla de datos) y el encabezado/pie de página— que deben pertenecer al mismo formato para que el documento sea coherente. No es válido combinar, por ejemplo, un cuerpo en Excel con un encabezado en PDF.


1. **¿El sistema crea un único tipo de producto que varía por formato, o varios productos relacionados que deben mantenerse coherentes entre sí dentro del mismo formato?**
   El sistema no crea un único tipo de producto que varía por formato: crea dos productos relacionados (`ReportBody` y `ReportHeaderFooter`) que deben mantenerse consistentes entre sí dentro del mismo formato. Esto apunta a una familia de productos, no a un producto aislado.

2. **Al agregar el futuro formato CSV, ¿basta con agregar una nueva implementación de un único producto, o hay que agregar una familia completa de piezas relacionadas?**
   Agregar CSV no significa añadir una sola implementación nueva de una clase existente, sino una familia completa nueva: un `CsvReportBody` y un `CsvHeaderFooter`, encapsulados en una nueva fábrica `CsvReportFactory`. El código existente (interfaces `ReportBody`, `ReportHeaderFooter`, `ReportFormatFactory`, y las fábricas de PDF/Excel/HTML) no se modifica.

3. **¿El riesgo real del problema es "se instancia la clase equivocada" o es "se mezclan piezas de familias distintas y el documento queda inconsistente"?**
   El riesgo no es "se instanció la clase equivocada" de forma aislada, sino mezclar piezas de familias distintas (cuerpo de un formato con   encabezado/pie de otro), lo que dejaría el documento final inconsistente.

**Por qué se descartó Factory Method:**

Factory Method resuelve bien el caso de un único producto que varía por
subclase/formato mediante un método de creación (`createReportDocument()`, por ejemplo). Sin embargo, aquí el problema no es un producto único: son dos productos que deben permanecer acoplados por formato. Factory Method no ofrece, por sí solo, ninguna garantía de que el cuerpo y el encabezado/pie creados pertenezcan al mismo formato esa garantía tendría que reforzarse con lógica
adicional fuera del patrón. Abstract Factory, en cambio, modela esa restricción directamente en su estructura: cada fábrica concreta (`PdfReportFactory`, `ExcelReportFactory`, `HtmlReportFactory`) es responsable de producir ambas piezas de una misma familia, haciendo imposible por diseño mezclar piezas de formatos distintos.
