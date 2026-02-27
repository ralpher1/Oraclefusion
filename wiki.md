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

---

### Tester — OF-25: Oracle Fusion Module Acceptance Criteria & Interactive Labs Testing

**Parent story**: OF-22 — Interactive Labs and API Demonstrations
**Scope**: Acceptance criteria per Oracle Fusion cloud module, plus testing strategy for interactive labs and live API demo flows.

---

## 11. Oracle Fusion Module Acceptance Criteria (OF-25)

Each module section below covers: unit test scope, integration mock stubs, E2E lab flows, and acceptance criteria.

---

### 11.1 HCM — Human Capital Management

**What the platform teaches**: Core HR, Workforce Management, Talent Management, Payroll concepts via Oracle HCM Cloud REST APIs.

**Unit Test Scope**
- `HcmContentService`: fetches, caches, and maps HCM API responses to lesson DTOs.
- `HcmLabService`: generates lab exercises (e.g., create employee, run payroll simulation).
- Controllers: `GET /api/labs/hcm/{exerciseId}` — 200 with lab spec, 404 on missing exercise.

**WireMock Stubs Required**
| Stub | Endpoint | Response |
|-|-|-|
| Employee list | `GET /hcmRestApi/resources/latest/workers` | Array of worker objects |
| Single employee | `GET /hcmRestApi/resources/latest/workers/{id}` | Worker detail object |
| Payroll run | `POST /hcmRestApi/resources/latest/payrollRuns` | Payroll run confirmation |
| Talent profile | `GET /hcmRestApi/resources/latest/talentProfiles` | Profile list |

**Acceptance Criteria**
- Given a student opens the HCM lab, when the exercise loads, then a sandboxed employee dataset renders within 3s.
- Given a student submits a "create employee" lab step, when validated, then the mock HCM API call is logged and success feedback shown.
- Given the HCM API stub returns 403, when lab step executes, then "Insufficient permissions — check your API role setup" guidance displayed.
- Given a student completes all HCM lab steps, when marked done, then HCM module badge awarded and progress updated.
- Given a quiz on Core HR concepts, when submitted with >= 70% score, then "HCM Fundamentals" certificate unlocked.

**Performance Target**: HCM lab exercise load (with mock data) p95 < 600ms under 50 concurrent lab sessions.

---

### 11.2 ERP — Enterprise Resource Planning

**What the platform teaches**: General Ledger, Accounts Payable/Receivable, Fixed Assets, Cash Management via Oracle Financials Cloud.

**Unit Test Scope**
- `ErpContentService`: maps Financials Cloud API responses to tutorial content.
- `ErpLabService`: generates journal entry, invoice, and ledger exercise scenarios.
- Controllers: `GET /api/labs/erp/{exerciseId}`, `POST /api/labs/erp/{exerciseId}/submit`.

**WireMock Stubs Required**
| Stub | Endpoint | Response |
|-|-|-|
| Ledger list | `GET /fscmRestApi/resources/latest/ledgers` | Ledger array |
| Journal entries | `GET /fscmRestApi/resources/latest/journalEntries` | Journal entry list |
| Create invoice | `POST /fscmRestApi/resources/latest/invoices` | Invoice confirmation |
| AP transactions | `GET /fscmRestApi/resources/latest/payablesTransactions` | Transaction list |

**Acceptance Criteria**
- Given a student opens the ERP General Ledger lab, when loaded, then a chart of accounts renders with sample data within 3s.
- Given a student creates a journal entry in the lab, when submitted, then debit/credit balance validation runs client-side and mock API call confirmed.
- Given an unbalanced journal entry submitted, when validated, then "Debits must equal credits" error shown before API call.
- Given a student completes the Accounts Payable exercise, when marked done, then next ERP lesson (AR) unlocks.
- Given a quiz on GL fundamentals, when passed, then ERP pathway progress advances to next module.
- Given Oracle Financials stub returns 500, when lab step executes, then student sees "API temporarily unavailable — results saved locally" message.

**Performance Target**: ERP lab with ledger data render p95 < 700ms under 40 concurrent sessions.

---

### 11.3 SCM — Supply Chain Management

