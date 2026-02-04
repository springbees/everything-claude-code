---
name: java-unit-testing
description: Java unit testing standards including AIR principles (Automatic, Independent, Repeatable), BCDE testing strategies, test coverage requirements, mocking best practices, and JUnit/TestNG guidelines. Use this when writing unit tests or reviewing test code.
---

# Java Unit Testing Standards

## Overview
Comprehensive unit testing standards based on Alibaba Java Development Handbook, covering test design principles, testing strategies, and best practices.

## When to Use
- Writing unit tests
- Reviewing test code
- Setting up test infrastructure
- Improving test coverage

## AIR Principles

### A - Automatic
Tests must run automatically without manual intervention.

```java
// Good: fully automated
@Test
public void testCalculateTotal() {
    Order order = new Order();
    order.addItem(new Item("Product A", new BigDecimal("100")));
    order.addItem(new Item("Product B", new BigDecimal("50")));

    BigDecimal total = order.calculateTotal();

    assertEquals(new BigDecimal("150"), total);
}

// Bad: requires manual verification
@Test
public void testDisplayOrder() {
    Order order = createTestOrder();
    order.display();
    System.out.println("Check if display is correct");  // Don't do this!
}
```

### I - Independent
Tests must not depend on each other or execution order.

```java
// Good: independent tests
public class UserServiceTest {
    @Test
    public void testCreateUser() {
        User user = userService.create("test@example.com");
        assertNotNull(user.getId());
    }

    @Test
    public void testUpdateUser() {
        User user = userService.create("test@example.com");
        user.setName("Updated Name");
        userService.update(user);

        User updated = userService.findById(user.getId());
        assertEquals("Updated Name", updated.getName());
    }
}

// Bad: dependent on execution order
public class UserServiceTest {
    private User savedUser;

    @Test
    public void test1_CreateUser() {
        savedUser = userService.create("test@example.com");
    }

    @Test
    public void test2_UpdateUser() {
        savedUser.setName("Updated");  // Fails if test1 doesn't run first!
    }
}
```

### R - Repeatable
Tests must produce same results regardless of environment.

```java
// Good: uses test data setup
@Test
public void testQueryByDate() {
    LocalDate testDate = LocalDate.of(2024, 1, 1);
    orderRepository.save(createTestOrder(testDate));

    List<Order> orders = orderRepository.findByDate(testDate);

    assertFalse(orders.isEmpty());
}

// Bad: depends on external state
@Test
public void testQueryToday() {
    List<Order> orders = orderRepository.findByDate(LocalDate.now());
    // Fails on different days!
}
```

## BCDE Testing Strategy

### B - Border (Boundary Values)
```java
@Test
public void testAgeValidation() {
    // Test minimum boundary
    assertFalse(validator.validateAge(0));
    assertFalse(validator.validateAge(17));  // below limit
    assertTrue(validator.validateAge(18));   // exact limit

    // Test maximum boundary
    assertTrue(validator.validateAge(65));   // exact limit
    assertFalse(validator.validateAge(66));  // above limit
    assertFalse(validator.validateAge(150));
}
```

### C - Correct (Normal Input)
```java
@Test
public void testNormalUserCreation() {
    UserCreateRequest request = new UserCreateRequest();
    request.setUsername("testuser");
    request.setEmail("test@example.com");
    request.setPassword("SecurePass123");

    User user = userService.createUser(request);

    assertNotNull(user);
    assertEquals("testuser", user.getUsername());
    assertEquals("test@example.com", user.getEmail());
}
```

### D - Design (Test Against Design)
```java
@Test
public void testPasswordStrengthPerRequirements() {
    // Requirement: at least 8 chars, 1 uppercase, 1 lowercase, 1 number
    assertTrue(validator.isPasswordValid("SecurePass123"));

    assertFalse(validator.isPasswordValid("short"));      // too short
    assertFalse(validator.isPasswordValid("alllowercase123")); // no uppercase
    assertFalse(validator.isPasswordValid("ALLUPPERCASE123")); // no lowercase
    assertFalse(validator.isPasswordValid("NoNumbers"));   // no numbers
}
```

