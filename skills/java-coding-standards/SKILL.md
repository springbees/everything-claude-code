---
name: java-coding-standards
description: Alibaba Java Coding Standards including naming conventions, constant definitions, code format, OOP rules, date/time handling, collections, concurrency, control statements, and comments. Use this when writing Java code, conducting code reviews, or ensuring adherence to industry-standard Java coding practices.
---

# Alibaba Java Coding Standards

## Overview
This skill provides comprehensive Java coding standards based on the Alibaba Java Development Handbook (Huangshan Edition). It covers naming conventions, code style, OOP principles, and best practices for writing maintainable, readable Java code.

## When to Use This Skill
Invoke this skill when you need to:
- Write new Java code following industry standards
- Review existing Java code for compliance
- Refactor code to meet coding standards
- Establish team coding guidelines
- Ensure consistent naming and formatting across projects

## Key Standards Covered

### 1. Naming Conventions

**Class Names:**
- Use `UpperCamelCase` for class names
- Exceptions: DO/BO/VO/DTO/UID
- Abstract classes start with `Abstract` or `Base`
- Exception classes end with `Exception`
- Test classes end with `Test`

```java
// Good
public class UserService {}
public abstract class AbstractController {}
public class ResourceNotFoundException {}
public class UserServiceTest {}

// Bad
public class userService {}
public class resourceexception {}
```

**Method and Variable Names:**
- Use `lowerCamelCase` for methods, parameters, and variables
- Boolean variables should NOT use `is` prefix (framework serialization issues)

```java
// Good
private String userName;
public void getUserById() {}
private boolean deleted;

// Bad
private String UserName;
public void Getuserbyid() {}
private boolean isDeleted;  // causes serialization issues
```

**Constant Names:**
- Use `UPPER_SNAKE_CASE` for constants
- Strive for semantic completeness

```java
// Good
public static final int MAX_STOCK_COUNT = 1000;
public static final String CACHE_EXPIRED_TIME = "3600";

// Bad
public static final int maxCount = 1000;
public static final int EXPIRED = 3600;
```

**Package Names:**
- Use all lowercase letters
- Use singular form
- One natural English word between dots

```java
// Good
package com.alibaba.ei.kunlun.aap.util;

// Bad
package com.alibaba.ei.Kunlun.AAP.Util;
```

### 2. Code Formatting

**Braces and Indentation:**
- Use 4 spaces for indentation (no tabs)
- Opening brace on same line for most constructs
- Consistent brace style

```java
// Good
public void method() {
    if (condition) {
        doSomething();
    }
}

// Bad
public void method()
{
  if ( condition )
  {
    doSomething();
  }
}
```

**Line Length:**
- Maximum line length: 120 characters
- Break long lines at logical points

### 3. OOP Conventions

**Object-Oriented Principles:**
- Favor composition over inheritance
- Use interfaces for abstraction
- Follow SOLID principles
- Avoid static methods for business logic

```java
// Good - using interface
public interface UserRepository {
    User findById(Long id);
}

// Bad - tight coupling to implementation
public class UserService {
    private UserRepositoryImpl userRepository;
}
```

**equals() and hashCode():**
- Always override both together
- Use same fields for both methods
- Handle null values properly

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    User user = (User) o;
    return Objects.equals(id, user.id);
}

@Override
public int hashCode() {
    return Objects.hash(id);
}
```

### 4. Collection Handling

**Initialization:**
- Specify initial capacity for collections when size is known

```java
// Good
Map<String, User> userMap = new HashMap<>(16);
List<String> items = new ArrayList<>(100);

// Bad - may cause multiple resizes
Map<String, User> userMap = new HashMap<>();
```

**Traversal:**
- Use enhanced for-loop or forEach for read-only traversal
- Use Iterator when removal is needed

```java
// Good
for (User user : users) {
    System.out.println(user.getName());
}

// Good for removal
Iterator<User> iterator = users.iterator();
while (iterator.hasNext()) {
    if (shouldRemove(iterator.next())) {
        iterator.remove();
    }
}

// Bad - ConcurrentModificationException
for (User user : users) {
    if (shouldRemove(user)) {
        users.remove(user);
    }
}
```

**Conversion:**
- Use Arrays.asList() for array to list
- Use collection.toArray() for list to array

```java
// Good
String[] array = {"a", "b", "c"};
List<String> list = new ArrayList<>(Arrays.asList(array));

// Good
String[] result = list.toArray(new String[0]);
```

### 5. Concurrency

**Thread Creation:**
- Use thread pools instead of creating threads directly

```java
// Good
ExecutorService executor = Executors.newFixedThreadPool(10);

// Bad
new Thread(() -> doSomething()).start();
```

**Synchronization:**
- Minimize synchronized block scope
- Use concurrent collections when appropriate

```java
// Good
synchronized(this) {
    // minimal code
}

// Good - for high concurrency
ConcurrentHashMap<String, User> map = new ConcurrentHashMap<>();

// Bad - entire method
public synchronized void doSomething() {
    // lots of code
}
```

**Volatile:**
- Use volatile for visibility guarantees only
- Not suitable for compound actions

### 6. Control Statements

**Switch Statements:**
- Always include break (or comment if intentional fall-through)
- Include default case

```java
// Good
switch (status) {
    case PENDING:
        handlePending();
        break;
    case APPROVED:
        handleApproved();
        break;
    default:
        handleUnknown();
}

