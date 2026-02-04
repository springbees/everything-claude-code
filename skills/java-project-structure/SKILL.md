---
name: java-project-structure
description: Java project structure standards including application layering, domain models (DO/DTO/BO/VO), Maven dependency management, and server configuration. Use this when structuring Java projects, organizing code, or managing dependencies.
---

# Java Project Structure Standards

## Overview
Standards for Java application architecture, layering, dependency management, and server configuration based on Alibaba Java Development Handbook.

## When to Use
- Designing application architecture
- Organizing project structure
- Managing Maven dependencies
- Configuring application servers

## Layered Architecture

### Recommended Layers
```
┌─────────────────────────────────────┐
│      Open API Layer                 │  RPC/HTTP interfaces
├─────────────────────────────────────┤
│      Display Layer                  │  Template rendering
├─────────────────────────────────────┤
│      Web Layer (Controller)         │  Access control, validation
├─────────────────────────────────────┤
│      Service Layer                  │  Business logic
├─────────────────────────────────────┤
│      Manager Layer                  │  General business processing
├─────────────────────────────────────┤
│      DAO Layer                      │  Data access
├─────────────────────────────────────┤
│  External Services / Database       │
└─────────────────────────────────────┘
```

### Layer Responsibilities

**Web Layer (Controller):**
- Request/response handling
- Parameter validation
- Access control forwarding
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService userService;

    @PostMapping
    public ApiResponse<UserDTO> create(@RequestBody @Valid UserCreateRequest request) {
        UserDTO user = userService.createUser(request);
        return ApiResponse.success(user);
    }
}
```

**Service Layer:**
- Core business logic
- Transaction boundaries
- Domain model operations
```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
    @Autowired
    private UserManager userManager;

    @Transactional
    public UserDTO createUser(UserCreateRequest request) {
        // Business logic
        userManager.checkDuplicate(request.getEmail());
        UserDO userDO = convertToDO(request);
        userDao.insert(userDO);
        return convertToDTO(userDO);
    }
}
```

**Manager Layer:**
- Third-party service encapsulation
- Multi-DAO composition
- Common capability sinking (caching, middleware)
```java
@Component
public class UserManager {
    @Autowired
    private UserDao userDao;
    @Autowired
    private CacheClient cacheClient;

    public UserDO getUserWithCache(Long userId) {
        // Cache handling
        UserDO user = cacheClient.get("user:" + userId);
        if (user == null) {
            user = userDao.findById(userId);
            cacheClient.set("user:" + userId, user);
        }
        return user;
    }
}
```

**DAO Layer:**
- Database operations only
- No business logic
```java
@Repository
public interface UserDao {
    UserDO findById(Long id);
    void insert(UserDO user);
    void update(UserDO user);
}
```

## Domain Models

### Model Types
```java
// DO - Data Object (1:1 with database table)
public class UserDO {
    private Long id;
    private String userName;
    private String email;
    private Boolean isDeleted;
    private Date createTime;
    private Date updateTime;
}

// DTO - Data Transfer Object (Service output)
public class UserDTO {
    private Long id;
    private String userName;
    private String email;
    private List<String> roles;
}

// BO - Business Object (Encapsulates business logic)
public class OrderBO {
    private OrderDO order;
    private List<OrderItemDO> items;
    private UserDO user;

    public BigDecimal calculateTotal() {
        return items.stream()
            .map(item -> item.getPrice().multiply(new BigDecimal(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// VO - View Object (Presentation layer)
public class UserVO {
    private String displayName;
    private String avatarUrl;
    private String status;
}

// Query - Query parameters
public class UserQuery {
    private String userName;
    private Integer status;
    private Integer page = 1;
    private Integer size = 20;
}
```

## Maven Dependency Management

### GAV Rules
```xml
<!-- GroupID: com.{company}.{business_line}[.{sub_business_line}] -->
<groupId>com.alibaba.usercenter</groupId>
<artifactId>usercenter-api</artifactId>
<version>1.0.0</version>
```

### Version Numbering
- Format: `major.minor.patch`
- Start from `1.0.0` (NOT `0.0.1`)
```xml
<version>2.1.3</version>
<!-- 2 = major: incompatible changes -->
<!-- 1 = minor: new features, mostly compatible -->
<!-- 3 = patch: bug fixes, fully compatible -->
```

### Dependency Management
```xml
<!-- Parent POM: define versions in dependencyManagement -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-framework-bom</artifactId>
            <version>${spring.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Child POM: reference without version -->
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>
</dependencies>
```

### Prohibited Practices
```xml
<!-- Bad: SNAPSHOT in production -->
<version>1.0.0-SNAPSHOT</version>

<!-- Bad: Same GAV, different version in sub-projects -->
<!-- Causes: works locally, fails in production -->

<!-- Bad: Enum in interface return value -->
public interface UserService {
    UserStatus getStatus();  // Bad: enum in return
}
```

## Server Configuration

### JVM Settings
```bash
# OOM dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heapdump.hprof

# Memory: set Xms = Xmx
-Xms2g
-Xmx2g

# GC (example: G1GC)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

### Timeout Configuration
```java
// Remote calls MUST have timeout
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(5000);  // 5 seconds
        factory.setReadTimeout(10000);    // 10 seconds
        return new RestTemplate(factory);
    }
}
```

### Thread Pool Isolation
```java
@Configuration
public class ThreadPoolConfig {
    @Bean("slowServiceExecutor")
    public Executor slowServiceExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("slow-svc-");
        executor.initialize();
        return executor;
    }
}

// Usage: isolate slow services
@Async("slowServiceExecutor")
public CompletableFuture<Result> callSlowService(Request request) {
    // slow operation
}
```

## Project Directory Structure

```
project-name/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           ├── controller/      # Web layer
│   │   │           ├── service/         # Service layer
│   │   │           ├── manager/         # Manager layer
│   │   │           ├── dao/             # DAO layer
│   │   │           ├── domain/          # Domain models
│   │   │           │   ├── DO/
│   │   │           │   ├── DTO/
│   │   │           │   ├── BO/
│   │   │           │   └── VO/
│   │   │           └── config/          # Configuration
│   │   └── resources/
│   │       ├── mapper/                  # MyBatis XML
│   │       ├── application.yml
│   │       └── logback.xml
│   └── test/
│       └── java/
│           └── com/
│               └── example/
│                   └── service/        # Unit tests
├── pom.xml
└── README.md
```

## Quick Reference

### Exception Handling by Layer
```
DAO Layer:     catch(Exception) -> throw DAOException(no log)
Manager Layer: catch(Exception) -> throw DAOException(no log, if co-located)
Service Layer: catch(Exception) -> log + throw BusinessException
Web Layer:     catch(Exception) -> convert to error code, no exception thrown
```

### Dependency Guidelines
- Group ID: `com.{company}.{business}[.{sub-business}]`
- Artifact ID: `{product}-{module}`
- Version: `major.minor.patch` starting from `1.0.0`
- No SNAPSHOT in production
- No enum in API return values

### Code Review Checklist
- [ ] Clear layer separation
- [ ] Domain models properly used (DO/DTO/BO/VO/Query)
- [ ] Dependencies managed in parent POM
- [ ] Remote calls have timeouts
- [ ] Thread pools isolated for slow services
- [ ] Exception handling follows layer standards

---

**Source:** Alibaba Java Development Handbook (Huangshan Edition 1.7.1)