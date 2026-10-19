# La arquitectura hexagonal es más fácil de lo que piensas.

*Tres etapas: hexagonal pura, arquitectura limpia, DDD. Solo necesitas la primera para empezar.*

La mayoría de las explicaciones de la arquitectura hexagonal muestran el conjunto completo de una sola vez: puertos, adaptadores, casos de uso, comandos, agregados, objetos de valor y eventos de dominio. Los lectores terminan pensando que el patrón es enorme, y los equipos lo rechazan porque "no necesitamos todo eso".

El patrón en sí es sencillo. Casi todo lo que se asocia con él proviene de la Arquitectura Limpia o del Diseño Orientado a Dominios, que son decisiones independientes que se superponen. Este artículo lo desarrolla en tres etapas para que puedas ver qué aporta cada concepto y cuál es su coste. Los ejemplos de cada etapa provienen de tres proyectos ejecutables de Spring Boot que implementan el mismo servicio de pedidos con la misma API HTTP; los fragmentos se han acortado para facilitar la lectura.

## El punto de partida

Aquí hay un servicio que la mayoría de nosotros hemos escrito en algún momento:

```java
@Service
public class OrderService {

    @Autowired private OrderRepository repository;      // JPA
    @Autowired private StripeClient stripe;             // HTTP
    @Autowired private KafkaTemplate<String, String> kafka;

    @Transactional
    public OrderDto placeOrder(OrderDto dto) {
        OrderEntity entity = new OrderEntity();
        entity.setCustomerId(dto.getCustomerId());
        entity.setTotal(calculateTotal(dto));
        if (entity.getTotal().compareTo(new BigDecimal("10000")) > 0) {
            throw new ResponseStatusException(HttpStatus.UNPROCESSABLE_ENTITY);
        }
        stripe.charge(entity.getTotal(), "chf");
        entity.setStatus("PAID");
        kafka.send("orders", entity.getId().toString());
        return OrderMapper.toDto(repository.save(entity));
    }
}
```

Para ser justos, este código no es descuidado. Devuelve un DTO, la entidad nunca sale de la capa de servicio y funciona. Los problemas son estructurales. La regla de negocio se encuentra entre una llamada a Stripe y una llamada a Kafka. La entidad es a la vez esquema de base de datos y modelo de dominio, por lo que cualquier cambio de esquema implica un cambio de dominio. Hay un código de estado HTTP dentro de una regla de negocio, lo cual no tiene sentido para un trabajo por lotes. Y probar la regla implica iniciar un contexto de Spring y simular tres clientes.

Los cuatro problemas tienen una misma causa: el núcleo conoce el mundo exterior.

**[ Insertar imagen aquí: `slides/img/schichten_en.png` ]**

*El punto de partida en capas. Los DTO se mueven hacia arriba, pero cada dependencia apunta hacia abajo, a la tecnología, y la entidad sirve como esquema y modelo de dominio al mismo tiempo.*

## Stage 1: Puertos y adaptadores, nada más.

En 2005, Alistair Cockburn describió este patrón como Puertos y Adaptadores. La idea principal se resume en una frase: las dependencias apuntan hacia adentro. El núcleo de la aplicación define las interfaces para todo lo que necesita del exterior, y el código técnico las implementa.

En el proyecto de ejemplo, toda la etapa tiene esta estructura de paquetes:

```
org.unsch.orders
├── domain                Order, OrderLine, OrderStatus
├── application
│   ├── port.in           OrderService
│   ├── port.out          OrderRepository, PaymentPort
│   └── service           DefaultOrderService
├── adapter
│   ├── in.web            OrderController, request/response DTOs, exception handler
│   ├── in.tx             TransactionalOrderService
│   ├── out.persistence   OrderJpaEntity, OrderLineJpaEntity, adapter, mapper
│   └── out.payment       PspPaymentAdapter, InMemoryPaymentAdapter
└── config                ApplicationConfiguration
```

Tres zonas.  `domain` y `application` son el interior, `adapter` es el exterior, y `config` es la única costura donde se unen. Los paquetes `port.in` y `port.out` hacen visible la dirección en el nombre del paquete.

El núcleo define dos tipos de interfaces. Un puerto de control (puerto de entrada) describe lo que se puede hacer con el sistema:

```java
public interface OrderService {
    UUID placeOrder(String customerId, String currency, List<OrderLine> lines);
    void cancelOrder(UUID orderId, String reason);
    Order getOrder(UUID orderId);
}
```

Los puertos controlados (puertos de salida) describen lo que el sistema necesita del exterior:

```java
public interface PaymentPort {
    String charge(UUID orderId, BigDecimal amount, String currency);
    void refund(String paymentReference);
}
```

