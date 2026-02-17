# @SpringBootApplication, @Bean vs @Component, and the Bean Context

## @SpringBootApplication

`@SpringBootApplication` is a convenience annotation placed on the main class of a Spring Boot application. It combines three annotations:

| Annotation | Purpose |
|------------|---------|
| **@Configuration** | Marks the class as a source of bean definitions. Allows `@Bean` methods in the class. |
| **@EnableAutoConfiguration** | Triggers Spring Boot’s auto-configuration (e.g. embedded Tomcat, default DataSource, etc.) based on classpath and properties. |
| **@ComponentScan** | Scans the package of the annotated class (and sub-packages) for `@Component`, `@Service`, `@Repository`, `@Controller`, etc., and registers them as beans. |

So a class annotated with `@SpringBootApplication` is both a configuration class and the entry point for component scanning and auto-configuration.

---

## @Bean vs @Component

| | @Bean | @Component (and @Service, @Repository, @Controller) |
|---|--------|-----------------------------------------------------|
| **Where** | Method inside a `@Configuration` (or `@Component`) class. | Type-level on a class. |
| **What it defines** | A single bean from the method’s return value. You control exactly how the instance is created. | The class itself is the bean. Spring instantiates it (no-arg constructor by default). |
| **Use case** | Third-party classes you can’t annotate, or when you need custom construction logic (builders, factories, conditional logic). | Your own classes that Spring should manage. |
| **Naming** | Default bean name = method name. | Default bean name = class name with first letter lowercased. |

**@Component** is the base; **@Service**, **@Repository**, **@Controller** are specializations with the same registration behavior but clearer semantics (e.g. `@Repository` adds exception translation for persistence).

**Summary:** Use `@Component` (or stereotypes) for your own types; use `@Bean` when you need to expose an instance you construct yourself or when the type is not under your control.

---

## The Bean Context (ApplicationContext)

The **application context** is the runtime container that holds and manages all beans. It:

- **Creates and wires beans** — instantiates beans, resolves dependencies (constructor/setter injection), and applies lifecycle callbacks (`@PostConstruct`, `InitializingBean`, etc.).
- **Scopes beans** — default is *singleton* (one instance per context); other scopes include *prototype* (new instance per request/lookup), *request*, *session*, etc.
- **Provides lookup** — you can get a bean by type or name via `context.getBean(...)` (prefer injection over manual lookup when possible).

Common context types in Spring Boot:

- **AnnotationConfigApplicationContext** — Java `@Configuration` and `@Component` classes.
- **SpringApplication** (used by Boot) typically creates an **AnnotationConfigApplicationContext** (or web variant), loading your `@SpringBootApplication` class and activating component scan and auto-configuration.

The context is created at startup and closed on shutdown; all singleton beans live inside it for the application’s lifetime.

---

## Interview Q&A

**Q: What is @SpringBootApplication and what does it do?**

A: It’s a convenience annotation that combines @Configuration, @EnableAutoConfiguration, and @ComponentScan. It marks the main class as the configuration source, enables auto-configuration from the classpath, and triggers component scanning from that class’s package (and sub-packages).

**Q: Why would you use @Bean instead of @Component?**

A: Use @Bean when you can’t put annotations on the class (e.g. third-party library), or when you need custom creation logic (builder, factory, conditional logic). Use @Component (or @Service, @Repository, @Controller) for your own classes that Spring should instantiate.

**Q: What is the default scope of a Spring bean? What other scopes do you know?**

A: Default is singleton (one instance per context). Others: prototype (new instance per getBean/lookup), request and session (web), application (ServletContext). You can also define custom scopes.

**Q: What is the difference between ApplicationContext and BeanFactory?**

A: BeanFactory is the core container (create beans, wire dependencies). ApplicationContext extends BeanFactory and adds messaging (i18n), event publishing, resource loading, and often AOP. In practice we use ApplicationContext; BeanFactory is the minimal SPI.

**Q: How does @ComponentScan work and how can you change what gets scanned?**

A: It scans the package of the annotated class and sub-packages for @Component and stereotype annotations. You can set basePackages / basePackageClasses or use @ComponentScan on a @Configuration class to point to specific packages or marker classes.

**Q: What is @EnableAutoConfiguration and how does it decide what to configure?**

A: It enables Spring Boot auto-configuration. Boot uses conditions (e.g. @ConditionalOnClass, @ConditionalOnProperty) on auto-configuration classes; if the condition matches (class on classpath, property set, etc.), the corresponding beans are registered. Configuration is typically in META-INF/spring.factories (or spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports in newer Boot).

**Q: When is a singleton bean created?**

A: By default at context startup when the context is being refreshed. You can make it lazy with @Lazy so it’s created on first use instead.

**Q: What lifecycle callbacks can you use for a bean?**

A: @PostConstruct (after construction and injection), InitializingBean#afterPropertiesSet, and custom init-method. For destruction: @PreDestroy, DisposableBean#destroy, custom destroy-method. In that order within each phase.
