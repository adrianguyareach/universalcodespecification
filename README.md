# Universal Code Specification (Language-Agnostic)

**Version:** 2.0  
**Goal:** Produce code that is simple, readable, testable, maintainable, and performant.  
**Scope:** Applies to all languages, frameworks, and runtime environments.  
**Audience:** Humans and AI code generators.

---

## 1) Core Rules (Non-Negotiable)

1. **Clarity over cleverness.**  
   Prefer obvious code to compact tricks.

2. **Single responsibility per unit.**  
   Each function/class/module should have one reason to change.

3. **Depend on abstractions, not concretions.**  
   Business rules must not depend directly on infrastructure details.

4. **Small, composable units.**  
   Favor short functions and focused modules.

5. **Side effects are explicit.**  
   Hidden mutation and implicit global state are forbidden.

6. **Testability by design.**  
   Code must be easy to test in isolation.

7. **Performance is intentional.**  
   Start simple; optimize measured bottlenecks only.

8. **Secure by default.**  
   Never trust external input. Protect secrets. Apply least privilege.

---

## 2) SOLID + Clean Code Requirements

### S — Single Responsibility

A unit handles one cohesive concern. If a description needs "and", split it.

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

### O — Open/Closed

Extend behavior through composition or new implementations. Do not modify stable core logic for every new variant.

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

### L — Liskov Substitution

Subtypes must preserve parent expectations. Never strengthen preconditions or weaken guarantees.

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

### I — Interface Segregation

Expose minimal interfaces. Clients should not depend on methods they do not use.

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

### D — Dependency Inversion

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

### Clean Code Constraints

- Meaningful names, no ambiguity.
- Functions do one thing.
- Minimize arguments (prefer 0-3; group related data).
- Avoid boolean flags that alter behavior.
- Replace magic numbers/strings with named constants.
- Comment _why_, not _what_ (code should express what).

```text
BAD:
FUNCTION process(data, true, 86400)

GOOD:
SESSION_TIMEOUT = 86400

FUNCTION process_order(order_data, SESSION_TIMEOUT)
```

---

## 3) Architecture and Boundaries

Use clear layers:

```text
[ Interface Layer ]     -> API / CLI / UI
[ Application Layer ]   -> Use Cases / Orchestration
[ Domain Layer ]        -> Business Logic (pure)
[ Infrastructure ]      -> DB, APIs, File System, External Tools
```

- **Domain/Core:** business rules, pure logic, no framework calls.
- **Application/Use Cases:** orchestrates domain actions.
- **Infrastructure/Adapters:** database, network, file system, external APIs.
- **Interface Layer:** HTTP/CLI/UI handlers.

**Rule:** Dependencies point inward (outer layers depend on inner layers only). Inner layers must NOT depend on outer layers. Business logic must be framework-independent.

---

## 4) Function and Module Conventions

### Function Rules

- Max **20-30 lines** per function.
- Max **3-5 parameters** (group related data into structures).
- One level of abstraction per function.
- Prefer **pure functions** for transformations.
- Keep I/O at edges.
- Validate inputs early; fail fast with explicit errors.
- Return structured results, not ambiguous primitives.
- One module should expose a small public API and hide internals.

### Standard Function Template

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

### Early Returns (Guard Clauses)

Keep nesting shallow. Reject invalid state upfront.

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

### Dependency Injection Example

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

Why this is good:

- Explicit dependencies (`clock`, `repo`, `tool_executor`)
- Clear responsibilities
- Testable with fakes/stubs
- Side effects isolated (`repo.save`)

---

## 5) Error Handling Specification

- Do not swallow errors silently.
- Convert low-level errors into meaningful domain/application errors.
- Include context without leaking secrets.
- Errors are explicit values or structured objects.
- Use consistent error types/categories:
  - `VALIDATION_ERROR`
  - `NOT_FOUND`
  - `CONFLICT`
  - `EXTERNAL_FAILURE`
  - `INTERNAL_ERROR`
- Emit structured logs/events for failures.

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

