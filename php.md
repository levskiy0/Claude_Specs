# PHP Style Guide

This style guide adapts core programming principles to modern PHP (8.0+), based on PSR standards and PHP best practices.

## Core Principles

### 1. Store Repeating Values as Constants

Use class constants, global constants, or configuration files appropriately.

```php
<?php
declare(strict_types=1);

// Good: Class constants for class-specific values
class OrderStatus
{
    public const PENDING = 'pending';
    public const PROCESSING = 'processing';
    public const COMPLETED = 'completed';
    public const CANCELLED = 'cancelled';
}

// Good: Global constants for application-wide values
define('APP_VERSION', '2.1.0');
define('MAX_UPLOAD_SIZE', 10 * 1024 * 1024); // 10MB

// Good: Configuration file approach
// config/app.php
return [
    'name' => $_ENV['APP_NAME'] ?? 'My Application',
    'environment' => $_ENV['APP_ENV'] ?? 'production',
    'debug' => $_ENV['APP_DEBUG'] ?? false,
];

// Bad: Magic values scattered in code
if ($order->status === 'pending') { // Don't do this
    // ...
}
```

### 2. Reuse Code - Don't Duplicate

Use traits, inheritance, and composition to eliminate duplication.

```php
<?php
// Good: Trait for common functionality
trait Timestampable
{
    protected DateTime $createdAt;
    protected DateTime $updatedAt;
    
    public function touch(): void
    {
        $this->updatedAt = new DateTime();
    }
    
    public function getCreatedAt(): DateTime
    {
        return $this->createdAt;
    }
}

// Good: Abstract class for shared behavior
abstract class Repository
{
    public function __construct(
        protected PDO $db
    ) {}
    
    abstract protected function getTableName(): string;
    
    public function findById(int $id): ?array
    {
        $stmt = $this->db->prepare(
            "SELECT * FROM {$this->getTableName()} WHERE id = :id"
        );
        $stmt->execute(['id' => $id]);
        
        return $stmt->fetch(PDO::FETCH_ASSOC) ?: null;
    }
}

// Good: Composition over inheritance
class UserService
{
    public function __construct(
        private UserRepository $repository,
        private EmailService $emailService,
        private Logger $logger
    ) {}
}
```

### 3. Don't Write Monolithic Code

Follow SOLID principles and maintain proper separation of concerns.

```php
<?php
// Good: Single Responsibility Principle
class UserValidator
{
    public function validate(array $data): array
    {
        $errors = [];
        
        if (empty($data['email'])) {
            $errors['email'] = 'Email is required';
        } elseif (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            $errors['email'] = 'Invalid email format';
        }
        
        return $errors;
    }
}

class UserRepository
{
    public function __construct(
        private PDO $db
    ) {}
    
    public function create(array $userData): User
    {
        $stmt = $this->db->prepare(
            'INSERT INTO users (email, name) VALUES (:email, :name)'
        );
        $stmt->execute($userData);
        
        return new User($this->db->lastInsertId(), $userData);
    }
}

// Bad: God class doing everything
class UserManager
{
    public function validateAndCreateUserAndSendEmail($data) {
        // 100+ lines doing validation, database operations, and email sending
    }
}
```

### 4. Don't Make Things Up

Always verify with PHP documentation and acknowledge limitations.

```php
<?php
// Good: Check documentation at php.net
// Use php --rf function_name to check function details

// Good: Acknowledge when unsure
class DataProcessor
{
    public function processComplexData($data): void
    {
        // TODO: Research PHP Fibers for async processing
        // I don't know how to implement this efficiently yet
        throw new \RuntimeException('Not implemented: need to research async patterns');
    }
}
```

### 5. Don't Invent Non-existent Methods

Verify function/method existence before use.

```php
<?php
// Good: Check if function exists
if (function_exists('imagecreate')) {
    $image = imagecreate(100, 100);
} else {
    throw new RuntimeException('GD extension not installed');
}

// Good: Check if method exists
if (method_exists($object, 'customMethod')) {
    $object->customMethod();
} else {
    throw new BadMethodCallException("Method 'customMethod' does not exist");
}

// Good: Use is_callable for more complete check
if (is_callable([$object, 'methodName'])) {
    call_user_func([$object, 'methodName']);
}
```

## PHP-Specific Best Practices

### PSR Standards

```php
<?php
// PSR-12: Extended Coding Style
declare(strict_types=1);

namespace App\Services;

use App\Models\User;
use App\Repositories\UserRepository;
use Psr\Log\LoggerInterface;

final class UserService
{
    public function __construct(
        private UserRepository $userRepository,
        private LoggerInterface $logger
    ) {
    }
    
    public function createUser(array $data): User
    {
        $this->logger->info('Creating new user', ['email' => $data['email']]);
        
        return $this->userRepository->create($data);
    }
}
```

### Type Declarations

