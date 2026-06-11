---
name: state-provider
description: "Creates state providers for read operations in API Platform. Use whenever GET data needs custom retrieval or shaping — computed or enriched fields, transforming entities to different DTOs, decorating built-in Doctrine providers, sortable computed fields via repositoryMethod — or any 'the response should also include X' request, even if the user doesn't say 'provider'."
---

# Creating State Providers

State Providers retrieve data for read operations (Get, GetCollection).

## Basic Structure

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\Metadata\CollectionOperationInterface;
use ApiPlatform\State\ProviderInterface;

/**
 * @implements ProviderInterface<YourResource|null>
 */
final class YourResourceProvider implements ProviderInterface
{
    public function provide(
        Operation $operation,
        array $uriVariables = [],
        array $context = []
    ): object|array|null {
        if ($operation instanceof CollectionOperationInterface) {
            return $this->repository->findAll();
        }

        return $this->repository->find($uriVariables['id']);
    }
}
```

## Decorating Built-in Providers

Wrap the default provider to add custom logic (computed fields, enrichment):

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class EnrichedItemProvider implements ProviderInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine.orm.state.item_provider')]
        private ProviderInterface $itemProvider,
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): ?object
    {
        $data = $this->itemProvider->provide($operation, $uriVariables, $context);

        if (null === $data) {
            return null; // API Platform returns 404 automatically
        }

        // Enrich with computed fields
        $data->computedScore = $this->computeScore($data);

        return $data;
    }
}
```

### Doctrine service names

| Persistence | Item Provider | Collection Provider |
|---|---|---|
| **ORM** | `api_platform.doctrine.orm.state.item_provider` | `api_platform.doctrine.orm.state.collection_provider` |
| **MongoDB ODM** | `api_platform.doctrine_mongodb.odm.state.item_provider` | `api_platform.doctrine_mongodb.odm.state.collection_provider` |

> **Laravel (Eloquent):** `ProviderInterface` is identical and resources reference a
> provider by class-string (`provider: YourProvider::class`). There are no
> `api_platform.doctrine.*` service ids — the built-in Eloquent providers are bound
> in the container by their class name
> (`ApiPlatform\Laravel\Eloquent\State\ItemProvider` /
> `…\State\CollectionProvider`). To decorate one, type-hint that concrete class in
> your constructor (the container resolves it). Stable scalar/computed-field sorting
> uses `stateOptions: new Options(modelClass: …)` on the Eloquent `Options` and an
> `OrderFilter` parameter rather than `repositoryMethod`.

## Enriching Paginated Collections

Iterate over paginated results to add computed fields:

```php
use ApiPlatform\State\Pagination\PaginatorInterface;

final class MailboxListProvider implements ProviderInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine_mongodb.odm.state.collection_provider')]
        private ProviderInterface $collectionProvider,
        private MailboxHelper $helper,
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): PaginatorInterface
    {
        $mailboxes = $this->collectionProvider->provide($operation, $uriVariables, $context);

        foreach ($mailboxes as $item) {
            $item->totalMessages = $this->helper->countMessages($item->getId());
        }

        return $mailboxes;
    }
}
```

## Cross-Resource Providers (Returning a Different Type)

A provider can fetch one resource type and return another:

```php
final class RawMessageProvider implements ProviderInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine_mongodb.odm.state.item_provider')]
        private ProviderInterface $itemProvider,
        private RawMessageBuilderInterface $builder,
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): ?RawMessage
    {
        // Fetch the Document\Message using Doctrine
        $message = $this->itemProvider->provide($operation, $uriVariables, $context);
        if (!$message) {
            return null;
        }

        // Transform to a completely different API resource
        $raw = new RawMessage();
        $raw->id = $message->getId();
        $raw->content = $this->builder->build($message);
        return $raw;
    }
}
```

## Provider Parameters

- **$operation**: Metadata about the operation (Get, GetCollection, etc.)
- **$uriVariables**: URI path variables (e.g., `['id' => '123', 'accountId' => '456']`)
- **$context**: Additional context:
  - `request`: The Symfony HttpFoundation Request
  - `resource_class`: The API resource class
  - `filters`: Applied query parameter filters

## Assigning to Resource

```php
#[Get(provider: YourProvider::class)]
#[GetCollection(provider: YourCollectionProvider::class)]
class YourResource {}
```

## Computed / Sortable Fields via a Repository Method (Doctrine)

When a field is a SQL aggregate that must also be **sortable** in the database, a
plain provider can't help — sorting happens in the query. The cleanest way is to
point the built-in Doctrine provider at a custom repository method that returns a
`QueryBuilder`, via `stateOptions` (API Platform 4.4+). The provider runs your
query builder, then layers pagination, filters and link handlers on top of it.

```php
// Repository: return a QueryBuilder, not results
class CartRepository extends EntityRepository
{
    public function getCartsWithTotalQuantity(): QueryBuilder
    {
        return $this->createQueryBuilder('o')
            ->leftJoin('o.items', 'items')
            ->addSelect('COALESCE(SUM(items.quantity), 0) AS totalQuantity')
            ->addGroupBy('o.id');
    }
}
```

```php
use ApiPlatform\Doctrine\Orm\State\Options;

#[ORM\Entity(repositoryClass: CartRepository::class)]
#[GetCollection(
    stateOptions: new Options(repositoryMethod: 'getCartsWithTotalQuantity'),
    processor: [self::class, 'process'],
    write: true,
    parameters: [
        'sort[:property]' => new QueryParameter(filter: new SortFilter(), properties: ['totalQuantity']),
    ],
)]
class Cart
{
    public int|string|null $totalQuantity;

    // A non-HIDDEN scalar addSelect makes rows arrive as [entity, 'totalQuantity' => N].
    // Flatten them back onto the entity with a processor:
    public static function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        foreach ($data as &$value) {
            $cart = $value[0];
            $cart->totalQuantity = $value['totalQuantity'] ?? 0;
            $value = $cart;
        }

        return $data;
    }
}
```

> Use a **Doctrine collection extension** (see **securing-collections**) instead when
> the query change must apply to *every* operation/query for the resource (e.g.
> multi-tenant isolation). `repositoryMethod` scopes one operation's base query;
> an extension mutates them all.

## Per-Property Serialization Context

Embed the same related resource at different depths by switching the
normalization group on one property with `#[Context]`:

```php
use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Attribute\Groups;

#[ApiResource(normalizationContext: ['groups' => ['initial']])]
class Order
{
    #[Groups(['initial'])]
    public ?Customer $customer = null;

    #[Groups(['initial'])]
    #[Context(['normalization' => ['groups' => ['summary']]])]
    public ?Customer $billedTo = null; // serialized with the 'summary' group only
}
```

## Best Practices

1. Return `null` for missing items — API Platform handles the 404
2. Use `CollectionOperationInterface` to distinguish collection vs item
3. When decorating, always delegate to the built-in provider first
4. For computed fields on collections, iterate over the paginator (it's lazy)
5. For *sortable* computed fields, use `stateOptions(repositoryMethod:)` — not a provider
