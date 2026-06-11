---
name: api-resource
description: "Creates or modifies API Platform resources with DTOs and Object Mapper. Use whenever the user wants to add an API endpoint, expose an entity or any data over HTTP, create or reshape a resource, define input/output DTOs, configure nested sub-resources with uriVariables, or map entities/documents to API representations — even if they just say 'add an endpoint for X' or 'expose X in the API'."
---

# Creating API Platform Resources

## Design-First Principle

An API Platform resource is built from three concerns (see
<https://api-platform.com/docs/core/design/>):

1. **Resource declaration** — a plain PHP object marked `#[ApiResource]` describing
   the public shape. Single source of truth for Hydra, OpenAPI and GraphQL.
2. **Data retrieval** — a state **provider** hydrates that object (Get, GetCollection).
3. **Data persistence** — a state **processor** writes it (Post, Put, Patch, Delete).

Design the public shape first; the resource class doesn't have to be a Doctrine
entity. How you wire it to persistence is a separate decision:

| Approach | When | Hookup |
|---|---|---|
| **Entity as Resource** | Prototyping, plain CRUD, internal model == public shape | `#[ApiResource]` on the entity; built-in Doctrine provider/processor (zero wiring) |
| **DTO with Object Mapper** | Decoupled public shape over a Doctrine entity/document — whether fields match 1:1 or need renames/transforms | `stateOptions: new Options(entityClass:/documentClass:)` **and** `#[Map]` on the DTO |
| **Input DTOs per Operation** | Write model differs from read model (stricter create/update payloads) | per-operation `input:` + processor |
| **Custom Provider/Processor** | Non-CRUD, external data, complex domain logic, CQRS | hand-written `ProviderInterface`/`ProcessorInterface` (see **state-provider**/**state-processor**) |

Rule of thumb: entity-as-resource is convenient but couples your public contract to
your schema — fine for prototypes, *probably not* for large or non-CRUD systems.
Decouple with a DTO as soon as the two shapes diverge.

## Entity as Resource (Simplest)

Mark the entity itself — built-in Doctrine providers/processors handle everything.
No provider, processor, or mapping to write:

```php
use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Book
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    public ?int $id = null;

    #[ORM\Column]
    public string $title = '';
}
```

Use this for prototypes and straight CRUD. Migrate to a DTO (strategies below) once
the API shape must differ from the schema.

## DTO with Object Mapper (Recommended)

One mechanism, not two: API Platform's `ObjectMapperProvider` /
`ObjectMapperProcessor` activate only when **both** are present — `stateOptions`
naming the `entityClass:` (or `documentClass:`) **and** a `#[Map]` attribute on the
DTO (and on the input entity for writes). On read it runs
`map($entity, $resourceClass)`; on write `map($inputDto, $entityClass)` then persists
via the Doctrine processor. There is no Object-Mapper mode without `stateOptions`.

Fields that line up by name map automatically; add `#[Map(source: …)]` to rename or
`transform:` to convert. The same config covers both the trivial 1:1 case and
arbitrary renames/transforms:

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

## Input DTOs per Operation

When the write model differs from the read model, give individual operations their
own `input:` DTO and processor (the resource class stays the read model):

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

## Backed Enums

### As a property

A backed enum property serializes to its `->value` automatically:

```php
#[ApiResource]
class Person
{
    public GenderType $genderType; // backed enum → {"genderType": "female"}
}
```

### As a resource

Expose a backed enum as a read-only resource so clients can discover allowed values:

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
enum AvailabilityStatus: string
{
    case InStock = 'InStock';
    case OutOfStock = 'OutOfStock';
}
```

`GET /availability_statuses` lists all cases; `GET /availability_statuses/InStock`
returns one.

## High-Precision Numbers

For monetary/scientific values that must avoid float drift, type a property as
`\BcMath\Number` (PHP 8.4 native, requires ext-bcmath). It serializes as a string
to preserve precision:

```php
class Invoice
{
    public ?\BcMath\Number $total; // → "300.55"
}
```

## Laravel (Eloquent)

The design-first split, DTOs, Object Mapper, per-operation `input:`, nested
`uriVariables`/`handleLinks`, custom operations and backed-enum resources all work on
Laravel. Deltas:

- **Model as resource:** `#[ApiResource]` on a class extending
  `Illuminate\Database\Eloquent\Model`; the Eloquent provider/processor handle CRUD
  with zero wiring.
- **Eloquent `Options`** (`ApiPlatform\Laravel\Eloquent\State\Options`) uses
  `modelClass:` (not `entityClass:`/`documentClass:`); it carries `modelClass` +
  `handleLinks` only — no `repositoryMethod`.
- **Object Mapper** is supported: pair `stateOptions: new Options(modelClass: ProductModel::class)`
  with `#[Map(source: ProductModel::class)]` on the DTO; per-property `#[Map]` and
  `TransformCallableInterface` are identical.
- **Properties:** Eloquent models have no typed properties, so declare `#[ApiProperty]`
  at the **class level** with `property:` (`#[ApiProperty(property: 'title', identifier: true)]`).
  DTO/`ApiResource` classes use property-level attributes. `\BcMath\Number` and backed
  enums behave the same.

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
