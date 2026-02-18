---
name: api-filter
description: Adds filters to API Platform collections. Use when implementing search, sorting, date ranges, boolean filters, or custom query parameters on GetCollection operations. Covers both the parameters approach and the ApiFilter attribute.
---

# Adding Filters to Collections

## Approach 1: `parameters` on Operations (Recommended)

Inline filter configuration directly on the operation:

```php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Doctrine\Orm\Filter\ExactFilter;

#[GetCollection(
    parameters: [
        'status' => new QueryParameter(filter: new ExactFilter()),
    ],
)]
class Order {}
```

Client: `GET /orders?status=shipped`

## Approach 2: `#[ApiFilter]` on Entity/Document

Declare filters at class level — applies to all GetCollection operations:

```php
use ApiPlatform\Doctrine\Odm\Filter\SearchFilter;
use ApiPlatform\Doctrine\Odm\Filter\BooleanFilter;
use ApiPlatform\Metadata\ApiFilter;

#[ApiFilter(SearchFilter::class, properties: ['address' => 'partial'])]
#[ApiFilter(BooleanFilter::class, properties: ['isActive'])]
class Account {}
```

Client: `GET /accounts?address=alice&isActive=true`

Both approaches are valid. Use `parameters` for per-operation control, `#[ApiFilter]` for class-wide defaults.

## Common Filter Types (parameters approach)

### Exact Match
```php
'status' => new QueryParameter(filter: new ExactFilter())
```

### Partial Search (LIKE)
```php
use ApiPlatform\Doctrine\Orm\Filter\PartialSearchFilter;

'q' => new QueryParameter(filter: new PartialSearchFilter(), property: 'title')
```

### Boolean
```php
use ApiPlatform\Doctrine\Orm\Filter\BooleanFilter;

'active' => new QueryParameter(filter: new BooleanFilter())
```

### Date Range
```php
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

'createdAt' => new QueryParameter(filter: new DateFilter())
```
Client: `GET /orders?createdAt[after]=2024-01-01&createdAt[before]=2024-12-31`

### Relation (IRI)
```php
use ApiPlatform\Doctrine\Orm\Filter\IriFilter;

'author' => new QueryParameter(filter: new IriFilter())
```
Client: `GET /books?author=/authors/1`

### Sorting
```php
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

'order' => new QueryParameter(filter: new OrderFilter(), properties: ['name', 'createdAt'])
```
Client: `GET /books?order[createdAt]=desc`

### Combined OR Search
```php
use ApiPlatform\Doctrine\Orm\Filter\FreeTextQueryFilter;
use ApiPlatform\Doctrine\Orm\Filter\OrFilter;
use ApiPlatform\Doctrine\Orm\Filter\ExactFilter;

'autocomplete' => new QueryParameter(
    filter: new FreeTextQueryFilter(new OrFilter(new ExactFilter())),
    properties: ['name', 'email', 'description']
)
```

## ORM vs MongoDB ODM Filter Namespaces

Filters exist in separate namespaces depending on your persistence layer:

| Filter | ORM | MongoDB ODM |
|---|---|---|
| Search | `ApiPlatform\Doctrine\Orm\Filter\SearchFilter` | `ApiPlatform\Doctrine\Odm\Filter\SearchFilter` |
| Boolean | `ApiPlatform\Doctrine\Orm\Filter\BooleanFilter` | `ApiPlatform\Doctrine\Odm\Filter\BooleanFilter` |
| Date | `ApiPlatform\Doctrine\Orm\Filter\DateFilter` | `ApiPlatform\Doctrine\Odm\Filter\DateFilter` |
| Order | `ApiPlatform\Doctrine\Orm\Filter\OrderFilter` | `ApiPlatform\Doctrine\Odm\Filter\OrderFilter` |

Use the namespace matching your persistence layer.

## Validating Filter Parameters

```php
use Symfony\Component\Validator\Constraints as Assert;

'length' => new QueryParameter(
    filter: new ExactFilter(),
    constraints: [new Assert\Length(max: 100)]
)
```

## Default Ordering

Set default sort order on GetCollection:

```php
new GetCollection(order: ['createdAt' => 'DESC'])
```
