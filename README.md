# Arquitectura hexagonal con Java 25 y Spring Boot

Material del curso para una **sesión de 120 minutos**: aproximadamente 70 minutos de presentación y 50 minutos de debate.
Público objetivo: desarrolladores Java experimentados que conozcan Spring Boot y la arquitectura clásica en capas.

## Contenido

```
hexagonal/
├── slides/
│   ├── hexagonal-arch-core.md      main talk: pure form → Clean → DDD
│   ├── hexagonal-arch-pitfalls.md  deep dive: costs, tuning knobs, events
│   ├── start.sh                    starts the Marp server (port 8081) with an overview page
│   └── img/
│       ├── hexagon.svg / .png      ports-and-adapters picture
│       ├── mapping.svg / .png      map of all translation points
│       ├── schichten.svg / .png    classic layered architecture (opening slide)
│       ├── *_en.svg / .png         the same diagrams with English labels
│       └── build_*.py              regenerate the SVGs (SVG_LANG=en for the _en variants)
├── course-material/
│   ├── script.md                   companion script: deep dives, tuning knobs, domain events
│   └── moderation.md               facilitation guide for the discussion
├── articulo/
│   └── hexagonal-architecture.md   Medium article (draft, Spanish)
├── hexagonal-pure/                   stage 1: hexagonal only - ports & adapters, DIP
├── hexagonal-clean/                  stage 2: + use cases and command objects (Clean Architecture)
└── hexagonal-ddd/                    stage 3: + DDD and domain events (41 tests, all green)
```

## Tres ejemplos de proyectos que se complementan entre sí.

Los tres proyectos implementan la misma aplicación empresarial (realizar, consultar y cancelar pedidos) con la misma 
API HTTP. La diferencia entre las dos etapas muestra exactamente qué aporta cada concepto y cuál es su coste:

| | `hexagonal-pure` | `hexagonal-clean` | `hexagonal-ddd` |
|---|---|---|---|
| Puertos y adaptadores, DIP, core sin Spring | ✅ | ✅ | ✅ |
| Casos de uso (un puerto por caso de uso) | ❌ | ✅ | ✅ |
| Objetos de comando | ❌ | ✅ | ✅ |
| Objetos de valor, agregados con comportamiento | ❌ | ❌ | ✅ |
| Domain events, outbox, second context | ❌ | ❌ | ✅ |

Cada proyecto comienza de la misma manera (`mvn spring-boot:run -Dspring-boot.run.profiles=demo`)
y distribuye una interfaz de usuario Swagger en  <http://localhost:8080/swagger-ui.html>.
Los detalles se encuentran en el archivo README de cada proyecto.

## Apertura de la sesión de laboratorio

**La forma más fácil:**

```bash
slides/start.sh
```

Inicia el servidor Marp en el puerto 8081 (para que un proyecto de ejemplo pueda ejecutarse en paralelo en el puerto
8080) y abre la página de resumen, donde se selecciona la presentación deseada. Se utilizan las teclas de flecha para 
navegar,  **`P`** se abre el modo presentador con notas del orador y una vista previa de la siguiente diapositiva,
**`F`** en pantalla completa. Los cambios en los archivos `.md` se actualizan automáticamente.

**Manualmente:**

```bash
npx @marp-team/marp-cli slides/hexagonal-architecture-core_en.md --preview
# or as a local server
npx @marp-team/marp-cli slides/ --server
```

**En VS Code:** instala la extensión *Marp for VS Code*, abre el archivo y haz clic en el icono de vista previa en la
parte superior derecha.

## Generando las diapositivas

