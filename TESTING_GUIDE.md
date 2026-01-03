# Testing Guide - Interview Perspective

## What Are These Tests? (Simple Explanation)

Think of testing like **quality control** in a factory. You want to make sure each part of your application works correctly before users use it. There are different types of tests that check different parts:

---

## 1. Unit Tests for Service Layer

### **What They Do:**
Test **ONE piece of business logic** in isolation. Like testing a single function to make sure it does its job correctly.

### **Why They Matter:**
- **Fast**: Run in milliseconds (no database, no network)
- **Isolated**: Test only the logic, not dependencies
- **Catch bugs early**: Find problems before integration

### **How They Work:**
You **mock** (fake) all dependencies (database, other services) and test just the service method.

### **Real Example from Your Codebase:**

**Service to Test:** `AuthService.register()` method

```java
// What the method does:
// 1. Checks if username exists
// 2. Checks if email exists  
// 3. Creates new user
// 4. Generates JWT token
// 5. Returns token

// Unit Test Example:
@Test
void testRegister_WhenUsernameExists_ShouldThrowException() {
    // Arrange (Setup)
    RegisterRequestDto request = new RegisterRequestDto();
    request.setUsername("john");
    request.setEmail("john@email.com");
    request.setPassword("password123");
    
    // Mock the repository to return existing user
    when(userRepository.findByUsername("john"))
        .thenReturn(Optional.of(new User()));
    
    // Act & Assert (Test)
    assertThrows(RuntimeException.class, () -> {
        authService.register(request);
    });
    
    // Verify: Make sure save() was NEVER called
    verify(userRepository, never()).save(any());
}

@Test
void testRegister_WhenValidInput_ShouldCreateUserAndReturnToken() {
    // Arrange
    RegisterRequestDto request = new RegisterRequestDto();
    request.setUsername("newuser");
    request.setEmail("new@email.com");
    request.setPassword("password123");
    
    // Mock: No existing user
    when(userRepository.findByUsername("newuser")).thenReturn(Optional.empty());
    when(userRepository.findByEmail("new@email.com")).thenReturn(Optional.empty());
    
    // Mock: Password encoder
    when(passwordEncoder.encode("password123")).thenReturn("hashed_password");
    
    // Mock: User details service
    UserDetails userDetails = mock(UserDetails.class);
    when(userDetailsService.loadUserByUsername("newuser")).thenReturn(userDetails);
    
    // Mock: JWT token generator
    when(jwtTokenUtil.generateToken(userDetails)).thenReturn("fake_jwt_token");
    
    // Act
    AuthResponseDto response = authService.register(request);
    
    // Assert
    assertEquals("fake_jwt_token", response.getToken());
    
    // Verify: User was saved with correct data
    verify(userRepository).save(argThat(user -> 
        user.getUsername().equals("newuser") &&
        user.getEmail().equals("new@email.com") &&
        user.getPassword().equals("hashed_password")
    ));
}
```

**Interview Answer:**
> "Unit tests for services test business logic in isolation. For example, I test `AuthService.register()` by mocking the database and checking that it correctly validates duplicate usernames, hashes passwords, and generates tokens. This catches logic errors fast without needing a real database."

---

## 2. Integration Tests for Repositories

### **What They Do:**
Test **database operations** with a **real database** (usually in-memory like H2). Verify that your repository methods actually save, find, and query data correctly.

### **Why They Matter:**
- **Real database**: Tests actual SQL queries
- **Catch SQL errors**: Find problems with queries, relationships, constraints
- **Verify JPA mappings**: Make sure entities map to tables correctly

### **How They Work:**
Use a test database (H2 or Testcontainers) and test repository methods with real data.

### **Real Example from Your Codebase:**

**Repository to Test:** `UserRepository.findByUsername()`

```java
@SpringBootTest
@Transactional  // Rollback after each test
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindByUsername_WhenUserExists_ShouldReturnUser() {
        // Arrange: Create and save a real user in test database
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@email.com");
        user.setPassword("hashed");
        userRepository.save(user);
        
        // Act: Query the database
        Optional<User> found = userRepository.findByUsername("testuser");
        
        // Assert: Verify it was found
        assertTrue(found.isPresent());
        assertEquals("testuser", found.get().getUsername());
        assertEquals("test@email.com", found.get().getEmail());
    }
    
    @Test
    void testFindByUsername_WhenUserNotExists_ShouldReturnEmpty() {
        // Act
        Optional<User> found = userRepository.findByUsername("nonexistent");
        
        // Assert
        assertFalse(found.isPresent());
    }
    
    @Test
    void testFindByEmail_WhenEmailExists_ShouldReturnUser() {
        // Arrange
        User user = new User();
        user.setUsername("user1");
        user.setEmail("unique@email.com");
        user.setPassword("pass");
        userRepository.save(user);
        
        // Act
        Optional<User> found = userRepository.findByEmail("unique@email.com");
        
        // Assert
        assertTrue(found.isPresent());
        assertEquals("unique@email.com", found.get().getEmail());
    }
}
```

