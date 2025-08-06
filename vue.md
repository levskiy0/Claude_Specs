# Vue.js Style Guide

## Component Naming
- Multi-word names (except root `App`).
- PascalCase in JS, kebab-case in templates.

## Structure
- One component per `.vue` file.
- Filename: consistent PascalCase or kebab-case.

## Data and Props
- `data` must be a function returning an object.
- Props: define type, required, and validators.

## Lists and Conditionals
- Always use `key` with `v-for`.
- Avoid `v-if` and `v-for` on same element.

## DRY Principle
- Use mixins/composables for shared logic.
- Create base components for reusable UI.

## Styling
- Consistent attribute casing.
- Order component options logically.

## Tools
- ESLint with Vue plugin.
- Prefer Vue features over direct DOM manipulation.
