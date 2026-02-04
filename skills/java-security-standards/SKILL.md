---
name: java-security-standards
description: Java security standards covering authentication, authorization, input validation, SQL injection prevention, XSS/CSRF protection, data encryption, and secure coding practices. Use this when implementing security features or conducting security reviews.
---

# Java Security Standards

## Overview
Comprehensive security standards for Java applications based on Alibaba Java Development Handbook, covering authentication, authorization, input validation, and common vulnerability prevention.

## When to Use
- Implementing authentication/authorization
- Validating user input
- Preventing common vulnerabilities (SQL injection, XSS, CSRF)
- Handling sensitive data
- Conducting security code reviews

## 1. Authentication & Authorization

### Permission Control
```java
// Good: check permissions for user-specific resources
@GetMapping("/user/{userId}/profile")
public ApiResponse<UserProfile> getUserProfile(
    @PathVariable Long userId,
    @AuthenticationPrincipal CurrentUser currentUser) {

    // Horizontal permission check
    if (!currentUser.getId().equals(userId)) {
        throw new ForbiddenException("Access denied");
    }

    UserProfile profile = userService.getProfile(userId);
    return ApiResponse.success(profile);
}

// Bad: no permission check
@GetMapping("/user/{userId}/profile")
public UserProfile getUserProfile(@PathVariable Long userId) {
    return userService.getProfile(userId);  // Can access anyone's profile!
}
```

### Role-Based Access Control
```java
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/admin/users")
public ApiResponse<User> createUser(@RequestBody UserCreateRequest request) {
    // Only admins can create users
}

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
@PutMapping("/admin/users/{userId}")
public ApiResponse<User> updateUser(@PathVariable Long userId,
                                    @RequestBody UserUpdateRequest request) {
    // Admins and managers can update users
}
```

## 2. Input Validation

### Parameter Validation
```java
// Good: comprehensive validation
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ApiResponse<Order> createOrder(
        @RequestBody @Valid OrderCreateRequest request) {

        // Additional business validation
        if (request.getAmount().compareTo(new BigDecimal("1000000")) > 0) {
            throw new BadRequestException("Amount exceeds limit");
        }

        Order order = orderService.createOrder(request);
        return ApiResponse.success(order);
    }
}

// Validation with annotations
public class OrderCreateRequest {

    @NotBlank(message = "Product ID is required")
    private String productId;

    @NotNull(message = "Quantity is required")
    @Min(value = 1, message = "Quantity must be at least 1")
    @Max(value = 999, message = "Quantity cannot exceed 999")
    private Integer quantity;

    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.01", message = "Amount must be at least 0.01")
    @DecimalMax(value = "1000000", message = "Amount cannot exceed 1000000")
    private BigDecimal amount;
}

// Bad: no validation
public void createOrder(OrderCreateRequest request) {
    // Direct processing without validation
}
```

### Pagination Validation
```java
// Good: validate pagination parameters
@GetMapping("/users")
public ApiResponse<PageResult<UserDTO>> listUsers(
    @RequestParam(defaultValue = "1") @Min(1) Integer page,
    @RequestParam(defaultValue = "20") @Min(1) @Max(100) Integer size) {

    if (size > 100) {
        throw new BadRequestException("Page size cannot exceed 100");
    }

    PageResult<UserDTO> users = userService.listUsers(page, size);
    return ApiResponse.success(users);
}

// Bad: no limits on page size
@GetMapping("/users")
public List<User> listUsers(
    @RequestParam(defaultValue = "20") Integer size) {

    // Can request 1 million records!
    return userService.listUsers(size);
}
```

## 3. SQL Injection Prevention

### Parameterized Queries
```java
// Good: parameterized query
@Mapper
public interface UserDao {
    @Select("SELECT * FROM user WHERE username = #{username}")
    UserDO findByUsername(@Param("username") String username);
}

// MyBatis XML
<select id="findByUsername" resultMap="BaseResultMap">
    SELECT id, username, email FROM user
    WHERE username = #{username}
</select>

// Bad: string concatenation
@Select("SELECT * FROM user WHERE username = '${username}'")
UserDO findByUsername(@Param("username") String username);

// Very bad: direct SQL construction
String sql = "SELECT * FROM user WHERE username = '" + username + "'";
```

