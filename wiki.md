### Tester — OF-5: Test Strategy & Acceptance Criteria

---

## Test Strategy — Oraclefusion Spring Boot Educational Platform

**Scope**: Planning document for the future build team. No tests are executed in this session.
**Platform**: Spring Boot 3.x + Oracle Fusion Cloud API integration + web frontend

---

## 1. Testing Frameworks & Toolchain

| Layer | Tool | Purpose |
|-|-|-|
| Unit | JUnit 5 + Mockito | Service, controller, utility class isolation |
| Spring context | Spring Boot Test | Full/slice context loading |
| Web layer | MockMvc | HTTP endpoint testing without full server |
| Database | Testcontainers (Oracle XE or H2) | Real DB integration in CI |
| API mocking | WireMock / MockRestServiceServer | Oracle Fusion REST API simulation |
| E2E | Playwright (Java) or Selenium 4 | Browser-level user flow validation |
| Performance | Gatling | Load & throughput testing on content delivery |
| Coverage | JaCoCo | Line/branch coverage reporting |
| Mutation | PIT (Pitest) | Optional — mutation score on critical modules |

---

## 2. Unit Testing Strategy

### 2.1 Service Layer
- Each `@Service` class has a corresponding `*ServiceTest` class.
- Dependencies injected via `@Mock` / `@InjectMocks` (Mockito).
- Test cases: happy path, null/empty input, edge cases, exception propagation.
- Oracle Fusion API clients are always mocked at this layer.

```
Example: CourseServiceTest
- getCourseById_returnsDto_whenFound
- getCourseById_throwsNotFoundException_whenMissing
- enrollUser_createsEnrollment_whenEligible
- enrollUser_throwsConflict_whenAlreadyEnrolled
```

### 2.2 Controller Layer
- Use `@WebMvcTest` + `MockMvc` for HTTP slice tests.
- Mock all service dependencies with `@MockBean`.
- Validate: HTTP status codes, response body JSON structure, error response format.
- Test authentication guards (401 on missing token, 403 on insufficient role).

```
Example: CourseControllerTest
- GET /api/courses — 200 with list
- GET /api/courses/{id} — 200 or 404
- POST /api/courses/enroll — 201 created / 409 conflict
- Any endpoint without auth token — 401
```

### 2.3 Repository Layer
- Use `@DataJpaTest` with Testcontainers (Oracle XE image) for real SQL validation.
- Test: custom JPQL queries, pagination, constraint violations.
- H2 acceptable for simple CRUD; Oracle XE required for Oracle-specific syntax.

---

## 3. Integration Testing

### 3.1 Spring Context Integration Tests
- `@SpringBootTest(webEnvironment = RANDOM_PORT)` for full-stack slice.
- Use `TestRestTemplate` or `WebTestClient` for HTTP calls.
- One integration test suite per major module (Auth, Courses, Lessons, Quizzes, Progress).

### 3.2 Database Integration
- Testcontainers spins up Oracle XE (or PostgreSQL as fallback) per test run.
- Flyway/Liquibase migrations run automatically — no manual schema setup.
- `@Transactional` on tests to roll back after each case.
- Test data loaded via `@Sql` scripts or test fixtures.

### 3.3 Oracle Fusion API Integration
- **No live Oracle Fusion calls in CI** — all mocked via WireMock stubs.
- WireMock server started via `@WireMockTest` or `WireMockExtension`.
- Stub files stored in `src/test/resources/wiremock/mappings/`.
- One real smoke-test suite (tagged `@Tag("live")`) excluded from CI, runnable manually.
- `MockRestServiceServer` used for `RestTemplate`-based clients as lightweight alternative.

---

## 4. E2E Testing

### 4.1 User Flows (Playwright Java)
All flows tested against a locally running Spring Boot instance with seeded test data.

| Flow | Steps | Pass Criteria |
|-|-|-|
| User Registration | Navigate → Fill form → Submit | Redirect to login, confirmation email triggered |
| Login | Enter credentials → Submit | Dashboard loads, user name shown |
| Browse Courses | Navigate catalog → Filter by module | Course cards render, Oracle Fusion badge visible |
| Start Lesson | Open course → Click lesson | Video/content loads, progress bar appears |
| Complete Lesson | Finish content → Mark done | Progress updates, next lesson unlocked |
| Take Quiz | Open quiz → Answer questions → Submit | Score displayed, pass/fail state persists |
| View Progress | Navigate to profile | Completion % accurate, badges awarded |
| Admin: Add Course | Login as admin → Create course form | Course appears in catalog |

### 4.2 Playwright Configuration
- Viewport: 1280x720 (desktop baseline).
- Tests run headless in CI, headed locally for debugging.
- Screenshots on failure saved to `test-results/screenshots/`.
- Base URL configured via `application-test.yml`.

---

## 5. API Testing (Oracle Fusion Mocking)

### 5.1 WireMock Stub Strategy
Oracle Fusion REST endpoints to stub:

| Endpoint | Method | Stub Response |
|-|-|-|
| `/fscmRestApi/resources/.../financials` | GET | Sample financial module JSON |
| `/fscmRestApi/resources/.../projects` | GET | Sample projects list |
| `/fscmRestApi/resources/.../procurement` | GET | Procurement data sample |
| `/crmRestApi/resources/.../opportunities` | GET | CRM opportunity records |
| OAuth2 token endpoint | POST | Mock JWT token |

### 5.2 Stub File Layout
```
src/test/resources/
  wiremock/
    mappings/
      oracle-financials-get.json
      oracle-projects-get.json
      oracle-crm-get.json
      oracle-oauth-token.json
    __files/
      financials-response.json
      projects-response.json
```

