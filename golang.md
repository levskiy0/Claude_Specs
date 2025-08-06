# Golang Style Guide

## Formatting and Indentation
- Use Go’s automatic formatter `gofmt` (or `go fmt`).
- Tabs for indentation, default `gofmt` style.
- No strict line length, wrap long lines logically.

## Naming Conventions
- camelCase for variables/functions.
- PascalCase for exported identifiers.
- Package names: lowercase, no underscores or dashes.

## Constants and Magic Numbers
- Avoid magic values; use `const` with descriptive names.
- Use `iota` for enumerations.

## DRY Principle
- Avoid duplicating code; extract helpers/functions.

## Small, Focused Functions
- Functions should do one thing well.

## Idiomatic Patterns
- Use Go idioms (short variable names in small scope, error handling early).

## Code Reuse
- Use standard library when possible.

## Avoid Inventing Patterns
- Stick to language-supported features.

## Tools
- Use `go vet`, `golangci-lint`, and tests.