**Interview Answer:**
> "Integration tests for repositories verify database operations work correctly. I test `UserRepository.findByUsername()` with a real test database to ensure the query works, handles nulls properly, and returns the correct user. This catches SQL errors and JPA mapping issues."

---

## 3. Controller Tests with MockMvc

### **What They Do:**
Test **HTTP endpoints** (REST APIs) without starting the full server. Simulate HTTP requests and verify responses, status codes, and JSON structure.

### **Why They Matter:**
- **Test API contracts**: Verify endpoints return correct status codes and JSON
- **Test request/response mapping**: Ensure DTOs are converted correctly
- **Test security**: Verify authentication/authorization works
- **Fast**: No need to start full Spring Boot application

### **How They Work:**
Use MockMvc to simulate HTTP requests (GET, POST, etc.) and check responses.

### **Real Example from Your Codebase:**

**Controller to Test:** `AuthController.register()` endpoint

```java
@WebMvcTest(AuthController.class)  // Only loads controller, not full app
class AuthControllerTest {
    
    @Autowired
    private MockMvc mockMvc;  // Simulates HTTP requests
    
    @MockBean  // Mock the service (don't use real service)
    private AuthService authService;
    
    @Autowired
    private ObjectMapper objectMapper;  // Convert objects to JSON
    
    @Test
    void testRegister_WhenValidRequest_ShouldReturn200AndToken() throws Exception {
        // Arrange
        RegisterRequestDto request = new RegisterRequestDto();
        request.setUsername("newuser");
        request.setEmail("new@email.com");
        request.setPassword("password123");
        
        AuthResponseDto response = new AuthResponseDto("jwt_token_123");
        when(authService.register(any(RegisterRequestDto.class)))
            .thenReturn(response);
        
        // Act & Assert: Simulate POST request
        mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())  // Check status code 200
            .andExpect(jsonPath("$.token").value("jwt_token_123"));  // Check JSON response
    }
    
    @Test
    void testRegister_WhenInvalidJson_ShouldReturn400() throws Exception {
        // Act & Assert: Send invalid JSON
        mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{ invalid json }"))
            .andExpect(status().isBadRequest());  // Should return 400
    }
    
    @Test
    void testLogin_WhenValidCredentials_ShouldReturn200() throws Exception {
        // Arrange
        AuthRequestDto request = new AuthRequestDto();
        request.setUsername("user");
        request.setPassword("pass");
        
        AuthResponseDto response = new AuthResponseDto("token");
        when(authService.login(any())).thenReturn(response);
        
        // Act & Assert
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.token").exists());
    }
}
```