### E - Error (Error Conditions)
```java
@Test(expected = IllegalArgumentException.class)
public void testNullUsername() {
    userService.create(null, "test@example.com", "password");
}

@Test(expected = DuplicateUserException.class)
public void testDuplicateUsername() {
    userService.create("testuser", "test1@example.com", "pass1");
    userService.create("testuser", "test2@example.com", "pass2");  // Should throw
}
```

## Test Coverage Requirements

### Coverage Targets
- Overall: 70% statement coverage
- Core modules: 100% statement and branch coverage
- Must cover: DAO, Manager, reusable Service layers

```java
// Good: comprehensive test with multiple scenarios
public class OrderServiceTest {

    @Test
    public void testCreateOrder() {
        // Normal case
        OrderRequest request = createValidRequest();
        Order order = orderService.createOrder(request);
        assertNotNull(order);

        // Insufficient inventory
        request.setQuantity(999999);
        try {
            orderService.createOrder(request);
            fail("Should throw InsufficientInventoryException");
        } catch (InsufficientInventoryException e) {
            // Expected
        }

        // Invalid payment amount
        request.setQuantity(1);
        request.setAmount(new BigDecimal("-100"));
        try {
            orderService.createOrder(request);
            fail("Should throw InvalidAmountException");
        } catch (InvalidAmountException e) {
            // Expected
        }
    }
}
```

## Test Structure (AAA Pattern)

### Arrange-Act-Assert
```java
@Test
public void testCalculateDiscount() {
    // Arrange: set up test data
    User user = new User();
    user.setLevel(UserLevel.VIP);
    BigDecimal originalAmount = new BigDecimal("100");

    // Act: execute the method under test
    BigDecimal discountedAmount = pricingService
        .calculateDiscount(user, originalAmount);

    // Assert: verify results
    assertEquals(new BigDecimal("80"), discountedAmount);  // 20% VIP discount
}
```

## Mocking Best Practices

### Using Mockito
```java
@RunWith(MockitoJUnitRunner.class)
public class UserServiceTest {

    @Mock
    private UserDao userDao;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;

    @Test
    public void testCreateUser() {
        // Arrange
        UserCreateRequest request = new UserCreateRequest();
        request.setUsername("testuser");

        UserDO savedUser = new UserDO();
        savedUser.setId(1L);
        savedUser.setUsername("testuser");

        when(userDao.insert(any(UserDO.class))).thenReturn(1);
        when(userDao.findById(1L)).thenReturn(savedUser);

        // Act
        UserDTO result = userService.createUser(request);

        // Assert
        assertEquals(1L, result.getId().longValue());
        verify(userDao).insert(any(UserDO.class));
        verify(emailService).sendWelcomeEmail(any(UserDO.class));
    }
}
```

### Avoid Over-Mocking
```java
// Bad: too many mocks
@Test
public void testComplexLogic() {
    when(mock1.method1()).thenReturn(value1);
    when(mock2.method2()).thenReturn(value2);
    when(mock3.method3()).thenReturn(value3);
    when(mock4.method4()).thenReturn(value4);
    when(mock5.method5()).thenReturn(value5);
    // ... fragile test, too many dependencies
}

// Good: focus on core logic
@Test
public void testComplexLogic() {
    // Test the business logic directly
    // Use real implementations where possible
    // Mock only external dependencies
}
```

## Test Naming Conventions

### Method Naming
```java
// Good: descriptive naming
@Test
public void testCreateUser_Success_ReturnsUserWithId() { }

@Test
public void testCreateUser_DuplicateEmail_ThrowsException() { }

@Test
public void testCreateUser_NullEmail_ThrowsException() { }

// Bad: non-descriptive
@Test
public void test1() { }
@Test
public void testCreateUser() { }  // Doesn't indicate what's being tested
```

