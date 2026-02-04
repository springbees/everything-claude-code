---
name: java-exception-logging
description: Java exception handling and logging best practices including error code standards, exception handling patterns, logging framework usage (SLF4J), and unit testing standards. Use this when handling exceptions, implementing logging, or writing error handling code in Java applications.
---

# Java Exception Handling & Logging Standards

## Overview
This skill provides comprehensive standards for exception handling, error codes, logging, and unit testing in Java applications, based on the Alibaba Java Development Handbook.

## When to Use This Skill
Invoke this skill when you need to:
- Handle exceptions properly in your code
- Implement error codes and error handling strategies
- Set up logging with SLF4J
- Write effective unit tests
- Debug production issues through logs

## 1. Error Code Standards

### Error Code Format
Error codes follow a 5-character format: `[Source][Number]`

**Source Types:**
- `A` - User-side errors
- `B` - System errors
- `C` - Third-party service errors

**Examples:**
```
00000 - Success (all OK)
A0001 - User error (level 1)
A0100 - User registration error (level 2)
A0110 - Username validation failed (level 3)
B0001 - System execution error (level 1)
B0100 - System timeout (level 2)
C0001 - Third-party service error (level 1)
C0110 - RPC service error (level 2)
```

### Error Code Implementation
```java
public enum ErrorCode {
    SUCCESS("00000", "Success"),
    USER_ERROR("A0001", "User error"),
    USER_NOT_FOUND("A0201", "User account not found"),
    USER_PASSWORD_ERROR("A0210", "User password error"),
    SYSTEM_ERROR("B0001", "System execution error"),
    SYSTEM_TIMEOUT("B0100", "System timeout"),
    THIRD_PARTY_ERROR("C0001", "Third-party service error");

    private final String code;
    private final String message;

    ErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() { return code; }
    public String getMessage() { return message; }
}

// Usage in API response
public class ApiResponse<T> {
    private String code;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(ErrorCode.SUCCESS.getCode());
        response.setMessage(ErrorCode.SUCCESS.getMessage());
        response.setData(data);
        return response;
    }

    public static <T> ApiResponse<T> error(ErrorCode errorCode) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(errorCode.getCode());
        response.setMessage(errorCode.getMessage());
        return response;
    }
}
```

## 2. Exception Handling Best Practices

### Do Not Catch Avoidable Exceptions
```java
// Bad - catching NPE
try {
    obj.method();
} catch (NullPointerException e) {
    // handle
}

// Good - prevent NPE with null check
if (obj != null) {
    obj.method();
}

// Good - use Optional
Optional.ofNullable(obj).ifPresent(Object::method);
```

### Do Not Use Exceptions for Flow Control
```java
// Bad
try {
    if (userExists(username)) {
        throw new UserExistsException();
    }
} catch (UserExistsException e) {
    return ERROR;
}

// Good
if (userExists(username)) {
    return ERROR_USER_EXISTS;
}
```

### Catch Specific Exceptions
```java
// Good
try {
    // code
} catch (NumberFormatException e) {
    logger.error("Invalid number format", e);
    throw new BusinessException("Invalid number format", e);
} catch (IOException e) {
    logger.error("IO error", e);
    throw new BusinessException("IO error occurred", e);
}

// Bad - too broad
try {
    // code
} catch (Exception e) {
    // handle everything
}
```

### Finally Block Best Practices
```java
// Good - try-with-resources (Java 7+)
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // use resources
} catch (SQLException e) {
    logger.error("Database error", e);
    throw new DAOException(e);
}

// Good - manual close with null check
Connection conn = null;
try {
    conn = dataSource.getConnection();
    // use connection
} catch (SQLException e) {
    logger.error("Database error", e);
    throw new DAOException(e);
} finally {
    if (conn != null) {
        try {
            conn.close();
        } catch (SQLException e) {
            logger.error("Failed to close connection", e);
        }
    }
}

// Bad - no return in finally
private int x = 0;
public int checkReturn() {
    try {
        return ++x;
    } finally {
        return ++x;  // This will override the try block's return!
    }
}
```

### Transaction Rollback
```java
// Good - explicit rollback on exception
@Transactional
public void updateUserData(User user) {
    try {
        userDao.update(user);
        // other operations
    } catch (Exception e) {
        logger.error("Failed to update user", e);
        TransactionAspectSupport.currentTransactionStatus()
            .setRollbackOnly();
        throw new ServiceException("Failed to update user", e);
    }
}
```

### NPE Prevention Checklist
- Use `Optional` for nullable return values
- Validate input parameters
- Avoid cascading calls (`obj.getA().getB().getC()`)
- Use `Objects.equals()` for null-safe comparison
- Add `@NonNull` / `@Nullable` annotations

