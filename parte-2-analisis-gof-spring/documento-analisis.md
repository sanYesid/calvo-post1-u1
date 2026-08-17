# Análisis de Patrones de Diseño GoF en Spring Framework

**Nombre completo:** José Yesid Calvo Quintero
**Código estudiantil:** 02240131050
**Curso:** Patrones de Diseño de Software
**Unidad:** Unidad 1 — Fundamentos de Patrones de Diseño
**Fecha:** 17/08/26

---

## 1. Introducción

Spring Framework es uno de los frameworks de desarrollo Java más utilizados en la industria para la construcción de aplicaciones empresariales, y su diseño interno constituye un caso de estudio particularmente valioso para comprender cómo los patrones de diseño Gang of Four (GoF) se aplican en software real y a gran escala. Lejos de ser conceptos exclusivamente teóricos, estos patrones resuelven problemas concretos de acoplamiento, extensibilidad y duplicación de código que surgen al construir un contenedor de inversión de control (IoC) capaz de administrar miles de objetos colaborativos.

El objetivo de este documento es identificar y analizar tres patrones de diseño GoF presentes en el código fuente de Spring Framework, uno de cada categoría (Creacional, Estructural y de Comportamiento), examinando para cada uno la clase o interfaz concreta donde se manifiesta, el problema específico que resuelve en el contexto de Spring Boot, la evidencia de código que confirma su implementación y el principio SOLID que refuerza. La clasificación de los veintitrés patrones en estas tres categorías, así como sus definiciones canónicas, provienen del catálogo original propuesto por Gamma et al. (1994), que sigue siendo la referencia estándar para este tipo de análisis. Los patrones seleccionados son **Factory Method** (creación de beans a través de `BeanFactory`), **Proxy** (interceptación de llamadas mediante `JdkDynamicAopProxy` en Spring AOP) y **Template Method** (ejecución del flujo JDBC mediante `JdbcTemplate`).

---

## 2. Análisis de Patrón 1 — Factory Method (Creacional)

### 2.1 Patrón y categoría

El **Factory Method** es un patrón creacional que define una interfaz para crear un objeto, pero delega en las subclases (o en una implementación concreta) la decisión de qué clase instanciar (Gamma et al., 1994). Su propósito general es desacoplar el código cliente del proceso de construcción de objetos, de modo que el cliente trabaje siempre contra una abstracción y nunca invoque directamente al operador `new` sobre una clase concreta.

### 2.2 Ubicación en Spring Framework

Este patrón aparece de forma central en la interfaz `org.springframework.beans.factory.BeanFactory`, ubicada en el módulo `spring-beans` (Spring Team, s.f.). `BeanFactory` define la familia de métodos sobrecargados `getBean(...)`, que constituyen el punto de acceso mediante el cual cualquier componente de la aplicación obtiene una instancia de un bean administrado por el contenedor, sin conocer los detalles de su construcción. La implementación real de la creación ocurre en clases internas del contenedor, como `AbstractBeanFactory` y `AbstractAutowireCapableBeanFactory`, que heredan y especializan el comportamiento de instanciación (VMware, Inc., 2024b).

### 2.3 Problema que resuelve

En una aplicación Spring Boot típica, los objetos (beans) pueden requerir procesos de construcción muy distintos entre sí: algunos se instancian directamente con `new`, otros a través de un método de fábrica definido por el propio desarrollador (`@Bean`), otros mediante inyección de dependencias con autowiring, y otros aplicando lógica condicional (`@Conditional`). Si el código de la aplicación tuviera que instanciar cada uno de estos objetos manualmente, quedaría fuertemente acoplado a los detalles de construcción de cada clase, y cualquier cambio en dicha construcción (por ejemplo, pasar de un singleton a un prototype, o agregar un nuevo parámetro al constructor) obligaría a modificar todo el código cliente disperso por la aplicación. El patrón Factory Method resuelve este problema centralizando la lógica de creación detrás de la abstracción `BeanFactory`: el desarrollador simplemente solicita un bean por nombre o por tipo (`getBean(...)`) y delega en el contenedor la responsabilidad de decidir cómo construirlo, resolver sus dependencias y determinar su ciclo de vida (singleton, prototype, request, session, etc.).

### 2.4 Evidencia de código

La propia documentación de la interfaz `BeanFactory` señala explícitamente que este mecanismo reemplaza el uso manual de patrones creacionales por parte del desarrollador (VMware, Inc., 2024b; Spring Team, s.f.). Un extracto representativo (adaptado y comentado por el estudiante) de la interfaz es el siguiente:

