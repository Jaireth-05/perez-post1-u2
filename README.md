# Post-contenido — Unidad 2: Patrones Creacionales

## Descripción
Repositorio del post-contenido de la Unidad 2 de Patrones de Diseño de Software (Sexto Semestre). Es un único proyecto Maven (`exportador-reportes/`) enfocado en resolver la exportación de reportes académicos en diferentes formatos (Parte 1), para luego extenderlo mediante configuración compleja y evaluar si el patrón Singleton aplica realmente al sistema (Parte 2).

## Cómo ejecutar
cd exportador-reportes
mvn compile
mvn exec:java -Dexec.mainClass="com.patrones.u2.Main"

## Decisiones de diseño

### Decisión 1 — Factory Method vs. Abstract Factory (Parte 1)

**Patrón elegido:** Abstract Factory.

**Justificación:**

Para tomar esta decisión, nos basamos en responder las tres preguntas diagnósticas de la Guía Teórica (Sección 7.1):

* **¿Un solo producto o una familia de productos relacionados?** El sistema no genera una sola pieza aislada que cambia de formato. En realidad, produce dos elementos estrechamente vinculados (`ReportBody` y `ReportHeaderFooter`) que deben mantener coherencia entre sí. No tendría ningún sentido ni validez combinar un cuerpo de reporte en Excel con un encabezado formateado para PDF.
* **Al agregar el futuro formato CSV, ¿basta una implementación nueva o hace falta una familia completa?** Requiere toda una familia nueva. Habría que implementar un `CsvReportBody` y un `CsvHeaderFooter`, agrupándolos dentro de una nueva fábrica `CsvReportFactory`. Todo el código previo (las interfaces y fábricas que ya existen para PDF, Excel y HTML) se mantiene intacto sin sufrir modificaciones.
* **¿Cuál es el riesgo real del problema?** El peligro principal no radica en instanciar la clase equivocada individualmente, sino en mezclar componentes de distintas familias (por ejemplo, el cuerpo en un formato y el pie de página en otro), lo que rompería la consistencia del documento exportado.

**Por qué se descartó Factory Method:** Aunque Factory Method funciona bastante bien para crear un producto único que varía según la subclase o el formato, no ofrece garantías por sí solo de que dos piezas dependientes pertenezcan a la misma variante. Abstract Factory, en cambio, soluciona esto directamente desde su estructura. Cada fábrica concreta (`PdfReportFactory`, `ExcelReportFactory`, `HtmlReportFactory`) se encarga de crear ambas partes respetando la misma familia, previniendo por diseño cualquier mezcla indebida.

### Decisión 2 — Mecanismo de extensibilidad de formatos (Parte 1)

**Opción elegida:** Registro dinámico usando `Map<String, Supplier<ReportFormatFactory>>` (`ReportFactoryRegistry`).

**Justificación:**

Escribir un `switch` o encadenar estructuras `if/else` para evaluar el tipo de formato obligaría a modificar la lógica central cada vez que se incorpore una variante (como el formato CSV planificado). Esto rompería de forma directa el Principio Abierto/Cerrado (OCP). 

Con un registro dinámico logramos separar qué formatos están disponibles de cómo se resuelven internamente. Gracias a `ReportFactoryRegistry.register(String, Supplier<ReportFormatFactory>)`, es posible dar de alta una nueva alternativa sin necesidad de tocar la clase `ReportFactoryRegistry`. Además, el método `resolve()` simplemente consulta el mapa y arroja una excepción explícita (`IllegalArgumentException`) si el formato solicitado no existe, evitando condicionales extensos que crezcan a medida que la aplicación escala.

### Decisión 3 — Builder vs. constructor telescópico vs. setters (Parte 2)

**Patrón elegido:** Builder (implementado como la clase interna `Builder` dentro de `ExportConfig`).

**Justificación:**

