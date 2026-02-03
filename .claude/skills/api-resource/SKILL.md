---
name: api-resource
description: Create or modify API Platform resources with DTOs. Use when adding new API endpoints, creating resources, defining input/output DTOs, or configuring resource attributes.
---

# Creating API Platform Resources

When creating or modifying API resources for this project, follow these guidelines.

## Design-First Principle

This project follows **design-first** methodology:
1. First design the **public shape** (DTO) of your API endpoint
2. The DTO represents your API contract and maps to documentation
3. DTOs don't need to be Doctrine entities

## Implementation Strategies

### 1. Simple DTO as Resource (Recommended)

Use a DTO with custom State Providers/Processors:

```php
<?php
namespace App\Api\Resource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use App\State\YourProvider;
use App\State\YourProcessor;

#[ApiResource(
    operations: [
        new Get(provider: YourProvider::class),
        new GetCollection(provider: YourProvider::class),
        new Post(processor: YourProcessor::class)
    ]
)]
class YourResource
{
    public int $id;
    public string $name;
}
```

### 2. DTO with stateOptions (When Fields Match Entity)

When DTO and Entity have similar fields, use `stateOptions` with `#[Map]`:

```php
<?php
namespace App\Api\Resource;

use ApiPlatform\Doctrine\Orm\State\Options;
use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\ObjectMapper\Attribute\Map;
use App\Entity\YourEntity;

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

### 3. Different Input DTOs for Write Operations

When write model differs from read model:

```php
<?php
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Patch;

#[ApiResource(
    operations: [
        new Post(input: CreateYourResourceInput::class),
        new Patch(input: UpdateYourResourceInput::class),
    ]
)]
class YourResource { /* ... */ }
```

## Checklist

When creating a new resource:
- [ ] Create the DTO class in `src/Api/Resource/`
- [ ] Add `#[ApiResource]` with appropriate operations
- [ ] Define provider for read operations
- [ ] Define processor for write operations
- [ ] Add validation constraints to input DTOs
- [ ] Consider schema.org semantics for property naming

## Reference

For detailed patterns, see:
- [Data Modeling Guide](../../../skills/data-model/AGENTS.md)
- [State Management Guide](../../../skills/state.md)