The naming matters. The port is called `PaymentPort`, not `StripeClient`. If the provider shows through in the interface, the port is cut wrong. This is Dependency Inversion, applied consistently at every system boundary. Before: the service depends on `StripeClient` and will not compile without it. After: the service depends on an interface it owns, and the Stripe adapter implements that interface. The call still flows toward Stripe at runtime, but the compile-time dependency points inward.

**[ Insert image here: `slides/img/hexagon_en.png` ]**

*The whole pattern in one picture. The red dashed line on the right is the point: the call goes outward, the dependency points inward. On the left both directions coincide, which is why the driven side is the one that so often gets built wrong.*

The implementation of the driving port (inbound port) is plain Java. No `@Service`, no `@Transactional`, no Spring import. One configuration class wires core and framework together:

```java
@Configuration
class ApplicationConfiguration {

    @Bean
    OrderService orderService(OrderRepository repo, PaymentPort payment) {
        return new TransactionalOrderService(new DefaultOrderService(repo, payment));
    }
}
```

Transactions are technology, so a decorator wraps them around the driving port (inbound) instead of putting the annotation into the core. On the driven side (outbound), the persistence adapter implements `OrderRepository` with Spring Data behind it, mapping between the domain class and a separate JPA entity. The payment port has two adapters, a real one and an in-memory one activated by a Spring profile, so the whole application runs locally without any external service.

That is the entire pattern. A handful of interfaces, a handful of adapters, one wiring class.

## What you get

The payoff shows up in the test suite. The core is tested with two hand-written fakes, about forty lines each:

```java
@Test
void doesNotSaveWhenPaymentFails() {
    payment.failNext();
    assertThatThrownBy(() -> service.placeOrder("cust-1", "CHF", lines()))
            .isInstanceOf(PaymentFailedException.class);
    assertThat(repository.size()).isZero();
}
```

This runs in milliseconds without Spring. In the example project, the seven core tests take about 10 ms, while the full application smoke test takes close to two seconds. The point is not fast tests as such. The point is that the code with the highest rate of change, the business logic, gets the fastest feedback. In a typical `@SpringBootTest` suite it is exactly the other way around.

The dependency rule itself becomes executable with ArchUnit:

```java
@ArchTest
static final ArchRule application_is_free_of_spring =
        noClasses().that().resideInAPackage("..application..")
                .should().dependOnClassesThat().resideInAnyPackage(
                        "org.springframework..", "jakarta.persistence..");
```

Without such a test, any architecture erodes within a few sprints. Not out of malice, but because the shortcut import always wins under deadline pressure.

> **Worth pausing here.** Everything up to this point is relatively simple, and it makes your life as a developer easier rather than harder. The code stays lean, and so does your mental model: the technical particularities are factored out automatically, and you can concentrate fully on the business logic. And the strongest argument is testability. A core you can test in milliseconds, without Spring and without mocking foreign systems, is hard to beat.

## Stage 2: Use cases and commands

Clean Architecture keeps the same dependency rule, and the part of it that teams actually adopt adds structure to the driving side (the inbound ports). The book goes further, with output boundaries and presenters on the way back out, but almost nobody carries those into production. Instead of one `OrderService` interface, there is one port per use case: `PlaceOrderUseCase`, `CancelOrderUseCase`, `GetOrderQuery`. And instead of parameter lists, each use case takes a command object:

```java
public interface PlaceOrderUseCase {

    UUID placeOrder(PlaceOrderCommand command);

    record PlaceOrderCommand(String customerId, List<Item> items) {

        // the command validates itself, but only syntactically;
        // business rules stay in the core
        public PlaceOrderCommand {
            if (customerId == null || customerId.isBlank()) {
                throw new IllegalArgumentException("customerId is required");
            }
            if (items == null || items.isEmpty()) {
                throw new IllegalArgumentException("at least one line item is required");
            }
            items = List.copyOf(items);   // defensive copy, the command stays immutable
        }

        public record Item(String productId, int quantity, BigDecimal unitPrice, String currency) { }
    }
}
```

With three methods on one interface, the split is theoretical. With twenty, every caller depends on the whole thing, and the split starts paying. Commands are worth it from about four parameters on: they name their fields, copy lists defensively, and validate themselves in the compact constructor. The cost: every use case now brings its own interface and its own service class. For CRUD-heavy services the pattern runs empty, which is why this is a stage and not a requirement.

> **A side note on reputation.** I suspect this stage is responsible for hexagonal architecture's reputation as boilerplate. Turning every read and write method into its own interface feels like overkill in many codebases, and a command object for each method on top of that is probably the point where most developers decided not to bother. That verdict then lands on the whole pattern, even though stage 1 never asked for any of it.

