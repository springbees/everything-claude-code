---
name: java-design-standards
description: Java design principles and architecture patterns including SOLID principles, design patterns, dependency management, system architecture, and UML documentation standards. Use this when designing software architecture, making design decisions, or applying design patterns in Java projects.
---

# Java Design Standards & Architecture Principles

## Overview
This skill provides comprehensive design standards and architecture principles for Java applications, based on the Alibaba Java Development Handbook. It covers SOLID principles, design patterns, system architecture design, and documentation standards.

## When to Use This Skill
Invoke this skill when you need to:
- Design system architecture and module boundaries
- Apply SOLID principles and design patterns
- Create design documentation (UML diagrams)
- Plan for system scalability and maintainability
- Design for fault tolerance and degradation
- Make architectural trade-off decisions

## Core Design Principles

### 1. SOLID Principles

**Single Responsibility Principle (SRP):**
- A class should have one reason to change
- One responsibility per class

```java
// Good - separate responsibilities
public class UserService {
    public void createUser(User user) { }
    public void deleteUser(Long id) { }
}

public class UserValidator {
    public boolean validate(User user) { }
}

// Bad - mixed responsibilities
public class UserService {
    public void createUser(User user) {
        if (!validateEmail(user.getEmail())) { }
        // save to database
        // send email
        // log activity
    }
}
```

**Open-Closed Principle (OCP):**
- Open for extension, closed for modification
- Use abstraction to allow behavior extension

```java
// Good - using strategy pattern
public interface PaymentStrategy {
    void pay(BigDecimal amount);
}

public class CreditCardPayment implements PaymentStrategy {
    public void pay(BigDecimal amount) { }
}

public class WeChatPayment implements PaymentStrategy {
    public void pay(BigDecimal amount) { }
}

// Usage - add new payment methods without modifying existing code
public class PaymentService {
    public void processPayment(PaymentStrategy strategy, BigDecimal amount) {
        strategy.pay(amount);
    }
}
```

**Liskov Substitution Principle (LSP):**
- Subtypes must be substitutable for base types
- Don't violate base class contracts

```java
// Good - proper inheritance
public abstract class Bird {
    public void fly() { }
    public void eat() { }
}

public class Sparrow extends Bird {
    // can fly and eat
}

// Bad - violates LSP (penguins can't fly)
public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException();
    }
}

// Better design
public abstract class Bird {
    public void eat() { }
}

public abstract class FlyingBird extends Bird {
    public abstract void fly();
}

public class Penguin extends Bird {
    // no fly method
}
```

**Interface Segregation Principle (ISP):**
- Clients shouldn't depend on interfaces they don't use
- Split large interfaces into focused ones

```java
// Bad - bloated interface
public interface UserService {
    void createUser(User user);
    void deleteUser(Long id);
    void sendEmail(User user);
    void generateReport(User user);
}

// Good - segregated interfaces
public interface UserCrudService {
    void create(User user);
    void delete(Long id);
}

public interface UserNotificationService {
    void sendEmail(User user);
}

public interface UserReportService {
    Report generateReport(User user);
}
```

**Dependency Inversion Principle (DIP):**
- Depend on abstractions, not concretions
- High-level modules shouldn't depend on low-level modules

```java
// Good - depends on abstraction
public class OrderService {
    private final PaymentRepository paymentRepository;
    private final NotificationService notificationService;

    public OrderService(PaymentRepository paymentRepository,
                       NotificationService notificationService) {
        this.paymentRepository = paymentRepository;
        this.notificationService = notificationService;
    }
}

// Bad - depends on concrete implementations
public class OrderService {
    private final MySqlPaymentRepository paymentRepository;
    private final EmailNotificationService notificationService;
}
```

### 2. Design Patterns

**Factory Pattern:**
```java
public interface MessageService {
    void send(String message);
}

public class EmailService implements MessageService {
    public void send(String message) { }
}

public class SmsService implements MessageService {
    public void send(String message) { }
}

public class MessageServiceFactory {
    public static MessageService createService(String type) {
        switch (type) {
            case "email":
                return new EmailService();
            case "sms":
                return new SmsService();
            default:
                throw new IllegalArgumentException("Unknown type: " + type);
        }
    }
}
```