```java
// org.springframework.beans.factory.BeanFactory
public interface BeanFactory {

    // El cliente solicita el objeto por nombre; no lo construye directamente.
    Object getBean(String name) throws BeansException;

    // Sobrecarga por tipo: el contenedor decide qué implementación concreta
    // instanciar y devolver, aplicando su propia lógica interna.
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    <T> T getBean(Class<T> requiredType) throws BeansException;
}
```

La propia documentación oficial de la interfaz indica que un `BeanFactory` permite ser usado como reemplazo del patrón de diseño Singleton o Prototype (VMware, Inc., 2024b), lo que confirma de forma explícita que el rol de esta interfaz es actuar como un creador abstracto que oculta la construcción concreta del objeto solicitado. En el proyecto de la Parte 1 de esta actividad, la interfaz `DiscountStrategy` junto con sus implementaciones (`VipDiscount`, `RegularDiscount`, `NoDiscount`) sigue un espíritu similar, aunque aplicado como patrón Strategy más que como Factory Method puro.

### 2.5 Principio SOLID asociado

Este patrón refuerza principalmente el **Principio de Inversión de Dependencias (DIP)**, ya que el código cliente depende únicamente de la abstracción `BeanFactory` (o de `ApplicationContext`, que la extiende) y no de las clases concretas que finalmente son instanciadas. De forma secundaria, también refuerza el **Principio de Abierto/Cerrado (OCP)**, porque es posible incorporar nuevas estrategias de creación de beans (por ejemplo, un nuevo `BeanPostProcessor` o un nuevo scope personalizado) sin modificar el código que consume `getBean(...)`.

**Análisis contrafactual:** si Spring Boot no aplicara este patrón y expusiera directamente las clases concretas de cada componente, cada módulo de la aplicación tendría que conocer y instanciar manualmente sus dependencias con `new`, lo que eliminaría la posibilidad de inyección de dependencias, dificultaría enormemente las pruebas unitarias (al no poder sustituir implementaciones por mocks) y produciría un acoplamiento extremadamente alto entre capas.

---

## 3. Análisis de Patrón 2 — Proxy (Estructural)

### 3.1 Patrón y categoría

El **Proxy** es un patrón estructural que provee un objeto sustituto o intermediario para controlar el acceso a otro objeto (Gamma et al., 1994). El proxy implementa la misma interfaz que el objeto real, de modo que resulta indistinguible para el cliente, pero puede añadir comportamiento adicional antes o después de delegar la llamada al objeto original (el "target").

### 3.2 Ubicación en Spring Framework

Este patrón se encuentra implementado en la clase `org.springframework.aop.framework.JdkDynamicAopProxy`, perteneciente al módulo `spring-aop` (Spring Team, s.f.). Esta clase implementa las interfaces `AopProxy`, `java.lang.reflect.InvocationHandler` y `Serializable`, y constituye una de las dos estrategias que Spring utiliza para generar proxies dinámicos (la otra es `CglibAopProxy`, para clases sin interfaz) (VMware, Inc., 2024b). `JdkDynamicAopProxy` se apoya en los proxies dinámicos nativos de Java (`java.lang.reflect.Proxy`) para generar, en tiempo de ejecución, un objeto que implementa las mismas interfaces que el bean original.

### 3.3 Problema que resuelve

En una aplicación Spring Boot es habitual necesitar comportamientos transversales —transaccionalidad (`@Transactional`), seguridad (`@PreAuthorize`), caché (`@Cacheable`), auditoría o logging— que no forman parte de la lógica de negocio de una clase, pero que deben ejecutarse antes o después de invocar sus métodos. Si esta lógica se escribiera directamente dentro de cada clase de servicio, se violaría la separación de responsabilidades y el mismo código transversal se duplicaría en decenas de clases distintas. El patrón Proxy resuelve este problema interceptando las llamadas a los métodos del bean real: `JdkDynamicAopProxy` implementa `InvocationHandler`, de modo que cada invocación sobre el proxy pasa primero por su método `invoke(Object proxy, Method method, Object[] args)`, donde Spring AOP ejecuta la cadena de advices (interceptores) configurada —por ejemplo, abrir una transacción— antes de delegar la ejecución real al objeto de negocio original y, finalmente, ejecutar la lógica posterior (como hacer commit o rollback).

