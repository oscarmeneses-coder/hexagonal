# La arquitectura hexagonal es más fácil de lo que piensas.

*Tres etapas: hexagonal pura, arquitectura limpia, DDD. Solo necesitas la primera para empezar.*

La mayoría de las explicaciones de la arquitectura hexagonal muestran el conjunto completo de una sola vez: puertos, adaptadores, casos de uso, comandos, agregados, objetos de valor y eventos de dominio. Los estudiantes terminan pensando que el patrón es enorme, y los equipos lo rechazan porque "no necesitamos todo eso".

El patrón en sí es sencillo. Casi todo lo que se asocia con él, proviene de la Arquitectura Limpia o del Diseño Orientado a Dominios, que son decisiones independientes que se superponen. Este documento lo desarrolla en tres etapas para que puedas ver qué aporta cada concepto y cuál es su coste. Los ejemplos de cada etapa provienen de tres proyectos ejecutables de Spring Boot que implementan el mismo servicio de pedidos con la misma API HTTP; los fragmentos se han acortado para facilitar la lectura.

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

**![](././slides/img/schichten_es.svg)[`Capas` ]**

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

La nomenclatura importa. El puerto se llama `PaymentPort`, no `StripeClient`. Si el proveedor se muestra en la interfaz, el puerto está mal cortado. Esto es Inversión de Dependencias, aplicada consistentemente en cada límite del sistema. Antes: el servicio depende de `StripeClient` y no compilará sin él. Después: el servicio depende de una interfaz de su propiedad, y el adaptador Stripe implementa esa interfaz. La llamada sigue fluyendo hacia Stripe en tiempo de ejecución, pero la dependencia en tiempo de compilación apunta hacia adentro.

![](./slides/img/hexagon_es.svg)
**[`Hexagonal` ]**

*Todo el patrón en una sola imagen. La línea discontinua roja de la derecha indica el punto clave: la llamada se dirige hacia afuera, mientras que la dependencia apunta hacia adentro. A la izquierda, ambas direcciones coinciden, razón por la cual el lado controlado es el que suele construirse incorrectamente.*

La implementación del puerto de control (puerto de entrada) es Java puro. No `@Service`, no `@Transactional`, y no hay importación de Spring. Una clase de configuración conecta el núcleo y el framework:

```java
@Configuration
class ApplicationConfiguration {

    @Bean
    OrderService orderService(OrderRepository repo, PaymentPort payment) {
        return new TransactionalOrderService(new DefaultOrderService(repo, payment));
    }
}
```

Las transacciones son tecnología, por lo que un decorador las envuelve alrededor del puerto de entrada en lugar de colocar la anotación en el núcleo. En el lado de salida, el adaptador de persistencia se implementa `OrderRepository` con Spring Data, mapeando entre la clase de dominio y una entidad JPA independiente. El puerto de pago tiene dos adaptadores: uno real y otro en memoria activado por un perfil de Spring, de modo que toda la aplicación se ejecuta localmente sin ningún servicio externo.

Ese es el patrón completo. Un puñado de interfaces, un puñado de adaptadores, una sola clase de cableado.

## Lo que obtienes

La recompensa se aprecia en el conjunto de pruebas. El núcleo se prueba con dos falsificaciones escritas a mano, de unas cuarenta líneas cada una:

```java
@Test
void doesNotSaveWhenPaymentFails() {
    payment.failNext();
    assertThatThrownBy(() -> service.placeOrder("cust-1", "CHF", lines()))
            .isInstanceOf(PaymentFailedException.class);
    assertThat(repository.size()).isZero();
}
```

Esto se ejecuta en milisegundos sin Spring. En el proyecto de ejemplo, las siete pruebas principales tardan unos 10 ms, mientras que la prueba de humo de la aplicación completa tarda casi dos segundos. La cuestión no son las pruebas rápidas en sí, sino que el código con mayor tasa de cambio, la lógica de negocio, recibe la retroalimentación más rápida. En una `@SpringBootTest` suite típica, ocurre justo lo contrario.

La regla de dependencia en sí misma se vuelve ejecutable con ArchUnit:

