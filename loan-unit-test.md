# Unit Testing Guidelines for Loan Service (Spring Boot, JUnit 5, Mockito)

This guideline outlines best practices for writing unit tests in a Spring Boot application using JUnit 5 and Mockito. It is personalized for the **Loan Service team** and covers setup, mocking dependencies, testing service methods, controllers, and handling Spring Boot–specific scenarios.

---

## 1. Introduction

Unit tests in Spring Boot focus on testing **individual components** (e.g., services, controllers) in isolation, mocking dependencies like repositories, external services, or other beans.  

**Why unit testing matters for Loan Service**:
- Validates **loan lifecycle flows** (creation, approval, disbursement, repayment, closure).  
- Ensures **business rules** (amount limits, tenor checks, repayment schedules) are enforced.  
- Confirms **interactions with external services** (journal posting, audit logs, account opening, notifications).  
- Detects regressions early and improves team confidence during refactoring.  

### Key Principles
- **Isolation** → test one component at a time, mock all external dependencies.  
- **Speed** → unit tests must run quickly without booting the entire Spring context.  
- **Clarity** → tests should be easy to read and serve as living documentation.  
- **Repeatability** → consistent results regardless of environment.  
- **Independence** → no reliance on shared state across tests.  

---

## 2. Dependencies in Loan Service (Single Service)

The Loan Service depends on multiple layers and external services:

- **Repositories**  
  - `LoanRepository` → CRUD for loan entities.  
  - `TransactionRepository` → handles loan-related transactions.  

- **External Clients**  
  - `TransactionJournalPostingService` → posts loan transactions to the general ledger.  
  - `CustomerProfileRetrieverService` → fetches customer details.  
  - `AccountOpeningClient` → opens accounts tied to loans.  

- **Other Services**  
  - `AuditLogPublisherService` → publishes loan events for audit trail.  
  - `NotificationService` → sends SMS/email to customers.  

### Mocking Strategy
- Mock all external clients and services with **Mockito**.  
- Repositories should be mocked in unit tests, unless using `@DataJpaTest` for query verification.  
- Do not mock value objects like DTOs; build real ones for clarity.  

---

## 3. Configuration Setup

### 3.1 Maven/Gradle Dependencies

**Maven**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**Gradle**
```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.mockito:mockito-core'
    testImplementation 'org.mockito:mockito-junit-jupiter'
}
```

---

### 3.2 JUnit 5 Setup
- Use `@ExtendWith(MockitoExtension.class)` to enable Mockito.  
- Prefer plain unit tests with `Mockito` instead of loading the full Spring context.  

---

### 3.3 Base Test Class
A reusable base setup for all LoanService tests:

```java
@ExtendWith(MockitoExtension.class)
public abstract class BaseLoanServiceTest {

    @Mock protected LoanRepository loanRepository;
    @Mock protected TransactionRepository transactionRepository;
    @Mock protected TransactionJournalPostingService transactionJournalPostingService;
    @Mock protected CustomerProfileRetrieverService customerProfileRetrieverService;
    @Mock protected AccountOpeningClient accountOpeningClient;
    @Mock protected AuditLogPublisherService auditLogPublisherService;
    @Mock protected NotificationService notificationService;

    @InjectMocks
    protected LoanService loanService;

    protected Loan loan;

    @BeforeEach
    void init() {
        loan = new Loan("LN123", "CUST01", BigDecimal.valueOf(50000), LoanStatus.PENDING);
    }
}
```

---

### 3.4 Test Properties
For integration-style tests:  
`src/test/resources/application-test.properties`
```properties
spring.datasource.url=jdbc:h2:mem:loansdb;DB_CLOSE_DELAY=-1
spring.datasource.driverClassName=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
```

---

## 4. Organizing Test Code

- **Test class names** → `LoanServiceTest`, `LoanControllerTest`.  
- **Method names** → `shouldCreateLoanWhenValidRequest()`.  
- **Structure** → follow Arrange → Act → Assert (AAA).  
- **Mirror package structure** → `src/test/java/com.example.loan/...`.  

---

## 5. Writing Unit Tests for Loan Service

### 5.1 Service Example
```java
@ExtendWith(MockitoExtension.class)
class LoanServiceTest extends BaseLoanServiceTest {

    @Test
    void shouldCreateLoanWhenValidRequest() {
        LoanRequest request = new LoanRequest("CUST01", 50000, 12);
        when(customerProfileRetrieverService.getProfile("CUST01"))
            .thenReturn(Optional.of(new Customer("CUST01", "John Doe")));
        when(loanRepository.save(any(Loan.class))).thenReturn(loan);

        Loan result = loanService.createLoan(request);

        assertEquals(50000, result.getAmount().intValue());
        verify(customerProfileRetrieverService).getProfile("CUST01");
        verify(loanRepository).save(any(Loan.class));
    }

    @Test
    void shouldThrowExceptionWhenCustomerNotFound() {
        LoanRequest request = new LoanRequest("INVALID", 50000, 12);
        when(customerProfileRetrieverService.getProfile("INVALID"))
            .thenReturn(Optional.empty());

        assertThrows(IllegalArgumentException.class,
            () -> loanService.createLoan(request));

        verify(loanRepository, never()).save(any());
    }
}
```

