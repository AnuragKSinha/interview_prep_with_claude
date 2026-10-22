# Spring Boot Auto-configuration

Written: 2026-10-22 (pulled forward from its Spring-thread week) · Revise: 23 Oct, 29 Oct, 21 Nov, 20 Jan
Builds on: [beans-and-di.md](beans-and-di.md)

## What `@SpringBootApplication` really is

| Annotation | Job |
| --- | --- |
| `@SpringBootConfiguration` | Marks this class as a configuration class |
| `@ComponentScan` | Finds *your* beans |
| `@EnableAutoConfiguration` | Lets Boot add *its* beans |

## The flow

1. **Add a starter**, e.g. `spring-boot-starter-data-jpa`. A starter is just a bundle of dependencies, so Hibernate and a connection pool land on the classpath.
2. **Boot reads the candidate list** from each module's `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file. (Older Boot versions used `spring.factories`.)
3. **Each candidate is a normal `@Configuration` class guarded by conditions:**
   - `@ConditionalOnClass(DataSource.class)`: the library is on the classpath.
   - `@ConditionalOnMissingBean`: you haven't defined this bean yourself.
   - `@ConditionalOnProperty("...")`: a property is set.
4. **Your configuration is processed first, auto-config after.** So when an auto-config checks `@ConditionalOnMissingBean`, your bean already exists and Boot backs off.

Not every auto-config uses `@ConditionalOnMissingBean`. It is on the beans you are meant to replace (`ObjectMapper`, `DataSource`, and so on).

## In plain-Java terms

```java
if (classExists("com.zaxxer.hikari.HikariDataSource") && !context.hasBean(DataSource.class)) {
    context.register(new HikariDataSource(propsFrom("spring.datasource.*")));
}
```

## Tools

| Need | How |
| --- | --- |
| See what Boot configured and why | `debug=true` in `application.properties` → condition evaluation report; look under **Positive matches** |
| Same report at runtime | Actuator `/actuator/conditions` |
| Turn one auto-config off | `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` or `spring.autoconfigure.exclude` |
| Tweak without replacing | Properties, e.g. `spring.datasource.hikari.maximum-pool-size=20` |

## Worked examples

- **I define my own `ObjectMapper`:** Jackson's auto-config sees it (`@ConditionalOnMissingBean`) and skips the default.
- **A `DataSource` appears that I never asked for:** a JDBC driver or HikariCP is on the classpath. The debug report's Positive matches show `DataSourceAutoConfiguration` and the condition that matched.

## One-line answer

"Starters put libraries on the classpath. Boot loads auto-configuration classes from the `AutoConfiguration.imports` files. Each one is guarded by `@Conditional` checks, and `@ConditionalOnMissingBean` means my own bean always wins."

## Self-test (quizzes on 22 Oct 2026)

**Q1. I add my own `ObjectMapper` bean. What happens to Boot's default, and which annotation decides it?**
Answer: Boot's default is not created. My configuration is processed before auto-configuration, so when Jackson's auto-config checks `@ConditionalOnMissingBean`, my bean already exists and it backs off.
My result: right.
**Gap:** not *every* auto-config uses `@ConditionalOnMissingBean`, only the ones that create beans you're meant to replace. Also say *why* mine wins (my config is processed first).

**Q2. The app starts with a `DataSource` I never asked for. How do I find which auto-config created it?**
Answer: set `debug=true` and read the condition evaluation report. Under **Positive matches**, `DataSourceAutoConfiguration` shows the condition that matched (usually a JDBC driver or HikariCP on the classpath). At runtime, Actuator's `/actuator/conditions` shows the same report.
My result: right.
**Gap:** name the *Positive matches* section and `/actuator/conditions`. Those sound senior.

**Q3. I want Boot's `DataSource` but with a pool size of 50. Do I write my own `DataSource` bean?**
Answer: no. Set a property and keep everything else Boot gives you:
`spring.datasource.hikari.maximum-pool-size=50`
Writing my own bean throws away Boot's defaults and property binding just to change one number.
My result: right (I wasn't sure, but I was).
**Gap:** properties go in `application.properties` or `application.yml`, not JSON.

**Q4. A shared team library's auto-config creates a `RestClient`. How do I make sure my service's own `RestClient` is used?**
Answer: define my own `RestClient` `@Bean` in a `@Configuration` class. If the library's auto-config uses `@ConditionalOnMissingBean`, it backs off.
If it doesn't, there are now two beans and startup fails. Then either:
- exclude that auto-config (`spring.autoconfigure.exclude` or `@SpringBootApplication(exclude = ...)`), or
- mark mine `@Primary`.

My result: right for the normal case.
**Gap:** cover the "what if it has no `@ConditionalOnMissingBean`" case. That's the senior part of the answer.
