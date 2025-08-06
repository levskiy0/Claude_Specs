# Universal Code Best Practices

This guide provides language-agnostic best practices based on the 5 core principles that apply across all programming languages.

## Core Principles

### 1. Store Repeating Values as Constants

**Why**: Centralizing values makes code maintainable, reduces errors, and enables easy configuration changes.

**Universal Guidelines**:
- Define constants at the highest appropriate scope
- Use descriptive, self-documenting names
- Group related constants together
- Separate configuration from code
- Use environment variables for deployment-specific values

**Naming Conventions**:
- Use UPPER_SNAKE_CASE for true constants
- Use descriptive prefixes for grouped constants
- Be consistent within your codebase

**Anti-patterns to Avoid**:
- Magic numbers and strings
- Hardcoded values scattered throughout code
- Duplicate constant definitions
- Using constants for values that change

### 2. Reuse Code - Don't Duplicate

**Why**: DRY (Don't Repeat Yourself) principle reduces maintenance burden and bugs.

**Universal Strategies**:
- Extract common functionality into functions/methods
- Use composition over inheritance
- Create utility/helper modules
- Implement design patterns appropriately
- Build reusable components/libraries

**Code Reuse Hierarchy**:
1. Functions/Methods - Smallest unit of reuse
2. Classes/Modules - Encapsulated functionality
3. Libraries/Packages - Shared across projects
4. Services/APIs - Shared across systems

**Anti-patterns to Avoid**:
- Copy-paste programming
- Near-duplicate functions with slight variations
- Reimplementing standard library functionality
- Over-engineering for unlikely reuse scenarios

### 3. Don't Write Monolithic Code

**Why**: Modular code is easier to understand, test, maintain, and scale.

**Universal Guidelines**:
- Follow Single Responsibility Principle (SRP)
- Keep functions/methods small and focused
- Separate concerns into different modules
- Use clear interfaces between components
- Maintain loose coupling, high cohesion

**Modularization Strategies**:
- **Horizontal Layering**: Separate by technical concerns (MVC, etc.)
- **Vertical Slicing**: Separate by features/domains
- **Hexagonal Architecture**: Core logic independent of external systems
- **Microservices**: Separate by bounded contexts (when appropriate)

**Anti-patterns to Avoid**:
- God classes/objects
- Functions doing multiple unrelated things
- Deep nesting and complex conditionals
- Circular dependencies
- Premature optimization leading to complexity

### 4. Don't Make Things Up

**Why**: Using established solutions prevents bugs and leverages community knowledge.

**Universal Guidelines**:
- Research before implementing
- Use official documentation
- Follow established patterns and practices
- Ask for help when stuck
- Document your learning process

**When You Don't Know**:
// Good: Acknowledge gaps in knowledge

// TODO: Research the best approach for [specific problem]

// I don't know how to implement this efficiently yet

// Bad: Guessing or implementing half-understood concepts

**Research Resources**:
- Official language/framework documentation
- Stack Overflow (verify answers)
- GitHub - study well-maintained projects
- Technical books and courses
- Community forums and chat groups

### 5. Don't Invent Non-existent Methods

**Why**: Using non-existent APIs causes runtime errors and confusion.

**Universal Guidelines**:
- Verify method existence in documentation
- Use IDE autocomplete and type checking
- Test code before assuming it works
- Check version compatibility
- Provide fallbacks when necessary

**Verification Methods**:
- Read official API documentation
- Use language-specific inspection tools
- Write tests to verify behavior
- Use static analysis tools
- Enable strict mode/type checking

## Universal Best Practices

### Code Organization

**File Structure**:
```
project/
├── src/           # Source code
├── tests/         # Test files
├── docs/          # Documentation
├── config/        # Configuration files
├── scripts/       # Build/deployment scripts
└── README.md      # Project overview
```

**Naming Conventions**:
- Be consistent within your project
- Use meaningful, descriptive names
- Avoid abbreviations and acronyms
- Follow language-specific conventions
- Consider future developers (including yourself)

### Error Handling

**Universal Principles**:
1. **Fail Fast**: Detect errors early
2. **Be Specific**: Provide detailed error messages
3. **Handle Gracefully**: Don't crash unnecessarily
4. **Log Appropriately**: Record errors for debugging
5. **Secure Information**: Don't expose sensitive data

**Error Handling Hierarchy**:
1. Prevent errors through validation
2. Catch and handle expected errors
3. Log unexpected errors with context
4. Provide user-friendly error messages
5. Have a fallback for catastrophic failures

## Summary

I've created comprehensive style guides for each requested programming language and framework, plus a universal best practices guide. Each guide:

1. **Adapts your 5 core principles** to the specific language/framework
2. **Includes practical code examples** showing correct and incorrect approaches
3. **References official documentation** and established best practices
4. **Provides verification methods** for checking API/method existence
5. **Offers language-specific patterns** for code organization and reusability

The guides are based on extensive research from official documentation, community standards, and recognized experts in each ecosystem. They emphasize practical, real-world applications while maintaining consistency with your core principles.