```java
// Good - NPE prevention
public void processUser(User user) {
    // Early return for null
    if (user == null) {
        throw new IllegalArgumentException("User cannot be null");
    }

    // Null-safe comparison
    if (Objects.equals(user.getId(), targetId)) {
        // process
    }

    // Optional for nested objects
    Optional.ofNullable(user.getAddress())
        .map(Address::getCity)
        .ifPresent(city -> processCity(city));
}
```

## 3. Logging Standards

### Use SLF4J (Not Log4j or Logback directly)
```java
// Good
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);

    public void createUser(User user) {
        logger.info("Creating user: {}", user.getName());
        // implementation
    }
}

// Bad - direct Log4j usage
import org.apache.log4j.Logger;

public class UserService {
    private static final Logger logger = Logger.getLogger(UserService.class);
}
```

### Log Levels
```java
// ERROR - System errors that require immediate attention
logger.error("Failed to connect to database", exception);

// WARN - Harmful situations that don't stop the system
logger.warn("Retry attempt {} for operation: {}", retryCount, operationName);

// INFO - Important business process milestones
logger.info("Order {} created successfully", orderId);

// DEBUG - Detailed diagnostic information
logger.debug("Processing request with parameters: {}", params);

// TRACE - Very detailed diagnostic information
logger.trace("Entering method processOrder with orderId: {}", orderId);
```

### Parameterized Logging
```java
// Good - parameterized (only converts if log level enabled)
logger.info("User {} logged in from IP: {}", username, ipAddress);

// Bad - string concatenation (always creates String)
logger.info("User " + username + " logged in from IP: " + ipAddress);
```

### Log Content Guidelines
```java
// Good - includes context
logger.error("Failed to process payment for order {}: {}", orderId, errorMessage, exception);

// Good - structured data
logger.info("Order processed: orderId={}, userId={}, amount={}",
    orderId, userId, amount);

// Bad - insufficient information
logger.error("Payment failed");
logger.info("Order processed");
```

### Guard Clauses for Debug Logging
```java
// Good - check log level
if (logger.isDebugEnabled()) {
    logger.debug("Complex object: {}", expensiveToString());
}

// This pattern is automatically handled by SLF4J for parameterized logging
logger.debug("User data: {}", user);  // Only converts if DEBUG enabled
```

### Log File Configuration
- Keep logs for at least 15 days
- For security-relevant logs: 6 months
- Daily rotation or size-based rotation
- Structured file naming: `{app}_{type}_{name}.log`

Example configuration (logback.xml):
```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.log.%d{yyyy-MM-dd}</fileNamePattern>
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.example.business" level="INFO" additivity="false">
        <appender-ref ref="FILE" />
    </logger>

    <root level="INFO">
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

### Sensitive Data Masking
```java
// Good - mask sensitive data
public String maskPhone(String phone) {
    if (phone == null || phone.length() != 11) {
        return phone;
    }
    return phone.substring(0, 3) + "****" + phone.substring(7);
}

public String maskEmail(String email) {
    if (email == null || !email.contains("@")) {
        return email;
    }
    String[] parts = email.split("@");
    return parts[0].charAt(0) + "***@" + parts[1];
}

// Usage
logger.info("User login: phone={}, email={}",
    maskPhone(user.getPhone()),
    maskEmail(user.getEmail()));
```

## 4. Unit Testing Standards (AIR Principles)

### A - Automatic
```java
// Good - fully automated
@Test
public void testCalculateTotal() {
    Order order = new Order();
    order.addItem(new Item("Product A", new BigDecimal("100")));
    order.addItem(new Item("Product B", new BigDecimal("50")));

    BigDecimal total = order.calculateTotal();

    assertEquals(new BigDecimal("150"), total);
}

// Bad - requires manual verification
@Test
public void testDisplayOrder() {
    Order order = createTestOrder();
    order.display();  // Manual verification needed
    System.out.println("Check if display is correct");  // Don't do this!
}
```

### I - Independent
```java
// Good - independent tests
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

// Bad - dependent on execution order
private User savedUser;
@Test
public void test1_CreateUser() {
    savedUser = userService.create("test@example.com");
}
@Test
public void test2_UpdateUser() {
    savedUser.setName("Updated");  // Fails if test1 doesn't run first
}
```

### R - Repeatable
```java
// Good - uses test data setup
@Test
public void testQueryByDate() {
    // Arrange
    LocalDate testDate = LocalDate.of(2024, 1, 1);
    orderRepository.save(createTestOrder(testDate));

    // Act
    List<Order> orders = orderRepository.findByDate(testDate);

    // Assert
    assertFalse(orders.isEmpty());
}

// Bad - depends on external state
@Test
public void testQueryToday() {
    List<Order> orders = orderRepository.findByDate(LocalDate.now());
    // Fails on different days
}
```

### BCDE Principles for Test Cases

```java
// B - Border (boundary values)
@Test
public void testAgeBoundary() {
    // Test minimum age
    assertFalse(validator.validateAge(0));
    // Test maximum age
    assertFalse(validator.validateAge(150));
}

