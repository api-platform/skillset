---
name: state-provider
description: Creates state providers for reading data in API Platform. Use when implementing custom data retrieval, enriching collections with computed fields, transforming entities to different DTOs, or decorating built-in Doctrine providers.
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

## Best Practices

1. Return `null` for missing items — API Platform handles the 404
2. Use `CollectionOperationInterface` to distinguish collection vs item
3. When decorating, always delegate to the built-in provider first
4. For computed fields on collections, iterate over the paginator (it's lazy)