**What the platform teaches**: Procurement, Inventory, Order Management, Manufacturing via Oracle SCM Cloud.

**Unit Test Scope**
- `ScmContentService`: fetches procurement and inventory data for tutorial content.
- `ScmLabService`: generates purchase order, shipment, and inventory exercises.
- Controllers: `GET /api/labs/scm/{exerciseId}`, `POST /api/labs/scm/{exerciseId}/submit`.

**WireMock Stubs Required**
| Stub | Endpoint | Response |
|-|-|-|
| Purchase orders | `GET /fscmRestApi/resources/latest/purchaseOrders` | PO list |
| Create PO | `POST /fscmRestApi/resources/latest/purchaseOrders` | PO confirmation |
| Inventory items | `GET /fscmRestApi/resources/latest/inventoryItems` | Item catalog |
| Shipments | `GET /fscmRestApi/resources/latest/shipments` | Shipment list |

**Acceptance Criteria**
- Given a student opens the SCM Procurement lab, when loaded, then a supplier catalog with sample items renders within 3s.
- Given a student creates a purchase order in the lab, when submitted, then PO number generated and confirmation screen shown.
- Given a student runs the inventory check exercise, when completed, then stock levels update in the lab UI to reflect the mock transaction.
- Given a student completes all SCM lab exercises, when marked done, then SCM pathway certificate awarded.
- Given a quiz on procurement workflows, when passed with >= 70%, then Order Management module unlocked.
- Given concurrent PO creation attempts (10 users), when executed simultaneously, then no data collision errors in lab state.

**Performance Target**: SCM lab PO creation round-trip p95 < 800ms under 30 concurrent sessions.

---

### 11.4 CX — Customer Experience (CRM)

**What the platform teaches**: Sales, Service, Marketing, Loyalty via Oracle CX Cloud (formerly Oracle CRM).

**Unit Test Scope**
- `CxContentService`: maps CRM API responses to tutorial and quiz content.
- `CxLabService`: generates opportunity, case, and campaign exercises.
- Controllers: `GET /api/labs/cx/{exerciseId}`, `POST /api/labs/cx/{exerciseId}/submit`.

**WireMock Stubs Required**
| Stub | Endpoint | Response |
|-|-|-|
| Opportunities | `GET /crmRestApi/resources/latest/opportunities` | Opportunity list |
| Create opportunity | `POST /crmRestApi/resources/latest/opportunities` | Opportunity record |
| Service requests | `GET /crmRestApi/resources/latest/serviceRequests` | SR list |
| Contacts | `GET /crmRestApi/resources/latest/contacts` | Contact list |
| Campaigns | `GET /crmRestApi/resources/latest/campaigns` | Campaign list |

**Acceptance Criteria**
- Given a student opens the CX Sales lab, when loaded, then a pipeline of sample opportunities renders within 3s.
- Given a student creates a new sales opportunity, when submitted, then opportunity appears in the pipeline with correct stage.
- Given a student closes an opportunity as "Won", when confirmed, then revenue contribution shown in the lab dashboard.
- Given a student opens the Service lab, when a case is created, then SLA timer starts and priority level enforced.
- Given a quiz on Oracle CX sales process, when passed, then CX module badge awarded.
- Given the CRM stub is slow (>2s response — simulated via WireMock delay), when lab loads, then loading spinner shown and no timeout error until 10s threshold.

**Performance Target**: CX opportunity pipeline render p95 < 500ms under 60 concurrent sessions.

---

### 11.5 Analytics — Oracle Analytics Cloud (OAC)

**What the platform teaches**: Data visualization, dashboards, report building, and AI/ML insights via Oracle Analytics Cloud.

**Unit Test Scope**
- `AnalyticsContentService`: fetches OAC dataset metadata and embeds for tutorial rendering.
- `AnalyticsLabService`: generates dashboard-building and report exercises.
- Controllers: `GET /api/labs/analytics/{exerciseId}`, `POST /api/labs/analytics/{exerciseId}/submit`.