// C - Correct (normal input)
@Test
public void testValidAge() {
    assertTrue(validator.validateAge(25));
}

// D - Design (test against design)
@Test
public void testAgeValidationPerRequirement() {
    // Per design spec: age must be between 18 and 65
    assertTrue(validator.validateAge(18));
    assertFalse(validator.validateAge(17));
}

// E - Error (error conditions)
@Test(expected = IllegalArgumentException.class)
public void testNullAge() {
    validator.validateAge(null);
}
```

### Test Coverage Requirements
- Overall statement coverage: 70%
- Core modules: 100% statement and branch coverage
- Test DAO, Manager, Service layers thoroughly
```java
// Good test with multiple scenarios
@Test
public void testTransfer() {
    // Normal case
    accountService.transfer(fromAccount, toAccount, amount);
    assertEquals(expectedBalance, fromAccount.getBalance());

    // Insufficient funds
    try {
        accountService.transfer(fromAccount, toAccount,
            fromAccount.getBalance().add(BigDecimal.ONE));
        fail("Should throw exception");
    } catch (InsufficientFundsException e) {
        // Expected
    }

    // Negative amount
    try {
        accountService.transfer(fromAccount, toAccount, BigDecimal.ONE.negate());
        fail("Should throw exception");
    } catch (IllegalArgumentException e) {
        // Expected
    }
}
```

### Test Location and Naming
- Location: `src/test/java` (not in main source)
- Naming: `{ClassName}Test.java`
- Methods: `test{MethodName}_{Scenario}_{ExpectedResult}`

```java
// Good
public class UserServiceTest {
    @Test
    public void testCreateUser_Success_ReturnsUserWithId() { }

    @Test
    public void testCreateUser_DuplicateEmail_ThrowsException() { }

    @Test
    public void testCreateUser_NullEmail_ThrowsException() { }
}

// Bad
public class UserServiceTest {
    @Test
    public void test1() { }

    @Test
    public void testCreateUser() { }  // Not specific enough
}
```

## 5. Custom Exception Hierarchy

```java
// Base exception
public abstract class BaseException extends RuntimeException {
    private final ErrorCode errorCode;

    public BaseException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public BaseException(ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}

// Business exception
public class BusinessException extends BaseException {
    public BusinessException(ErrorCode errorCode, String message) {
        super(errorCode, message);
    }

    public BusinessException(ErrorCode errorCode, String message, Throwable cause) {
        super(errorCode, message, cause);
    }
}

// DAO exception
public class DAOException extends BaseException {
    public DAOException(String message, Throwable cause) {
        super(ErrorCode.SYSTEM_ERROR, message, cause);
    }
}

// Usage
public User findByUsername(String username) {
    try {
        return userRepository.findByUsername(username);
    } catch (EmptyResultDataAccessException e) {
        throw new BusinessException(ErrorCode.USER_NOT_FOUND,
            "User not found: " + username, e);
    } catch (DataAccessException e) {
        throw new DAOException("Database error", e);
    }
}
```

## Quick Reference

### Common Exception Patterns

**1. Validate-Then-Act:**
```java
public void withdraw(BigDecimal amount) {
    if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Invalid amount");
    }
    if (amount.compareTo(balance) > 0) {
        throw new InsufficientFundsException();
    }
    balance = balance.subtract(amount);
}
```

**2. Try-With-Resources:**
```java
try (InputStream in = new FileInputStream(file);
     OutputStream out = new FileOutputStream(output)) {
    byte[] buffer = new byte[8192];
    int len;
    while ((len = in.read(buffer)) > 0) {
        out.write(buffer, 0, len);
    }
}
```

**3. Exception Translation:**
```java
public void saveData(Data data) {
    try {
        externalService.save(data);
    } catch (RemoteException e) {
        throw new BusinessException(ErrorCode.THIRD_PARTY_ERROR,
            "Failed to save data to external service", e);
    }
}
```

## Code Review Checklist

**Exception Handling:**
- [ ] Specific exceptions caught (not Exception)
- [ ] No empty catch blocks
- [ ] Resources properly closed (try-with-resources)
- [ ] Finally blocks don't have return statements
- [ ] Custom exceptions extend appropriate base class
- [ ] Error codes follow standard format

**Logging:**
- [ ] SLF4J used (not Log4j directly)
- [ ] Appropriate log levels used
- [ ] Parameterized logging for performance
- [ ] Sensitive data masked
- [ ] Context information included
- [ ] Debug logging has guards

**Unit Tests:**
- [ ] Tests are fully automated
- [ ] Tests are independent of each other
- [ ] Tests are repeatable
- [ ] Test coverage meets requirements (70%+)
- [ ] Tests follow BCDE principles
- [ ] Proper test data setup and cleanup

---

**Source:** Based on Alibaba Java Development Handbook (Huangshan Edition 1.7.1)
**Last Updated:** 2022.02.03