### 5.3 Error Scenario Coverage
- 401 Unauthorized from Oracle Fusion → app returns 502 with user-friendly message.
- 503 Service Unavailable → retry logic tested (exponential backoff stub).
- Malformed JSON response → graceful parsing failure, error logged.

---

## 6. Performance Testing (Gatling)

### 6.1 Scenarios
- **Course Catalog Load**: 100 concurrent users browsing catalog for 60s. Target: p95 < 500ms.
- **Lesson Content Delivery**: 50 concurrent users streaming lesson content. Target: p95 < 1s.
- **Quiz Submission Burst**: 200 quiz submissions in 30s. Target: 0% error rate, p99 < 2s.
- **Oracle Fusion API Proxy**: 50 concurrent API proxy requests. Target: p95 < 800ms.

### 6.2 Gatling Setup
- Simulations in `src/gatling/scala/simulations/`.
- Run via `mvn gatling:test -Pperformance`.
- Reports output to `target/gatling/`.
- Baseline established in first build; regression alert if p95 degrades >20%.

---

## 7. Acceptance Criteria Templates

For each Jira story, the build team MUST define acceptance criteria using this template:

```
Given [precondition / system state]
When  [user action or API call]
Then  [expected outcome — specific, measurable]
And   [additional assertions if needed]
```

### Per-Module Acceptance Criteria

#### Module: User Authentication & Authorization
- Given a valid email/password, when login submitted, then JWT issued and dashboard rendered within 2s.
- Given an invalid password, when login submitted, then 401 returned with "Invalid credentials" message.
- Given an expired token, when any protected API called, then 401 returned, user redirected to login.
- Given an admin role user, when accessing admin panel, then admin controls visible.
- Given a student role user, when accessing admin panel, then 403 returned.

#### Module: Course Catalog
- Given the catalog page is open, when loaded, then all published courses render within 3s.
- Given courses exist with tags, when filter applied, then only matching courses shown.
- Given a course card, when clicked, then course detail page loads with full description and module list.
- Given no courses match filter, when filter applied, then "No courses found" message shown.

#### Module: Lesson Content
- Given a student is enrolled, when lesson opened, then content renders and progress tracked.
- Given a student completes a lesson, when marked done, then next lesson unlocked within 1s.
- Given a student partially completes a lesson, when returning, then progress position restored.
- Given Oracle Fusion API is unavailable, when lesson loads API-dependent content, then fallback static content shown.

#### Module: Quiz Engine
- Given a lesson is completed, when quiz accessed, then questions load with no duplicates.
- Given all questions answered, when submitted, then score calculated correctly within 1s.
- Given score >= passing threshold (configurable, default 70%), when submitted, then "Passed" badge awarded.
- Given score < threshold, when submitted, then "Try again" prompt shown with review option.
- Given a quiz already passed, when re-attempted, then previous best score preserved.

#### Module: Progress Tracking
- Given a student completes lessons, when viewing profile, then completion percentage accurate to 1%.
- Given a student earns a badge, when viewing achievements, then badge displayed with earned date.
- Given an admin, when viewing student progress report, then all enrolled students listed with status.

#### Module: Oracle Fusion API Integration
- Given valid Oracle Fusion credentials configured, when content module loads, then live data rendered.
- Given stale cached data, when cache TTL expires, then fresh data fetched from Oracle Fusion.
- Given Oracle Fusion returns error, when content requested, then error logged and user sees friendly fallback.

---

## 8. Quality Gates

### 8.1 Code Coverage (JaCoCo)
| Scope | Threshold |
|-|-|
| Overall line coverage | >= 80% |
| Service layer line coverage | >= 90% |
| Controller layer line coverage | >= 85% |
| Branch coverage (overall) | >= 75% |

CI fails if any threshold is not met.

### 8.2 CI Pipeline Checks (all must pass to merge)
1. Unit tests — zero failures
2. Integration tests — zero failures
3. JaCoCo coverage gate — thresholds met
4. Static analysis (Checkstyle / SpotBugs) — zero critical violations
5. Dependency vulnerability scan (OWASP Dependency Check) — no CVSS >= 7.0
6. Build artifact produced (`mvn package` succeeds)

### 8.3 PR Review Requirements
- Minimum 1 reviewer approval required.
- No unresolved review comments before merge.
- All CI checks green.
- New features must include corresponding test file in the same PR.

### 8.4 Definition of Done (per story)
- [ ] Feature implemented and passing all unit tests
- [ ] Integration test written covering the happy path
- [ ] Acceptance criteria from Jira story verified manually or via E2E test
- [ ] JaCoCo thresholds still met after changes
- [ ] No new Checkstyle/SpotBugs violations introduced
- [ ] PR reviewed and approved
- [ ] Jira story transitioned to Done

---

## 9. Test Data Strategy

- **Unit tests**: Inline builders / `ObjectMother` pattern — no external dependencies.
- **Integration tests**: SQL fixture scripts (`src/test/resources/sql/`) loaded via `@Sql`.
- **E2E tests**: Seeded via a `TestDataSeeder` Spring component activated with `test` profile.
- **No production data** in any test environment — synthetic data only.
- Oracle Fusion test data: entirely WireMock-stubbed, stored as JSON fixtures.

---

## 10. Test Environment Configuration

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:tc:oracle:thin:@localhost/XEPDB1  # Testcontainers Oracle XE
  jpa:
    hibernate:
      ddl-auto: validate
oracle:
  fusion:
    base-url: http://localhost:${wiremock.port}  # WireMock stub server
    oauth-token-url: http://localhost:${wiremock.port}/oauth/token
```

---

*Document prepared by the Tester agent — planning session 2026-02-27. For execution by the build team.*
