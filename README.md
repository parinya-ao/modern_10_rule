# Standard Operating Procedure: The Modern Power of 10 for Software Engineering Reliability

## Introduction

This document sets the Standard Operating Procedure (SOP) for software development, based on and adapted from "NASA's Power of 10 Rules for Safety-Critical Code" to fit modern development environments (Python, TypeScript, Go).

**Objective:** To ensure the quality, reliability, and maintainability of software systems, especially in organizations with high staff turnover. These standards are designed to reduce technical debt and lower the cognitive load for new developers who join the team.

---

## 1. Simple Flow & Async Clarity

**Principle:** The control flow of the program should be as linear as possible. Code should be easy to read and understand in one pass, without jumping between contexts.

### Details and Guidelines

In modern systems, complexity often comes from asynchronous work, state management, and concurrency. Common problems are callback hell or complex goroutines/threads that are hard to follow.

1. **Avoid Callback Hell:** Do not use nested callbacks more than 2 levels deep. Use language features that help code look sequential.
2. **Explicit Data Flow:** Data sent between threads or goroutines must have a clear direction. Avoid sharing complex state.

### Implementation Guidelines

**TypeScript:**
Use `async/await` instead of `Promise.then().catch()` so the `try/catch` block covers all logic. Long promise chains make the call stack unclear when errors happen.

**Go:**
Avoid creating channels that send data back and forth without direction. Use patterns like pipeline or fan-out/fan-in with clear control, and use the `select` statement for blocking operations.

---

## 2. Timeouts & Rate Limits

**Principle:** The system must have deterministic latency. All blocking operations must have a clear end. Never let a process wait for resources forever.

### Details and Guidelines

Waiting for I/O (network, disk, database) without a timeout is a main cause of system hangs and resource exhaustion when the other system has problems.

1. **Mandatory Timeouts:** Always set a timeout for network requests, database queries, or waiting for locks.
2. **Circuit Breaker Pattern:** If an external system fails many times, disconnect quickly (fail fast) to prevent a build-up of waiting requests.

**Go:**
Use `context.Context` to send deadlines to all functions that do I/O.

---

## 3. Strict Resource Management

**Principle:** Resource Acquisition Is Initialization (RAII). Whoever requests a resource (memory, file, socket, database connection) is responsible for releasing it as soon as the task is done.

### Details and Guidelines

Garbage collectors only manage heap memory, not other system resources. Ignoring this leads to memory leaks and file descriptor exhaustion.

1. **Scope-based Cleanup:** Resources must be released as soon as they leave the working scope.
2. **Explicit Cleanup in UI:** In frontend frameworks, always unsubscribe event listeners or subscriptions during component teardown.

**Python:**
Always use context managers (`with` statement) for opening files or network connections. Do not call `.close()` by hand, as errors may happen before that line.

**Go:**
Use `defer` right after getting a resource to ensure cleanup, even if a panic happens.

---

## 4. Small Functions & Single Responsibility

**Principle:** Reduce complexity by strictly following the Single Responsibility Principle (SRP). Each function should have only one reason to change.

### Details and Guidelines

Large functions are hard to unit test and increase the cognitive load for new developers.

1. **Visual Limit:** Functions should not be longer than one screen (about 40-60 lines), not counting comments.
2. **Abstraction Level:** Every line in a function should be at the same level of abstraction. Do not mix high-level business logic with low-level details.

**Refactoring Strategy:**
If you need comments to explain different sections in a function, split it into smaller functions.

---

## 5. Validate at the Edge

**Principle:** Use zero trust for external data. All input is unsafe until it passes structure and type validation.

### Details and Guidelines

Scattered validation inside business logic makes code messy and hard to maintain. Validation should happen at the "edge" of the system, like API handlers or UI form submits.

1. **Strict Schema:** Use libraries to define and check schemas instead of writing if/else checks by hand.
2. **Fail Fast:** If data is in the wrong format, reject it immediately with a clear error.

**TypeScript:**
Use `Zod` or `Yup` for runtime validation to make sure TypeScript types match real data at runtime.

---

## 6. No Global Mutable State

**Principle:** Avoid shared mutable state, which causes race conditions and unpredictable side effects. This makes debugging and testing hard.

### Details and Guidelines

Global variables that anyone can change make the system highly coupled and hide real dependencies.

1. **Encapsulation:** Keep state inside the relevant object or structure only.
2. **Dependency Injection (DI):** Pass dependencies (like database config, user session) as parameters or constructors instead of using global variables.

**Go:**
Do not use `var` to declare global variables at the package level, except for constants.

---

## 7. Handle Errors Explicitly

**Principle:** Errors must not be ignored. Every error must be checked, logged, or passed on properly.

### Details and Guidelines

Empty catch blocks or ignoring error return values hides real problems, making it hard to find the cause when the system fails.

1. **No Silent Failures:** Never write code that "swallows" errors without logging.
2. **Error Wrapping:** When passing on errors, always add context (for example, change "File not found" to "Cannot load config: File not found").

**Python:**
Never use a bare `except:`. Always specify the exception class.

**Go:**
Always check `if err != nil`. Never use `_` to ignore errors.

---

## 8. Avoid "Magic" Code

**Principle:** Explicit is better than implicit. Code should be easy to read and understand without knowing complex mechanisms behind the scenes.

### Details and Guidelines

Using reflection, complex decorators, or magic methods makes it hard for IDEs to autocomplete or do static analysis, and confuses new developers.

1. **Avoid Reflection:** Do not use reflection in normal runtime paths unless really needed (like writing a library).
2. **Traceability:** Code should let you "go to definition" in the IDE and find the real working point immediately.

Write code in a straightforward (imperative) way, even if it means some repetition, for clarity and easy checking.

---

## 9. Immutability First

**Principle:** Treat data as immutable whenever possible. If you need to change data, make a copy instead of mutating the original.

### Details and Guidelines

Mutation causes bugs like "spooky action at a distance" and concurrency problems.

1. **Prefer Const/Final:** Declare variables as constants whenever possible.
2. **Pure Functions:** Try to write pure functions (output depends only on input, with no side effects).

**JavaScript / React:**
Do not use array methods that change the original (like `push`, `splice`). Use methods that return a new value (like `map`, `filter`, spread operator).

---

## 10. Linting & Static Analysis

**Principle:** Rules must be enforced by code (infrastructure as code), not just by memory or human review.

### Details and Guidelines

Humans make mistakes, and code review standards are not always consistent. Static analysis tools must act as gatekeepers to stop low-quality code from entering the repository.

1. **CI/CD Integration:** The pipeline must fail immediately if there are warnings or errors from the linter.
2. **Zero Tolerance:** No warnings are allowed to remain in the codebase.

**Tools Requirement:**
* **Python:** `Ruff` (strict mode), `MyPy` (disallow any)
* **TypeScript:** `ESLint` (strict rules), `Prettier`
* **Go:** `GolangCI-Lint` (enable `errcheck`, `gocyclo`, `gosec`)

**Workflow:**
1. **Pre-commit Hook:** Check code on the developer's machine before commit.
2. **Pull Request Check:** CI checks again. If it fails, do not allow merge.

---

**Conclusion**
Strictly following these 10 standards will help build a strong engineering foundation, reduce the risk of serious errors, and allow the team to deliver high-quality software continuously, even with staff changes.