### Like Query Special Characters
```java
// Good: escape special characters
public List<UserDO> searchByUsername(String keyword) {
    String escaped = keyword.replace("\\", "\\\\")
                            .replace("%", "\\%")
                            .replace("_", "\\_");
    return userDao.findByUsernameLike("%" + escaped + "%");
}

// MyBatis
<select id="findByUsernameLike" resultMap="BaseResultMap">
    SELECT * FROM user
    WHERE username LIKE CONCAT('%', #{keyword}, '%')
    ESCAPE '\\'
</select>
```

## 4. XSS Prevention

### Output Encoding
```java
// Good: HTML escape user input
import org.apache.commons.text.StringEscapeUtils;

@GetMapping("/profile")
public String viewProfile(@RequestParam String name, Model model) {
    String escapedName = StringEscapeUtils.escapeHtml4(name);
    model.addAttribute("userName", escapedName);
    return "profile";
}

// Or use template engine auto-escaping (Thymeleaf does this by default)
// Thymeleaf template:
// <p th:text="${userName}">Username</p>  <!-- Auto-escaped -->
// <p th:utext="${userName}">Username</p>  <!-- NOT escaped -->

// Bad: unescaped output
@GetMapping("/profile")
public String viewProfile(@RequestParam String name, Model model) {
    model.addAttribute("userName", name);  // XSS vulnerability!
    return "profile";
}
```

### Content Security Policy
```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers()
            .contentSecurityPolicy("default-src 'self'; script-src 'self' 'unsafe-inline'");
        return http.build();
    }
}
```

## 5. CSRF Protection

### Enable CSRF Protection
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf()
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
        return http.build();
    }
}
```

### Stateless API (if using JWT)
```java
// For stateless APIs (JWT), CSRF can be disabled
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        return http.build();
    }
}
```

## 6. Sensitive Data Handling

### Data Masking
```java
// Good: mask sensitive data
public class UserVO {
    private String username;
    private String phone;      // Masked
    private String email;      // Masked
    private String idCard;     // Masked
}

public class UserService {

    private String maskPhone(String phone) {
        if (phone == null || phone.length() != 11) return phone;
        return phone.substring(0, 3) + "****" + phone.substring(7);
    }

    private String maskEmail(String email) {
        if (email == null || !email.contains("@")) return email;
        String[] parts = email.split("@");
        return parts[0].charAt(0) + "***@" + parts[1];
    }

    private String maskIdCard(String idCard) {
        if (idCard == null || idCard.length() < 18) return idCard;
        return idCard.substring(0, 6) + "********" + idCard.substring(14);
    }

    public UserVO getUserVO(UserDO userDO) {
        UserVO vo = new UserVO();
        vo.setUsername(userDO.getUsername());
        vo.setPhone(maskPhone(userDO.getPhone()));
        vo.setEmail(maskEmail(userDO.getEmail()));
        vo.setIdCard(maskIdCard(userDO.getIdCard()));
        return vo;
    }
}
```

### Password Encryption
```java
// Good: use BCrypt
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

@Service
public class UserService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    public void createUser(UserCreateRequest request) {
        UserDO user = new UserDO();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        userDao.insert(user);
    }
}
```

### Configuration File Encryption
```yaml
# application.yml
# Bad: plaintext password
spring:
  datasource:
    password: mypassword123

# Good: encrypted password
spring:
  datasource:
    password: ENC(AQIDBAUGBwgJCgsMDQ4PEBESExQVFhcYGRobHB0eHyA=)

# Or use external secret management
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      scheme: http
```

## 7. File Upload Security

### File Type Validation
```java
@Controller
public class FileUploadController {

    private static final Set<String> ALLOWED_EXTENSIONS =
        Set.of("jpg", "jpeg", "png", "pdf", "doc", "docx");

    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB

    @PostMapping("/upload")
    public ApiResponse<String> uploadFile(@RequestParam("file") MultipartFile file) {
        // Check if file is empty
        if (file.isEmpty()) {
            throw new BadRequestException("File is empty");
        }

        // Check file size
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new BadRequestException("File size exceeds limit");
        }