**Strategy Pattern:**
```java
public interface DiscountStrategy {
    BigDecimal applyDiscount(BigDecimal originalPrice);
}

public class VIPDiscountStrategy implements DiscountStrategy {
    public BigDecimal applyDiscount(BigDecimal originalPrice) {
        return originalPrice.multiply(new BigDecimal("0.8"));
    }
}

public class RegularDiscountStrategy implements DiscountStrategy {
    public BigDecimal applyDiscount(BigDecimal originalPrice) {
        return originalPrice.multiply(new BigDecimal("0.95"));
    }
}

public class PricingService {
    private DiscountStrategy discountStrategy;

    public void setDiscountStrategy(DiscountStrategy strategy) {
        this.discountStrategy = strategy;
    }

    public BigDecimal calculatePrice(BigDecimal originalPrice) {
        return discountStrategy.applyDiscount(originalPrice);
    }
}
```

**Observer Pattern:**
```java
public interface OrderEventListener {
    void onOrderCreated(Order order);
    void onOrderCancelled(Order order);
}

public class EmailNotificationListener implements OrderEventListener {
    public void onOrderCreated(Order order) {
        // send email notification
    }
    public void onOrderCancelled(Order order) {
        // send cancellation email
    }
}

public class InventoryUpdateListener implements OrderEventListener {
    public void onOrderCreated(Order order) {
        // update inventory
    }
    public void onOrderCancelled(Order order) {
        // restore inventory
    }
}

public class OrderService {
    private List<OrderEventListener> listeners = new ArrayList<>();

    public void addListener(OrderEventListener listener) {
        listeners.add(listener);
    }

    public void createOrder(Order order) {
        // create order logic
        for (OrderEventListener listener : listeners) {
            listener.onOrderCreated(order);
        }
    }
}
```

### 3. System Architecture Design

**Architecture Design Goals:**
1. Define system boundaries (what to build, what not to build)
2. Define module relationships and dependencies
3. Establish design principles for evolution
4. Address non-functional requirements (security, availability, scalability)

**Layered Architecture:**
```
┌─────────────────────────────────────┐
│      API Layer (Open API/Web)       │
├─────────────────────────────────────┤
│         Business Layer              │
│  ┌─────────────────────────────┐   │
│  │      Service Layer          │   │
│  ├─────────────────────────────┤   │
│  │      Manager Layer          │   │
│  └─────────────────────────────┘   │
├─────────────────────────────────────┤
│      Data Access Layer (DAO)        │
├─────────────────────────────────────┤
│    External Services / Database     │
└─────────────────────────────────────┘
```

**Domain Model Standards:**
- **DO (Data Object):** Maps 1:1 to database tables
- **DTO (Data Transfer Object):** Transfers data between layers
- **BO (Business Object):** Encapsulates business logic
- **VO (View Object):** Used for presentation layer
- **Query:** Encapsulates query parameters (avoid Map for >2 params)

```java
// DO - database mapping
public class UserDO {
    private Long id;
    private String name;
    private String email;
    // getters/setters
}

// DTO - service layer output
public class UserDTO {
    private Long id;
    private String name;
    private String email;
    private List<String> roles;
    // getters/setters
}

// BO - business logic
public class OrderBO {
    private OrderDO order;
    private List<OrderItemDO> items;
    private UserDO user;

    public BigDecimal calculateTotal() {
        // business logic
    }
    // getters/setters
}

// VO - presentation layer
public class UserVO {
    private String displayName;
    private String avatar;
    private String status;
    // getters/setters
}

// Query - query parameters
public class UserQuery {
    private String name;
    private Integer status;
    private Integer page;
    private Integer size;
    // getters/setters
}
```

### 4. Fault Tolerance & Degradation

**Identify Weak Dependencies:**
- Services whose failure doesn't break core functionality
- Examples: recommendations, notifications, analytics