**WireMock Stubs Required**
| Stub | Endpoint | Response |
|-|-|-|
| Dataset list | `GET /api/20210901/datasets` | Dataset array |
| Dataset detail | `GET /api/20210901/datasets/{id}` | Dataset schema |
| Workbook list | `GET /api/20210901/workbooks` | Workbook list |
| Create workbook | `POST /api/20210901/workbooks` | Workbook confirmation |
| Embedded report | `GET /api/20210901/visualizations/{id}/embed` | Embed URL/token |

**Acceptance Criteria**
- Given a student opens the Analytics lab, when loaded, then sample datasets render in a data browser within 3s.
- Given a student drags a dimension to the canvas, when dropped, then a chart automatically generates with sample data.
- Given a student saves a workbook, when confirmed, then workbook appears in "My Work" panel within 1s.
- Given a student runs a pre-built report, when executed, then data table renders with correct row count matching the stub dataset.
- Given a quiz on OAC visualization types, when passed with >= 70%, then Analytics certificate unlocked.
- Given an embedded OAC report iframe, when loaded, then no X-Frame-Options CSP errors appear in browser console.
- Given 80 concurrent students viewing dashboards, when load test runs, then p95 < 1.2s and 0% error rate.

**Performance Target**: Analytics lab dashboard render p95 < 1.2s under 80 concurrent sessions (highest load — visualization-heavy).

---

## 12. Interactive Labs Testing Strategy (OF-25)

### 12.1 Lab Execution Model
Each interactive lab consists of:
1. **Exercise spec** — instructions rendered from DB, fetched via `GET /api/labs/{module}/{exerciseId}`
2. **Step runner** — student submits each step, validated server-side against expected mock API call
3. **Result feedback** — immediate pass/fail per step with guidance
4. **Lab completion** — all steps passed → progress event emitted → badge/certificate logic triggered

### 12.2 Lab-Specific Integration Tests
```
HcmLabIntegrationTest
  - loadExercise_returnsSpec_whenFound
  - submitStep_callsMockHcmApi_andReturnsSuccess
  - submitStep_handlesApiError_gracefully
  - completeAllSteps_triggersBadgeEvent

ErpLabIntegrationTest  [same pattern per module]
ScmLabIntegrationTest
CxLabIntegrationTest
AnalyticsLabIntegrationTest
```

### 12.3 E2E Lab Flows (Playwright)
| Flow | Steps | Pass Criteria |
|-|-|-|
| HCM Lab: Create Employee | Login → Open HCM lab → Fill form → Submit | Success message, step marked done |
| ERP Lab: Journal Entry | Open ERP lab → Enter debit/credit → Submit | Balance validated, entry confirmed |
| SCM Lab: Purchase Order | Open SCM lab → Select supplier → Create PO | PO number generated, confirmation shown |
| CX Lab: Sales Opportunity | Open CX lab → Create opportunity → Advance stage | Stage updated in pipeline |
| Analytics Lab: Build Dashboard | Open Analytics lab → Drag dimension → Save workbook | Chart renders, workbook saved |

### 12.4 API Demo Flow Testing
For "live API demonstration" pages (read-only, no student interaction):
- Verify mock stub returns data within SLA.
- Verify rendered data matches stub fixture (no transformation errors).
- Verify error state renders correctly when stub returns 5xx.
- No authentication to Oracle Fusion production — all demo calls hit WireMock in test and staging.

---

*OF-25 section prepared by Tester agent — 2026-02-27. Parent: OF-22 Interactive Labs and API Demonstrations.*

---

### Builder — OF-3: Architecture Outline and Technology Stack

**Parent story**: OF-2 — Project Setup and Spring Boot Architecture

---

## 13. Technology Stack

| Layer | Technology | Version | Purpose |
|-|-|-|-|
| Runtime | Java | 17+ (LTS) | Application language |
| Framework | Spring Boot | 3.2+ | Web application framework |
| Web | Spring MVC + Thymeleaf | - | Server-side rendering with REST API support |
| Security | Spring Security | 6.x | Authentication, authorization, CSRF, session mgmt |
| Persistence | Spring Data JPA + Hibernate | - | ORM and repository abstraction |
| Database | PostgreSQL | 15+ | Primary relational data store |
| Caching | Redis + Spring Cache | - | Session store, API response caching |
| Migration | Flyway | 9+ | Database schema versioning |
| HTTP Client | Spring WebClient (WebFlux) | - | Non-blocking Oracle Fusion API calls |
| Build | Maven | 3.9+ | Build and dependency management |
| Docs | SpringDoc OpenAPI | 2.x | Swagger UI for REST API documentation |
| Monitoring | Spring Boot Actuator + Micrometer | - | Health checks, metrics, Prometheus export |

