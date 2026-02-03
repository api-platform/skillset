---
name: state-provider
description: Create state providers for reading data in API Platform. Use when implementing custom data retrieval, fetching from external APIs, aggregating data sources, or optimizing queries.
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
    public function __construct(
        private YourRepository $repository,
    ) {}

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

## Decorating Built-in Doctrine Provider

Wrap the default provider to add custom logic:

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class CustomProvider implements ProviderInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine.orm.state.item_provider')]
        private ProviderInterface $itemProvider,
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        $data = $this->itemProvider->provide($operation, $uriVariables, $context);

        // Add custom logic (computed fields, transformations, etc.)

        return $data;
    }
}
```

## Provider Parameters

- **$operation**: Metadata about the operation (Get, GetCollection, etc.)
- **$uriVariables**: URI path variables (e.g., `['id' => 123]`)
- **$context**: Additional context including request, filters, serialization context

## Common Context Keys

- `request`: The Symfony HTTP request object
- `resource_class`: The resource class being operated on
- `filters`: Applied query filters

## Assigning to Resource

```php
#[ApiResource]
#[Get(provider: YourResourceProvider::class)]
#[GetCollection(provider: YourResourceProvider::class)]
class YourResource {}
```

## Best Practices

1. Return `null` for missing items (API Platform handles 404)
2. Use `CollectionOperationInterface` to distinguish collection vs item
3. Inject only what you need via constructor
4. For complex queries, optimize with eager loading
5. Consider caching for expensive operations

## Reference

For detailed patterns, see:
- [State Management Guide](../../../skills/state.md)
- [State Providers/Processors Guide](../../../skills/state/AGENTS.md)
