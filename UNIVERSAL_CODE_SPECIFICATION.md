# Universal Code Specification

This document is a language-agnostic specification for writing code that is readable, testable, maintainable, and performant. It applies to all languages, frameworks, and runtime environments. It is designed to be directly usable by engineers, code reviewers, and AI coding agents.

All code produced under this specification must conform to Uncle Bob's Clean Code principles and the SOLID design principles. All examples use language-agnostic pseudocode.

This spec is a companion to the [SDLC Specification](./SDLC.md), which governs how software is built end-to-end.

---

## Table of Contents

1. [Core Rules](#1-core-rules)
2. [SOLID and Clean Code](#2-solid-and-clean-code)
3. [Architecture and Boundaries](#3-architecture-and-boundaries)
4. [Function and Module Conventions](#4-function-and-module-conventions)
5. [Error Handling](#5-error-handling)
6. [Control Flow](#6-control-flow)
7. [State and Concurrency](#7-state-and-concurrency)
8. [Performance](#8-performance)
9. [Testing](#9-testing)
10. [Naming and Readability](#10-naming-and-readability)
11. [Documentation and Comments](#11-documentation-and-comments)
12. [Events and Observability](#12-events-and-observability)
13. [Reusability and Composition](#13-reusability-and-composition)
14. [Configuration Management](#14-configuration-management)
15. [Security](#15-security)
16. [Logging](#16-logging)
17. [Retry and Resilience](#17-retry-and-resilience)
18. [Async and Resource Management](#18-async-and-resource-management)
19. [Code Organization](#19-code-organization)
20. [Versioning and Backwards Compatibility](#20-versioning-and-backwards-compatibility)
21. [AI Code Generation Contract](#21-ai-code-generation-contract)
22. [Definition of Done](#22-definition-of-done)

---

## 1. Core Rules

These rules are non-negotiable. All other sections in this specification derive from them.

1. **Clarity over cleverness.** Obvious code is preferred to compact tricks.
2. **Single responsibility per unit.** Each function, class, or module has one reason to change.
3. **Depend on abstractions, not concretions.** Business rules do not depend directly on infrastructure details.
4. **Small, composable units.** Functions are short. Modules are focused.
5. **Side effects are explicit.** Hidden mutation and implicit global state are forbidden.
6. **Testability by design.** Code is easy to test in isolation.
7. **Performance is intentional.** Start simple. Optimize measured bottlenecks only.
8. **Secure by default.** External input is never trusted. Secrets are never in code. Least privilege applies.

---

## 2. SOLID and Clean Code

### 2.1 Single Responsibility

A unit handles one cohesive concern. If a description needs "and", the unit is split.

```text
BAD:
FUNCTION process_user():
    validate_user()
    save_user()
    send_email()

GOOD:
FUNCTION validate_user()
FUNCTION save_user()
FUNCTION notify_user()

FUNCTION register_user(user):
    validate_user(user)
    save_user(user)
    notify_user(user)
```

### 2.2 Open/Closed

Behavior is extended through composition or new implementations. Stable core logic is not modified for every new variant.

```text
BAD:
FUNCTION calculate_discount(customer):
    IF customer.type == "VIP":
        RETURN amount * 0.20
    ELSE IF customer.type == "EMPLOYEE":
        RETURN amount * 0.30
    -- Must edit this function for every new type

GOOD:
INTERFACE DiscountStrategy:
    FUNCTION apply(amount) -> Number

CLASS VipDiscount IMPLEMENTS DiscountStrategy:
    FUNCTION apply(amount):
        RETURN amount * 0.20

CLASS EmployeeDiscount IMPLEMENTS DiscountStrategy:
    FUNCTION apply(amount):
        RETURN amount * 0.30

FUNCTION calculate_discount(strategy: DiscountStrategy, amount):
    RETURN strategy.apply(amount)
```

### 2.3 Liskov Substitution

Subtypes preserve parent expectations. Preconditions are never strengthened. Guarantees are never weakened.

```text
BAD:
CLASS Bird:
    FUNCTION fly()

CLASS Penguin EXTENDS Bird:
    FUNCTION fly():
        RAISE Error("Penguins can't fly")  -- Violates parent contract

GOOD:
INTERFACE Movable:
    FUNCTION move()

CLASS Sparrow IMPLEMENTS Movable:
    FUNCTION move():
        fly_through_air()

CLASS Penguin IMPLEMENTS Movable:
    FUNCTION move():
        walk_on_ground()
```

### 2.4 Interface Segregation

Interfaces are minimal. Clients do not depend on methods they do not use.

```text
BAD:
INTERFACE Worker:
    FUNCTION write_code()
    FUNCTION design_ui()
    FUNCTION manage_servers()
    -- Forces every implementor to handle all three

GOOD:
INTERFACE CodeWriter:
    FUNCTION write_code()

INTERFACE Designer:
    FUNCTION design_ui()

INTERFACE Operator:
    FUNCTION manage_servers()
```

### 2.5 Dependency Inversion

High-level policy depends on interfaces. Concrete adapters are injected at boundaries.

```text
BAD:
FUNCTION checkout():
    processor = StripeProcessor()  -- Hardcoded dependency
    processor.charge(100)

GOOD:
INTERFACE PaymentProcessor:
    FUNCTION charge(amount)

FUNCTION checkout(processor: PaymentProcessor):
    processor.charge(100)
```

### 2.6 Clean Code Constraints

- Names are meaningful with no ambiguity.
- Functions do one thing.
- Arguments are minimized (0-3 preferred; related data is grouped into structures).
- Boolean flags that alter behavior are avoided.
- Magic numbers and strings are replaced with named constants.
- Comments explain *why*, not *what*. Code expresses what.

```text
BAD:
FUNCTION process(data, true, 86400)

GOOD:
SESSION_TIMEOUT = 86400

FUNCTION process_order(order_data, SESSION_TIMEOUT)
```

---

## 3. Architecture and Boundaries

All systems use layered architecture. Dependencies point inward. Inner layers never depend on outer layers. Business logic is framework-independent.

```text
[ Interface Layer ]     -> API / CLI / UI
[ Application Layer ]   -> Use Cases / Orchestration
[ Domain Layer ]        -> Business Logic (pure)
[ Infrastructure ]      -> DB, APIs, File System, External Tools
```

- **Domain/Core.** Business rules, pure logic, no framework calls.
- **Application/Use Cases.** Orchestrates domain actions.
- **Infrastructure/Adapters.** Database, network, file system, external APIs.
- **Interface Layer.** HTTP/CLI/UI handlers.

---

## 4. Function and Module Conventions

### 4.1 Function Rules

- Maximum 20-30 lines per function.
- Maximum 3-5 parameters. Related data is grouped into structures.
- One level of abstraction per function.
- Pure functions are preferred for transformations.
- I/O is kept at edges.
- Inputs are validated early. Invalid state fails fast with explicit errors.
- Structured results are returned, not ambiguous primitives.
- Modules expose a small public API and hide internals.

### 4.2 Standard Function Template

```text
FUNCTION perform_action(input) -> Result:

    -- 1. Validate
    VALIDATE input

    -- 2. Transform
    processed = transform(input)

    -- 3. Execute
    result = execute(processed)

    -- 4. Return
    RETURN result
```

### 4.3 Early Returns

Nesting is kept shallow. Invalid state is rejected upfront with guard clauses.

```text
BAD:
FUNCTION process(order):
    IF order IS NOT NULL:
        IF order.items IS NOT EMPTY:
            IF order.is_valid:
                RETURN complete_order(order)

GOOD:
FUNCTION process(order):
    IF order IS NULL:
        RETURN Error("Missing order")

    IF order.items IS EMPTY:
        RETURN Error("No items")

    IF NOT order.is_valid:
        RETURN Error("Invalid order")

    RETURN complete_order(order)
```

### 4.4 Dependency Injection

All external collaborators are injected. This makes functions testable with fakes and stubs and isolates side effects to boundaries.

```text
INTERFACE Clock:
    FUNCTION now() -> Timestamp

INTERFACE SessionRepo:
    FUNCTION save(session) -> Void

FUNCTION process_input(session, input, clock, repo, tool_executor) -> Result:
    VALIDATE input IS NOT EMPTY

    updated = session.with_state(PROCESSING)
    updated = updated.add_user_turn(input, clock.now())

    tool_result = tool_executor.execute_for(updated)

    final = updated.add_tool_result(tool_result).with_state(IDLE)
    repo.save(final)

    RETURN Result.ok(final)
```

---

## 5. Error Handling

Errors are never swallowed silently. Low-level errors are converted into meaningful domain or application errors. Context is included without leaking secrets. Errors are explicit values or structured objects.

Standard error categories:

- `VALIDATION_ERROR`
- `NOT_FOUND`
- `CONFLICT`
- `EXTERNAL_FAILURE`
- `INTERNAL_ERROR`

Structured logs and events are emitted for all failures.

```text
BAD:
FUNCTION divide(a, b):
    TRY:
        RETURN a / b
    CATCH:
        RETURN NULL  -- Silent failure, caller has no idea what went wrong

GOOD:
FUNCTION divide(a, b) -> Result:
    IF b == 0:
        RETURN Error(
            type    = VALIDATION_ERROR,
            message = "Division by zero",
            context = { numerator = a }
        )

    RETURN Result.ok(a / b)
```

---

## 6. Control Flow

### 6.1 Simple Loops

```text
FOR EACH item IN items:
    process(item)
```

### 6.2 No Deep Nesting

```text
BAD:
IF a:
    IF b:
        IF c:
            do_thing()

GOOD:
IF NOT a: RETURN
IF NOT b: RETURN
IF NOT c: RETURN
do_thing()
```

### 6.3 Named Predicates

Complex conditionals are extracted into intention-revealing functions.

```text
BAD:
IF user.age >= 18 AND user.has_id AND NOT user.is_banned AND user.balance > 0:
    allow_purchase()

GOOD:
FUNCTION is_eligible_for_purchase(user):
    RETURN user.age >= 18
       AND user.has_id
       AND NOT user.is_banned
       AND user.balance > 0

IF is_eligible_for_purchase(user):
    allow_purchase()
```

---

## 7. State and Concurrency

- Immutable updates are preferred for critical flows.
- When state is mutated, mutation points are centralized and documented.
- Shared mutable state is synchronized or avoided.
- State transitions are centralized.
- Parallelism is allowed only when operations are independent, ordering is not required, and failures are collected and handled deterministically.

### 7.1 State Machine Example

```text
FUNCTION process_input(session, user_input):
    session.state = PROCESSING

    IF session.abort_signaled:
        RETURN

    LOOP:
        IF exceeded_limits(session):
            BREAK

        response = call_model(session)

        IF no_tool_calls(response):
            BREAK

        execute_tools(session, response)

    session.state = IDLE
```

---

## 8. Performance

1. **Baseline first.** The clear solution is implemented first.
2. **Measure before optimizing.** Real workloads are profiled.
3. **Optimize hotspots only.** No speculative micro-optimizations.
4. **Data structures are chosen intentionally** based on access patterns.
5. **Expensive operations are bounded.** Retries, loops, queue sizes, and payload sizes have limits.
6. **Readability is preserved after optimization.** Non-obvious tradeoffs are explained.

### 8.1 Patterns

```text
-- Before: repeated linear scans in loop (O(n*m))
FOR EACH item IN items:
    owner = FIND users WHERE users.id == item.user_id

-- After: build index once (O(n+m))
user_by_id = INDEX_BY(users, key = id)
FOR EACH item IN items:
    owner = user_by_id[item.user_id]
```

```text
-- Cache expensive operations
IF cache.contains(key):
    RETURN cache[key]

result = expensive_computation(key)
cache.set(key, result)
RETURN result
```

```text
BAD:
FOR EACH user IN users:
    db.save(user)         -- N database calls

GOOD:
db.save_all(users)        -- 1 database call
```

---

## 9. Testing

### 9.1 Test Pyramid

- **Unit tests (majority).** Pure logic, fast, deterministic.
- **Integration tests.** Boundaries — database, APIs, filesystem.
- **End-to-end tests (minimal).** Critical user journeys only.

### 9.2 Testable Design

- All external collaborators are injected as dependencies.
- External systems are mocked or stubbed.
- Clocks and randomness are deterministic (injected, not called directly).

### 9.3 Test Quality

- One behavior per test.
- Arrange-Act-Assert structure.
- No hidden test coupling.
- Coverage includes happy path, edge cases, failure modes, and concurrency-sensitive behavior where relevant.

### 9.4 Example

```text
TEST "should calculate total correctly":
    -- Arrange
    items = [10, 20, 30]

    -- Act
    result = calculate_total(items)

    -- Assert
    ASSERT result == 60

TEST "should return error for empty list":
    -- Arrange
    items = []

    -- Act
    result = calculate_total(items)

    -- Assert
    ASSERT result IS Error
    ASSERT result.message == "No items to total"
```

---

## 10. Naming and Readability

- Names reveal intent. `calculate_total_price`, not `doStuff`.
- Abbreviations are avoided unless domain-standard.
- Scope is kept tight. Local is preferred over global.
- Long conditionals are replaced with intention-revealing predicates.
- Files are focused. They are split when cognitive load grows.

```text
BAD:  fn, val, tmp, proc, mgr
GOOD: calculate_invoice_total, validated_order, temp_file_path
```

### 10.1 Boolean Naming

Booleans read as true/false questions.

```text
BAD:  status, check, flag
GOOD: is_valid, has_items, can_execute, should_retry
```

---

## 11. Documentation and Comments

Every public module includes:

- Purpose
- Inputs and outputs
- Invariants
- Failure modes
- Performance characteristics (if non-trivial)

Comments are allowed for rationale, tradeoffs, non-obvious algorithmic constraints, and external protocol quirks.

Comments are not allowed for restating obvious code or stale TODOs without owner or context.

---

## 12. Events and Observability

### 12.1 Key Transitions

```text
session.emit(USER_CREATED, user_id = user.id)
session.emit(PAYMENT_FAILED, order_id = order.id, reason = error.message)
session.emit(ORDER_COMPLETED, order_id = order.id)
```

### 12.2 Rules

- Events describe what happened, not how.
- Events include enough context to reconstruct the scenario.
- Events are used for debugging, monitoring, and audit trails.
- Secrets and PII are never included in event payloads.

---

## 13. Reusability and Composition

- Reusable logic is extracted into small, focused units.
- Duplication is avoided (DRY), but code is not over-abstracted for hypothetical reuse.
- Composition is preferred over inheritance.

```text
BAD (inheritance chain):
CLASS Animal
CLASS Mammal EXTENDS Animal
CLASS Dog EXTENDS Mammal
CLASS GuideDog EXTENDS Dog  -- Deep hierarchy, fragile

GOOD (composition):
FUNCTION calculate_tax(amount)
FUNCTION calculate_discount(amount)

FUNCTION calculate_total(amount):
    tax = calculate_tax(amount)
    discount = calculate_discount(amount)
    RETURN amount + tax - discount
```

```text
GOOD (composed behaviors):
CLASS GuideDog:
    walker   = WalkBehavior()
    guider   = GuideBehavior()
    feeder   = FeedBehavior()

    FUNCTION perform_duties():
        walker.walk()
        guider.guide_owner()
```

---

## 14. Configuration Management

- No hardcoded values in business logic.
- Configuration objects or environment-sourced values are used.
- Configuration is validated at startup. Missing or invalid values fail fast.
- Configuration is separated by environment (dev, staging, production).

```text
BAD:
FUNCTION retry_request():
    max_retries = 3              -- Magic number buried in logic
    timeout = 5000               -- What unit? Milliseconds? Seconds?

GOOD:
CONFIG:
    max_retries       = 3
    timeout_ms        = 5000
    base_url          = ENV("API_BASE_URL")
    feature_flag_x    = ENV("ENABLE_FEATURE_X", default = false)

FUNCTION retry_request(config):
    FOR i IN RANGE(config.max_retries):
        result = request(config.base_url, timeout = config.timeout_ms)
        IF result.ok:
            RETURN result
```

---

## 15. Security

- External input is never trusted. All data from users, APIs, files, and environment is validated and sanitized.
- Least privilege. Only the minimum permissions required are granted.
- Secrets are never in code. API keys, passwords, and tokens come from secure config or vaults.
- Queries are parameterized. User input is never interpolated into queries or commands.
- Data is validated at every trust boundary (between layers, services, and external systems).

```text
BAD:
query = "SELECT * FROM users WHERE id = " + user_input  -- Injection risk

GOOD:
query = "SELECT * FROM users WHERE id = ?"
params = [user_input]
db.execute(query, params)
```

```text
BAD:
API_KEY = "sk-12345-secret"   -- Secret in source code

GOOD:
API_KEY = ENV("API_KEY")      -- Loaded from secure config
```

---

## 16. Logging

- Structured log formats are used (key-value or JSON).
- Logs use appropriate levels: `DEBUG`, `INFO`, `WARN`, `ERROR`.
- Secrets, tokens, passwords, and PII are never logged.
- Correlation IDs are included for tracing across services.
- Logs describe what happened and why it matters, not implementation noise.

```text
BAD:
LOG("something went wrong")
LOG("user data: " + user.to_string())  -- Leaks PII

GOOD:
LOG(level = ERROR, message = "Payment failed",
    order_id = order.id,
    error_code = error.code,
    correlation_id = request.trace_id)

LOG(level = INFO, message = "Order completed",
    order_id = order.id,
    duration_ms = elapsed)
```

---

## 17. Retry and Resilience

- Transient failures use exponential backoff.
- Every external call has a timeout.
- Failing services are protected with circuit breakers.
- Operations that may be retried are idempotent.
- Retries are bounded. Indefinite retry is forbidden.

```text
FUNCTION call_with_retry(action, config) -> Result:
    attempts = 0

    LOOP:
        IF attempts >= config.max_retries:
            RETURN Error("Max retries exceeded")

        result = action(timeout = config.timeout_ms)

        IF result.ok:
            RETURN result

        IF NOT is_retriable(result.error):
            RETURN result

        wait(config.base_delay_ms * (2 ^ attempts))
        attempts += 1
```

```text
FUNCTION call_with_breaker(service, breaker) -> Result:
    IF breaker.is_open():
        RETURN Error("Circuit open, service unavailable")

    result = service.call()

    IF result.ok:
        breaker.record_success()
    ELSE:
        breaker.record_failure()

    RETURN result
```

---

## 18. Async and Resource Management

### 18.1 Async Conventions

- Every async call has a timeout.
- Cancellation is propagated.
- Fire-and-forget is forbidden. Results are always handled or awaited.
- Structured concurrency is preferred. Child tasks are scoped to parent lifetime.

```text
BAD:
FUNCTION handle_request():
    fire_and_forget(send_email())  -- No error handling, no cancellation

GOOD:
FUNCTION handle_request(cancellation_token) -> Result:
    result = AWAIT send_email(timeout = 5000, cancel = cancellation_token)
    IF result.is_error:
        LOG(level = ERROR, message = "Email failed", error = result.error)
    RETURN result
```

### 18.2 Resource Cleanup

Resources (connections, file handles, locks) are always released when done. Language-appropriate cleanup patterns are used (finally blocks, defer, using/with, destructors). Garbage collection is never relied on for resource release.

```text
BAD:
FUNCTION read_file(path):
    handle = open(path)
    data = handle.read()
    RETURN data               -- handle never closed

GOOD:
FUNCTION read_file(path):
    handle = open(path)
    TRY:
        data = handle.read()
        RETURN data
    FINALLY:
        handle.close()
```

---

## 19. Code Organization

### 19.1 File and Folder Structure

Code is grouped by feature or domain, not by technical role. Related code is co-located (handler, service, tests together). Files are split when they exceed approximately 200-300 lines.

```text
BAD (grouped by role):
/controllers/
    user_controller
    order_controller
/services/
    user_service
    order_service
/models/
    user_model
    order_model

GOOD (grouped by feature):
/users/
    user_handler
    user_service
    user_repository
    user_test
/orders/
    order_handler
    order_service
    order_repository
    order_test
/shared/
    logger
    config
    errors
```

### 19.2 Module Boundaries

Each module exposes a small public API and hides internals. Barrel/index files are used sparingly; explicit imports are preferred. Circular dependencies between modules are forbidden.

---

## 20. Versioning and Backwards Compatibility

- Existing consumers are never broken silently. Changing a public interface is a breaking change.
- Interfaces are extended with new optional fields or methods rather than altering existing ones.
- Old behavior is deprecated with a migration path before removal.
- Externally consumed interfaces are explicitly versioned.

```text
BAD:
-- v1 returns { name, email }
-- v2 renames email to contact_email with no notice
-- All consumers break

GOOD:
-- v1 returns { name, email }
-- v2 returns { name, email, contact_email }
-- v2 marks email as deprecated with migration guide
-- v3 removes email after consumers have migrated
```

---

## 21. AI Code Generation Contract

When generating code, the agent:

1. States assumptions when requirements are ambiguous.
2. Produces small, composable units with clear names.
3. Injects dependencies via interfaces and abstractions.
4. Keeps business logic framework-independent.
5. Includes or updates tests for changed behavior.
6. Does not introduce unnecessary complexity.
7. Explains any optimization beyond straightforward code.
8. Prefers consistency with existing project conventions.
9. Separates logic, state, and I/O into distinct units.
10. Emits events for key transitions and errors.
11. Uses early returns and explicit naming.
12. Avoids deep nesting, hidden side effects, and over-abstraction.

---

## 22. Definition of Done

Code is complete when:

- [ ] Functions are small and focused (max 20-30 lines).
- [ ] Each unit has single responsibility.
- [ ] Dependencies are inverted through interfaces.
- [ ] Naming is clear, consistent, and intent-revealing.
- [ ] No duplicated logic without reason.
- [ ] All edge cases are handled.
- [ ] Error handling is structured and meaningful.
- [ ] Side effects are explicit and boundary-contained.
- [ ] Tests exist and pass (happy path, edge, failure).
- [ ] Performance-sensitive paths are measured or bounded.
- [ ] No hardcoded secrets or configuration values.
- [ ] Resources are properly released.
- [ ] Code is understandable without heavy comments.
- [ ] Code is readable in under 30 seconds.