---

### 5.2 Controller Example
```java
@WebMvcTest(LoanController.class)
class LoanControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private LoanService loanService;

    @Test
    void shouldReturnCreatedLoan() throws Exception {
        Loan loan = new Loan("LN123", "CUST01", BigDecimal.valueOf(50000), LoanStatus.APPROVED);
        when(loanService.createLoan(any())).thenReturn(loan);

        mockMvc.perform(post("/api/loans")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"customerId\":\"CUST01\",\"amount\":50000,\"tenor\":12}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.loanId").value("LN123"))
                .andExpect(jsonPath("$.amount").value(50000));
    }
}
```

---

### 5.3 Repository Example (Optional Integration Test)
```java
@DataJpaTest
class LoanRepositoryTest {
    @Autowired private LoanRepository loanRepository;

    @Test
    void shouldFindLoansByCustomerId() {
        Loan loan = new Loan("LN001", "CUST01", BigDecimal.valueOf(7500), LoanStatus.PENDING);
        loanRepository.save(loan);

        List<Loan> loans = loanRepository.findByCustomerId("CUST01");

        assertFalse(loans.isEmpty());
        assertEquals("CUST01", loans.get(0).getCustomerId());
    }
}
```

---

## 6. Advanced Testing Features

- **Parameterized Tests** → cover multiple inputs.  
- **Async Testing** → for methods annotated with `@Async`.  
- **Static Mocking** → using `MockedStatic`.  
- **ArgumentCaptor** → verify saved objects.  

---

## 7. Best Practices

- Test **happy path** + **edge cases**.  
- Keep unit tests **fast and isolated**.  
- Avoid over-verifying mocks.  
- Use `@WebMvcTest` for controllers, not `@SpringBootTest`.  
- Use `@DataJpaTest` sparingly, only for repository queries.  
- Maintain descriptive test names.  

---

## 8. Example: Complete Loan Flow Test

```java
@ExtendWith(MockitoExtension.class)
class LoanServiceFlowTest {

    @Mock private LoanRepository loanRepository;
    @Mock private TransactionJournalPostingService journalPostingService;

    @InjectMocks private LoanService loanService;

    @Test
    void shouldCreateLoanAndPostToJournal() {
        Loan loan = new Loan("LN123", "CUST01", BigDecimal.valueOf(10000), LoanStatus.PENDING);
        when(loanRepository.save(any())).thenReturn(loan);
        doNothing().when(journalPostingService).postLoanTransaction(anyString(), anyString(), any());

        Loan result = loanService.createLoan(new LoanRequest("CUST01", 10000, 12));

        assertEquals("LN123", result.getLoanId());
        verify(loanRepository).save(any(Loan.class));
        verify(journalPostingService).postLoanTransaction("LN123", "CUST01", BigDecimal.valueOf(10000));
    }
}
```

---

## 9. Pre-Commit Hook Setup

To ensure **tests pass before committing or pushing**, add a Git pre-commit hook.  

1. Install **pre-commit** package manager if not already installed:
   ```bash
   pip install pre-commit
   pre-commit install
   ```

2. Create a `.pre-commit-config.yaml` at the project root:

   ```yaml
   repos:
     - repo: local
       hooks:
         - id: run-maven-tests
           name: Run Unit Tests
           entry: mvn test
           language: system
           pass_filenames: false
   ```

   > For Gradle users, replace `mvn test` with `./gradlew test`.

3. Make it executable (if using `.git/hooks` manually):  
   ```bash
   chmod +x .git/hooks/pre-commit
   ```

4. Now every time you `git commit`, unit tests will run. If they fail, commit is blocked.  

---

## 10. Loan Service Testing Checklist

- [ ] Unit test **all public service methods**.
- [ ] Cover loan creation, approval, repayment, closure.  
- [ ] Cover exceptions: invalid customer, negative/zero amount, duplicate loan.  
- [ ] Controller tests for validation & mapping.  
- [ ] Repository query tests (with `@DataJpaTest` where needed).  
- [ ] Add JaCoCo plugin, enforce **70–80% coverage**.  
- [ ] Run tests in CI/CD pipeline.  
- [ ] Peer-review test cases in PRs.  

---

## 11. Next Steps

1. **Identify all Loan Application Services, Repositories and Controller methods** that are testable and map tests for each.
2. **Add JaCoCo to Maven/Gradle** for coverage reports.  
3. **Integrate tests in CI/CD** to block merges if coverage drops below threshold.  
4. **Run a coverage review session** with the team after first rollout.  
5. **Document recurring patterns** (mocking external services, loan lifecycle flows).  

---

## 12. Resources

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)  
- [Mockito Documentation](https://site.mockito.org/)  
- [Spring Boot Testing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)  
- [Pre-commit Framework](https://pre-commit.com/)  