## Test Data Management

### Test Data Builders
```java
public class UserTestDataBuilder {

    public static UserDO.Builder aUser() {
        return UserDO.builder()
            .username("testuser")
            .email("test@example.com")
            .status(UserStatus.ACTIVE);
    }

    public static UserDO.Builder aVIPUser() {
        return aUser()
            .level(UserLevel.VIP)
            .points(1000);
    }
}

// Usage
@Test
public void testVIPDiscount() {
    User user = UserTestDataBuilder.aVIPUser().build();
    // test...
}
```

### Database Test Setup
```java
@Test
public void testDatabaseOperation() {
    // Good: insert test data programmatically
    UserDO user = new UserDO();
    user.setUsername("testuser");
    user.setEmail("test@example.com");
    userDao.insert(user);

    // Test...
}

// Bad: assume data exists
@Test
public void testDatabaseOperation() {
    // Assumes user with ID 1 exists - fragile!
    User user = userDao.findById(1L);
}

// Good: use rollback or cleanup
@Transactional
@Test
public void testWithRollback() {
    // Changes will be rolled back automatically
}
```

## Test Organization

### Test Class Structure
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class OrderServiceIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private TestEntityManager entityManager;

    private UserDO testUser;
    private ProductDO testProduct;

    @Before
    public void setUp() {
        // Set up common test data
        testUser = createTestUser();
        testProduct = createTestProduct();
    }

    @Test
    public void testCreateOrder() {
        // Test...
    }

    @Test
    public void testCancelOrder() {
        // Test...
    }

    @After
    public void tearDown() {
        // Clean up if needed
    }
}
```

## Common Testing Patterns

### Exception Testing
```java
// JUnit 4
@Test(expected = InsufficientFundsException.class)
public void testWithdrawInsufficientFunds() {
    account.withdraw(new BigDecimal("10000"));
}

// JUnit 5
@Test
void testWithdrawInsufficientFunds() {
    InsufficientFundsException exception = assertThrows(
        InsufficientFundsException.class,
        () -> account.withdraw(new BigDecimal("10000"))
    );
    assertEquals("Insufficient funds", exception.getMessage());
}
```

### Async Testing
```java
@Test
public void testAsyncMethod() throws Exception {
    CompletableFuture<String> future = asyncService.getMessage();

    // Wait for result
    String result = future.get(5, TimeUnit.SECONDS);

    assertEquals("Hello", result);
}
```

## Test File Location

### Directory Structure
```
src/
├── main/
│   └── java/
│       └── com/example/
│           └── UserService.java
└── test/
    └── java/
        └── com/example/
            └── UserServiceTest.java  // Tests go here!
```

**Important:**
- Tests MUST be in `src/test/java`
- NOT in `src/main/java`
- Test class mirrors main class structure

## Quick Reference

### Test Template
```java
public class SomeClassTest {

    private SomeClass someClass;

    @Before
    public void setUp() {
        someClass = new SomeClass();
    }

    @Test
    public void testMethodName_Scenario_ExpectedResult() {
        // Arrange
        // set up test data

        // Act
        // execute method

        // Assert
        // verify results
    }

    @After
    public void tearDown() {
        // cleanup
    }
}
```

### Testing Checklist
- [ ] Test is automatic (no manual verification)
- [ ] Test is independent (no dependencies on other tests)
- [ ] Test is repeatable (works in any environment)
- [ ] Test covers boundary values
- [ ] Test covers normal cases
- [ ] Test covers error conditions
- [ ] Test uses proper mocking (don't over-mock)
- [ ] Test has descriptive name
- [ ] Test data is properly set up
- [ ] Test is in src/test/java
- [ ] Code coverage meets requirements

---

**Source:** Alibaba Java Development Handbook (Huangshan Edition 1.7.1)