### 3.4 Evidencia de código

Un extracto simplificado y comentado que ilustra el mecanismo de interceptación es el siguiente (adaptado por el estudiante a partir del código fuente original; Spring Team, s.f.):

```java
// org.springframework.aop.framework.JdkDynamicAopProxy
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

    private final AdvisedSupport advised; // configuración de advices/interceptores

    // Punto único de entrada de TODAS las llamadas al bean proxied.
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. Obtiene el objeto real (target) detrás del proxy.
        Object target = this.advised.getTargetSource().getTarget();

        // 2. Construye la cadena de interceptores (p. ej. transacción, seguridad).
        List<Object> chain =
            this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, target.getClass());

        // 3. Ejecuta la cadena y, al final, delega en el método real del target.
        MethodInvocation invocation =
            new ReflectiveMethodInvocation(proxy, target, method, args, target.getClass(), chain);
        return invocation.proceed();
    }
}
```

Este fragmento confirma que el proxy no contiene lógica de negocio propia: su única responsabilidad es interceptar la llamada, ejecutar el comportamiento adicional configurado y delegar en el objeto real.

### 3.5 Principio SOLID asociado

El patrón Proxy refuerza principalmente el **Principio de Responsabilidad Única (SRP)**, dado que separa la lógica transversal (transacciones, seguridad, logging) de la lógica de negocio propia de la clase de servicio, evitando que esta última tenga más de una razón para cambiar. También refuerza el **Principio de Abierto/Cerrado (OCP)**, porque es posible añadir nuevos aspectos (advices) a un bean existente sin modificar una sola línea de su código fuente, simplemente registrando un nuevo interceptor en la configuración de AOP.

**Análisis contrafactual:** sin este patrón, cada clase de servicio en Spring Boot tendría que implementar manualmente el manejo de transacciones, la verificación de permisos y el registro de auditoría dentro de sus propios métodos, lo que produciría una enorme duplicación de código, un alto acoplamiento con la infraestructura y una violación sistemática del principio de responsabilidad única en prácticamente toda la capa de servicios.

---

## 4. Análisis de Patrón 3 — Template Method (Comportamiento)

### 4.1 Patrón y categoría

El **Template Method** es un patrón de comportamiento que define en una clase el esqueleto invariable de un algoritmo, dejando que ciertos pasos variables sean proporcionados por el código cliente, ya sea mediante subclases que sobrescriben métodos o mediante objetos de callback que se pasan como parámetro (Gamma et al., 1994). El objetivo es reutilizar la estructura común de un proceso y evitar que cada cliente tenga que reimplementar los pasos repetitivos.

### 4.2 Ubicación en Spring Framework

Este patrón se manifiesta en la clase `org.springframework.jdbc.core.JdbcTemplate`, del módulo `spring-jdbc` (Spring Team, s.f.). `JdbcTemplate` expone métodos como `execute(StatementCallback action)`, `execute(PreparedStatementCreator psc, PreparedStatementCallback action)` y `query(...)`, que constituyen el "esqueleto fijo" del acceso a datos JDBC (VMware, Inc., 2024a). La parte variable del algoritmo —la sentencia SQL concreta y la extracción de resultados— se entrega a través de interfaces de callback como `StatementCallback`, `PreparedStatementCallback` y `RowMapper`, definidas en el mismo paquete.

### 4.3 Problema que resuelve

Trabajar con JDBC "a mano" obliga a repetir, en cada operación de acceso a datos, la misma secuencia de pasos: obtener una conexión del `DataSource`, crear el `Statement` o `PreparedStatement`, ejecutar la operación, recorrer el `ResultSet`, capturar las excepciones `SQLException` y, finalmente, cerrar de forma segura la conexión, el statement y el result set (habitualmente en bloques `finally` anidados). Este código repetitivo es una fuente frecuente de errores, como fugas de conexiones no cerradas. `JdbcTemplate` resuelve el problema encapsulando ese flujo fijo dentro de sus métodos `execute(...)`, y exponiendo únicamente el punto variable del proceso —qué SQL ejecutar y cómo interpretar el resultado— a través de un objeto callback que el desarrollador implementa. De esta manera, Spring Boot logra que el desarrollador solo escriba la parte que realmente cambia entre una consulta y otra.

### 4.4 Evidencia de código

Un extracto simplificado y comentado que ilustra el esqueleto fijo del algoritmo es el siguiente (adaptado por el estudiante a partir del código fuente original; Spring Team, s.f.):