```php
<?php
declare(strict_types=1);

// Good: Use type declarations everywhere
class Calculator
{
    public function add(int $a, int $b): int
    {
        return $a + $b;
    }
    
    // PHP 8.0 union types
    public function process(int|float $value): string
    {
        return (string) $value;
    }
    
    // PHP 8.0 named arguments
    public function createUser(
        string $email,
        string $name,
        ?string $phone = null,
        bool $active = true
    ): User {
        return new User(
            email: $email,
            name: $name,
            phone: $phone,
            active: $active
        );
    }
}
```

### Error Handling

```php
<?php
// Good: Custom exception classes
class ValidationException extends \Exception
{
    public function __construct(
        private array $errors,
        string $message = 'Validation failed'
    ) {
        parent::__construct($message);
    }
    
    public function getErrors(): array
    {
        return $this->errors;
    }
}

// Good: Proper error handling
class FileProcessor
{
    public function processFile(string $filename): string
    {
        if (!file_exists($filename)) {
            throw new \InvalidArgumentException("File not found: {$filename}");
        }
        
        $content = file_get_contents($filename);
        if ($content === false) {
            throw new \RuntimeException("Failed to read file: {$filename}");
        }
        
        return $content;
    }
}

// Good: Try-catch with specific handling
try {
    $processor = new FileProcessor();
    $result = $processor->processFile('data.txt');
} catch (\InvalidArgumentException $e) {
    // Handle missing file
    error_log("File error: " . $e->getMessage());
} catch (\RuntimeException $e) {
    // Handle read error
    error_log("Runtime error: " . $e->getMessage());
} catch (\Throwable $e) {
    // Handle any other error
    error_log("Unexpected error: " . $e->getMessage());
}
```

### Security Best Practices

```php
<?php
// Good: Prepared statements for database queries
class UserRepository
{
    public function __construct(
        private PDO $pdo
    ) {
        $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    }
    
    public function findByEmail(string $email): ?User
    {
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE email = :email');
        $stmt->execute(['email' => $email]);
        
        $userData = $stmt->fetch(PDO::FETCH_ASSOC);
        return $userData ? User::fromArray($userData) : null;
    }
}

// Good: Output escaping
function safeOutput(string $data): string
{
    return htmlspecialchars($data, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

// Good: Password handling
class PasswordManager
{
    public function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3,
        ]);
    }
    
    public function verifyPassword(string $password, string $hash): bool
    {
        return password_verify($password, $hash);
    }
}
```

### PHP 8+ Features

```php
<?php
declare(strict_types=1);

// Constructor property promotion
class Point
{
    public function __construct(
        public readonly float $x,
        public readonly float $y,
        public readonly float $z = 0.0,
    ) {
    }
}

// Enums (PHP 8.1+)
enum Status: string
{
    case PENDING = 'pending';
    case APPROVED = 'approved';
    case REJECTED = 'rejected';
    
    public function getLabel(): string
    {
        return match($this) {
            self::PENDING => 'Awaiting Review',
            self::APPROVED => 'Approved',
            self::REJECTED => 'Rejected',
        };
    }
}

// Match expression (PHP 8.0+)
function getStatusColor(Status $status): string
{
    return match($status) {
        Status::PENDING => 'yellow',
        Status::APPROVED => 'green',
        Status::REJECTED => 'red',
    };
}

// Nullsafe operator (PHP 8.0+)
$country = $user?->getAddress()?->getCountry();
```

### Testing with PHPUnit

```php
<?php
declare(strict_types=1);

use PHPUnit\Framework\TestCase;

final class CalculatorTest extends TestCase
{
    private Calculator $calculator;
    
    protected function setUp(): void
    {
        $this->calculator = new Calculator();
    }
    
    public function testAddition(): void
    {
        $result = $this->calculator->add(2, 3);
        $this->assertSame(5, $result);
    }
    
    /**
     * @dataProvider divisionProvider
     */
    public function testDivision(float $a, float $b, float $expected): void
    {
        $result = $this->calculator->divide($a, $b);
        $this->assertSame($expected, $result);
    }
    
    public static function divisionProvider(): array
    {
        return [
            'positive numbers' => [10.0, 2.0, 5.0],
            'negative dividend' => [-10.0, 2.0, -5.0],
            'decimal result' => [10.0, 3.0, 3.333333333333333],
        ];
    }
    
    public function testDivisionByZero(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('Division by zero');
        
        $this->calculator->divide(10.0, 0.0);
    }
}
```

## Tools and Verification

- **PHP Manual**: Official documentation at php.net
- **PHPStan/Psalm**: Static analysis tools
- **PHP CS Fixer**: Automatic code style fixing
- **Composer**: Dependency management
- **Xdebug**: Debugging and profiling

## References

- [PHP Manual](https://www.php.net/manual/)
- [PHP-FIG PSR Standards](https://www.php-fig.org/psr/)
- [PHP The Right Way](https://phptherightway.com/)
- [Modern PHP](https://www.modernphp.com/)