# PHP Serializer

A simple, secure, and flexible serializer that converts any PHP object or data structure into a simple scalar array, ready to be sent via REST API.

The serializer automatically handles backward and forward compatibility, security concerns, and supports a wide range of PHP types including DTOs, entities, enums, DateTime objects, and more.

## :bulb: Key Principles

- **Zero dependencies** - Works standalone without any required external packages
- **Security-first** - Automatically hides sensitive data (passwords, credit cards, PINs)
- **Type-safe** - Full support for PHP 8.0+ typed properties and modern features
- **Recursion protection** - Detects and prevents infinite loops in object graphs
- **Depth limiting** - Protects against overly deep nested structures (max 32 levels)
- **Convention-based** - Configurable behavior through the convention system
- **Bridge interfaces** - Extensible architecture for custom type handling

## :building_construction: Architecture

The package follows a simple yet powerful architecture with clear separation of concerns:

```
┌───────────────────────────────────────────────────────────┐
│                        Serializer                         │
│  (Main entry point - singleton or instance-based usage)   │
├───────────────────────────────────────────────────────────┤
│                   SerializerConvention                    │
│  (Configuration: date format, null handling, hidden keys) │
├───────────────────────────────────────────────────────────┤
│                    Bridge Interfaces                      │
│  ┌─────────────────────┐  ┌───────────────────────────┐  │
│  │ ItemsListInterface  │  │  StatusCountInterface     │  │
│  │ (List collections)  │  │  (Status with count)      │  │
│  └─────────────────────┘  └───────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
                             │
                             ▼
┌───────────────────────────────────────────────────────────┐
│                     Supported Types                       │
│  - Scalars (string, int, float, bool, null)               │
│  - Arrays (indexed and associative)                       │
│  - Objects (DTOs, entities, stdClass)                     │
│  - DateTimeInterface                                      │
│  - UnitEnum (PHP 8.1+)                                    │
│  - Nette\Utils\Paginator                                  │
│  - Baraja\Localization\Translation                        │
│  - Baraja\EcommerceStandard\DTO\PriceInterface            │
│  - Objects with __toString() method                       │
└───────────────────────────────────────────────────────────┘
```

### :gear: Core Components

#### Serializer

The main class responsible for converting PHP data structures to arrays. It can be used as a singleton via `Serializer::get()` or instantiated with a custom convention.

**Key features:**
- Processes objects using reflection to access all properties (including private)
- Automatically skips internal properties (those starting with `_`)
- Tracks object instances to detect circular references
- Enforces depth limit to prevent stack overflow

#### SerializerConvention

Configuration class that controls serialization behavior:

| Property | Default | Description |
|----------|---------|-------------|
| `dateTimeFormat` | `'Y-m-d H:i:s'` | Format for DateTime serialization |
| `rewriteTooStringMethod` | `true` | Use `__toString()` for stringable objects |
| `rewriteNullToUndefined` | `false` | Remove null values from output |
| `keysToHide` | `['password', 'passwd', ...]` | Keys containing sensitive data |

#### Bridge Interfaces

**ItemsListInterface** - For objects representing a collection of items:
```php
interface ItemsListInterface
{
    /** @return array<int, array<string, mixed>> */
    public function getData(): array;
}
```

**StatusCountInterface** - For status objects with a count (useful for filters, tabs):
```php
interface StatusCountInterface
{
    public function getKey(): string;
    public function getLabel(): string;
    public function getCount(): int;
}
```

## :package: Installation

It's best to use [Composer](https://getcomposer.org) for installation, and you can also find the package on [Packagist](https://packagist.org/packages/baraja-core/serializer) and [GitHub](https://github.com/baraja-core/serializer).

To install, simply use the command:

```shell
$ composer require baraja-core/serializer
```

You can use the package manually by creating an instance of the internal classes.

### Requirements

- PHP 8.0 or higher
- No required dependencies (optional integrations available)

## :rocket: Basic Usage

### Simple DTO Serialization