## 6) Control Flow

### Prefer Simple Loops

```text
FOR EACH item IN items:
    process(item)
```

### Avoid Deep Nesting

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

### Replace Complex Conditionals with Named Predicates

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

## 7) State and Concurrency Rules

- Prefer immutable updates for critical flows.
- If mutating state, mutation points must be centralized and documented.
- Shared mutable state must be synchronized or avoided.
- Centralize state transitions.
- Parallelism allowed only when:
  - operations are independent
  - ordering is not required
  - failures are collected and handled deterministically

### State Machine Example

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

## 8) Performance Rules (Fast + Understandable)

1. **Baseline first:** implement clear solution.
2. **Measure before optimizing:** profile real workloads.
3. **Optimize hotspots only:** no speculative micro-optimizations.
4. **Choose data structures intentionally:** based on access patterns.
5. **Bound expensive operations:** retries, loops, queue sizes, payload sizes.
6. **Preserve readability after optimization:** explain non-obvious tradeoffs.

### Patterns

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
-- Batch operations instead of one-at-a-time
BAD:
FOR EACH user IN users:
    db.save(user)         -- N database calls

GOOD:
db.save_all(users)        -- 1 database call
```

---

## 9) Testing Standard

### Test Pyramid

- **Unit tests (majority):** pure logic, fast, deterministic.
- **Integration tests:** boundaries (DB, APIs, filesystem).
- **End-to-end tests (minimal):** critical user journeys.

### Testable Design Requirements

- Dependency injection for all external collaborators.
- Mock/stub external systems.
- Deterministic clocks and randomness (inject time and randomness).

### Test Quality Rules

- One behavior per test.
- Arrange-Act-Assert structure.
- No hidden test coupling.
- Cover:
  - happy path
  - edge cases
  - failure modes
  - concurrency-sensitive behavior where relevant

### Test Example

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

## 10) Naming and Readability

- Names must reveal intent (`calculate_total_price`, not `doStuff`).
- Avoid abbreviations unless domain-standard.
- Keep scope tight; prefer local over global.
- Replace long conditionals with intention-revealing predicates.
- Keep files focused; split when cognitive load grows.

```text
BAD:  fn, val, tmp, proc, mgr
GOOD: calculate_invoice_total, validated_order, temp_file_path
```

### Boolean Naming

Booleans must read as true/false questions:

```text
BAD:  status, check, flag
GOOD: is_valid, has_items, can_execute, should_retry
```

---

## 11) Documentation and Comments

Every public module should include:

- purpose
- inputs/outputs
- invariants
- failure modes
- performance characteristics (if non-trivial)

Comments allowed for:

- rationale/tradeoffs
- non-obvious algorithmic constraints
- external protocol quirks

Comments not allowed for:

- restating obvious code
- stale TODOs without owner/context

---

## 12) Events and Observability

### Emit Events for Key Transitions

```text
session.emit(USER_CREATED, user_id = user.id)
session.emit(PAYMENT_FAILED, order_id = order.id, reason = error.message)
session.emit(ORDER_COMPLETED, order_id = order.id)
```

### Rules

- Events describe **what happened**, not how.
- Include enough context to reconstruct the scenario.
- Useful for debugging, monitoring, and audit trails.
- Never include secrets or PII in event payloads.

---

## 13) Reusability and Composition

### Rules

- Extract reusable logic into small, focused units.
- Avoid duplication (DRY), but do not over-abstract for hypothetical reuse.
- **Prefer composition over inheritance.**

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

## 14) Configuration Management

### Rules

- No hardcoded values in business logic.
- Use configuration objects or environment-sourced values.
- Validate configuration at startup, fail fast on missing/invalid values.
- Separate config by environment (dev, staging, production).

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

## 15) Security Basics

### Rules

- **Never trust external input.** Validate and sanitize all data from users, APIs, files, and environment.
- **Least privilege.** Grant only the minimum permissions required.
- **No secrets in code.** API keys, passwords, tokens must come from secure config or vaults.
- **Parameterize queries.** Never interpolate user input into queries or commands.
- **Boundary trust zones.** Validate data at every trust boundary (e.g., between layers, services).

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

## 16) Logging Standards

### Rules

- Use structured log formats (key-value or JSON).
- Log at appropriate levels: `DEBUG`, `INFO`, `WARN`, `ERROR`.
- **Never log secrets, tokens, passwords, or PII.**
- Include correlation IDs for tracing across services.
- Log _what happened_ and _why it matters_, not implementation noise.

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

## 17) Retry and Resilience Patterns

### Rules

- **Retry with backoff.** Use exponential backoff for transient failures.
- **Set timeouts.** Every external call must have a timeout.
- **Circuit breakers.** Stop calling a failing service after repeated failures.
- **Idempotency.** Operations that may be retried must produce the same result when repeated.
- **Bound retries.** Never retry indefinitely.

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
-- Circuit breaker pattern
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

## 18) Async and Resource Management

### Async/Await Conventions

- Every async call must have a **timeout**.
- Support **cancellation** propagation.
- Avoid fire-and-forget; always handle or await results.
- Prefer **structured concurrency** (child tasks are scoped to parent lifetime).

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

### Resource Cleanup

- Always release resources (connections, file handles, locks) when done.
- Use language-appropriate cleanup patterns (finally blocks, defer, using/with, destructors).
- Never rely on garbage collection for resource release.

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

## 19) Code Organization Conventions

### File and Folder Structure

- Group by **feature or domain**, not by technical role.
- Co-locate related code (handler + service + tests together).
- Keep file size manageable; split when a file exceeds ~200-300 lines.

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

### Module Boundaries

- Each module exposes a **small public API** and hides internals.
- Use barrel/index files sparingly; prefer explicit imports.
- Circular dependencies between modules are forbidden.

---

## 20) Versioning and Backwards Compatibility

### Rules

- **Never break existing consumers silently.** Changing a public interface is a breaking change.
- **Add, don't modify.** Extend interfaces with new optional fields/methods rather than altering existing ones.
- **Deprecate before removing.** Mark old behavior as deprecated with a migration path before removal.
- **Version your APIs.** Use explicit versioning for any interface consumed externally.

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

## 21) AI Code Generation Contract

When generating code, the model **must**:

1. **State assumptions** when requirements are ambiguous.
2. **Produce small, composable units** with clear names.
3. **Inject dependencies** via interfaces/abstractions.
4. **Keep business logic** framework-independent.
5. **Include or update tests** for changed behavior.
6. **Avoid introducing unnecessary complexity.**
7. **Explain any optimization** beyond straightforward code.
8. **Prefer consistency** with existing project conventions.
9. **Separate logic, state, and I/O** into distinct units.
10. **Emit events** for key transitions and errors.
11. **Use early returns** and explicit naming.
12. **Avoid deep nesting**, hidden side effects, and over-abstraction.

---

## 22) Definition of Done

Code is complete when:

- [ ] Functions are small and focused (max 20-30 lines).
- [ ] Unit has single responsibility.
- [ ] Dependencies inverted through interfaces.
- [ ] Naming is clear, consistent, and intent-revealing.
- [ ] No duplicated logic without reason.
- [ ] All edge cases handled.
- [ ] Error handling is structured and meaningful.
- [ ] Side effects are explicit and boundary-contained.
- [ ] Tests exist and pass (happy path, edge, failure).
- [ ] Performance-sensitive paths are measured or bounded.
- [ ] No hardcoded secrets or configuration values.
- [ ] Resources are properly released.
- [ ] Code is understandable without heavy comments.
- [ ] Code is readable in under 30 seconds.

---

## Final Philosophy

> "Code should read like a story, not a puzzle."

Good code:

- **Explains itself** through structure and naming.
- **Fails loudly** with clear, actionable errors.
- **Scales naturally** through composition and clean boundaries.
- **Resists decay** because each piece is small, tested, and independent.