```java
// org.springframework.jdbc.core.JdbcTemplate
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations {

    // Paso fijo del algoritmo: obtener conexión, crear Statement,
    // manejar excepciones y liberar recursos SIEMPRE de la misma forma.
    public <T> T execute(StatementCallback<T> action) throws DataAccessException {
        Connection con = DataSourceUtils.getConnection(obtainDataSource());
        Statement stmt = null;
        try {
            stmt = con.createStatement();
            applyStatementSettings(stmt);

            // Paso variable: delega en el callback que el desarrollador implementó.
            T result = action.doInStatement(stmt);
            return result;
        }
        catch (SQLException ex) {
            throw translateException("StatementCallback", getSql(action), ex);
        }
        finally {
            JdbcUtils.closeStatement(stmt);
            DataSourceUtils.releaseConnection(con, getDataSource());
        }
    }
}
```

```java
// org.springframework.jdbc.core.StatementCallback
@FunctionalInterface
public interface StatementCallback<T> {
    // Único método que el desarrollador debe implementar: la parte variable.
    T doInStatement(Statement stmt) throws SQLException, DataAccessException;
}
```

### 4.5 Principio SOLID asociado

Este patrón refuerza el **Principio de Responsabilidad Única (SRP)**, ya que `JdbcTemplate` se encarga exclusivamente de la gestión del ciclo de vida de los recursos JDBC (conexión, statement, manejo de excepciones), mientras que el callback proporcionado por el desarrollador se encarga exclusivamente de la lógica de negocio de la consulta. También refuerza el **Principio de Abierto/Cerrado (OCP)**, porque es posible ejecutar nuevas operaciones SQL simplemente implementando una nueva instancia de `StatementCallback`, `PreparedStatementCallback` o `RowMapper`, sin necesidad de modificar la clase `JdbcTemplate`.

**Análisis contrafactual:** si `JdbcTemplate` no existiera, cada clase DAO de la aplicación tendría que reimplementar manualmente el manejo de conexiones, la apertura y cierre de statements y la traducción de excepciones `SQLException`, multiplicando el riesgo de fugas de recursos y de inconsistencias en el manejo de errores en toda la capa de persistencia.

---

## 5. Conclusiones

El análisis del código fuente de Spring Framework evidencia que los patrones de diseño GoF no son un ejercicio meramente académico, sino herramientas que un framework maduro utiliza de forma sistemática para resolver problemas reales de acoplamiento, duplicación y extensibilidad. El patrón Factory Method, presente en `BeanFactory`, permite que el contenedor de Spring controle la creación de objetos sin que el código cliente conozca los detalles de instanciación, reforzando la inversión de dependencias que es el corazón mismo del contenedor IoC. El patrón Proxy, implementado en `JdkDynamicAopProxy`, demuestra cómo es posible añadir comportamiento transversal a una aplicación sin ensuciar la lógica de negocio, sosteniendo así el principio de responsabilidad única a escala de todo un framework. El patrón Template Method, presente en `JdbcTemplate`, muestra cómo encapsular un algoritmo repetitivo y propenso a errores, dejando únicamente la parte variable en manos del desarrollador. La lección más relevante para el diseño propio de software es que estos patrones rara vez aparecen aislados: en Spring conviven y se refuerzan mutuamente (el proxy es devuelto por una factory, y ambos dependen de abstracciones bien definidas), lo que confirma que aplicar SOLID de manera consistente es, en la práctica, lo que habilita el uso efectivo de los patrones GoF descritos originalmente por Gamma et al. (1994).

---

## 6. Referencias

Gamma, E., Helm, R., Johnson, R., & Vlissides, J. (1994). *Design patterns: Elements of reusable object-oriented software*. Addison-Wesley.

Refactoring.Guru. (s.f.). *Design patterns*. Recuperado el 17 de agosto de 2026, de https://refactoring.guru/design-patterns

Spring Team. (s.f.). *spring-framework* [Código fuente]. GitHub. Recuperado el 17 de agosto de 2026, de https://github.com/spring-projects/spring-framework

VMware, Inc. (2024a). *Spring Framework reference documentation: Using the JDBC core classes*. Spring.io. https://docs.spring.io/spring-framework/reference/data-access/jdbc/core.html

VMware, Inc. (2024b). *BeanFactory (Spring Framework API)*. Spring.io. https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/BeanFactory.html