```java
@ArchTest
static final ArchRule application_is_free_of_spring =
        noClasses().that().resideInAPackage("..application..")
                .should().dependOnClassesThat().resideInAnyPackage(
                        "org.springframework..", "jakarta.persistence..");
```

Sin una prueba de este tipo, cualquier arquitectura se deteriora en cuestión de sprints. No por mala intención, sino porque la importación rápida siempre prevalece bajo la presión de los plazos de entrega.

> **Vale la pena detenerse aquí.** Todo lo anterior es relativamente sencillo, lo que facilita la labor del desarrollador en lugar de complicarla. El código se mantiene conciso, al igual que el modelo mental: las particularidades técnicas se eliminan automáticamente, permitiendo concentrarse plenamente en la lógica de negocio. Y el argumento más sólido es la capacidad de prueba. Un núcleo que se puede probar en milisegundos, sin Spring y sin simular sistemas externos, es difícil de superar.

## Stage 2: Casos de uso y comandos

La arquitectura limpia mantiene la misma regla de dependencia, y la parte que los equipos adoptan realmente añade estructura al lado impulsor (los puertos de entrada). El libro va más allá, con límites de salida y presentadores en el camino de regreso, pero casi nadie los implementa en producción. En lugar de una inbterfaz `OrderService`, hay un puerto por caso de uso: `PlaceOrderUseCase`, `CancelOrderUseCase`, `GetOrderQuery`. Y en lugar de listas de parámetros, cada caso de uso toma un objeto de comando:

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

Con tres métodos en una interfaz, la división es teórica. Con veinte, cada llamador depende de la interfaz completa, y la división comienza a ser rentable. Los comandos valen la pena a partir de cuatro parámetros: nombran sus campos, copian listas de forma segura y se validan a sí mismos en el constructor compacto. El costo: cada caso de uso ahora trae consigo su propia interfaz y su propia clase de servicio. Para servicios con muchas operaciones CRUD, el patrón se ejecuta sin necesidad de una interfaz, por lo que esto es una etapa y no un requisito.

> **Una nota al margen sobre la reputación.** Sospecho que esta etapa es responsable de la reputación de la arquitectura hexagonal como código repetitivo. Convertir cada método de lectura y escritura en su propia interfaz resulta excesivo en muchos casos, y añadir un objeto de comando para cada método es probablemente el punto en el que la mayoría de los desarrolladores decidieron no molestarse. Este veredicto recae entonces sobre todo el patrón, aunque la etapa 1 nunca lo exigió.

## Stage 3: Un modelo de dominio que se defiende a sí mismo.

Hasta este punto, la clase de dominio era un contenedor de datos con métodos get y set, y las reglas de negocio residían en el servicio como una lista de verificación. Cualquiera que creara un pedido sin pasar por el servicio no verificaba nada. El total del pedido se calculaba en dos lugares. El importe y la moneda se transmitían a través del puerto de pago como dos parámetros independientes que el compilador no podía diferenciar.

DDD cierra estas brechas. Estrictamente hablando, esta etapa solo toma prestados los patrones tácticos de DDD; el diseño estratégico (contextos delimitados, lenguaje ubicuo) es un tema aparte. El agregado toma las reglas:

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

Un `Order` ue existe es válido. No hay setters. Los objetos de valor como `Money` y `OrderId` reemplazan a `BigDecimal` y `UUID`, por lo que las comprobaciones de redondeo y moneda se realizan exactamente una vez y los argumentos no se pueden intercambiar. El servicio de la aplicación se reduce a una coreografía. Un negocio `if` dentro de un servicio de aplicación ahora es un olor a revisión: la regla probablemente debería estar en el agregado.

Los eventos de dominio siguen la misma disciplina. El agregado los recopila, pero no los publica, ya que la publicación requeriría una dependencia externa. El servicio de aplicación los publica después de guardarlos, a través de un puerto controlado (saliente). Un segundo contexto de negocio puede reaccionar sin que el primero lo sepa.

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

Spring realiza el despacho por tipo de parámetro, con almacenamiento en búfer hasta la confirmación. Se pueden agregar nuevas reacciones sin modificar el contexto del pedido, y una regla de ArchUnit mantiene el límite intacto. Si se debe garantizar la entrega, un adaptador de bandeja de salida en el mismo puerto de evento escribe los eventos en una tabla dentro de la misma transacción, y un relé los entrega al menos una vez.