// Bad - missing breaks
switch (status) {
    case PENDING:
        handlePending();
    case APPROVED:
        handleApproved();
}
```

**If-Else:**
- Use guard clauses to reduce nesting
- Keep conditions simple

```java
// Good
public void process(User user) {
    if (user == null) {
        return;
    }
    // process user
}

// Bad - excessive nesting
public void process(User user) {
    if (user != null) {
        if (user.isActive()) {
            if (user.isValid()) {
                // process
            }
        }
    }
}
```

### 7. Comments and Documentation

**Javadoc:**
- All public classes/methods must have Javadoc
- Include @param, @return, @throws as needed

```java
/**
 * Validates user input and returns validation result.
 *
 * @param user the user to validate, must not be null
 * @return true if user is valid, false otherwise
 * @throws IllegalArgumentException if user is null
 */
public boolean validate(User user) {
    // implementation
}
```

**Code Comments:**
- Comment "why" not "what"
- Keep comments up to date
- Avoid commented-out code

```java
// Good - explains reasoning
// Using String.intern() to reduce memory for common values
String normalized = value.intern();

// Bad - obvious comment
// Set the name
user.setName(name);
```

### 8. Date/Time Handling

**Use java.time (Java 8+):**
```java
// Good
LocalDate today = LocalDate.now();
LocalDateTime now = LocalDateTime.now();
ZonedDateTime utc = ZonedDateTime.now(ZoneOffset.UTC);

// Bad - outdated
Date date = new Date();
Calendar calendar = Calendar.getInstance();
```

**Formatting:**
```java
// Good
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String formatted = now.format(formatter);

// Bad
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
```

### 9. Exception Handling

**Specific Exceptions:**
```java
// Good
try {
    // code
} catch (NumberFormatException e) {
    // handle specifically
} catch (IllegalArgumentException e) {
    // handle specifically
}

// Bad - too broad
try {
    // code
} catch (Exception e) {
    // handle everything
}
```

**NPE Prevention:**
```java
// Good - use Optional
Optional<User> user = userRepository.findById(id);
if (user.isPresent()) {
    return user.get();
}

// Good - early return
public void process(User user) {
    if (user == null || user.getName() == null) {
        return;
    }
    // process
}
```

### 10. Miscellaneous Best Practices

**Avoid Magic Numbers:**
```java
// Good
private static final int MAX_RETRY_COUNT = 3;
if (retryCount >= MAX_RETRY_COUNT) {
    // handle
}

// Bad
if (retryCount >= 3) {
    // handle
}
```

**Use Constants for Strings:**
```java
// Good
private static final String CACHE_KEY_PREFIX = "user:";
String cacheKey = CACHE_KEY_PREFIX + userId;

// Bad
String cacheKey = "user:" + userId;
```

**Logging:**
- Use SLF4J with appropriate log levels
- Use parameterized logging

```java
// Good
logger.info("User {} logged in at {}", username, timestamp);
logger.debug("Processing order: {}", orderId);

// Bad
logger.info("User " + username + " logged in at " + timestamp);
```

## Quick Reference

### Common Patterns

**Singleton Pattern:**
```java
public enum Singleton {
    INSTANCE;
    public void doSomething() {
        // implementation
    }
}
```

**Builder Pattern:**
```java
public class User {
    private final String name;
    private final String email;

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
    }

    public static class Builder {
        private String name;
        private String email;

        public Builder name(String name) {
            this.name = name;
            return this;
        }

        public Builder email(String email) {
            this.email = email;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.Builder()
    .name("John")
    .email("john@example.com")
    .build();
```

## Common Pitfalls to Avoid

1. **Using == for String comparison**
   ```java
   // Bad
   if (str1 == str2) { }

   // Good
   if (str1.equals(str2)) { }
   if (Objects.equals(str1, str2)) { }
   ```

2. **Ignoring exceptions**
   ```java
   // Bad
   try {
       // code
   } catch (Exception e) {
       // empty catch
   }

   // Good
   try {
       // code
   } catch (Exception e) {
       logger.error("Operation failed", e);
       // handle or rethrow
   }
   ```

3. **Resource leaks**
   ```java
   // Good - try-with-resources
   try (Connection conn = dataSource.getConnection()) {
       // use connection
   }

   // Bad - manual close that may be skipped
   Connection conn = dataSource.getConnection();
   // use connection
   conn.close();
   ```

4. **String concatenation in loops**
   ```java
   // Bad
   String result = "";
   for (String item : items) {
       result += item;
   }

   // Good
   StringBuilder sb = new StringBuilder();
   for (String item : items) {
       sb.append(item);
   }
   String result = sb.toString();
   ```

## Code Review Checklist

- [ ] Naming follows conventions (classes, methods, variables)
- [ ] No magic numbers or strings
- [ ] Proper exception handling (specific catches, no empty catches)
- [ ] Resources properly closed (try-with-resources)
- [ ] Thread-safe when needed
- [ ] Collections initialized with proper capacity
- [ ] Javadoc present for public APIs
- [ ] No commented-out code
- [ ] Proper logging (SLF4J, parameterized)
- [ ] equals() and hashCode() overridden together

---

**Source:** Based on Alibaba Java Development Handbook (Huangshan Edition 1.7.1)
**Last Updated:** 2022.02.03