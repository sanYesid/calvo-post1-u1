# calvo-post1-u1
Post-contenido — Refactorización SOLID y análisis de patrones GoF en Spring

## Descripción
Repositorio del post-contenido de la Unidad 1 de Patrones de Diseño
de Software — Sexto Semestre. Contiene dos partes: refactorización
SOLID de un God Object (parte-1-refactorizacion-solid/) y análisis
de patrones GoF en Spring Framework (parte-2-analisis-gof-spring/)

## Parte 1 — Refactorización SOLID
Proyecto Maven que refactoriza OrderProcessor aplicando SRP, OCP y
DIP. Ver parte-1-refactorizacion-solid/.

| Principio | Método/Sección afectada | Descripción de la violación |
|-----------|-------------------------|-----------------------------|
| SRP | calculateTotal + applyDiscount + saveOrder + sendEmail + printReport | La clase OrderProcessor se encarga de distintos aspectos del sistema: lógica del negocio, aplicación de descuentos, persistencia de órdenes, notificación al cliente y generación de reportes. Esto significa que la clase tiene más de una razón para cambiar, ya que una modificación en cualquiera de estos aspectos obliga a tocar la misma clase que maneja el cálculo de totales o el envío de correos. También, dificulta las pruebas unitarias, porque no se puede aislar un solo comportamiento sin arrastrar las demás responsabilidades, y aumenta el riesgo de que un cambio en una parte del código termine rompiendo, de forma no evidente, otra parte que no tiene relación lógica con la que se modificó. |
| OCP | applyDiscount (if/else sobre customerType) | El método applyDiscount viola OCP porque la lógica de descuentos está resuelta mediante una cadena de condicionales (if/else) que compara directamente el valor de customerType. Si en el futuro se necesita agregar un nuevo tipo de cliente como "PREMIUM", es necesario abrir y modificar un método que ya está en funcionamiento y que ya fue probado, en lugar de poder extender el comportamiento sin alterar el código existente. Esto contradice el principio de estar "abierto a la extensión, cerrado a la modificación", ya que cada nueva regla de negocio implica un riesgo de introducir errores en una lógica que antes funcionaba correctamente. |
| DIP | Toda la clase (dependencias internas sin abstracciones) | OrderProcessor no depende de ninguna abstracción para las operaciones de persistencia ni de notificación, sino que implementa directamente los detalles concretos dentro de sus propios métodos (System.out.println simulando el guardado en base de datos y el envío del correo). Esto implica que un módulo de alto nivel, como es la lógica de procesamiento de una orden, queda acoplado directamente a implementaciones de bajo nivel. Al no existir un punto de abstracción, como una interfaz OrderRepository o NotificationService, cualquier cambio en la forma de persistir los datos o de notificar al cliente obliga a modificar directamente la clase OrderProcessor, en lugar de simplemente reemplazar la implementación concreta que se está usando. |


## Parte 2 — Análisis de Patrones GoF en Spring
| # | Patrón | Categoría | Clase en Spring |
|---|--------|-----------|-----------------|
| 1 | Factory Method | Creacional | `org.springframework.beans.factory.BeanFactory` |
| 2 | Proxy | Estructural | `org.springframework.aop.framework.JdkDynamicAopProxy` |
| 3 | Template Method | Comportamiento | `org.springframework.jdbc.core.JdbcTemplate` |
 
Ver parte-2-analisis-gof-spring/documento-analisis.md.
 
## Herramientas utilizadas
- Java 17, Apache Maven, VS Code, Git, GitHub
- Código fuente de Spring Framework (investigación)
## Conclusiones
Este post-contenido permitió comprender los principios SOLID no solo como reglas teóricas, sino como criterios concretos para dividir responsabilidades, extraer abstracciones e inyectar dependencias en código legacy real. La refactorización del God Object `OrderProcessor` mostró en la práctica cómo separar responsabilidades (SRP), abrir el sistema a nuevas reglas de negocio sin modificar código existente (OCP) y depender de abstracciones en lugar de implementaciones concretas (DIP) mejora directamente la mantenibilidad y la capacidad de prueba de un sistema. Por su parte, el análisis de Spring Framework evidenció que los patrones GoF —Factory Method, Proxy y Template Method— no son ejercicios académicos aislados, sino soluciones que un framework maduro aplica de forma sistemática para resolver problemas reales de acoplamiento, extensibilidad y duplicación de código. La lección más importante de ambas partes es que los patrones de diseño y los principios SOLID se refuerzan mutuamente: aplicar SOLID de manera consistente es, en la práctica, lo que habilita el uso efectivo de los patrones de diseño en cualquier proyecto propio.