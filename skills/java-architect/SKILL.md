---
name: java-architect
description: Enterprise Java: Spring Boot 3, virtual threads (Project Loom), records, pattern matching, GraalVM native images, and modernization strategies.
---

# Java Architect

Design and build enterprise Java applications using modern Java 21+, Spring Boot 3, virtual threads, and GraalVM native images. Covers idiomatic patterns, security hardening, and architecture decisions for production-grade systems.

## When to Use

- Designing Spring Boot 3 microservices or modular monoliths
- Modernizing legacy Java codebases (Java 8/11 to 21+)
- Adopting virtual threads to replace reactive or thread-pool concurrency
- Configuring GraalVM native-image builds for fast startup
- Implementing DDD, hexagonal architecture, or CQRS in Java
- Setting up Spring Security with OAuth2/JWT

## Table of Contents

1. [Modern Java Features](#1-modern-java-features)
2. [Virtual Threads](#2-virtual-threads)
3. [Spring Boot 3](#3-spring-boot-3)
4. [Spring Data](#4-spring-data)
5. [Spring Security](#5-spring-security)
6. [GraalVM Native](#6-graalvm-native)
7. [Testing](#7-testing)
8. [Architecture Patterns](#8-architecture-patterns)

---

## 1. Modern Java Features

### Records

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount must be non-negative");
        Objects.requireNonNull(currency, "Currency is required");
    }
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new CurrencyMismatchException(this.currency, other.currency);
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

**Anti-pattern:** Do not use records as JPA entities. Records are immutable and lack a no-arg constructor.

```java
// BAD — will fail at runtime
@Entity public record Order(Long id, String status) {}

// GOOD — use records for DTOs/projections only
public record OrderSummary(Long id, String status, BigDecimal total) {}
```

### Sealed Classes and Pattern Matching

```java
public sealed interface PaymentMethod
        permits CreditCard, BankTransfer, DigitalWallet {}

public record CreditCard(String number, YearMonth expiry) implements PaymentMethod {}
public record BankTransfer(String iban, String bic) implements PaymentMethod {}
public record DigitalWallet(String provider, String token) implements PaymentMethod {}

public BigDecimal calculateFee(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc    -> cc.expiry().isBefore(YearMonth.now())
                ? BigDecimal.ZERO : new BigDecimal("0.029");
        case BankTransfer bt  -> new BigDecimal("0.005");
        case DigitalWallet dw -> switch (dw.provider()) {
            case "ApplePay", "GooglePay" -> new BigDecimal("0.015");
            default -> new BigDecimal("0.025");
        };
    }; // No default — compiler enforces exhaustiveness for sealed types
}
```

**Anti-pattern:** Avoid `instanceof` chains when sealed types with pattern matching give compile-time exhaustiveness.

### Text Blocks and `var`

```java
String query = """
        SELECT o.id, o.status, o.total FROM orders o
        WHERE o.customer_id = :customerId AND o.created_at > :since
        ORDER BY o.created_at DESC
        """;

var orders = orderRepository.findRecentByCustomer(customerId); // type obvious
```

**Anti-pattern:** Do not use `var` when the type is unclear: `var result = service.process(input);`

---

## 2. Virtual Threads

Virtual threads (Project Loom, Java 21) allow millions of concurrent tasks without platform-thread overhead.

### Creating Virtual Threads

```java
Thread.startVirtualThread(() -> {
    var data = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
    processResponse(data);
});

ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
List<Future<OrderResult>> futures = orders.stream()
        .map(order -> executor.submit(() -> processOrder(order)))
        .toList();
for (var future : futures) {
    results.add(future.get()); // blocking is fine on virtual threads
}
```

### Structured Concurrency (Preview)

```java
OrderDetails fetchOrderDetails(Long orderId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var orderTask    = scope.fork(() -> orderService.findById(orderId));
        var customerTask = scope.fork(() -> customerService.findByOrderId(orderId));
        var linesTask    = scope.fork(() -> orderLineService.findByOrderId(orderId));
        scope.join();
        scope.throwIfFailed();
        return new OrderDetails(orderTask.get(), customerTask.get(), linesTask.get());
    } // All subtasks guaranteed complete — no leaked threads
}
```

### Spring Boot Integration

```properties
# application.properties — enables virtual threads for Tomcat + async executors
spring.threads.virtual.enabled=true
```

### Migration Anti-Patterns

```java
// BAD: synchronized pins the virtual thread to a platform thread
public synchronized String fetchData() {
    return httpClient.send(request, BodyHandlers.ofString()).body();
}

// GOOD: ReentrantLock allows virtual threads to unmount while waiting
private final ReentrantLock lock = new ReentrantLock();
public String fetchData() {
    lock.lock();
    try { return httpClient.send(request, BodyHandlers.ofString()).body(); }
    finally { lock.unlock(); }
}

// BAD: pooling defeats the purpose of virtual threads
ExecutorService pool = Executors.newFixedThreadPool(100);
// GOOD: one virtual thread per task
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

---

## 3. Spring Boot 3

Spring Boot 3 requires Java 17+ and Jakarta EE 9+ (`jakarta.*` namespace).

### Configuration Properties

```java
@ConfigurationProperties(prefix = "app.billing")
@Validated
public record BillingProperties(
        @NotBlank String apiKey,
        @Positive int maxRetries,
        @DurationUnit(ChronoUnit.SECONDS) Duration timeout,
        @DefaultValue("true") boolean auditEnabled
) {}
```

```yaml
app:
  billing:
    api-key: ${BILLING_API_KEY}
    max-retries: 3
    timeout: 30s
    audit-enabled: true
```

### Profiles

```yaml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
---
spring.config.activate.on-profile: local
spring.datasource.url: jdbc:h2:mem:devdb
---
spring.config.activate.on-profile: prod
spring.datasource:
  url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
  hikari:
    maximum-pool-size: 20
    minimum-idle: 5
```

### Actuator

```yaml
management:
  endpoints.web.exposure.include: health, info, metrics, prometheus
  endpoint.health:
    show-details: when-authorized
    probes.enabled: true  # /actuator/health/liveness + /readiness
```

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    private final PaymentGatewayClient client;
    public PaymentGatewayHealthIndicator(PaymentGatewayClient client) { this.client = client; }

    @Override
    public Health health() {
        try { client.ping(); return Health.up().withDetail("gateway", "reachable").build(); }
        catch (Exception e) { return Health.down(e).withDetail("gateway", "unreachable").build(); }
    }
}
```

### Docker Support

```bash
# Buildpack-based (no Dockerfile needed)
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest
```

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## 4. Spring Data

### JPA Entities and Repositories

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING) @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    @Version private Integer version; // optimistic locking

    protected Order() {}
    public Order(OrderStatus status) { this.status = status; this.createdAt = Instant.now(); }
}
```

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByStatusAndCreatedAtAfter(OrderStatus status, Instant since);

    @Query("""
            SELECT new com.example.dto.OrderSummary(o.id, o.status, SUM(ol.price * ol.quantity))
            FROM Order o JOIN o.lines ol WHERE o.status = :status
            GROUP BY o.id, o.status
            """)
    List<OrderSummary> findSummariesByStatus(@Param("status") OrderStatus status);

    @Query(value = "SELECT * FROM orders WHERE status = :status FOR UPDATE SKIP LOCKED LIMIT :limit",
           nativeQuery = true)
    List<Order> findAndLockForProcessing(@Param("status") String status, @Param("limit") int limit);
}
```

### Specifications for Dynamic Queries

```java
public class OrderSpecs {
    public static Specification<Order> hasStatus(OrderStatus s) {
        return (root, q, cb) -> cb.equal(root.get("status"), s);
    }
    public static Specification<Order> createdAfter(Instant since) {
        return (root, q, cb) -> cb.greaterThan(root.get("createdAt"), since);
    }
}

// Compose dynamically
var spec = Specification.where(hasStatus(PENDING)).and(createdAfter(oneWeekAgo));
var orders = orderRepository.findAll(spec, PageRequest.of(0, 50));
```

### Flyway Migrations

```sql
-- V1__create_orders.sql
CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    status     VARCHAR(32) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version    INTEGER     NOT NULL DEFAULT 0
);
CREATE INDEX idx_orders_status_created ON orders (status, created_at);
```

**Anti-pattern:** Never use `ddl-auto=update` in production. Use Flyway/Liquibase and set `ddl-auto=validate`.

### JOOQ for Complex Queries

```java
@Repository
public class OrderQueryRepository {
    private final DSLContext dsl;
    public OrderQueryRepository(DSLContext dsl) { this.dsl = dsl; }

    public List<OrderSummary> findTopOrders(int limit) {
        return dsl.select(ORDERS.ID, ORDERS.STATUS,
                        sum(ORDER_LINES.PRICE.multiply(ORDER_LINES.QUANTITY)).as("total"))
                .from(ORDERS).join(ORDER_LINES).on(ORDER_LINES.ORDER_ID.eq(ORDERS.ID))
                .where(ORDERS.STATUS.eq("COMPLETED"))
                .groupBy(ORDERS.ID, ORDERS.STATUS)
                .orderBy(field("total").desc()).limit(limit)
                .fetchInto(OrderSummary.class);
    }
}
```

---

## 5. Spring Security

### SecurityFilterChain

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"))
                .cors(cors -> cors.configurationSource(corsSource()))
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/public/**", "/actuator/health/**").permitAll()
                        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter())))
                .build();
    }

    @Bean
    JwtAuthenticationConverter jwtAuthConverter() {
        var authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");
        authorities.setAuthorityPrefix("ROLE_");
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }

    @Bean
    CorsConfigurationSource corsSource() {
        var config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### Method-Level Security

```java
@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN') or #customerId == authentication.principal.claims['sub']")
    public List<Order> getOrdersForCustomer(String customerId) {
        return orderRepository.findByCustomerId(customerId);
    }

    @PreAuthorize("@orderAuthz.canCancel(#orderId, authentication)")
    public void cancelOrder(Long orderId) { /* ... */ }
}

@Component("orderAuthz")
public class OrderAuthorizationLogic {
    private final OrderRepository orderRepository;
    public OrderAuthorizationLogic(OrderRepository repo) { this.orderRepository = repo; }

    public boolean canCancel(Long orderId, Authentication auth) {
        var order = orderRepository.findById(orderId).orElse(null);
        if (order == null) return false;
        var userId = ((Jwt) auth.getPrincipal()).getSubject();
        return order.getOwnerId().equals(userId) && order.isCancellable();
    }
}
```

**Anti-pattern:** Never blanket-disable CSRF (`http.csrf(AbstractHttpConfigurer::disable)`) for browser-facing endpoints. Disable selectively for stateless API paths only.

---

## 6. GraalVM Native

GraalVM native images compile Java ahead-of-time into standalone executables with sub-200ms startup.

### Setup

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
./mvnw -Pnative native:compile        # standalone binary
./mvnw -Pnative spring-boot:build-image  # native container image
```

### Runtime Hints for Reflection

```java
@Configuration
@ImportRuntimeHints(AppRuntimeHints.class)
public class NativeConfig {}

public class AppRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(OrderSummary.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS);
        hints.resources().registerPattern("db/migration/*");
        hints.serialization().registerType(OrderEvent.class);
    }
}
```

### Tradeoffs

| Aspect | JVM | Native Image |
|---|---|---|
| Startup | 2-10 s | 50-200 ms |
| Memory (RSS) | 200-500 MB | 50-150 MB |
| Peak throughput | Higher (JIT) | Lower (no JIT) |
| Build time | 10-30 s | 2-10 min |
| Dynamic features | Full | Requires hints config |

**Anti-pattern:** Avoid dynamic class loading (`Class.forName`) in native builds. Use DI or a registry instead.

```java
// BAD: breaks native image
Class<?> clazz = Class.forName(config.getProcessorClassName());

// GOOD: registry pattern, native-compatible
@Component
public class ProcessorRegistry {
    private final Map<String, Processor> processors;
    public ProcessorRegistry(List<Processor> all) {
        this.processors = all.stream().collect(Collectors.toMap(Processor::getName, identity()));
    }
    public Processor get(String name) {
        return Optional.ofNullable(processors.get(name))
                .orElseThrow(() -> new IllegalArgumentException("Unknown: " + name));
    }
}
```

---

## 7. Testing

### JUnit 5

```java
@DisplayName("Order domain logic")
class OrderTest {

    @Test void addLineIncreasesTotal() {
        var order = new Order(OrderStatus.PENDING);
        order.addLine(new Product("Widget", new BigDecimal("19.99")), 3);
        assertThat(order.getTotal()).isEqualByComparingTo(new BigDecimal("59.97"));
    }

    @ParameterizedTest
    @CsvSource({"PENDING,true", "CONFIRMED,true", "SHIPPED,false", "DELIVERED,false"})
    void cancellableStatuses(OrderStatus status, boolean expected) {
        assertThat(new Order(status).isCancellable()).isEqualTo(expected);
    }

    @Test void cannotAddLineToShippedOrder() {
        var order = new Order(OrderStatus.SHIPPED);
        assertThatThrownBy(() -> order.addLine(new Product("Widget", BigDecimal.TEN), 1))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Cannot modify a shipped order");
    }
}
```

### Mockito

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository orderRepository;
    @Mock PaymentGateway paymentGateway;
    @InjectMocks OrderService orderService;

    @Test void confirmOrder_chargesAndUpdatesStatus() {
        var order = new Order(OrderStatus.PENDING);
        order.addLine(new Product("Widget", BigDecimal.TEN), 2);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentGateway.charge(any())).thenReturn(new PaymentResult(true, "tx-123"));

        orderService.confirmOrder(1L);

        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(paymentGateway).charge(argThat(r -> r.amount().compareTo(new BigDecimal("20.00")) == 0));
    }
}
```

### Testcontainers

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired OrderRepository orderRepository;

    @Test void findAndLock_returnsOnlyPending() {
        orderRepository.save(new Order(OrderStatus.PENDING));
        orderRepository.save(new Order(OrderStatus.PENDING));
        orderRepository.save(new Order(OrderStatus.SHIPPED));

        var locked = orderRepository.findAndLockForProcessing("PENDING", 10);
        assertThat(locked).hasSize(2).allMatch(o -> o.getStatus() == OrderStatus.PENDING);
    }
}
```

### Slice Tests

```java
@WebMvcTest(OrderController.class) @Import(SecurityConfig.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;

    @Test @WithMockUser(roles = "ADMIN")
    void getOrders_asAdmin_returnsOk() throws Exception {
        when(orderService.findAll(any())).thenReturn(
                new PageImpl<>(List.of(new OrderSummary(1L, "PENDING", BigDecimal.TEN))));
        mockMvc.perform(get("/api/v1/orders").param("page", "0").param("size", "20"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content[0].status").value("PENDING"));
    }
}
```

### ArchUnit

```java
@AnalyzeClasses(packages = "com.example.orders")
class ArchitectureTest {

    @ArchTest static final ArchRule domainIndependence = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..infrastructure..", "..adapter..", "org.springframework..");

    @ArchTest static final ArchRule noFieldInjection = noFields()
            .should().beAnnotatedWith(Autowired.class)
            .because("Use constructor injection instead");
}
```

---

## 8. Architecture Patterns

### Hexagonal Architecture (Ports and Adapters)

```
com.example.orders/
  domain/model/         # Entities, value objects, aggregates
  domain/port/in/       # Driving ports (use-case interfaces)
  domain/port/out/      # Driven ports (repository/gateway interfaces)
  application/          # Use-case implementations
  adapter/in/web/       # REST controllers
  adapter/out/persistence/  # JPA repositories
  adapter/out/payment/  # External gateway clients
```

```java
// Ports
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}
public interface OrderPersistencePort {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}

// Application service wires ports together
@Service @Transactional
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderPersistencePort persistence;
    private final PaymentPort payment;

    @Override
    public OrderId placeOrder(PlaceOrderCommand cmd) {
        var order = Order.create(cmd.customerId(), cmd.lines());
        var result = payment.charge(order.toPaymentRequest());
        if (!result.success()) throw new PaymentFailedException(result.reason());
        order.confirm(result.transactionId());
        return persistence.save(order).getId();
    }
}
```

### DDD Aggregates

```java
public class Order {
    private OrderId id;
    private OrderStatus status;
    private Money total;
    private final List<OrderLine> lines = new ArrayList<>();
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public static Order create(CustomerId customerId, List<LineItem> items) {
        var order = new Order();
        order.status = OrderStatus.DRAFT;
        items.forEach(i -> order.addLine(i.productId(), i.quantity(), i.price()));
        return order;
    }

    public void addLine(ProductId productId, int quantity, Money price) {
        guardModifiable();
        lines.add(new OrderLine(productId, quantity, price));
        recalculateTotal();
    }

    public void confirm(String txId) {
        guardModifiable();
        if (lines.isEmpty()) throw new InvalidOrderException("Cannot confirm empty order");
        this.status = OrderStatus.CONFIRMED;
        domainEvents.add(new OrderConfirmedEvent(this.id, txId, this.total));
    }

    private void guardModifiable() {
        if (status != OrderStatus.DRAFT && status != OrderStatus.PENDING)
            throw new InvalidOrderException("Order not modifiable in status: " + status);
    }
}
```

**Anti-pattern:** Never expose aggregate internals. All changes go through aggregate root methods.

```java
// BAD: bypasses invariants
order.getLines().add(new OrderLine(...));
// GOOD
order.addLine(productId, quantity, price);
```

### CQRS

```java
// Command side — rich domain model
@Service
public class OrderCommandService {
    @Transactional
    public OrderId handle(PlaceOrderCommand cmd) {
        var order = Order.create(cmd.customerId(), cmd.lines());
        return repository.save(order).getId();
    }
}

// Query side — optimized reads against denormalized views
@Repository
public class OrderQueryRepository {
    private final JdbcTemplate jdbc;
    public Page<OrderListView> findOrders(OrderSearchCriteria criteria, Pageable pageable) {
        var sql = """
                SELECT o.id, o.status, o.total, c.name AS customer_name, COUNT(ol.id) AS line_count
                FROM orders o JOIN customers c ON c.id = o.customer_id
                LEFT JOIN order_lines ol ON ol.order_id = o.id
                WHERE (:status IS NULL OR o.status = :status)
                GROUP BY o.id, c.name ORDER BY o.created_at DESC
                """;
        // ...
    }
}
```

### Event Sourcing

```java
public class Order extends EventSourcedAggregate {
    private OrderId id;
    private OrderStatus status;
    private final List<OrderLine> lines = new ArrayList<>();

    public void apply(OrderCreatedEvent e) { this.id = e.orderId(); this.status = OrderStatus.DRAFT; }
    public void apply(OrderLineAddedEvent e) { lines.add(new OrderLine(e.productId(), e.quantity(), e.price())); }
    public void apply(OrderConfirmedEvent e) { this.status = OrderStatus.CONFIRMED; }

    public void addLine(ProductId pid, int qty, Money price) {
        guardModifiable();
        raiseEvent(new OrderLineAddedEvent(this.id, pid, qty, price));
    }
}
```

### Modular Monolith

```java
// Module public API — the only type other modules depend on
public interface OrderModuleApi {
    OrderId placeOrder(PlaceOrderRequest request);
    Optional<OrderDto> getOrder(OrderId id);
    void cancelOrder(OrderId id, String reason);
}

// Internal implementation — package-private
@Service
class OrderService implements OrderModuleApi { /* hidden internals */ }
```

Enforce boundaries with ArchUnit:

```java
@ArchTest static final ArchRule moduleBoundaries = noClasses()
        .that().resideOutsideOfPackage("..order.internal..")
        .should().accessClassesThat().resideInAPackage("..order.internal..")
        .because("Other modules must use OrderModuleApi, not internal classes");
```

---

## Output Format

When generating Java architecture:

1. **Package structure** -- directory layout with module/layer separation
2. **Domain model** -- entities, value objects, aggregates with invariants
3. **Configuration** -- Spring Boot properties, security, Docker setup
4. **Data access** -- repositories, migrations, query optimization
5. **Tests** -- unit, integration, and architecture tests
6. **Build** -- Maven/Gradle config, native-image setup if applicable

Always prefer constructor injection over field injection, records over mutable DTOs, sealed types over marker interfaces, and composition over inheritance.