**Another Example: Testing UserController**

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testGetUserProfile_WhenAuthenticated_ShouldReturn200() throws Exception {
        // Arrange
        User user = new User();
        user.setId(1L);
        user.setUsername("john");
        user.setXp(100);
        
        when(userService.getUserProfile()).thenReturn(user);
        
        // Act & Assert
        mockMvc.perform(get("/api/users/profile")
                .header("Authorization", "Bearer fake_token"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("john"))
            .andExpect(jsonPath("$.xp").value(100));
    }
    
    @Test
    void testAddXp_WhenValidRequest_ShouldReturn200() throws Exception {
        // Arrange
        XpUpdateDto request = new XpUpdateDto();
        request.setXpAmount(50);
        
        User updatedUser = new User();
        updatedUser.setXp(150);
        
        when(userService.addXp(any())).thenReturn(updatedUser);
        
        // Act & Assert
        mockMvc.perform(post("/api/users/xp")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.xp").value(150));
    }
}
```

**Interview Answer:**
> "Controller tests with MockMvc verify REST endpoints work correctly. I test `POST /api/auth/register` by simulating HTTP requests and checking it returns status 200 and a JWT token in the JSON response. This ensures the API contract is correct and request/response mapping works."

---

## 4. Security and Authentication Tests

### **What They Do:**
Test that **security rules work correctly**. Verify:
- Protected endpoints require authentication
- Users can only access their own data
- JWT tokens are validated
- Unauthorized requests are rejected

### **Why They Matter:**
- **Prevent security breaches**: Catch vulnerabilities before production
- **Verify authorization**: Ensure users can't access others' data
- **Test JWT flow**: Verify token generation and validation

### **How They Work:**
Test endpoints with/without authentication tokens, test with different user roles.

### **Real Example from Your Codebase:**

**Testing Security for Protected Endpoints**

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtTokenUtil jwtTokenUtil;
    
    @Test
    void testGetUserProfile_WithoutToken_ShouldReturn401() throws Exception {
        // Act & Assert: No authentication token
        mockMvc.perform(get("/api/users/profile"))
            .andExpect(status().isUnauthorized());  // Should return 401
    }
    
    @Test
    void testGetUserProfile_WithValidToken_ShouldReturn200() throws Exception {
        // Arrange: Create user and generate token
        User user = new User();
        user.setUsername("testuser");
        user.setPassword(passwordEncoder.encode("password"));
        user.setEmail("test@email.com");
        userRepository.save(user);
        
        UserDetails userDetails = userDetailsService.loadUserByUsername("testuser");
        String token = jwtTokenUtil.generateToken(userDetails);
        
        // Act & Assert: Include token in header
        mockMvc.perform(get("/api/users/profile")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
    
    @Test
    void testGetUserProfile_WithInvalidToken_ShouldReturn401() throws Exception {
        // Act & Assert: Invalid token
        mockMvc.perform(get("/api/users/profile")
                .header("Authorization", "Bearer invalid_token_123"))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    void testAddXp_WithoutToken_ShouldReturn401() throws Exception {
        XpUpdateDto request = new XpUpdateDto();
        request.setXpAmount(50);
        
        mockMvc.perform(post("/api/users/xp")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    void testPublicEndpoint_WithoutToken_ShouldReturn200() throws Exception {
        // Some endpoints might be public (like /api/auth/login)
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"user\",\"password\":\"pass\"}"))
            .andExpect(status().isOk());  // Public endpoint, no token needed
    }
}
```

**Testing Authorization (User Can Only Access Own Data)**

```java
@Test
void testGetUserProfile_UserCanOnlySeeOwnProfile() throws Exception {
    // Arrange: Create two users
    User user1 = createUser("user1", "user1@email.com");
    User user2 = createUser("user2", "user2@email.com");
    
    String token1 = generateTokenForUser("user1");
    String token2 = generateTokenForUser("user2");
    
    // Act & Assert: User1's token should return user1's data
    mockMvc.perform(get("/api/users/profile")
            .header("Authorization", "Bearer " + token1))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.username").value("user1"));
    
    // User2's token should return user2's data
    mockMvc.perform(get("/api/users/profile")
            .header("Authorization", "Bearer " + token2))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.username").value("user2"));
}
```

**Interview Answer:**
> "Security tests verify authentication and authorization work correctly. I test that protected endpoints like `/api/users/profile` return 401 without a token, accept valid JWT tokens, and reject invalid tokens. I also verify users can only access their own data, preventing unauthorized access."

---

## Summary Table

| Test Type | What It Tests | Speed | Dependencies | Example |
|-----------|---------------|-------|--------------|---------|
| **Unit Tests (Service)** | Business logic | ⚡⚡⚡ Very Fast | Mocked | `AuthService.register()` logic |
| **Integration Tests (Repository)** | Database operations | ⚡⚡ Fast | Real test DB | `UserRepository.findByUsername()` |
| **Controller Tests (MockMvc)** | HTTP endpoints | ⚡⚡⚡ Very Fast | Mocked services | `POST /api/auth/register` |
| **Security Tests** | Authentication/Authorization | ⚡⚡ Medium | Real security config | JWT token validation |

---

## Interview Talking Points

### **Why Test Coverage Matters:**
1. **Catch bugs early**: Find problems before users do
2. **Refactor safely**: Change code knowing tests will catch breaks
3. **Documentation**: Tests show how code should be used
4. **Confidence**: Deploy knowing code works

### **How to Answer "What Tests Do You Write?"**

> "I write comprehensive tests at multiple levels:
> 
> 1. **Unit tests** for service layer business logic - like testing `AuthService.register()` validates duplicates and hashes passwords correctly, using mocks for dependencies.
> 
> 2. **Integration tests** for repositories - testing `UserRepository.findByUsername()` with a real test database to verify SQL queries and JPA mappings work.
> 
> 3. **Controller tests** with MockMvc - testing REST endpoints like `POST /api/auth/register` to ensure correct HTTP status codes and JSON responses.
> 
> 4. **Security tests** - verifying protected endpoints require authentication, JWT tokens are validated, and users can only access authorized resources.
> 
> This layered approach catches bugs at different levels and gives confidence when deploying."

---

## Quick Reference: Test Annotations

```java
// Unit Test (Service)
@ExtendWith(MockitoExtension.class)
class AuthServiceTest {
    @Mock
    private UserRepository userRepository;  // Mock dependency
    
    @InjectMocks
    private AuthService authService;  // Real service, injects mocks
}

// Integration Test (Repository)
@SpringBootTest
@Transactional
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;  // Real repository
}

// Controller Test
@WebMvcTest(AuthController.class)
class AuthControllerTest {
    @Autowired
    private MockMvc mockMvc;  // HTTP simulator
    
    @MockBean
    private AuthService authService;  // Mock service
}

// Security Test
@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {
    @Autowired
    private MockMvc mockMvc;
}
```

---

## Next Steps

To implement these tests in your project:

1. **Add test dependencies** to `pom.xml` (JUnit 5, Mockito, MockMvc)
2. **Create test classes** in `src/test/java/`
3. **Write tests** following the examples above
4. **Run tests** with `mvn test` or IDE

Would you like me to create actual test files for your codebase?

