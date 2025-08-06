# General Coding Rules and Best Practices

## 1. Honesty and Transparency
- **Do not invent non-existent methods or functions.** If a method/function/class does not exist in the language or framework, do not assume it is available without verification.
- If you don't know something or don't have the information — **state it immediately** (e.g., *"I don't know how to complete this task"*).

## 2. Constants and Repeated Values
- Extract repeated values into constants.
- Use descriptive constant names.
- Example:
  ```go
  const (
      MODE_TEST = "test"
      MODE_PROD = "prod"
  )
  ```

## 3. DRY (Don't Repeat Yourself)
- Avoid code duplication. If logic is repeated, move it into a separate function, module, or component.
- Reusing code improves reliability and simplifies maintenance.

## 4. KISS (Keep It Simple, Stupid)
- Code should be simple and readable.
- Avoid unnecessary complexity and overengineering.

## 5. YAGNI (You Aren't Gonna Need It)
- Do not implement functionality "just in case" if it is not needed right now.
- Write code only for the current requirements.

## 6. SOLID Principles
1. **S** — Single Responsibility Principle: each module/class should handle only one responsibility.
2. **O** — Open/Closed Principle: code should be open for extension but closed for modification.
3. **L** — Liskov Substitution Principle: subclasses should be replaceable by their parent classes without altering behavior.
4. **I** — Interface Segregation Principle: many small interfaces are better than one large one.
5. **D** — Dependency Inversion Principle: depend on abstractions, not concrete implementations.

## 7. Readability and Formatting
- Consistent coding style across the entire project.
- Use auto-formatters (`gofmt`, `eslint --fix`, `php-cs-fixer`, `prettier`).
- Write self-documenting code (meaningful variable, method, and class names).

## 8. Error Handling
- Handle errors explicitly.
- Do not suppress errors (`try/catch` or `if err != nil` in Go) without logging or handling them.
- Provide informative error messages.

## 9. Testing
- Write automated tests for critical logic.
- Keep tests up to date.

## 10. Architecture and Modularity
- Split code into modules/packages/components by functionality.
- Logic, UI, and data should be separated (Separation of Concerns).
- Reuse modules instead of copying code.

## 11. Documentation
- Document public APIs, complex algorithms, and architectural decisions.
- Comments should explain **why**, not **what** the code does (the "what" should be clear from the code itself).

## 12. Best Practices
- Minimize the use of global variables.
- Follow security best practices: escape input, validate data, avoid SQL injections and XSS.
- Optimize only after identifying actual bottlenecks.
- Conduct regular code reviews.
