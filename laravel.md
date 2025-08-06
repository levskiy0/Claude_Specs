# Laravel Style Guide

This style guide adapts core programming principles to Laravel 10+, based on official Laravel conventions and best practices.

## Core Principles

### 1. Store Repeating Values as Constants

Use configuration files and environment variables following Laravel conventions.

```php
<?php
// Good: Configuration file approach
// config/business.php
return [
    'order_status' => [
        'PENDING' => 'pending',
        'PROCESSING' => 'processing',
        'COMPLETED' => 'completed',
        'CANCELLED' => 'cancelled',
    ],
    'user_roles' => [
        'ADMIN' => 'administrator',
        'USER' => 'user',
        'MODERATOR' => 'moderator',
    ],
];

// Good: Environment-specific values
// .env
APP_NAME="My Laravel App"
APP_ENV=production
APP_DEBUG=false

// Usage in application
$status = config('business.order_status.PENDING');
$appName = config('app.name');

// Bad: Direct env() usage in application
$appName = env('APP_NAME'); // Don't do this outside config files
```

### 2. Reuse Code - Don't Duplicate

Use Actions, Services, Traits, and Laravel's built-in features.

```php
<?php
// Good: Action class for reusable business logic
namespace App\Actions\User;

use App\Models\User;
use Illuminate\Support\Facades\Hash;

class CreateUser
{
    public function handle(array $data): User
    {
        return User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
        ]);
    }
}

// Good: Trait for common model functionality
trait HasUuid
{
    protected static function bootHasUuid(): void
    {
        static::creating(function ($model) {
            $model->uuid = (string) Str::uuid();
        });
    }
    
    public function getRouteKeyName(): string
    {
        return 'uuid';
    }
}

// Good: Service class for complex operations
class OrderService
{
    public function __construct(
        private PaymentService $paymentService,
        private InventoryService $inventoryService,
        private NotificationService $notificationService
    ) {}
    
    public function processOrder(Order $order): void
    {
        DB::transaction(function () use ($order) {
            $this->paymentService->charge($order);
            $this->inventoryService->reserve($order->items);
            $this->notificationService->sendOrderConfirmation($order);
        });
    }
}
```

### 3. Don't Write Monolithic Code

Follow Laravel's modular architecture with thin controllers and service layers.

```php
<?php
// Good: Thin controller
class UserController extends Controller
{
    public function store(
        StoreUserRequest $request,
        CreateUser $createUser
    ) {
        $user = $createUser->handle($request->validated());
        
        return new UserResource($user);
    }
}

// Good: Single Action Controller
class PublishPostController extends Controller
{
    public function __invoke(Post $post, PublishPost $publishPost)
    {
        $this->authorize('publish', $post);
        
        $publishPost->handle($post);
        
        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'Post published successfully');
    }
}

// Bad: Fat controller with business logic
class BadController extends Controller
{
    public function store(Request $request)
    {
        // Validation
        $validated = $request->validate([...]);
        
        // Business logic (should be in service/action)
        $user = User::create([...]);
        
        // More business logic
        Mail::to($user)->send(new WelcomeEmail($user));
        
        // Even more logic
        Activity::create([...]);
        
        return response()->json($user);
    }
}
```

### 4. Don't Make Things Up

Always refer to official Laravel documentation and use established patterns.

```php
<?php
// Good: Document when unsure
class ComplexFeature
{
    public function handle()
    {
        // TODO: Research Laravel Octane for this use case
        // I don't know the best approach for handling WebSocket connections
        throw new \Exception('Not implemented: need to research Laravel Octane');
    }
}

// Good: Use Laravel's documented features
// Check laravel.com/docs for official methods
```

### 5. Don't Invent Non-existent Methods

Use IDE helpers and official documentation to verify Laravel methods.

```php
<?php
// Good: Install and use IDE Helper
// composer require --dev barryvdh/laravel-ide-helper
// php artisan ide-helper:generate

// Good: Verify Eloquent methods exist
if (method_exists(User::class, 'scopeActive')) {
    $activeUsers = User::active()->get();
}

// Good: Check facade methods
// Use official docs: laravel.com/docs/facades

// Bad: Using non-existent methods
$users = User::whereActiveTrue()->get(); // This method doesn't exist
```

## Laravel-Specific Best Practices

### Directory Structure