## Stage 3: A domain model that defends itself

Up to this point the domain class was a data holder with getters and setters, and the business rules lived in the service as a checklist. Anyone who builds an order without going through the service checks nothing. The order total was calculated in two places. Amount and currency traveled through the payment port as two loose parameters that the compiler cannot keep apart.

DDD closes these gaps. Strictly speaking, this stage borrows only DDD's tactical patterns; strategic design (bounded contexts, ubiquitous language) is a topic of its own. The aggregate takes over the rules:

```java
public class Order extends AggregateRoot {

    public static Order place(OrderId id, CustomerId customerId, List<OrderLine> lines) {
        // invariants checked here; the only way to create a new order
        Order order = new Order(id, customerId, lines, OrderStatus.PLACED, null);
        order.registerEvent(new OrderPlaced(id, customerId, order.total()));
        return order;
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.CANCELLED)
            throw new OrderNotCancellableException(id, status);
        OrderStatus previous = status;
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(id, previous));
    }
}
```

An `Order` that exists is valid. There are no setters. Value objects like `Money` and `OrderId` replace `BigDecimal` and `UUID`, so rounding and currency checks happen exactly once and arguments cannot be swapped. The application service shrinks to choreography. A business `if` inside an application service is now a review smell: the rule probably belongs in the aggregate.

Domain events follow the same discipline. The aggregate collects them but does not publish them, because publishing would require a dependency on the outside. The application service publishes after saving, through a driven port (outbound). A second business context can then react without the first one knowing about it:

```java
@Component
class OrderPaidListener {                          // shipping, adapter.in.event

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void on(OrderPaid event) {
        scheduleShipment.scheduleShipment(new ScheduleShipmentCommand(event.orderId().value()));
    }
}
```

Spring dispatches by parameter type, buffered until the commit. New reactions can be added without touching the order context, and an ArchUnit rule keeps the boundary intact. If delivery must be guaranteed, an outbox adapter on the same event port writes the events into a table in the same transaction, and a relay delivers them at least once.

## What I would not skip

The stages are choices you make per service. A few things are not.

Keep the dependency rule and enforce it with ArchUnit. Keep driven ports (outbound) for everything you do not want running in a test: foreign systems, the clock, randomness. And keep the persistence model separate from the domain model. Putting JPA annotations on the domain class looks like a shortcut, but it breaks the rule at its most important spot. JPA then dictates default constructors, mutability, and proxies, and dirty checking saves state nobody asked to save. The mapper is the price. Pay it.

## Notas de la experiencia del proyecto

Two observations after using this in real projects. The testability is unbeatable, and by now I personally find this structure easier to work with than a layered architecture. The hard part is not the pattern. It is the other developers who are reluctant to follow the conventions. The pattern is less widely known than layering, so the same question comes up again and again: where does this class go?

A few simple rules answer it in almost every case:

- If it depends on a technology, it is an adapter behind a driven port (outbound). Database, HTTP client, message broker, mail: outside.
- If it is a call from the outside into the business logic, it belongs behind a driving port (inbound), a use case or service interface.
- For everything else, look at the import list. Only `java.*` and your own packages means core. A framework import means adapter.
- Put the core into its own Maven or Gradle module. Then the dependency rule is enforced by the compiler, and cyclic dependencies cannot happen in the first place.

> **A warning from experience.** Do not split that core further into one module per domain right away. You often want to share the domain model, cross dependencies show up anyway, and untangling them early costs more than it protects. One large core module is easier to start with; split it once the boundaries have proven stable.

And when a rule still gets debated, encode it in ArchUnit. Then the build answers the question instead of a code review.

> **One more shortcut that works in practice.** You can skip the inbound ports entirely and let the adapters reference the service classes directly. The services are technology neutral either way, so this is usually not a problem. Interfaces do buy you flexibility later, a second caller or a decorator slots in without touching the service, so I treat this as a matter of taste rather than a rule.

As for the stages: start with plain hexagonal and stay there for a while. Clean Architecture you can save yourself in most projects; the use case interfaces and the command objects rarely pay for what they cost. From DDD, the part worth taking early is the event decoupling. A second context that reacts to a domain event without the first one knowing about it solves a real problem, not a stylistic one.

## Resumen

Three sentences to take home. Dependencies point inward, everything else follows from that. The core owns its interfaces, which is why the port is named `PaymentPort` and not `StripeClient`. Clean Architecture and DDD are extension stages, not obligations, and each one answers a specific pain you should actually have before you pay for it.

The three example projects (hexagonal-pure, hexagonal-clean, hexagonal-ddd) implement the same application at each stage, so a diff between two of them shows exactly what a concept costs and buys. The code, the slides, and a companion script are on GitHub: <>.