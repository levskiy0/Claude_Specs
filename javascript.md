# JavaScript Style Guide

This style guide adapts core programming principles to modern JavaScript (ES2020+), based on Airbnb, Google style guides, and MDN best practices.

## Core Principles

### 1. Store Repeating Values as Constants

Use const for all values that don't change, with descriptive UPPER_CASE names for true constants.

```javascript
// Good: Named constants
const API_BASE_URL = 'https://api.example.com';
const MAX_RETRY_ATTEMPTS = 3;
const HTTP_STATUS = {
  OK: 200,
  NOT_FOUND: 404,
  SERVER_ERROR: 500
};

// Good: Configuration object
const CONFIG = {
  api: {
    baseUrl: process.env.API_URL || 'https://api.example.com',
    timeout: 5000,
    retryAttempts: 3
  },
  features: {
    enableAnalytics: true,
    debugMode: false
  }
};

// Bad: Magic numbers and strings
if (response.status === 200) { // Don't do this
  setTimeout(retry, 5000); // Magic number
}
```

### 2. Reuse Code - Don't Duplicate

Use functions, modules, and higher-order functions to eliminate duplication.

```javascript
// Good: Reusable utility functions
const createValidator = (rules) => (value) => {
  return rules.every(rule => rule(value));
};

const isEmail = createValidator([
  value => value.includes('@'),
  value => value.length > 5,
  value => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)
]);

// Good: Higher-order functions
const withRetry = (fn, retries = 3) => {
  return async (...args) => {
    let lastError;
    for (let i = 0; i < retries; i++) {
      try {
        return await fn(...args);
      } catch (error) {
        lastError = error;
        if (i === retries - 1) throw lastError;
        await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, i)));
      }
    }
  };
};

// Bad: Duplicated logic
function validateEmail(email) {
  if (!email.includes('@')) return false;
  if (email.length < 5) return false;
  // Same validation logic repeated elsewhere
}
```

### 3. Don't Write Monolithic Code

Use ES6 modules and maintain single responsibility principle.

```javascript
// Good: Modular approach
// userService.js
export class UserService {
  constructor(apiClient, logger) {
    this.apiClient = apiClient;
    this.logger = logger;
  }
  
  async createUser(userData) {
    this.logger.info('Creating user', { email: userData.email });
    return this.apiClient.post('/users', userData);
  }
}

// apiClient.js
export class ApiClient {
  constructor(baseURL) {
    this.baseURL = baseURL;
  }
  
  async post(endpoint, data) {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    return response.json();
  }
}

// Bad: Everything in one file/function
```

### 4. Don't Make Things Up

Always verify JavaScript features with MDN documentation and check browser compatibility.

```javascript
// Good: Acknowledge limitations
function processData(data) {
  // TODO: Research proper implementation for WeakRef
  // I don't know how to properly handle memory management here
  console.warn('Feature not implemented: need to research WeakRef usage');
}

// Good: Check MDN for accurate information
// MDN: https://developer.mozilla.org/en-US/docs/Web/JavaScript
```

### 5. Don't Invent Non-existent Methods

Verify method existence and provide fallbacks when necessary.

```javascript
// Good: Feature detection
if ('IntersectionObserver' in window) {
  const observer = new IntersectionObserver(callback);
} else {
  console.warn('IntersectionObserver not supported');
  // Provide fallback
}

// Good: Check method existence
if (typeof Array.prototype.flatMap === 'function') {
  const result = array.flatMap(x => [x, x * 2]);
} else {
  // Polyfill or alternative implementation
  const result = array.reduce((acc, x) => acc.concat([x, x * 2]), []);
}

// Good: Use optional chaining for safety
const value = obj?.method?.();
```

## JavaScript-Specific Best Practices

### Variable Declarations

```javascript
// Good: Use const by default, let when reassignment needed
const API_KEY = process.env.API_KEY;
const users = [];

let currentIndex = 0;
let isLoading = false;

// Never use var
var oldStyle = 'avoid'; // Don't do this
```