---

## 14. Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Presentation Layer                │
│  Thymeleaf templates, REST controllers, error views │
├─────────────────────────────────────────────────────┤
│                    Service Layer                    │
│  Business logic, content orchestration, quiz engine │
├─────────────────────────────────────────────────────┤
│                  Integration Layer                  │
│  Oracle Fusion API clients, caching, retry logic    │
├─────────────────────────────────────────────────────┤
│                  Persistence Layer                  │
│  JPA repositories, entities, Flyway migrations      │
├─────────────────────────────────────────────────────┤
│                  Infrastructure                     │
│  PostgreSQL, Redis, Oracle Fusion Cloud (external)  │
└─────────────────────────────────────────────────────┘
```

### 14.1 Package Structure

```
com.oraclefusion.edu/
  config/           — Spring configuration (Security, WebClient, Cache, CORS)
  controller/
    web/            — Thymeleaf page controllers (dashboard, courses, labs)
    api/            — REST API controllers (progress, quizzes, admin)
  service/
    course/         — CourseService, LessonService, EnrollmentService
    quiz/           — QuizService, QuestionService, ScoreService
    lab/             — LabService, LabStepRunner, LabResultService
    progress/       — ProgressService, BadgeService, CertificateService
    user/           — UserService, RoleService
  integration/
    oracle/
      hcm/          — HcmApiClient, HcmDataMapper
      erp/          — ErpApiClient, ErpDataMapper
      scm/          — ScmApiClient, ScmDataMapper
      cx/           — CxApiClient, CxDataMapper
      analytics/    — AnalyticsApiClient, AnalyticsDataMapper
      auth/         — OracleOAuth2TokenProvider
    cache/          — CacheConfig, CacheKeyGenerator
  repository/       — JPA repositories (CourseRepo, UserRepo, ProgressRepo, etc.)
  entity/           — JPA entities (Course, Lesson, Quiz, User, Progress, Badge)
  dto/              — Data transfer objects for API and view layer
  exception/        — Custom exceptions, global error handler
  util/             — Helpers (date, formatting, validation)
```

### 14.2 Module Organization

Each Oracle Fusion module (HCM, ERP, SCM, CX, Analytics) is organized as a self-contained package under `integration/oracle/` with:
- **ApiClient**: WebClient-based HTTP client with retry, timeout, error mapping
- **DataMapper**: Transforms Oracle Fusion API JSON to internal DTOs
- **Configuration**: Module-specific properties (base URL, credentials, cache TTL)

Each module's educational content follows the same structure:
- Courses contain Lessons
- Lessons contain Content Blocks (text, video, interactive) and end with a Quiz
- Labs are standalone exercises tied to a module, consisting of ordered Steps

---

## 15. Database Schema (Key Entities)

```sql
-- Users and Auth
users (id, email, password_hash, display_name, role, created_at, last_login)
user_roles (user_id, role)  -- STUDENT, INSTRUCTOR, ADMIN

-- Course Content
courses (id, module, title, description, difficulty, estimated_hours, published)
lessons (id, course_id, sequence, title, content_type, content_body, duration_min)
content_blocks (id, lesson_id, sequence, block_type, body, media_url)

-- Quizzes
quizzes (id, lesson_id, passing_score, time_limit_min)
questions (id, quiz_id, sequence, text, question_type, correct_answer, options_json)
quiz_attempts (id, quiz_id, user_id, score, passed, started_at, completed_at)

-- Labs
labs (id, module, title, description, difficulty, prerequisite_lesson_id)
lab_steps (id, lab_id, sequence, instruction, expected_api_call, validation_rule)
lab_attempts (id, lab_id, user_id, status, started_at, completed_at)
lab_step_results (id, lab_attempt_id, lab_step_id, passed, response_data, submitted_at)