```
app/
├── Actions/           # Single-purpose action classes
├── Console/          # Artisan commands
├── DTOs/             # Data Transfer Objects
├── Events/           # Event classes
├── Exceptions/       # Custom exceptions
├── Http/
│   ├── Controllers/  # HTTP controllers
│   ├── Middleware/   # HTTP middleware
│   └── Requests/     # Form requests
├── Jobs/             # Queue jobs
├── Listeners/        # Event listeners
├── Mail/             # Mailables
├── Models/           # Eloquent models
├── Notifications/    # Notification classes
├── Policies/         # Authorization policies
├── Providers/        # Service providers
├── Repositories/     # Repository pattern (optional)
├── Rules/            # Custom validation rules
├── Services/         # Service classes
└── ViewModels/       # View models (optional)
```

### Naming Conventions

- **Controllers**: Plural resource name + Controller (`UsersController`)
- **Models**: Singular, StudlyCase (`User`, `BlogPost`)
- **Migrations**: Descriptive with timestamp (`2023_01_01_000000_create_users_table`)
- **Form Requests**: Action + Resource + Request (`StoreUserRequest`)
- **Jobs**: Describe the action (`SendWelcomeEmail`)
- **Events**: Past or present tense (`UserRegistered`, `OrderProcessing`)
- **Policies**: Model + Policy (`UserPolicy`)

### Eloquent Best Practices

```php
<?php
// Good: Use relationships and eager loading
$users = User::with(['posts', 'profile'])->active()->get();

// Good: Use scopes for reusable queries
class User extends Model
{
    public function scopeActive($query)
    {
        return $query->where('active', true);
    }
    
    public function scopeWithRole($query, string $role)
    {
        return $query->where('role', $role);
    }
}

// Good: Use accessors and mutators
class User extends Model
{
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => ucfirst($value),
            set: fn ($value) => strtolower($value),
        );
    }
}

// Good: Use casts
class Product extends Model
{
    protected $casts = [
        'price' => 'decimal:2',
        'metadata' => 'array',
        'published_at' => 'datetime',
        'is_featured' => 'boolean',
    ];
}
```

### Form Requests

```php
<?php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }
    
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'min:8', 'confirmed'],
            'role' => ['required', 'in:user,admin,moderator'],
        ];
    }
    
    public function messages(): array
    {
        return [
            'email.unique' => 'This email address is already registered.',
            'password.min' => 'Password must be at least :min characters.',
        ];
    }
    
    public function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower($this->email),
        ]);
    }
}
```

### Service Pattern

```php
<?php
namespace App\Services;

use App\DTOs\CreateUserDTO;
use App\Models\User;
use App\Actions\User\CreateUser;
use Illuminate\Support\Facades\DB;

class UserService
{
    public function __construct(
        private CreateUser $createUser,
        private EmailService $emailService
    ) {}
    
    public function registerUser(CreateUserDTO $dto): User
    {
        return DB::transaction(function () use ($dto) {
            $user = $this->createUser->handle($dto->toArray());
            
            $this->emailService->sendWelcomeEmail($user);
            
            event(new UserRegistered($user));
            
            return $user;
        });
    }
}
```

### API Resources

```php
<?php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'role' => $this->role,
            'created_at' => $this->created_at->toISOString(),
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'posts_count' => $this->when(
                $this->posts_count !== null,
                $this->posts_count
            ),
        ];
    }
}

// Usage
return UserResource::collection($users);
return new UserResource($user);
```

### Testing

```php
<?php
namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_can_create_user(): void
    {
        $userData = [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123',
            'password_confirmation' => 'password123',
        ];
        
        $response = $this->postJson('/api/users', $userData);
        
        $response->assertCreated()
            ->assertJsonStructure([
                'data' => ['id', 'name', 'email', 'created_at']
            ]);
        
        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
        ]);
    }
    
    public function test_validates_user_input(): void
    {
        $response = $this->postJson('/api/users', []);
        
        $response->assertUnprocessable()
            ->assertJsonValidationErrors(['name', 'email', 'password']);
    }
}
```

### Queue Jobs

```php
<?php
namespace App\Jobs;

use App\Models\User;
use App\Mail\WelcomeEmail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function __construct(
        public User $user
    ) {}
    
    public function handle(): void
    {
        Mail::to($this->user)->send(new WelcomeEmail($this->user));
    }
    
    public function failed(\Throwable $exception): void
    {
        \Log::error('Failed to send welcome email', [
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}

// Dispatching
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(5));
```

## Tools and Verification

- **Laravel IDE Helper**: Auto-completion for facades and models
- **Laravel Debugbar**: Development debugging
- **Laravel Telescope**: Debug assistant for local development
- **Artisan Commands**: Built-in CLI tools
- **Tinker**: Interactive REPL for testing code

## References

- [Laravel Documentation](https://laravel.com/docs)
- [Laravel Best Practices](https://github.com/alexeymezenin/laravel-best-practices)
- [Spatie Laravel Guidelines](https://spatie.be/guidelines/laravel-php)
- [Laravel News](https://laravel-news.com/)