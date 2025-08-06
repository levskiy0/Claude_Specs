# PHP Style Guide

## PSR Standards
- Follow PSR-1 and PSR-12.
- UTF-8 without BOM.
- Omit closing `?>` in pure PHP files.

## Naming
- Classes: PascalCase.
- Constants: UPPER_SNAKE_CASE.
- Methods/properties: camelCase.

## Formatting
- 4 spaces indentation.
- Braces: class/method brace on new line; control structure brace on same line.

## Strict Types
- `declare(strict_types=1);` at file top.

## DRY
- Centralize config values/constants.
- Avoid duplication.

## PHPDoc
- Document public APIs; avoid redundant comments.

## Code Reuse
- Use built-in functions and Composer packages.

## Structure
- One class per file; PSR-4 autoloading.