Las cartas están en formato Markdown [Marp](https://marp.app) . Two ways:

**VS Code:** instala la extensión *Marp for VS Code*, open the file, start the preview.
Export via the command palette (`Marp: Export Slide Deck…`).

**Command line:**

```bash
npx @marp-team/marp-cli slides/hexagonal-architecture-pitfalls_en.md --pdf --allow-local-files
npx @marp-team/marp-cli slides/hexagonal-architecture-pitfalls_en.md --html
npx @marp-team/marp-cli slides/hexagonal-architecture-pitfalls_en.md --pptx   # if PowerPoint is needed
```

For presenter mode with speaker notes: export `--html` and press `P` in the
browser. The notes live as HTML comments inside the slides.

## Ejecutando un proyecto de ejemplo (stage 3, `hexagonal-ddd`)

Prerequisite: JDK 25.

```bash
cd example-ddd

mvn verify                                             # 41 tests, ~25 seconds
mvn spring-boot:run -Dspring-boot.run.profiles=demo    # no external payment service needed
```

Entonces:

```bash
curl -i -X POST http://localhost:8080/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"customerId":"cust-1","items":[{"productId":"p-1","quantity":2,"unitPrice":12.50,"currency":"CHF"}]}'

curl http://localhost:8080/api/orders/<id-from-location-header>
```

Hay más solicitudes (incluidos los casos de error 400 y 422) en
`hexagonal-ddd/orders.http`.

## Estructura de `hexagonal-ddd` (stage 3)

Dos áreas de negocio en una misma implementación. La comparación es lo importante.

### `org.unsch.orders` — la construcción completa

```
├── domain
│   ├── model            Order, AggregateRoot, Money, OrderLine, business errors
│   └── event            DomainEvent, OrderPlaced, OrderPaid, OrderCancelled
├── application
│   ├── port.in          PlaceOrderUseCase, CancelOrderUseCase, GetOrderQuery
│   ├── port.out         LoadOrderPort, SaveOrderPort, PaymentPort, OrderEventPublisher
│   └── service          PlaceOrderService, CancelOrderService, GetOrderService
├── adapter
│   ├── in.web           OrderController, DTOs, OrderExceptionHandler
│   ├── in.tx            transaction decorators around the driving ports
│   ├── out.persistence  OrderJpaEntity, OrderLineJpaEntity (@OneToMany), adapter, mapper
│   ├── out.payment      PspPaymentAdapter (RestClient), InMemoryPaymentAdapter (profile "demo")
│   ├── out.event        SpringOrderEventPublisher, OrderEventLogger
│   └── out.outbox       OutboxOrderEventPublisher, OutboxRelay (profile "outbox")
└── config               ApplicationConfiguration (the single wiring point)
```

`domain` y `application` no contienen **ni una sola importación de Spring o JPA**.
`HexagonalArchitectureTest` lo impone con ArchUnit en la compilación normal.

### `org.unsch.shipping` — misma estructura, conectada a través de un evento

```
├── domain.model         Shipment, ShipmentStatus — behavior with preconditions, no setters
├── application
│   ├── port.in          ScheduleShipmentUseCase, DispatchShipmentUseCase, GetShipmentQuery
│   ├── port.out         LoadShipmentPort, SaveShipmentPort, TrackingPort
│   └── service          ScheduleShipmentService (idempotent), DispatchShipmentService, GetShipmentService
├── adapter
│   ├── in.event         OrderPaidListener — the driving adapter for events
│   ├── in.web           ShipmentController
│   ├── in.tx            transaction decorators
│   ├── out.persistence  ShipmentJpaEntity, ShipmentPersistenceAdapter, mapper
│   └── out.tracking     LoggingTrackingAdapter (external provider)
└── config               ShippingConfiguration
```

Lo interesante es el `adapter.in.event`: un adaptador de control es un adaptador de control, ya sea que el 
desencadenante sea HTTP o un evento. El `OrderPaidListener`
Traduce el evento del contexto vecino en una llamada a su propio puerto de control 
(`ScheduleShipmentUseCase`). Cómo llegan los eventos (bus de eventos de Spring, despacho por tipo de 
parámetro, `AFTER_COMMIT`, `REQUIRES_NEW`) está documentado paso a paso en el Javadoc de la clase.
`ShippingArchitectureTest` aplica las mismas reglas para `shipping` que para `orders`.

### Eventos de dominio

`Order` extiende `AggregateRoot` y registra sus propios eventos; el servicio de la aplicación los publica
**después de** guardarlos, a través del puerto `OrderEventPublisher`.
Tres adaptadores en el mismo puerto:

| Adapter | Reach | Delivery |
|---|---|---|
| `SpringOrderEventPublisher` (default) | this deployment | no guarantee |
| `OutboxOrderEventPublisher` (profile `outbox`) | foreign systems | at least once |
| `RecordingEventPublisher` (test) | none | — |

El `shipping` se engancha a `OrderPaid`; `orders` no sabe nada de envíos.
`ContextBoundariesTest` lo fija con ArchUnit.

Pruébalo:

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=demo,outbox   # events go into the outbox table
```

## Los diagramas

All hand-drawn (no licensing questions), available as SVG **and** PNG under
`slides/img/`. The corresponding `build_*.py` scripts regenerate them; content
sits in a list at the top of each script, layout below. Every diagram also
exists with English labels (`*_en.svg`, used by the `_en` decks); generate with
`SVG_LANG=en python3 build_*.py`.

**`schichten.svg`** shows classic layered architecture as the starting point:
DTOs flow upward, but all dependencies point downward at the technology, and
the entity serves as schema and domain model at the same time.

**`hexagon.svg`** shows ports and adapters on the concrete example code. The
real content is the red dashed line on the right side: the *call* goes outward,
the *dependency* points inward. On the left both directions coincide, which is
why nobody notices there, and why the right side is built wrong so often.

**`mapping.svg`** — see below.

## Donde se realiza la traducción (en `hexagonal-ddd`)

`slides/img/mapping.svg` shows all translation points at a glance. Two gaps to
keep apart: **outside world ↔ adapter model** is handled by the framework
(Jackson, Hibernate); **adapter model ↔ core** is written by you. In the order
context that is ten places:

| # | Translation | Where |
|---|---|---|
| 1 | `PlaceOrderRequest` → `PlaceOrderCommand` | `OrderWebMapper.toCommand` |
| 2 | `PlaceOrderCommand` → `Order`, `OrderLine` | `PlaceOrderService` |
| 3 | `Order` → `OrderResponse` | `OrderWebMapper.toResponse` |
| 4 | business error → `ProblemDetail` | `OrderExceptionHandler` |
| 5 | `Order` → `OrderJpaEntity` | `OrderPersistenceMapper.toEntity` |
| 6 | `OrderJpaEntity` → `Order` | `toDomain` + `Order.restore` |
| 7 | `OrderId`, `Money` → `ChargeRequest` | `PspPaymentAdapter` |
| 8 | `ChargeResponse` → `PaymentReference` | `PspPaymentAdapter` |
| 9 | domain event → event payload | **missing** — see `OutboxOrderEventPublisher` |
| 10 | `OrderPaid` → `UUID` in the neighboring context | `OrderPaidListener` |

`org.unsch.shipping` follows the same pattern with four hand-written
places (event → `UUID` in the listener, `Shipment` → response, persistence
there and back). The full accounting — which tuning knob removes which
mapping, and why 4 through 10 stay — is in `course-material/skript_en.md`,
chapter 6.2.

## Tiempos de ejecución de las pruebas (medidos en `hexagonal-ddd`)

| Test | Level | Tests | Runtime |
|---|---|---|---|
| `OrderTest` | domain, no framework | 7 | 0.004 s |
| `ShipmentTest` | domain, no framework | 4 | 0.004 s |
| `PlaceOrderServiceTest` | use case with fakes | 3 | 0.004 s |
| `CancelOrderServiceTest` | use case with fakes | 3 | 0.006 s |
| `OrderControllerTest` | `@WebMvcTest` | 4 | 0.4 s |
| `OrderPersistenceAdapterTest` | `@DataJpaTest` + H2 | 2 | 0.14 s |
| `OrdersApplicationSmokeTest` | `@SpringBootTest`, profile `demo` | 1 | 0.02 s |
| `OrderPaidListenerTest` | event across the context boundary | 3 | 1.9 s¹ |
| `HexagonalArchitectureTest` | ArchUnit, 7 rules | 7 | 0.2 s |
| `ShippingArchitectureTest` | ArchUnit, 5 rules | 5 | 0.35 s |
| `ContextBoundariesTest` | ArchUnit, context boundaries | 2 | 0.14 s |

<sup>¹ carries the context start as the first Spring test; the remaining
Spring tests reuse the context.</sup>

The business logic, the part with the highest rate of change, is tested in
under 100 milliseconds.

## Tecnología

Java 25 · Spring Boot 4.1.1 · H2 (in-memory) · JUnit 5 · AssertJ · ArchUnit 1.5.0