# Spring Beans and Dependency Injection

Written: 2026-10-22 · Revise: 23 Oct, 29 Oct, 21 Nov, 20 Jan
Ref: https://docs.spring.io/spring-boot/reference/using/spring-beans-and-dependency-injection.html

## The problem (plain Java)

```java
public class PaymentService {
    private final FraudChecker checker = new FraudChecker(new RulesRepo(new DbClient("jdbc:...")));
}
```

The class builds its own dependencies, so it is hard to test (no way to swap in a fake), hard to change, and every class makes its own copies.

## Core ideas

| Term | Plain English | Java equivalent |
| --- | --- | --- |
| Bean | An object Spring creates and manages | A normal Java object; Spring owns its life cycle |
| Container (`ApplicationContext`) | Creates all beans at startup and wires them | A `Map<Type, Object>` plus a factory you never wrote |
| Dependency injection | The class asks for what it needs; the container hands it over | Passing objects into the constructor instead of calling `new` |
| Inversion of control | The framework calls `new`, not your code | — |

## How Spring finds beans

1. **Your own classes:** `@Component`, `@Service`, `@Repository`, `@Controller`. Found by component scan, starting from the main class's package and going down.
2. **Classes you can't annotate** (library clients, a customised `ObjectMapper`): a `@Bean` method inside a `@Configuration` class.

```java
@Service
public class PaymentService {
    private final FraudChecker checker;

    public PaymentService(FraudChecker checker) {   // constructor injection
        this.checker = checker;
    }
}

@Configuration
public class ClientConfig {
    @Bean
    public RestClient fraudApiClient() {
        return RestClient.create("https://fraud.internal");
    }
}
```

## Why constructor injection

- Fields can be `final`, so the object is never half-built.
- Unit test = `new PaymentService(mockChecker)`. The constructor is plain Java, so no Spring container starts and the test runs in milliseconds.
- One constructor means no `@Autowired` needed.
- Avoid field injection (`@Autowired private X x;`): it hides dependencies and can't be `final`.

## Interview traps

- **Scope:** beans are singletons by default, so one instance is shared by every thread. Keep services stateless; never store request data in a field.
- **Two beans of the same type:** startup fails because Spring can't choose. Fix with `@Primary` on the default, or `@Qualifier("name")` where you inject it.

## One-line answer

"Spring's container creates beans at startup and injects them through constructors. That keeps classes loosely coupled, immutable and testable with plain `new` and mocks."

## Self-test (quizzes on 22 Oct 2026)

**Q1. Why can a constructor-injected service be unit-tested without starting Spring?**
Answer: the constructor is plain Java. In a test I call `new PaymentService(mockChecker)` myself, so no container and no context startup are needed, and the test runs in milliseconds.
My result: right.

**Q2. `PaymentService` keeps a `List<Transaction>` field and adds to it on every request. What goes wrong under load?**
Answer: the bean is a singleton, shared by every request thread at once.
- `ArrayList` isn't thread-safe: concurrent adds cause lost updates or a `ConcurrentModificationException`.
- The list only grows, so under load you get an `OutOfMemoryError`.
- With several instances behind a load balancer, each holds a different list.
- Fix: singletons stay stateless. Request data goes in local variables or the database, never in a field.

My result: partly right. I said "data is lost if the node goes down, so persist it". That's true but secondary.
**Gap:** say *thread safety* first whenever a singleton holds state.

**Q3. Two beans implement `FraudChecker` (`RulesFraudChecker`, `MlFraudChecker`) and startup fails. Two fixes?**
Answer: `@Primary` on the default bean, or `@Qualifier("name")` at the injection point. To use both, inject `List<FraudChecker>` and Spring gives you every implementation.
My result: right.
**Gap (wording):** they *implement the same interface*. They don't "extend the same bean".

**Q4. When do you use a `@Bean` method instead of `@Component`?**
Answer: for classes you can't annotate (third-party or library classes), for custom construction logic, and when you need two beans of the same class with different settings (e.g. two `RestClient`s for two downstream services).
My result: right. I missed the "two beans of the same class" case.
