# Laravel Style Guide

## Conventions
- Use Laravel's documented practices.
- Models: singular nouns.
- Controllers: singular + `Controller` suffix.

## Database
- Table names: plural snake_case.
- Pivot tables: alphabetical singular names joined with underscore.

## Relationships
- Singular for single-result, plural for multi-result.

## Structure
- Use default Laravel directories.
- Skinny controllers, move logic to models/services.

## Controllers
- Keep actions small.
- Use dependency injection and route model binding.

## Framework Features
- Use built-in auth, validation, Eloquent.
- Avoid reinventing features.

## Migrations
- Always implement `down()` method.

## Updates
- Stay on current stable Laravel.

## DRY
- Reuse helpers, Blade directives, config constants.

## Tools
- Use Laravel Pint for style.