-- Progress
enrollments (id, user_id, course_id, enrolled_at, completed_at, status)
lesson_progress (id, user_id, lesson_id, status, position, last_accessed)
badges (id, name, description, icon_url, criteria_type, criteria_value)
user_badges (id, user_id, badge_id, earned_at)
certificates (id, user_id, module, issued_at, certificate_url)
```

---

### Builder — OF-15: Content Delivery Architecture

**Parent story**: OF-14 — Educational Content Delivery System

---

## 16. Content Delivery Architecture

### 16.1 Tutorial Engine
- **Rendering**: Thymeleaf templates with fragment composition. Each lesson renders as a sequence of content blocks (text, code, video, interactive widget).
- **Content Storage**: Lesson body stored in DB as structured markdown. Rendered server-side via a MarkdownProcessor service.
- **Media**: Video content embedded via iframe (YouTube/Vimeo or self-hosted). Static assets served from `/static/media/` or CDN.
- **Navigation**: Prev/Next lesson navigation with progress bar. Breadcrumb: Module > Course > Lesson.

### 16.2 Quiz Engine
- **Question Types**: Multiple choice, true/false, fill-in-the-blank, code completion.
- **Scoring**: Server-side scoring only — no client-side answer validation to prevent cheating.
- **Time Limits**: Optional per-quiz timer enforced server-side. Timer starts on first question load.
- **Retry Policy**: Configurable max attempts per quiz (default: 3). Best score preserved.
- **Results**: Immediate score display. Detailed review showing correct/incorrect per question available after submission.

### 16.3 Lab Environment
- **Sandbox Model**: Each lab session creates an isolated context (in-memory state + mock API session).
- **Step Execution**: Sequential steps. Each step submission validated against expected Oracle Fusion API call pattern.
- **Feedback Loop**: Step result returned immediately. Hints available after first failure. Full solution revealed after 3 failed attempts.
- **State Persistence**: Lab attempt state persisted to DB. Students can resume incomplete labs.

### 16.4 Learning Dashboard
- **Student View**: Course progress bars, upcoming lessons, recent quiz scores, earned badges, module completion percentage.
- **Instructor View**: Student roster, aggregate completion rates, quiz score distributions, struggling students report.
- **Admin View**: Platform usage metrics, content management, user management, system health.

---

### Builder — OF-19: Authentication and Authorization Architecture

**Parent story**: OF-18 — User Authentication and Progress Tracking

---

## 17. Authentication Architecture

### 17.1 Spring Security Configuration
- **Authentication**: Form-based login with Spring Security's `formLogin()`. BCrypt password hashing.
- **Session Management**: Server-side sessions stored in Redis for horizontal scalability. `SessionCreationPolicy.IF_REQUIRED`.
- **Remember Me**: Token-based remember-me with persistent token repository (DB-backed).
- **CSRF**: Enabled for all form submissions. Thymeleaf auto-injects CSRF tokens.
- **CORS**: Configured for API endpoints if a separate frontend SPA is added later.

### 17.2 Role-Based Access Control

| Role | Access |
|-|-|
| STUDENT | Browse courses, take lessons, submit quizzes, run labs, view own progress |
| INSTRUCTOR | All student access + view student progress reports + manage course content |
| ADMIN | All instructor access + user management + system configuration + analytics |

### 17.3 Security Filter Chain
```
HTTP Request
  → CsrfFilter
  → SessionManagementFilter
  → UsernamePasswordAuthenticationFilter (login)
  → RememberMeAuthenticationFilter
  → AuthorizationFilter (role-based URL patterns)
  → Controller