La clase `ExportConfig` maneja 1 parámetro obligatorio (`format`) junto a 8 opcionales. Utilizar un constructor con los 9 parámetros obligaría a recordar el orden exacto al llamarlo, lo que incrementa el riesgo de traspasar valores del mismo tipo (`String` o `boolean`) en la posición errónea sin que el compilador lo detecte. 

Por otro lado, crear múltiples constructores sobrecargados derivaría en un descontrol de combinaciones muy difíciles de mantener. Finalmente, usar objetos mutables con setters independientes abre la puerta a dejar instancias en estados inconsistentes o incompletos, ya que no habría un momento claro para validar las combinaciones recibidas.

El patrón `Builder` soluciona estos inconvenientes. Ofrece métodos encadenables que solo configuran las variables necesarias mientras el resto conserva sus valores por defecto. Toda la validación previa ocurre centralizada en un único punto dentro de `build()`, bloqueando la creación de objetos inválidos (por ejemplo, intentar activar `compress=true` sin definir una ruta en `outputPath` arroja un `IllegalStateException` de inmediato).

### Decisión 4 — ¿ReportFactoryRegistry necesita ser Singleton? (Parte 2)

**Conclusión:** No resulta conveniente transformar `ReportFactoryRegistry` en un Singleton tradicional.

**Justificación:**

Al evaluar la situación bajo los criterios de la Guía Teórica (Secciones 2.1, 2.4 y 7.3), encontramos las siguientes razones:

* **Identidad de objeto:** El proyecto opera adecuadamente usando únicamente métodos estáticos. Ninguna parte del sistema requiere inyectar el registro por constructor, usar polimorfismo sobre él o reemplazarlo por un mock durante las pruebas unitarias. Su uso se reduce a llamadas directas mediante `ReportFactoryRegistry.resolve()` y `ReportFactoryRegistry.register()`.
* **Inicialización costosa:** No existen procesos pesados. El `Map` interno se llena únicamente con tres registros (PDF, Excel y HTML) dentro de un bloque `static`, sin necesidad de consultar bases de datos o leer archivos externos. El propio cargador de clases (classloading) resuelve esto de forma natural sin requerir inicialización perezosa (lazy initialization).
* **Fuente única de verdad:** La presencia del `Map` estático garantiza este comportamiento. Al ser un atributo `static`, vive una sola instancia compartida a nivel de JVM sin tener que añadir la complejidad de un Singleton (constructores privados adicionales, `getInstance()`, o manejo de hilos/sincronización).
* **Escenarios futuros razonables:** De hecho, implementar Singleton aquí podría resultar perjudicial a futuro. Si el proyecto evoluciona hacia una arquitectura multi-institución donde cada entidad necesite su propio registro independiente, la restricción de Singleton impediría tener más de una instancia activa. 

Por ende, la estructura ya construida en la Parte 1 (clase `final`, constructor privado y miembros estáticos) resulta más que suficiente. Añadir un Singleton convencional solo agregaría código innecesario sin aportar ningún beneficio real.

## Herramientas utilizadas
* Java 17
* Apache Maven
* VS Code
* Git y GitHub

## Conclusiones

Esta práctica dejó claro que aplicar un patrón creacional no debe ser una decisión automática o por cumplir con una plantilla. Dos problemas con apariencias similares (exportar un archivo en diferentes formatos) requieren soluciones completamente distintas dependiendo de si se maneja un producto aislado o una familia entera de componentes que deben coordinarse. 

De igual forma, comprendimos que centralizar un proceso no implica convertirlo en Singleton por inercia. La costumbre de vincular la palabra "registro" con "Singleton" debe reemplazarse por el análisis crítico de factores concretos: la identidad del objeto, los costos de inicialización y las necesidades reales del sistema a futuro. El aprendizaje más valioso es que justificar y documentar por qué se descarta una alternativa es tan importante como la implementación misma, pues nos exige respaldar la arquitectura en las necesidades reales del software y no solo en la intuición.