## Lo que no me saltaría

Las etapas son decisiones que tomas para cada servicio. Algunas cosas no lo son.

Mantén la regla de dependencia y aplícala con ArchUnit. Mantén puertos controlados (salientes) para todo lo que no quieras que se ejecute en una prueba: sistemas externos, el reloj, la aleatoriedad. Y mantén el modelo de persistencia separado del modelo de dominio. Poner anotaciones JPA en la clase de dominio parece un atajo, pero rompe la regla en su punto más importante. JPA entonces impone constructores predeterminados, mutabilidad y proxies, y la comprobación de cambios guarda un estado que nadie pidió guardar. El mapeador es el precio. Págalo.

## Notas de la experiencia del proyecto

Dos observaciones tras usarlo en proyectos reales. La capacidad de prueba es insuperable, y personalmente, ahora encuentro esta estructura más fácil de manejar que una arquitectura por capas. La dificultad no reside en el patrón en sí, sino en la reticencia de los demás desarrolladores a seguir las convenciones. El patrón es menos conocido que la arquitectura por capas, por lo que surge la misma pregunta una y otra vez: ¿dónde se ubica esta clase?

Unas pocas reglas sencillas lo solucionan en casi todos los casos:

- Si depende de una tecnología, es un adaptador detrás de un puerto controlado (saliente). Base de datos, cliente HTTP, message broker, correo electrónico: externo.
- Si se trata de una llamada desde el exterior hacia la lógica de negocio, debe estar detrás de un puerto de entrada (inbound), un caso de uso o una interfaz de servicio.
- Para todo lo demás, consulta la lista de importaciones. Solo `java.*` y tus propios paquetes significa core. Una importación de framework significa adapter.
- Coloca el núcleo en su propio módulo Maven o Gradle. De esta forma, el compilador aplicará la regla de dependencias y se evitarán las dependencias cíclicas.

> **Una advertencia basada en la experiencia.** no divida ese núcleo en módulos individuales por dominio de inmediato. A menudo se desea compartir el modelo de dominio, las dependencias cruzadas surgen de todos modos, y desenredarlas prematuramente cuesta más de lo que protege. Es más fácil comenzar con un módulo central grande; divídalo una vez que los límites se hayan estabilizado.

Y cuando una regla siga siendo objeto de debate, incorpórela a ArchUnit. De esta forma, la compilación responderá a la pregunta en lugar de una revisión de código.

> **Otro atajo que funciona en la práctica.** Puedes omitir por completo los puertos de entrada y dejar que los adaptadores hagan referencia directamente a las clases de servicio. Los servicios son tecnológicamente neutros en cualquier caso, así que esto no suele ser un problema. Las interfaces ofrecen flexibilidad posteriormente; un segundo llamador o un decorador se integra sin modificar el servicio, así que lo considero una cuestión de preferencia personal más que una regla.

En cuanto a las etapas: empieza con un diseño hexagonal simple y mantente así un tiempo. La arquitectura limpia te resultará útil en la mayoría de los proyectos; las interfaces de casos de uso y los objetos de comando rara vez justifican su coste. Desde la perspectiva de DDD, la parte que vale la pena abordar desde el principio es el desacoplamiento de eventos. Un segundo contexto que reacciona a un evento de dominio sin que el primero lo sepa resuelve un problema real, no uno estilístico.

## Resumen

Tres frases para recordar. Las dependencias apuntan hacia adentro, todo lo demás se deriva de eso. El núcleo controla sus interfaces, por eso el puerto se llama `PaymentPort` y no `StripeClient`. La arquitectura limpia y DDD son etapas de extensión, no obligaciones, y cada una responde a un problema específico que deberías tener antes de pagar por ella.

Los tres proyectos de ejemplo (hexagonal-pure, hexagonal-clean, hexagonal-ddd) implementan la misma aplicación en cada etapa, por lo que una comparación entre dos de ellos muestra exactamente cuánto cuesta y qué se obtiene con cada concepto. El código, las diapositivas y un script complementario están disponibles en GitHub.