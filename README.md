
# 💼 # Loan Team Unit Test and Code Chnage Structure

This service handles all loan-related functionality such as loan creation, disbursement, approvals, overdraft (e.g. AlwaysOn), and audit trail integration.
This is an extention of [Policy Document](https://github.com/ndrymesSoft/Policy-Documents) which speaks to our branching strategy.

---

## 📁 Repository Structure

```bash
src/
  main/
    java/
      com/project/loan/       # Core logic
    resources/
      application.yml         # Configurations
  test/
    java/
      com/project/loan/       # Unit and integration tests
```

---

## 🔀 Git Branching Strategy (GitFlow + Client Policy)

### 🟢 Core Branches

| Branch        | Purpose                            |
|---------------|-------------------------------------|
| `main`        | Production-ready code               |
| `develop`     | Internal integration branch         |
| `release/x.y.z` | General release prep              |
| `hotfix/x.y.z`  | Urgent production bug fix         |

### 🔵 Client-Specific Branches

| Branch                              | Purpose                              |
|-------------------------------------|--------------------------------------|
| `client/<client-name>/develop`      | Client-specific development base     |
| `release/<client-name>/x.y.z`       | Client-specific UAT/production release |
| `hotfix/<client-name>/x.y.z`        | Emergency fix for client environment |
| `feature/client/<client>/<feature>` | Client-specific feature branch       |

---

## ⚙️ Code Change & Testing Structure

### ✅ Unit Tests

- Write unit tests for every service, controller, utility, or core business logic added.
- All test classes must:
  - Use `@SpringBootTest` or `@WebMvcTest` (as needed)
  - Be placed under `/src/test/java`
  - Follow the same package structure as the code under test

### 📌 Example

```java
// Main class
src/main/java/com/project/loan/service/LoanService.java

// Test class
src/test/java/com/project/loan/service/LoanServiceTest.java
```

> Run all tests using:
```bash
mvn clean test
```

> Mocks should use **Mockito** unless integration testing is required.

---

## 🔁 Merge and PR Instructions

1. **Create a feature branch**:
   ```bash
   git checkout -b feature/loan-disbursement develop
   ```

2. **Client-specific feature**:
   ```bash
   git checkout -b feature/client/gtbank/overdraft-enhancement client/gtbank/develop
   ```

3. **Push and create a PR**:
   - All PRs must go to their base branch (`develop`, `client/.../develop`, etc.)
   - Title must include:
     - JIRA ID (if applicable)
     - Short description

4. **Get at least one review**
   - No self-approvals
   - Reviewers must verify tests pass

5. **Merge via PR** only
   - No direct commits to `develop`, `main`, `client/*`, or any release/hotfix branch
   - Rebase if necessary before merge to resolve conflicts

---

## 🚀 Deployment & CI/CD Policy

### General Release

```bash
git checkout -b release/v3.0.0 develop
```

- QA signs off
- Tag the release:
  ```bash
  git tag -a v3.0.0 -m "Release v3.0.0"
  ```
- Merge into:
  - `main`
  - `develop`
- Delete release branch

### Client Release

```bash
git checkout -b release/gtbank/v2.5.0 client/gtbank/develop
```

- Test with client-specific scenarios
- Tag and deploy:
  ```bash
  git tag -a gtbank-v2.5.0 -m "Client release for GTBank"
  ```

### Cherry-Pick Strategy (if needed)

```bash
git checkout -b patch-gtbank-v2.5.1 main
git cherry-pick <commit-hash>
```

---

## 🧑‍💻 Developer Responsibilities

| Task                  | Required ✅ |
|-----------------------|------------|
| Write/update unit tests | ✅        |
| Create clean feature branch | ✅   |
| Use PR (no direct commits) | ✅     |
| Keep client code isolated | ✅     |
| Ensure local test pass | ✅        |
| Raise flags for broken dev env | ✅ |

---

## 👥 Maintainers

- **Engineering Lead**: `Olusegun Adeoye`
- **Backend Lead**: `Oluwole Sumonu`
- **Backend Sub Lead**: `Joseph Abetang`
- **Team Lead**: `Oluwaseun Odeyemi`
- **Contributors**: Loan Engineering Team `Aminu Kabunu, Uchechukwu Obiakor`
