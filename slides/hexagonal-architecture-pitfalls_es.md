---
marp: true
theme: default
paginate: true
size: 16:9
header: 'Hexagonal Architecture with Java & Spring Boot'
footer: '60 minutes · Talk + Discussion'
style: |
  section {
    font-size: 25px;
    padding: 50px 60px;
  }
  section.lead {
    text-align: center;
  }
  h1 { color: #1c3d5a; font-size: 44px; }
  h2 { color: #1c3d5a; font-size: 34px; border-bottom: 2px solid #d8e2ea; padding-bottom: 6px; }
  code { font-size: 0.85em; }
  pre { font-size: 19px; line-height: 1.35; }
  section.code pre { font-size: 17px; }
  table { font-size: 22px; }
  blockquote {
    border-left: 5px solid #1c3d5a;
    padding-left: 18px;
    color: #33475b;
  }
  .cols { display: grid; grid-template-columns: 1fr 1fr; gap: 28px; }
  .small { font-size: 20px; }
  .muted { color: #6b7c8c; }
  footer, header { color: #8a9aa8; }
---

<!-- _class: lead -->
<!-- _paginate: false -->

# Hexagonal Architecture - inconvenientes

## Puertos y adaptadores con un ejemplo práctico
### Java 25 · Spring Boot 4.1

<br>

<span class="muted">60 minutos: ~35 minutos de información, ~25 minutos de debate.</span>

<!--
Welcome. Two sentences on the setup: the code we're about to see is a
runnable Maven project with 32 green tests. Everything on the slides
exists exactly like that in the repository.

Set expectations: this is not a talk about the perfect architecture, but
about a specific cost-benefit calculation. The interesting part is the
second half.
-->
---

## Lo que cuesta todo esto

<div class="cols">
<div>

**Más archivos**
El caso de uso "realizar pedido" afecta al dominio, puerto, servicio, DTO, mappper, adapter.

**Mappers**
Entre DTO web, entidad JPA y objeto de dominio: al menos dos traducciones por dirección.

</div>
<div>

**Indirección**
"¿Quién implementa esto realmente?" se convierte en una práctica habitual en los entornos de desarrollo integrados (IDE).

**Barrera de entrada**
Los nuevos miembros del equipo necesitan el mapa; de lo contrario, tendrán que adivinar dónde colocar su código.

</div>
</div>

<br>

> El precio es real y a menudo se subestima. Es prácticamente constante; el beneficio aumenta con la duración 
> y la complejidad de la lógica empresarial.

<!--
Definitely keep this slide in. A talk that only shows advantages loses
credibility with experienced devs - and the objections will come up in
review anyway.

The cost/benefit curve is the honest core: constant cost, growing benefit.
The next slide follows directly from that.
-->

---

## ¿Qué es lo molesto del modelo de Arquitectura Limpia?

<div class="cols">
<div class="small">

**Un caso de uso = seis archivos**
Puerto, comando, servicio, decorador, DTO, mapper. Para "cambiar dirección".

**Una interfaz con exactamente una implementación**
La segregación es teórica siempre que haya un único llamador.

**`Ctrl+Click` aterriza en la interfaz**
Cada navegación es un salto adicional.

</div>
<div class="small">

**Mantenimiento del Mapper**
Tocar un nuevo campo significa cuatro lugares: DTO, comando, agregado, entidad.

**Los casos de uso CRUD se ejecutan vacíos**
`repository.save(mapper.map(dto))` — el agregado no tiene ninguna regla.

**Proceso de incorporación**
Sin un mapa, la nueva compañera adivina dónde debe ir su código.

</div>
</div>

<br>

> La cuestión no es "todo o nada", sino: **¿qué partes llevo conmigo y cuáles no?**

<!--
This slide is important for credibility. If you only defend the pattern,
you lose the room with experienced devs - the objections come anyway, so
better to state them yourself, precisely.

The last two points on the right are the most serious: a pattern running
empty and a team that doesn't carry it are the two most common reasons
things derail.
-->

---

<!-- _footer: '' -->

## Donde el mapeo se realiza en todas partes (una vez finalizada la construcción)

<style scoped>img { display: block; margin: 6px auto 0; }</style>

![w:1080](img/mapping_es.svg)

<!--
The map that's missing from almost every discussion. Two messages:

1. The gap on the left is handled by the framework - nobody talks about
   that, and rightly so. The gap on the right you write yourself: ten
   places in the order context.

2. Four of them you expect (DTO, entity, PSP). The other six come as a
   surprise: error-to-status-code is a mapping, the event payload is one,
   and so is the translation at the context boundary.

The red line is the one deliberately left open in the example: there the
domain event goes raw to external systems - see the outbox slide.

For comparison, if someone asks: the lean `shipping` context has exactly
TWO hand-written mappings instead of ten. That's the math behind the
tuning knobs on the next slide.
-->

---

## Cinco perillas de afinación, cada una ajustable individualmente.

<style scoped>table { font-size: 18px; }</style>

<div class="small">

| Ajuste | Lo que ahorra | Lo que cuesta |
|---|---|---|
| **Elimina `port.in`**, usa las clases `@Service` directamente | 2  archivos por caso de uso | El controlador depende de la clase; Spring en el núcleo |
| **Elimina los objetos de comando**, use plain parameters | 1  archivo por caso de uso | Las firmas crecen; no hay autovalidación. |
| **Spring Data directamente** en lugar de cargar/guardar puerto | 3  archivos por caso de uso | Pruebas de base de datos en lugar de pruebas unitarias; `Page<T>` en el núcleo |
| **`@Transactional` en el servicio** en lugar de decorador | 1  archivo por caso de uso | Core ya no está libre de Spring |
| **DTO Web = comando** | 1 mapper | API y core están unidos. |

</div>

<br>

> La última fila es solo para API internas. **No es un parámetro de ajuste**: entidad JPA como agregado: eso rompe la regla en lugar de flexibilizarla.

<!--
This slide is the actual core of the answer to "this is too much for me".
The important message: these are five INDEPENDENT decisions, not a package
deal. You can keep ports and drop commands, or vice versa.

Make the bottom line explicit: "JPA entity as aggregate" deliberately does
NOT appear in the table. That's not a ceremony saving, it gives up the
separate domain model - JPA then dictates constructors, proxies and
mutability, and jakarta.persistence sits in the core. We advise against it.

What I would NOT drop otherwise, no matter how small the project:
- business logic in the aggregate instead of the service
- driven ports for external systems (payment, shipping, mail)
- the dependency rule itself, enforced via ArchUnit

That's the core; everything else is ceremony you can afford or not.
-->

---
<!-- _class: code -->

---

## Aquí es donde entran en juego los eventos DDD.

<style scoped>pre { font-size: 15px; }</style>

A domain event is a **statement about something that happened** — past tense, business language:
`OrderPlaced`, `OrderPaid`, `OrderCancelled`. No `OrderCreatedEvent`, no `OrderChangedNotification`.

<div class="cols">
<div class="small">

**Who creates them?**
The aggregate, at the very moment the rule applies. It does *not publish* them — that would be an outward dependency.

</div>
<div class="small">

**Who sends them?**
The application service, **after** saving, via a driven port. Exactly one port for all event types.

</div>
</div>

```java
public abstract class AggregateRoot {
    private final List<DomainEvent> domainEvents = new ArrayList<>();
    protected void registerEvent(DomainEvent event) { domainEvents.add(event); }

    public List<DomainEvent> pullDomainEvents() {          // deliverable exactly once
        List<DomainEvent> pulled = List.copyOf(domainEvents);
        domainEvents.clear();
        return pulled;
    }
}
```

<!--
The separation "collect vs. send" is the whole trick. An aggregate that
publishes itself would need the publisher as a field - and with that the
domain would have an outward dependency.

Important detail: Order.restore() from the persistence adapter registers
NO events. Loading is not a business occurrence. Miss that, and you send
events on every read access.

Spring Data offers the same thing ready-made with AbstractAggregateRoot
and @DomainEvents - but then Spring Data sits in the aggregate. Another
tuning knob.
-->

---
<!-- _class: code -->

## Save first, then announce — and one port for all

<style scoped>pre { font-size: 15px; }</style>

```java
// in the aggregate
public void markPaid(PaymentReference reference) {
    if (status != OrderStatus.PLACED) throw new IllegalStateException(...);
    this.status = OrderStatus.PAID;
    registerEvent(new OrderPaid(id, total(), reference));
}

// in the application service
saveOrderPort.save(order);
eventPublisher.publish(order.pullDomainEvents());     // the order is intentional
```

<div class="small">

| Adapter on the port | Reach | Delivery | When |
|---|---|---|---|
| `SpringOrderEventPublisher` | this deployment | no guarantee | default case, coupling within the process |
| `OutboxOrderEventPublisher` | external systems | at least once | when loss hurts |
| `RecordingEventPublisher` (test) | none | — | use-case tests without the framework |

</div>

<!--
One port, three adapters, swappable via profile - the same pattern as with
payment. This is the moment the structure pays off for the second time.

"Save first, then announce": the other way around you announce things that
may never have happened. Sounds trivial, but is wrong in a frightening
number of codebases.

One port with publish(List<DomainEvent>) instead of one method per event
type: new events then no longer change the port.
-->

---
<!-- _class: code -->

## Across the context boundary: `orders` knows nothing about `shipping`

```java
@Component
class OrderPaidListener {                                    // in the shipping package

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)   // ← without this: silent data loss
    void on(OrderPaid event) {
        scheduleShipment.scheduleShipment(
                new ScheduleShipmentCommand(event.orderId().value()));
    }
}
```

<div class="small">

**The pitfall you have to experience once:** after `AFTER_COMMIT` the transaction is
complete, but its `EntityManager` is still bound. A plain `@Transactional` attaches to it
and never commits — nothing is saved, nothing is logged. `REQUIRES_NEW` is mandatory.

**The visible coupling:** `shipping` knows the event type from `orders` and immediately translates it
into its own terms (`UUID` instead of `OrderId`). Across a real system boundary, a dedicated
integration event would be due here.

</div>

<!--
This pitfall happened to me while building the example - the tests were
green-looking but empty. That's exactly why it's on the slide: in real
life it costs half a day of searching.

ArchUnit enforces the boundary in both directions: ContextBoundariesTest
forbids orders -> shipping entirely and allows shipping -> orders only
into the domain.

If someone asks why not just a method call: because then the order context
would know about shipping, and two weeks later also about invoicing, the
loyalty program and the newsletter.
-->

---
<!-- _class: code -->

## When delivery must be guaranteed: Outbox

```java
@Component @Profile("outbox")
class OutboxOrderEventPublisher implements OrderEventPublisher {
    public void publish(List<DomainEvent> events) {
        events.forEach(e -> repository.save(
                new OutboxEntry(e.getClass().getSimpleName(), toJson(e), clock.instant())));
    }
}                                    // same transaction as the order: both or nothing
```

```
Outbox -> Broker: OrderPaid {"orderId":{"value":"769e..."},
                             "total":{"amount":49.90,"currency":"CHF","positive":true}}
```

<span class="small">This is **exactly** where missing translation takes its revenge: `positive` comes from `Money.isPositive()` and is
from now on part of the contract with external systems. Between event and wire there belongs a flat
payload DTO with its own versioning — the same rule as for the web adapter.</span>

<!--
The log excerpt is real, from the running example. Deliberately not
"cleaned up": in two lines it shows why domain objects don't belong on
the wire.

The relay is a @Scheduled job that delivers and marks the row as done.
"At least once" delivery - the receivers must be idempotent, just like
the ScheduleShipmentService in shipping.
-->

---

## What typically goes wrong with events

<div class="small">

1. **Announced before commit.** The receiver reacts to something that, after the rollback, never happened.
2. **Event where a command was meant.** `SendConfirmationMail` is not an event but an order — a port is more honest then.
3. **Domain objects serialized.** An internal rename breaks external systems (see previous slide).
4. **Receivers not idempotent.** "At least once" means: it arrives twice. Always.
5. **Events as a database substitute.** Packing the whole state into the event only relocates the coupling.
6. **No versioning plan.** The first field an event loses breaks a receiver nobody knows about.

</div>

<br>

> Events decouple components — and couple you to a schema in return. The contract moves, it does not disappear.

<!--
Point 2 is the underrated one: a great many "events" are actually commands
in disguise. Test: does the sender care whether anyone is listening? Then
it's a command, and a driven port is the more honest modeling.

The takeaway line at the bottom is a good transition into the discussion
if time is already short.
-->

---

## When is it worth it?

<div class="cols">
<div>

### Yes
- business logic with real rules and states
- lifespan > 2–3 years
- multiple input channels (REST + batch + events)
- swappable or unreliable external integrations
- a team that actually sticks to the rule

</div>
<div>

### Rather not
- CRUD over a single table
- prototype, PoC, throwaway service
- pure data transformation job
- small team, short lifetime, stable environment

</div>
</div>

<br>

<span class="small">Middle path: don't tighten all the tuning knobs — but the dependency rule, ports in front of external systems, and the separate persistence model stay.</span>

<!--
The middle-path line is practically the most important of the talk: you
don't have to take the pattern all or nothing - port.in, commands and the
decorator are negotiable. Non-negotiable: the dependency rule itself,
ports in front of external systems, and the separate persistence model.
Using the JPA entity as the domain model is not a shortcut, it breaks the
rule at its most important spot.

The last point on the left is a team point, not a technical one: an
architecture nobody understands or defends is gone after a year.
-->

---

## Legacy code: a viable path

1. **ArchUnit first** — write the rules, freeze the existing code with `FreezingArchRule`. From now on it doesn't get worse.
2. **Pick one use case** — the one that hurts the most.
3. **Extract a driven port** — interface in the core, the existing client becomes the adapter. Don't change behavior.
4. **Pull business logic into the aggregate** — rule by rule, each secured with a fast test.
5. **Only then** think about a separate persistence model — if needed at all.

<br>

> Step 3 alone already gives you testable use cases. None of this needs a big bang.

<!--
The most common question after a talk like this is "and now, with our
400,000 lines?". This slide is the answer.

Step 1 is the most important and costs half a day. Step 5 is optional -
many teams stop at 4, and that's fine.
-->

---

## Three sentences to take away

<br>

### 1. Dependencies point inward.
Everything else follows from that.

### 2. The core dictates the interfaces, not the technology.
The port is called `PaymentPort`, not `StripeClient`.

### 3. The costs are constant, the benefit grows.
That's why it is a decision — not a rule.

<!--
Pause briefly here and really let the three sentences sink in before the
discussion opens up.
-->

---

<!-- _class: lead -->

# Discussion

<span class="muted">~25 minutes</span>

<!--
From here on, moderate, don't lecture. The questions on the next slide are
starting points - if something original comes from the audience, that
takes priority.

Detailed moderation guide with counterarguments: course-material/moderation.md
-->

---

## Where do we start?

<div class="small">

1. **Where in our codebase would the dependency rule break today?** And what did that cost us most recently?
2. **Which of the five tuning knobs do we turn?** Concretely: keep `port.in` or drop it — and why?
3. **Where do we use JPA entities as our domain model today** — and what does that cost us?
4. **`charge()` runs inside the transaction.** Outbox, saga, or compensating transaction?
5. **Where do we already have events today — and are they really events?** Or commands in disguise?
6. **How do we cut our contexts,** and who enforces the boundaries?
7. **Do we introduce ArchUnit?** And who fixes the first red build?
8. **Where would we deliberately opt out** — and how do we recognize that in time?

</div>

<!--
Don't go through all eight - two or three in depth are worth more than
eight skimmed.

In my experience 1, 2 and 4 carry the furthest. Question 1 first, because
it's concrete and everyone has something to contribute.

If the room stays silent: put question 2 to a vote (show of hands per
tuning knob) - that gets the positions on the table immediately.

Detailed moderation guide with counterarguments: course-material/moderation.md
-->

---

## Further reading

<div class="small">

**Origin**
Alistair Cockburn — *Hexagonal Architecture* (2005), `alistair.cockburn.us/hexagonal-architecture`

**Most direct for Java developers**
Tom Hombergs — *Get Your Hands Dirty on Clean Architecture* (2nd edition, 2023)

**The core, cut along business lines**
Eric Evans — *Domain-Driven Design* (2003) · Vaughn Vernon — *Implementing DDD* (2013)

**Context**
Robert C. Martin — *Clean Architecture* (2017) · Jeffrey Palermo — *The Onion Architecture* (2008)

**Tooling**
ArchUnit — `archunit.org`

</div>

<br>

<span class="muted">Example project, script, and moderation guide are in the repository.</span>

<!--
If only one book: Hombergs. It's short, Java, and covers exactly the
questions that come up in everyday work.
-->

---

<!-- _footer: '' -->

## Appendix: the ten mapping points in detail

<style scoped>table { font-size: 13.5px; } h2 { font-size: 30px; }</style>

| # | Translation | Where in the code | Why — and what happens without it |
|---|---|---|---|
| 1 | `PlaceOrderRequest` → `PlaceOrderCommand` | `OrderWebMapper.toCommand` | API evolves independently; without it: a rename in the aggregate breaks clients |
| 2 | `PlaceOrderCommand` → `Order`, `OrderLine` | `PlaceOrderService` | core works only with its own types; cheap, the command already carries value objects |
| 3 | `Order` → `OrderResponse` | `OrderWebMapper.toResponse` | prevents the internal structure from becoming the API |
| 4 | Business error → `ProblemDetail` | `OrderExceptionHandler` | **also a mapping**; without it: HTTP status leaks into the domain |
| 5 | `Order` → `OrderJpaEntity` | `OrderPersistenceMapper.toEntity` | decouple schema and domain model |
| 6 | `OrderJpaEntity` → `Order` | `toDomain` + `Order.restore` | without it: lazy proxies and detached-entity trouble in the core |
| 7 | `OrderId`, `Money` → `ChargeRequest` | `PspPaymentAdapter` | vendor contract stays in the adapter |
| 8 | `ChargeResponse` → `PaymentReference` | `PspPaymentAdapter` | foreign model does not enter the core |
| 9 | Domain event → event payload | **missing** in `OutboxOrderEventPublisher` | without it: internal structure becomes a contract with external systems |
| 10 | `OrderPaid` → `UUID` in the neighboring context | `OrderPaidListener` | context boundary; across a real system boundary: integration event |

<span class="small">Tuning knobs eliminate 1, 2, 3. **4 through 10 stay** — 5 and 6 because the persistence model remains separate; at 4 and 7–10 a foreign contract begins.</span>

<!--
Backup slide for the discussion. Don't go through it in the talk - bring
it up when someone digs into question 3 (persistence model) or the
mapping effort.

The closing line is the actual insight: only mappings 1-3 (web DTO,
command) can be cut. 5 and 6 stay because we don't give up the separate
persistence model - and 4, 7-10 sit on foreign contracts. Cutting there
doesn't save ceremony, it gives up protection.
-->