        // Check file extension
        String filename = file.getOriginalFilename();
        String extension = filename.substring(filename.lastIndexOf(".") + 1)
            .toLowerCase();

        if (!ALLOWED_EXTENSIONS.contains(extension)) {
            throw new BadRequestException("Invalid file type");
        }

        // Check content type (additional validation)
        String contentType = file.getContentType();
        if (!isValidContentType(contentType)) {
            throw new BadRequestException("Invalid content type");
        }

        // Save file
        String savedPath = fileService.saveFile(file);
        return ApiResponse.success(savedPath);
    }
}
```

### Prevent Path Traversal
```java
// Good: validate filename
@PostMapping("/upload")
public String uploadFile(@RequestParam("file") MultipartFile file,
                        @RequestParam("path") String path) {

    // Remove path traversal attempts
    String safePath = path.replaceAll("\\.\\.", "");
    safePath = safePath.replaceAll("/", "");

    // Or use predefined paths
    Set<String> allowedPaths = Set.of("documents", "images", "avatars");
    if (!allowedPaths.contains(safePath)) {
        throw new BadRequestException("Invalid path");
    }

    String fullPath = UPLOAD_DIR + "/" + safePath + "/" + filename;
    file.transferTo(new File(fullPath));
    return "success";
}

// Bad: direct path usage
String fullPath = UPLOAD_DIR + "/" + path + "/" + filename;
```

## 8. Rate Limiting & Anti-Replay

### Rate Limiting
```java
@RestController
@RequestMapping("/api/sms")
public class SmsController {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @PostMapping("/send")
    public ApiResponse<Void> sendSms(@RequestParam String phone) {

        // Check rate limit: max 5 SMS per hour per phone
        String key = "sms:limit:" + phone;
        Long count = redisTemplate.opsForValue().increment(key);

        if (count == null) {
            redisTemplate.opsForValue().set(key, "1", 1, TimeUnit.HOURS);
        } else if (count > 5) {
            throw new TooManyRequestsException("Too many SMS requests");
        } else {
            redisTemplate.expire(key, 1, TimeUnit.HOURS);
        }

        // Send SMS
        smsService.sendVerificationCode(phone);
        return ApiResponse.success(null);
    }
}
```

### Anti-Replay Protection
```java
@RestController
@RequestMapping("/api/payment")
public class PaymentController {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @PostMapping("/pay")
    public ApiResponse<PaymentResult> pay(@RequestBody @Valid PaymentRequest request) {

        // Check for replay using unique request ID
        String requestId = request.getRequestId();
        String key = "payment:request:" + requestId;

        Boolean exists = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", 10, TimeUnit.MINUTES);

        if (Boolean.FALSE.equals(exists)) {
            throw new BadRequestException("Duplicate request");
        }

        // Process payment
        PaymentResult result = paymentService.processPayment(request);
        return ApiResponse.success(result);
    }
}
```

## 9. URL Redirection Security

### Whitelist Validation
```java
@Controller
public class RedirectController {

    private static final Set<String> ALLOWED_DOMAINS = Set.of(
        "example.com",
        "www.example.com",
        "app.example.com"
    );

    @GetMapping("/redirect")
    public String redirect(@RequestParam("url") String url) {

        try {
            URL targetUrl = new URL(url);
            String domain = targetUrl.getHost();

            // Check whitelist
            if (!ALLOWED_DOMAINS.contains(domain)) {
                throw new BadRequestException("Invalid redirect URL");
            }

            return "redirect:" + url;

        } catch (MalformedURLException e) {
            throw new BadRequestException("Invalid URL");
        }
    }
}
```

## Security Checklist

- [ ] Permission checks for user-specific resources
- [ ] Input validation for all parameters
- [ ] SQL parameterized queries (no string concatenation)
- [ ] HTML output encoding (XSS prevention)
- [ ] CSRF protection enabled
- [ ] Sensitive data masked in logs/responses
- [ ] Passwords encrypted (BCrypt)
- [ ] File uploads validated (type, size, content)
- [ ] Rate limiting for sensitive operations
- [ ] URL redirection whitelist
- [ ] Security headers configured (CSP, X-Frame-Options, etc.)

---

**Source:** Alibaba Java Development Handbook (Huangshan Edition 1.7.1)