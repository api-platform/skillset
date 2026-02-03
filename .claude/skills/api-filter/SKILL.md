---
name: api-filter
description: Add filters to API Platform collections. Use when implementing search, sorting, date ranges, boolean filters, or custom query parameters on GetCollection operations.
---

# Adding Filters to Collections

API Platform provides filtering via the `parameters` attribute on `GetCollection` operations.

## Basic Filter Setup

```php
<?php
namespace App\Api\Resource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Doctrine\Orm\Filter\ExactFilter;

#[ApiResource]
#[GetCollection(
    parameters: [
        'name' => new QueryParameter(filter: new ExactFilter()),
    ],
)]
class YourResource
{
    public string $name;
}
```

Client request: `GET /your_resources?name=value`

## Common Filter Types

### Exact Match
```php
'fieldName' => new QueryParameter(filter: new ExactFilter())
```

### Partial Search (LIKE)
```php
use ApiPlatform\Doctrine\Orm\Filter\PartialSearchFilter;

'search' => new QueryParameter(
    filter: new PartialSearchFilter(),
    property: 'name', // Maps 'search' param to 'name' property
)
```

### Boolean Filter
```php
use ApiPlatform\Doctrine\Orm\Filter\BooleanFilter;

'active' => new QueryParameter(filter: new BooleanFilter())
```
Client: `GET /resources?active=true`

### Date Filter
```php
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

'createdAt' => new QueryParameter(filter: new DateFilter())
```
Client: `GET /resources?createdAt[after]=2024-01-01&createdAt[before]=2024-12-31`

### IRI Filter (Relations)
```php
use ApiPlatform\Doctrine\Orm\Filter\IriFilter;

'author' => new QueryParameter(filter: new IriFilter())
```
Client: `GET /resources?author=/authors/1`

### Order Filter (Sorting)
```php
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

'order' => new QueryParameter(
    filter: new OrderFilter(),
    properties: ['name', 'createdAt']
)
```
Client: `GET /resources?order[name]=asc&order[createdAt]=desc`

## Combined Filters (OR Search)

Search across multiple fields:

```php
use ApiPlatform\Doctrine\Orm\Filter\FreeTextQueryFilter;
use ApiPlatform\Doctrine\Orm\Filter\OrFilter;
use ApiPlatform\Doctrine\Orm\Filter\ExactFilter;

'autocomplete' => new QueryParameter(
    filter: new FreeTextQueryFilter(new OrFilter(new ExactFilter())),
    properties: ['name', 'email', 'description']
)
```
Client: `GET /resources?autocomplete=searchterm`

## Complete Example

```php
#[GetCollection(
    parameters: [
        // Exact match
        'status' => new QueryParameter(filter: new ExactFilter()),

        // Partial search
        'q' => new QueryParameter(
            filter: new PartialSearchFilter(),
            property: 'title',
        ),

        // Date range
        'publishedAt' => new QueryParameter(filter: new DateFilter()),

        // Boolean
        'available' => new QueryParameter(filter: new BooleanFilter()),

        // Relation
        'author' => new QueryParameter(filter: new IriFilter()),

        // Sorting
        'order' => new QueryParameter(
            filter: new OrderFilter(),
            properties: ['publishedAt', 'title']
        ),
    ],
)]
class Book {}
```

## Validating Filter Parameters

Add constraints to filter parameters:

```php
use Symfony\Component\Validator\Constraints as Assert;

'length' => new QueryParameter(
    filter: new ExactFilter(),
    constraints: [new Assert\Length(max: 100)]
)
```

## Reference

For detailed patterns, see:
- [Filtering Guide](../../../skills/filtering/AGENTS.md)