```php
use Baraja\Serializer\Serializer;

class UserDTO
{
    public function __construct(
        public string $name,
        public string $email,
        public int $age,
    ) {
    }
}

$serializer = Serializer::get();

$user = new UserDTO(
    name: 'Jan Barasek',
    email: 'jan@example.com',
    age: 30,
);

$result = $serializer->serialize($user);
// Result: ['name' => 'Jan Barasek', 'email' => 'jan@example.com', 'age' => 30]
```

### Nested Objects

```php
class AddressDTO
{
    public function __construct(
        public string $street,
        public string $city,
    ) {
    }
}

class PersonDTO
{
    public function __construct(
        public string $name,
        public AddressDTO $address,
    ) {
    }
}

$person = new PersonDTO(
    name: 'Jan',
    address: new AddressDTO(street: 'Main St', city: 'Prague'),
);

$result = $serializer->serialize($person);
// Result: [
//     'name' => 'Jan',
//     'address' => ['street' => 'Main St', 'city' => 'Prague']
// ]
```

### Array Serialization

```php
$data = [
    'users' => [
        new UserDTO('Alice', 'alice@example.com', 25),
        new UserDTO('Bob', 'bob@example.com', 30),
    ],
    'total' => 2,
];

$result = $serializer->serialize($data);
```

### DateTime Handling

```php
class EventDTO
{
    public function __construct(
        public string $title,
        public \DateTimeInterface $startDate,
    ) {
    }
}

$event = new EventDTO(
    title: 'Conference',
    startDate: new \DateTime('2024-06-15 10:00:00'),
);

$result = $serializer->serialize($event);
// Result: ['title' => 'Conference', 'startDate' => '2024-06-15 10:00:00']
```

### Enum Serialization (PHP 8.1+)

```php
enum Status: string
{
    case Active = 'active';
    case Inactive = 'inactive';
}

class ItemDTO
{
    public function __construct(
        public string $name,
        public Status $status,
    ) {
    }
}

$item = new ItemDTO(name: 'Product', status: Status::Active);

$result = $serializer->serialize($item);
// Result: ['name' => 'Product', 'status' => 'active']
```

## :shield: Security Features

### Automatic Password Protection

The serializer automatically detects and hides sensitive keys. When a sensitive key is found, its value is replaced with `*****`:

```php
class LoginDTO
{
    public function __construct(
        public string $username,
        public string $password,
    ) {
    }
}

$login = new LoginDTO(username: 'admin', password: 'secret123');

$result = $serializer->serialize($login);
// Result: ['username' => 'admin', 'password' => '*****']
```

**Default hidden keys:** `password`, `passwd`, `pass`, `pwd`, `creditcard`, `credit card`, `cc`, `pin`

**Exception:** BCrypt hashes (format `$2[ayb]$...`) are allowed through, as they are already securely hashed.

### Internal Property Protection

Properties starting with underscore (`_`) are automatically excluded from serialization:

```php
class EntityDTO
{
    public function __construct(
        public string $name,
        public string $_internalId, // Will be excluded
    ) {
    }
}
```

### Recursion Detection

The serializer tracks object instances and throws an exception if circular references are detected:

```php
class Node
{
    public function __construct(
        public string $name,
        public ?Node $parent = null,
    ) {
    }
}

$node = new Node('A');
$node->parent = $node; // Circular reference

$serializer->serialize($node);
// Throws: InvalidArgumentException with recursion warning
```

## :wrench: Advanced Usage

### Custom Convention

Create a custom convention by extending `SerializerConvention`:

```php
use Baraja\Serializer\Serializer;
use Baraja\Serializer\SerializerConvention;

class CustomConvention extends SerializerConvention
{
    private string $dateTimeFormat = 'c'; // ISO 8601
    private bool $rewriteNullToUndefined = true;
}

$convention = new CustomConvention();
$serializer = new Serializer($convention);
```

### Using Bridge Interfaces

#### ItemsListInterface

Implement this interface for paginated or list results:

```php
use Baraja\Serializer\Bridge\ItemsListInterface;

class UserList implements ItemsListInterface
{
    /** @param UserDTO[] $users */
    public function __construct(
        private array $users,
    ) {
    }

    public function getData(): array
    {
        return array_map(
            fn(UserDTO $user) => [
                'name' => $user->name,
                'email' => $user->email,
            ],
            $this->users,
        );
    }
}
```

**Convention:** `ItemsListInterface` objects must be placed in a key named `items`.

#### StatusCountInterface

Useful for filter tabs or status counts:

```php
use Baraja\Serializer\Bridge\StatusCountInterface;

class OrderStatus implements StatusCountInterface
{
    public function __construct(
        private string $key,
        private string $label,
        private int $count,
    ) {
    }

    public function getKey(): string
    {
        return $this->key;
    }

    public function getLabel(): string
    {
        return $this->label;
    }

    public function getCount(): int
    {
        return $this->count;
    }
}

$statuses = [
    new OrderStatus('pending', 'Pending', 5),
    new OrderStatus('completed', 'Completed', 42),
];

$result = $serializer->serialize($statuses);
// Result: [
//     ['key' => 'pending', 'label' => 'Pending', 'count' => 5],
//     ['key' => 'completed', 'label' => 'Completed', 'count' => 42],
// ]
```

### Nette Paginator Support

If `nette/utils` is installed, Paginator objects are automatically serialized:

```php
use Nette\Utils\Paginator;

$paginator = new Paginator();
$paginator->setItemCount(100);
$paginator->setItemsPerPage(10);
$paginator->setPage(3);

$result = $serializer->serialize(['paginator' => $paginator]);
// Result: [
//     'paginator' => [
//         'page' => 3,
//         'pageCount' => 10,
//         'itemCount' => 100,
//         'itemsPerPage' => 10,
//         'firstPage' => 1,
//         'lastPage' => 10,
//         'isFirstPage' => false,
//         'isLastPage' => false,
//     ]
// ]
```

**Convention:** Paginator objects must be placed in a key named `paginator`.

### Translation Support

If `baraja-core/localization` is installed, Translation objects are converted to strings:

```php
use Baraja\Localization\Translation;

$translation = new Translation(['en' => 'Hello', 'cs' => 'Ahoj']);

$result = $serializer->serialize(['greeting' => $translation]);
// Result: ['greeting' => 'Hello'] (based on current locale)
```

### Price Support

If `baraja-core/ecommerce-standard` is installed, PriceInterface objects are serialized:

```php
// Assuming $price implements PriceInterface

$result = $serializer->serialize(['price' => $price]);
// Result: [
//     'price' => [
//         'value' => '199.00',
//         'currency' => 'USD',
//         'html' => '$199.00',
//         'isFree' => false,
//     ]
// ]
```

## :warning: Conventions and Constraints

### Enforced Conventions

1. **ItemsListInterface** must be in key `items`
2. **Paginator** must be in key `paginator`
3. **Maximum depth** is 32 levels
4. **Properties starting with `_`** are excluded

### Error Handling

The serializer throws exceptions for:

- **LogicException**: Structure depth exceeds 32 levels
- **InvalidArgumentException**: Circular reference detected
- **InvalidArgumentException**: ItemsListInterface not in `items` key
- **InvalidArgumentException**: Paginator not in `paginator` key
- **InvalidArgumentException**: Unsupported value type

## :warning: Security Logging

When a sensitive key with a non-BCrypt value is detected, the serializer logs a critical warning via Tracy Debugger (if available):

```
Security warning: User password may have been compromised!
The Baraja API prevented passwords being passed through the API in a readable form.
```

This helps identify potential security issues during development.

## :busts_in_silhouette: Author

**Jan Barasek** - [https://baraja.cz](https://baraja.cz)

## :page_facing_up: License

`baraja-core/serializer` is licensed under the MIT license. See the [LICENSE](https://github.com/baraja-core/serializer/blob/master/LICENSE) file for more details.