```java
// Good - degrade gracefully
public class ProductService {
    @Autowired
    private RecommendationService recommendationService;

    public ProductDetail getProductDetail(Long productId) {
        ProductDetail detail = getProductFromDB(productId);

        // Weak dependency - degrade if fails
        try {
            List<Product> recommendations =
                recommendationService.getRecommendations(productId);
            detail.setRecommendations(recommendations);
        } catch (Exception e) {
            // Log error but don't fail the request
            logger.warn("Failed to get recommendations", e);
            detail.setRecommendations(Collections.emptyList());
        }

        return detail;
    }
}
```

**Circuit Breaker Pattern:**
```java
public class ExternalServiceCaller {
    private final CircuitBreaker circuitBreaker;

    public ExternalServiceCaller() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .build();
        this.circuitBreaker = CircuitBreaker.of("externalService", config);
    }

    public String callExternalService() {
        Supplier<String> supplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                return externalServiceClient.call();
            });

        try {
            return supplier.get();
        } catch (CallNotPermittedException e) {
            return "Service unavailable - circuit open";
        } catch (Exception e) {
            throw new ServiceException("External service failed", e);
        }
    }
}
```

### 5. Documentation Standards

**When to Use UML Diagrams:**

1. **Use Case Diagram:** When >1 user type and >5 use cases
2. **State Diagram:** When business object has >3 states
3. **Sequence Diagram:** When call chain involves >3 objects
4. **Class Diagram:** When >5 model classes with complex dependencies
5. **Activity Diagram:** When >2 objects collaborate with complex processes

**Design Documentation Requirements:**
- Data structure design must be reviewed and documented
- Storage solutions need double-check before production
- Database schema changes require review

### 6. Design Anti-Patterns to Avoid

**Don't Repeat Yourself (DRY):**
```java
// Bad - duplicated validation logic
public void createUser(User user) {
    if (user.getName() == null || user.getName().isEmpty()) {
        throw new IllegalArgumentException("Invalid name");
    }
    // create user
}

public void updateUser(User user) {
    if (user.getName() == null || user.getName().isEmpty()) {
        throw new IllegalArgumentException("Invalid name");
    }
    // update user
}

// Good - extract common logic
public class UserValidator {
    public static void validateName(String name) {
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("Invalid name");
        }
    }
}

public void createUser(User user) {
    UserValidator.validateName(user.getName());
    // create user
}
```

**Avoid Over-Engineering:**
```java
// Bad - unnecessary interface for single implementation
public interface UserService {
    User createUser(User user);
}

public class UserServiceImpl implements UserService {
    public User createUser(User user) {
        // implementation
    }
}

// Good - simple class when only one implementation
public class UserService {
    public User createUser(User user) {
        // implementation
    }
}
```

### 7. Composition Over Inheritance

```java
// Bad - deep inheritance hierarchy
public class Animal {
    public void eat() { }
}

public class Mammal extends Animal {
    public void breathe() { }
}

public class Dog extends Mammal {
    public void bark() { }
}

public class RobotDog extends Dog {
    // Robot dog doesn't need eat() or breathe()
}

// Good - use composition
public interface Barkable {
    void bark();
}

public class Dog implements Barkable {
    private final Eater eater;
    private final Breather breather;

    public Dog(Eater eater, Breather breather) {
        this.eater = eater;
        this.breather = breather;
    }

    public void bark() {
        System.out.println("Woof!");
    }
}

public class RobotDog implements Barkable {
    public void bark() {
        System.out.println("Electronic woof!");
    }
}
```

## Design Review Checklist

**System Design:**
- [ ] System boundaries clearly defined
- [ ] Module dependencies documented
- [ ] Non-functional requirements addressed (security, performance, scalability)
- [ ] Weak dependencies identified with degradation plans
- [ ] Data storage design reviewed and documented

**Code Design:**
- [ ] Classes follow Single Responsibility Principle
- [ ] Composition favored over inheritance
- [ ] Dependencies inverted (depend on abstractions)
- [ ] Design patterns applied appropriately
- [ ] No code duplication (DRY principle)

**Documentation:**
- [ ] UML diagrams created for complex interactions
- [ ] Design decisions documented with rationale
- [ ] State transitions documented for complex state machines
- [ ] Architecture diagrams up to date

---

**Source:** Based on Alibaba Java Development Handbook (Huangshan Edition 1.7.1)
**Last Updated:** 2022.02.03