```

### 17.4 URL Authorization Rules
```
/                           — permitAll (landing page)
/login, /register           — permitAll
/courses/**, /lessons/**    — hasRole(STUDENT)
/labs/**                    — hasRole(STUDENT)
/api/progress/**            — hasRole(STUDENT)
/instructor/**              — hasRole(INSTRUCTOR)
/admin/**                   — hasRole(ADMIN)
/api/admin/**               — hasRole(ADMIN)
/actuator/**                — hasRole(ADMIN) or IP whitelist
```

### 17.5 User Registration Flow
1. Student fills registration form (email, password, display name)
2. Server validates: email uniqueness, password strength (min 8 chars, mixed case, digit)
3. BCrypt hash stored. Default role: STUDENT
4. Welcome email sent (async via Spring Events + email service)
5. Redirect to login page with success message

### 17.6 Progress Tracking Data Model
- **Enrollment**: Created when student starts a course. Tracks overall course status.
- **Lesson Progress**: Per-lesson record with status (NOT_STARTED, IN_PROGRESS, COMPLETED), position bookmark, last access timestamp.
- **Quiz Attempts**: Immutable records of each quiz attempt with score and answers.
- **Lab Attempts**: Records of lab sessions with per-step results.
- **Badges & Certificates**: Awarded automatically via event listeners when criteria met (e.g., course completion, quiz score threshold).

---

*Architecture planning content prepared by General Builder — Project Planning Architect. Session 2026-02-27.*

---

### Builder-General — OF-6 OF-10: Oracle Fusion Module Curriculum Outline

**Parent stories**: OF-6 (HCM Module), OF-10 (ERP Module)
**Scope**: Detailed course curriculum structure for all five Oracle Fusion product lines.

---

## 18. Course Curriculum Outline

### Course 1: Oracle HCM Cloud Fundamentals

**Target Audience**: HR Admins, Payroll Specialists new to Oracle HCM
**Difficulty**: Beginner
**Estimated Duration**: 8 hours

| Module | Title | Lessons | Key Topics |
|-|-|-|-|
| 1 | Introduction to Oracle HCM | 3 | Cloud overview, navigation, roles |
| 2 | Core HR | 5 | Worker model, positions, org structures |
| 3 | Talent Management | 4 | Goals, performance, succession |
| 4 | Absence Management | 3 | Absence types, accruals, self-service |
| 5 | HCM REST API Basics | 4 | Auth, worker API, HDL intro |

**Capstone Lab**: Configure a worker hierarchy, run a talent review, and query the HCM REST API for org data.

### Course 2: Oracle ERP Cloud Essentials

**Target Audience**: Finance Analysts, Accountants learning Oracle Financials
**Difficulty**: Beginner-Intermediate
**Estimated Duration**: 10 hours

| Module | Title | Lessons | Key Topics |
|-|-|-|-|
| 1 | Financials Overview | 2 | Cloud ERP landscape, key concepts |
| 2 | General Ledger | 5 | COA, journals, period close, reports |
| 3 | Accounts Payable | 4 | Suppliers, invoices, payments |
| 4 | Accounts Receivable | 4 | Customers, invoices, receipts |
| 5 | ERP REST & FBDI | 5 | FBDI templates, REST endpoints, automation |

**Capstone Lab**: Create a journal entry via REST API, generate an FBDI template, and import AP invoices.

### Course 3: Oracle SCM Cloud Practitioner

**Target Audience**: Supply Chain Analysts, Procurement Specialists
**Difficulty**: Intermediate
**Estimated Duration**: 10 hours

| Module | Title | Lessons | Key Topics |
|-|-|-|-|
| 1 | SCM Overview | 2 | Supply chain concepts, Fusion SCM modules |
| 2 | Inventory Management | 5 | Items, organizations, transactions |
| 3 | Procurement | 4 | PO lifecycle, requisitions, sourcing |
| 4 | Order Management | 4 | Sales orders, shipping, returns |
| 5 | SCM APIs | 4 | Item REST, PO REST, integration patterns |

**Capstone Lab**: Create a purchase order, receive inventory, and fulfill a sales order using REST APIs.

### Course 4: Oracle CX Cloud Sales & Service

**Target Audience**: Sales Reps, Service Agents, CRM Admins
**Difficulty**: Beginner-Intermediate
**Estimated Duration**: 8 hours

| Module | Title | Lessons | Key Topics |
|-|-|-|-|
| 1 | CX Platform Overview | 2 | Sales Cloud vs Service Cloud, navigation |
| 2 | Sales Cloud | 5 | Accounts, contacts, opportunities, forecasting |
| 3 | Service Cloud | 4 | Service requests, queues, SLAs |
| 4 | CX REST APIs | 4 | Opportunity, account, SR endpoints |

**Capstone Lab**: Manage a full sales cycle from account creation to closed-won, then escalate to service.

### Course 5: Oracle Analytics Cloud (OAC)

**Target Audience**: BI Analysts, Report Developers
**Difficulty**: Intermediate-Advanced
**Estimated Duration**: 12 hours

| Module | Title | Lessons | Key Topics |
|-|-|-|-|
| 1 | OAC Fundamentals | 3 | Datasets, data flows, projects |
| 2 | Visualization & Dashboards | 5 | Canvas, charts, KPIs, filters |
| 3 | Data Modeling | 5 | Subject areas, logical model, RPD |
| 4 | OAC APIs & Embedding | 4 | REST API, embedding SDK, export |
| 5 | AI & Machine Learning | 3 | Explain, auto insights, ML models |

**Capstone Lab**: Build a multi-page dashboard, embed it via the OAC SDK, and export via REST API.

---

## 19. Oracle Fusion API Integration Strategy (OF-22 / OF-24)

### 19.1 Authentication Flow

```
Spring Boot App
  └── FusionApiService
      └── WebClient (reactive, non-blocking)
          └── OAuth2AuthorizedClientManager
              └── ClientCredentials grant
                  └── Oracle IDCS / OCI IAM token endpoint
```

Environment configuration (never hardcoded, loaded via application.yml from env vars):

```yaml
oracle.fusion:
  base-url: ${FUSION_BASE_URL}
  client-id: ${FUSION_CLIENT_ID}
  client-secret: ${FUSION_CLIENT_SECRET}
  token-url: ${FUSION_TOKEN_URL}
  scope: ${FUSION_SCOPE}
```

### 19.2 Per-Module API Endpoints

| Module | Key Endpoint | HTTP Method | Demo Purpose |
|-|-|-|-|
| HCM | `/hcmRestApi/resources/11.13.18.05/workers` | GET | Live org chart in HCM lessons |
| ERP | `/fscmRestApi/resources/11.13.18.05/journalEntries` | POST | Journal entry creation demo |
| SCM | `/fscmRestApi/resources/11.13.18.05/inventoryItems` | GET | Live item catalog in SCM lessons |
| CX | `/crmRestApi/resources/11.13.18.05/opportunities` | GET | Pipeline visualization in CX lessons |
| OAC | `/api/20210901/embeddedContent` (OAC SDK) | GET | Embedded dashboard iframe |

### 19.3 API Demo Framework Design (OF-24)

**Live Demo Pages**: Read-only pages in each course module showing real-time Oracle Fusion data. No student input — purely observational with annotations explaining the API response structure.

**Mock Service Layer**: `FusionMockService` — activated via `@Profile("mock")` — returns fixture JSON from `src/main/resources/fusion-fixtures/{module}/`. Allows full platform functionality without live credentials.

**Fixture File Layout**:
```
src/main/resources/fusion-fixtures/
  hcm/
    workers-list.json          -- sample worker collection
    worker-detail.json         -- single worker with full attributes
  erp/
    journal-entries.json       -- sample journal entry collection
    ledgers.json               -- chart of accounts sample
  scm/
    inventory-items.json       -- item catalog sample
    purchase-orders.json       -- PO list sample
  cx/
    opportunities.json         -- pipeline data sample
    service-requests.json      -- SR queue sample
  oac/
    datasets.json              -- dataset metadata sample
    workbooks.json             -- workbook list sample
```

**Circuit Breaker**: If live Fusion API fails (network, auth, 5xx), the service auto-falls-back to fixture data with a visible "Demo Mode — Live API Unavailable" banner rendered in the UI.

### 19.4 API Rate Limiting & Caching

- Oracle Fusion REST APIs have per-tenant rate limits — cache all GET responses in Redis.
- Default cache TTL: 5 minutes for list endpoints, 15 minutes for detail endpoints.
- Cache keys: `fusion:{module}:{endpoint}:{params-hash}`
- Cache invalidation: manual admin endpoint `/api/admin/cache/flush/{module}`

---

*Module curriculum and API integration planning prepared by Builder-General agent — 2026-02-27.*
