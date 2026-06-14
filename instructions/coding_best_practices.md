# Coding Best Practices for Autonomous Agents

This document outlines the strict, language-agnostic coding standards required for all generated source code. The primary optimization target is **local reasoning**: code must be structurally obvious, explicitly typed, and aggressively declarative to ensure future agents and humans can safely refactor it without breaking hidden dependencies.

## 1. Core Architecture

### Functional Core, Imperative Shell (FCIS)
Enforce a strict boundary between business logic and environmental side effects.
- **Functional Core**: All business rules and domain logic must be implemented as pure functions. They must take inputs, return outputs, and possess zero side effects (no database calls, no network requests, no state mutations).
- **Imperative Shell**: All I/O (database, network, file system, UI) must reside in the outer shell. The shell gathers data, passes it to the core, and applies the returned results. 
- **Rule**: The shell calls the core; the core *never* calls the shell.

**Bad / Un-idiomatic:**
```javascript
// BAD: Business logic mixed with I/O side effects
async function calculateDiscountAndSave(userId, cartTotal) {
  const user = await db.users.find(userId);
  const discount = user.isVip ? cartTotal * 0.2 : 0;
  await db.orders.save({ userId, total: cartTotal - discount });
}
```

**Good / Idiomatic:**
```javascript
// GOOD: Pure core logic separated from shell I/O
function calculateDiscount(isVip, cartTotal) {
  return isVip ? cartTotal * 0.2 : 0;
}

// Shell handles I/O
async function checkout(userId, cartTotal) {
  const user = await db.users.find(userId);
  const discount = calculateDiscount(user.isVip, cartTotal);
  await db.orders.save({ userId, total: cartTotal - discount });
}
```

### Simplicity and YAGNI
Implement *only* the requested functionality. 
- **Rule**: Never build abstractions, interfaces, or generic structures to anticipate unstated future requirements. Over-engineering destroys context.

### Single Responsibility Principle (SRP)
Keep functions and modules small and single-purposed.
- **Rule**: If a function name requires "And" (e.g., `validateAndSave`), it violates SRP and must be split.

## 2. Control Flow & Legibility

### Flatten Conditional Logic (Guard Clauses)
Deep nesting ("Arrow Anti-Pattern") forces massive cognitive load.
- **Rule**: Always check for invalid conditions and return/throw immediately at the top of the function. The "happy path" must remain un-nested and aligned to the left margin.

**Bad / Un-idiomatic:**
```python
# BAD: Deep nesting
def process_order(order):
    if order is not None:
        if order.is_paid:
            if order.in_stock:
                ship_item(order)
```

**Good / Idiomatic:**
```python
# GOOD: Guard clauses flatten logic
def process_order(order):
    if not order:
        return
    if not order.is_paid:
        return
    if not order.in_stock:
        return
        
    ship_item(order)
```

### Explicit Naming and Comments
Clarity completely supersedes brevity.
- **Rule**: Variables must be long and descriptive (`active_user_count`, not `n`).
- **Rule**: Code must dictate *what* is happening. Comments must be reserved exclusively to explain *why* (business rationale, historical context, math constants). Never comment obvious mechanics.

## 3. Data & State Management

### Parse, Don't Validate
State management must be explicit to avoid unpredictable behavior.
- **Rule**: Parse raw inputs into strongly typed domain objects at the absolute boundary of the system. Once instantiated, their internal invariants guarantee validity, eliminating defensive checks deeper in the code.

**Bad / Un-idiomatic:**
```typescript
// BAD: Continual validation of raw primitives
function processEmail(email: string) {
  if (!email.includes('@')) throw new Error("Invalid");
  // ...do work
}
```

**Good / Idiomatic:**
```typescript
// GOOD: Parse into a validated domain object at the boundary
class Email {
  constructor(public readonly value: string) {
    if (!value.includes('@')) throw new Error("Invalid");
  }
}

// Core functions safely trust the parsed type
function processEmail(email: Email) {
  // ...do work safely
}
```

### Explicit Configuration
- **Rule**: Never hardcode configuration parameters, API keys, or environment secrets. Inject them dynamically from environment variables or secure secret managers.

## 4. Redundancy & Duplication

### The "Wrong Abstraction" Penalty
Duplication is far cheaper than the wrong abstraction. A brittle shared abstraction requires boolean flags and nested `if` statements to handle diverging business rules.
- **Rule of Three**: You must wait for at least three distinct instances of concrete behavioral duplication before extracting a shared abstraction.
- **Rule**: If an abstraction requires complex branching to support all consumers, inline the abstraction, duplicate the code, and let implementations evolve independently. Mark with a `// DUPE` comment if necessary.

## 5. Error Handling & Observability

### Fail Fast vs Recover
- **Rule**: For unrecoverable state errors or missing critical configuration, fail immediately and loudly (crash/throw).
- **Rule**: For transient errors (network timeouts, locked records), return explicit error types or Result objects that force the caller to handle the failure programmatically.

### Contextual Logging
- **Rule**: Catch exceptions at the highest logical boundary. When logging errors, include structured contextual metadata (User ID, request parameters) to ensure observability. Do not log naked stack traces without domain context.

## 6. Anti-Patterns to Avoid

The following behaviors actively degrade the codebase and are strictly forbidden:

1. **Ghost Architecture**: Implicit, emergent behaviors that bypass defined structure (e.g., unbounded database queries without pagination, accreted primary keys).
2. **Poltergeists**: Useless pass-through classes (often suffixed with `_Manager` or `_Coordinator`) that contain no business logic and simply forward data to another class.
3. **Lava Flow**: Leaving dead, deprecated, or unused code in the repository out of fear. Practice aggressive deletion.
4. **Metaprogramming**: Writing code that dynamically generates other code or properties at runtime using reflection. It destroys static analysis and blinds autonomous agents. Always favor explicit declarations.