### Functions

```javascript
// Good: Arrow functions for simple operations
const double = x => x * 2;
const sum = (a, b) => a + b;
const getUser = async (id) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

// Good: Regular functions for complex logic
function calculateCompoundInterest(principal, rate, time, n = 12) {
  return principal * Math.pow((1 + rate / n), n * time);
}

// Good: Default parameters
function greet(name = 'Guest', greeting = 'Hello') {
  return `${greeting}, ${name}!`;
}
```

### Object and Array Operations

```javascript
// Good: Destructuring
const { name, email, ...rest } = user;
const [first, second, ...others] = items;

// Good: Spread syntax
const newUser = { ...defaultUser, ...updates };
const combined = [...array1, ...array2];

// Good: Object shorthand
const name = 'John';
const age = 30;
const user = { name, age }; // Instead of { name: name, age: age }

// Good: Array methods
const doubled = numbers.map(n => n * 2);
const filtered = users.filter(user => user.active);
const total = prices.reduce((sum, price) => sum + price, 0);
```

### Async Patterns

```javascript
// Good: Async/await with proper error handling
async function fetchUserData(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error;
  }
}

// Good: Parallel operations
async function fetchAllData() {
  const [users, posts, comments] = await Promise.all([
    fetchUsers(),
    fetchPosts(),
    fetchComments()
  ]);
  
  return { users, posts, comments };
}

// Good: Error boundary with Promise.allSettled
async function fetchMultipleResources(urls) {
  const results = await Promise.allSettled(
    urls.map(url => fetch(url).then(r => r.json()))
  );
  
  return results
    .filter(result => result.status === 'fulfilled')
    .map(result => result.value);
}
```

### Error Handling

```javascript
// Good: Custom error classes
class ValidationError extends Error {
  constructor(field, message) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

class ApiError extends Error {
  constructor(status, message) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
  }
}

// Good: Comprehensive error handling
async function processUserData(userData) {
  try {
    validateUserData(userData);
    const user = await createUser(userData);
    await sendWelcomeEmail(user);
    return user;
  } catch (error) {
    if (error instanceof ValidationError) {
      console.error(`Validation failed for ${error.field}:`, error.message);
      throw error;
    } else if (error instanceof ApiError) {
      console.error(`API error ${error.status}:`, error.message);
      throw error;
    } else {
      console.error('Unexpected error:', error);
      throw new Error('Failed to process user data');
    }
  }
}
```

### Module Organization

```javascript
// Good: Named exports for utilities
// utils/validation.js
export const isEmail = (email) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
export const isPhone = (phone) => /^\d{10}$/.test(phone);

// Good: Default export for main functionality
// services/UserService.js
export default class UserService {
  // Implementation
}

// Good: Barrel exports for clean imports
// utils/index.js
export * from './validation';
export * from './formatting';
export * from './dates';
```

### Type Safety with JSDoc

```javascript
/**
 * Calculate the total price including tax
 * @param {Array<{price: number, quantity: number}>} items - Array of items
 * @param {number} [taxRate=0.1] - Tax rate as decimal
 * @returns {number} Total price including tax
 * @throws {TypeError} If items is not an array
 */
function calculateTotal(items, taxRate = 0.1) {
  if (!Array.isArray(items)) {
    throw new TypeError('Items must be an array');
  }
  
  const subtotal = items.reduce((sum, item) => {
    return sum + (item.price * item.quantity);
  }, 0);
  
  return subtotal * (1 + taxRate);
}
```

## Tools and Verification

- **MDN Web Docs**: Authoritative JavaScript reference
- **Can I Use**: Browser compatibility checking
- **ESLint**: Code quality and error prevention
- **Prettier**: Code formatting
- **TypeScript**: Optional static typing
- **Chrome DevTools**: Debugging and profiling

## References

- [MDN JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)
- [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)
- [Google JavaScript Style Guide](https://google.github.io/styleguide/jsguide.html)
- [You Don't Know JS](https://github.com/getify/You-Dont-Know-JS)