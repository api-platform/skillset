---
name: api-resource
description: Creates or modifies API Platform resources with DTOs and Object Mapper. Use when adding new API endpoints, creating resources, defining input/output DTOs, configuring nested sub-resources with uriVariables, or mapping entities to API representations.
---

# Creating API Platform Resources

## Design-First Principle

1. Design the **public DTO shape** first — this is your API contract
2. Map it to your persistence layer (Doctrine entity/document) via Object Mapper or custom state providers/processors
3. DTOs don't need to be Doctrine entities

## Strategy 1: DTO as Resource with Object Mapper (Recommended)

Use `#[Map]` to transform between Document/Entity and API Resource:

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Odm\State\Options;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use App\Document\Order as DocumentOrder;
use App\Transformer\CustomerTransformer;
use Symfony\Component\ObjectMapper\Attribute\Map;

#[ApiResource(
    operations: [
        new Get(stateOptions: new Options(documentClass: DocumentOrder::class)),
        new GetCollection(stateOptions: new Options(documentClass: DocumentOrder::class)),
    ]
)]
#[Map(target: DocumentOrder::class)]
final class Order
{
    public string $id;

    // Simple 1:1 field mapping (automatic when names match)
    public string $status;

    // Map from a different source field
    #[Map(source: 'customerName')]
    public string $buyer;

    // Transform with a custom callable
    #[Map(source: 'rawData', transform: new CustomerTransformer())]
    public CustomerDto $customer;
}
```

## Custom Transformers

Implement `TransformCallableInterface` for complex mappings:

```php
<?php
namespace App\Transformer;

use Symfony\Component\DependencyInjection\Attribute\Exclude;
use Symfony\Component\ObjectMapper\TransformCallableInterface;

#[Exclude]
final class CustomerTransformer implements TransformCallableInterface
{
    public function __construct(private readonly string $field = 'name') {}

    public function __invoke(mixed $value, object $source, ?object $target): mixed
    {
        // $value = source field value, $source = full source object
        $dto = new CustomerDto();
        $dto->name = $value[$this->field] ?? '';
        return $dto;
    }
}
```

Use `#[Exclude]` so Symfony's container doesn't try to autowire transformer constructor args.

## Strategy 2: stateOptions with `#[Map]` (Simple Field Mapping)

When DTO and Entity fields mostly match:

```php
#[ApiResource(
    stateOptions: new Options(entityClass: YourEntity::class)
)]
#[Map(source: YourEntity::class)]
class YourResource
{
    public int $id;

    #[Map(source: 'entityFieldName')]
    public string $apiFieldName;
}
```

## Strategy 3: Different Input DTOs

When write model differs from read model:

```php
#[ApiResource(
    operations: [
        new Post(input: CreateOrderInput::class, processor: OrderCreateProcessor::class),
        new Patch(input: UpdateOrderInput::class, processor: OrderUpdateProcessor::class),
    ]
)]
class Order { /* read model */ }
```

## Nested Sub-Resources with uriVariables

For resources nested under parents (e.g., `/accounts/{accountId}/mailboxes/{mailboxId}/messages`):

```php
#[ApiResource(
    operations: [
        new Get(
            uriTemplate: '/accounts/{accountId}/mailboxes/{mailboxId}/messages/{id}',
            uriVariables: ['accountId', 'mailboxId', 'id'],
            stateOptions: new Options(
                handleLinks: MessageLinkHandler::class,
                documentClass: Message::class,
            )
        ),
        new GetCollection(
            uriTemplate: '/accounts/{accountId}/mailboxes/{mailboxId}/messages',
            uriVariables: ['accountId', 'mailboxId'],
            stateOptions: new Options(
                handleLinks: MessageLinkHandler::class,
                documentClass: Message::class,
            ),
            itemUriTemplate: '/accounts/{accountId}/mailboxes/{mailboxId}/messages/{id}',
        ),
    ]
)]
```

The `handleLinks` class validates parent ownership and applies security filters. See the **securing-collections** skill for implementation details.

Hidden IRI fields for URI generation:

```php
#[ApiProperty(readable: false, writable: false)]
#[Map(source: 'account', transform: new DocumentIdTransformer())]
public string $accountId;
```

## Custom Named Operations

Add non-CRUD actions on the same resource:

```php
new Put(
    uriTemplate: '/orders/{id}/cancel',
    input: CancelOrderInput::class,
    processor: OrderCancelProcessor::class,
    name: '_api_order_cancel',
),
new Get(
    write: true, // triggers processor on GET
    uriTemplate: '/orders/{id}/download',
    processor: OrderDownloadProcessor::class,
    name: '_api_order_download',
),
```

Use `write: true` on Get operations that need a processor (e.g., file downloads).

## Checklist

When creating a new resource:
- [ ] Create the DTO class in `src/ApiResource/`
- [ ] Add `#[ApiResource]` with operations
- [ ] Define provider for read operations (or use `stateOptions`)
- [ ] Define processor for write operations
- [ ] Add validation constraints to input DTOs
- [ ] For nested resources: configure `uriVariables` and `handleLinks`
- [ ] Add `#[Map]` attributes for entity/document mapping
- [ ] Add `#[ApiProperty]` for OpenAPI documentation
