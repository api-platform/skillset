---
name: api-filter
description: "Adds filters to API Platform collections using the canonical QueryParameter approach. Use whenever the user wants search, sorting, date ranges, boolean/enum/numeric filtering, IRI lookups, free-text search, or any 'let users filter/search/sort the list by X' request on a collection — even if they never say 'filter'. Also use when migrating legacy #[ApiFilter]/SearchFilter code."
---

# Adding Filters to Collections

Declare filters with `QueryParameter` in the operation's `parameters:` array. The
key is the query-string parameter exposed to clients; the `filter:` is the filter
instance. This is the canonical approach for API Platform 4.4+ — it works with any
state provider and is per-operation explicit.

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

> **`#[ApiFilter]` is legacy.** The class-level `#[ApiFilter(SearchFilter::class, ...)]`
> attribute and the multi-strategy filters (`SearchFilter`, `BooleanFilter`,
> `NumericFilter`, `OrderFilter`, `BackedEnumFilter`) are **deprecated in 4.4 and
> removed in 6.0**. Don't teach or add them in new code. If you encounter them, the
> migration target is the `QueryParameter` + single-purpose filter set below.
> The mapping is in the table at the bottom.

## Canonical filter set

Each filter does one thing. They live in two parallel namespaces — pick the one
matching your persistence layer:

| Filter | Purpose | ORM | MongoDB ODM |
|---|---|---|---|
| `ExactFilter` | equality, multi-value (`IN`); booleans/ints/enums via `nativeType` | `ApiPlatform\Doctrine\Orm\Filter\ExactFilter` | `ApiPlatform\Doctrine\Odm\Filter\ExactFilter` |
| `PartialSearchFilter` | `LIKE %x%` substring | `…\Orm\Filter\PartialSearchFilter` | `…\Odm\Filter\PartialSearchFilter` |
| `ComparisonFilter` | `gt`/`gte`/`lt`/`lte`/`ne` (decorates an equality filter) | `…\Orm\Filter\ComparisonFilter` | `…\Odm\Filter\ComparisonFilter` |
| `SortFilter` | `ORDER BY` | `…\Orm\Filter\SortFilter` | `…\Odm\Filter\SortFilter` |
| `DateFilter` | `before`/`after` date ranges | `…\Orm\Filter\DateFilter` | `…\Odm\Filter\DateFilter` |
| `RangeFilter` | `between`/`gt`/`lt` on numbers | `…\Orm\Filter\RangeFilter` | `…\Odm\Filter\RangeFilter` |
| `ExistsFilter` | `IS NULL` / `IS NOT NULL` | `…\Orm\Filter\ExistsFilter` | `…\Odm\Filter\ExistsFilter` |
| `IriFilter` | relationship lookup by IRI | `…\Orm\Filter\IriFilter` | `…\Odm\Filter\IriFilter` |
| `OrFilter` | decorator: switches a primary's `WHERE` to `orWhere` | `…\Orm\Filter\OrFilter` | `…\Odm\Filter\OrFilter` |
| `FreeTextQueryFilter` | decorator: broadcasts one value across N properties | `…\Orm\Filter\FreeTextQueryFilter` | `…\Odm\Filter\FreeTextQueryFilter` |

> `ComparisonFilter` and `OrFilter` are still marked `@experimental` in 4.4 — they
> are the intended migration target but their API may shift before stabilizing.

## Common filters

### Exact match

```php
'status' => new QueryParameter(filter: new ExactFilter())
```
`GET /orders?status=shipped`. Arrays produce an `IN`: `?status[]=draft&status[]=sent`.

### Boolean / integer / enum — `ExactFilter` + `nativeType`

There is no `BooleanFilter` in the canonical set. Use `ExactFilter` and declare the
native type so values are cast and documented correctly:

```php
use Symfony\Component\TypeInfo\Type\BuiltinType;
use Symfony\Component\TypeInfo\TypeIdentifier;

'active' => new QueryParameter(
    filter: new ExactFilter(),
    nativeType: new BuiltinType(TypeIdentifier::BOOL),
),
```
`GET /orders?active=true`. Use `TypeIdentifier::INT` for integers; pass an enum's
native type for backed enums.

### Partial search (LIKE)

```php
use ApiPlatform\Doctrine\Orm\Filter\PartialSearchFilter;

'q' => new QueryParameter(filter: new PartialSearchFilter(), property: 'title')
```
`GET /books?q=harry`. `property:` maps the public param name to the entity field.

### Comparison (`gt`/`gte`/`lt`/`lte`/`ne`)

```php
use ApiPlatform\Doctrine\Orm\Filter\ComparisonFilter;

'price' => new QueryParameter(filter: new ComparisonFilter(new ExactFilter()))
```
`GET /products?price[gt]=100&price[lte]=500`.

### Date range

```php
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

'createdAt' => new QueryParameter(filter: new DateFilter())
```
`GET /orders?createdAt[after]=2024-01-01&createdAt[before]=2024-12-31`.

### Numeric range

```php
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;

'price' => new QueryParameter(filter: new RangeFilter())
```
`GET /products?price[between]=10..100`.

### Exists (null check)

```php
use ApiPlatform\Doctrine\Orm\Filter\ExistsFilter;

'deletedAt' => new QueryParameter(filter: new ExistsFilter())
```
`GET /orders?deletedAt[exists]=false`.

### Relation (IRI)

```php
use ApiPlatform\Doctrine\Orm\Filter\IriFilter;

'author' => new QueryParameter(filter: new IriFilter())
```
`GET /books?author=/authors/1`.

### Sorting — `SortFilter`

There is no `OrderFilter` in the canonical set. Use `SortFilter`. Two forms:

```php
use ApiPlatform\Doctrine\Orm\Filter\SortFilter;

// Per-property named parameter
'orderName' => new QueryParameter(filter: new SortFilter(), property: 'name'),
// → GET /books?orderName=desc

// Dynamic, OrderFilter-style: one parameter, any allowed property
'order[:property]' => new QueryParameter(filter: new SortFilter()),
// → GET /books?order[name]=asc&order[createdAt]=desc
```

`property:` can traverse relations (`'department.company.name'` sorts across joins).
Control null placement with `nullsComparison`:

```php
use ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface;

'orderHireDate' => new QueryParameter(
    filter: new SortFilter(nullsComparison: OrderFilterInterface::NULLS_ALWAYS_FIRST),
    property: 'hireDate',
),
```

### Free-text across multiple fields

`FreeTextQueryFilter` decorates a primary and broadcasts one value to several
properties; wrap with `OrFilter` to match any of them:

```php
use ApiPlatform\Doctrine\Orm\Filter\FreeTextQueryFilter;
use ApiPlatform\Doctrine\Orm\Filter\OrFilter;
use ApiPlatform\Doctrine\Orm\Filter\ExactFilter;

'autocomplete' => new QueryParameter(
    filter: new FreeTextQueryFilter(new OrFilter(new ExactFilter())),
    properties: ['name', 'email', 'description'],
),
```
`GET /users?autocomplete=alice`.

## Validating filter parameters

```php
use Symfony\Component\Validator\Constraints as Assert;

'length' => new QueryParameter(
    filter: new ExactFilter(),
    constraints: [new Assert\Length(max: 100)],
)
```
Invalid values yield a 422 before the query runs.

## Default ordering

Set a default sort directly on the operation (no parameter needed):

```php
new GetCollection(order: ['createdAt' => 'DESC'])
```

## Legacy → canonical migration map

| Legacy (deprecated 4.4, removed 6.0) | Canonical replacement |
|---|---|
| `#[ApiFilter(SearchFilter::class, ['x' => 'exact'])]` | `'x' => new QueryParameter(filter: new ExactFilter())` |
| `SearchFilter` strategy `partial` | `PartialSearchFilter` |
| `BooleanFilter` | `ExactFilter` + `nativeType: BOOL` |
| `NumericFilter` | `ExactFilter` + `nativeType: INT` |
| `BackedEnumFilter` | `ExactFilter` + enum `nativeType` |
| `OrderFilter` | `SortFilter` |
| `RangeFilter`, `DateFilter`, `ExistsFilter` | same class — survives as drop-in, just move to `QueryParameter` |

## Laravel (Eloquent)

The `QueryParameter` wiring is identical, but Laravel ships its **own** Eloquent
filter set under `ApiPlatform\Laravel\Eloquent\Filter\` — these are not the Doctrine
classes and the names differ. Declare them with class-level `#[QueryParameter]` /
`#[GetCollection(parameters: …)]` on the model (or DTO). The `key`, `property`,
`properties`, `:property` placeholder and `constraints` options work the same.

| Need | Doctrine (above) | Laravel Eloquent class |
|---|---|---|
| equality | `ExactFilter` | `EqualsFilter` |
| `LIKE %x%` | `PartialSearchFilter` | `PartialSearchFilter` |
| `LIKE x%` / `%x` | `StartSearchFilter` / `EndSearchFilter` | `StartSearchFilter` / `EndSearchFilter` |
| boolean | `ExactFilter` + `nativeType` | `BooleanFilter` |
| date range | `DateFilter` | `DateFilter` (`filterContext: ['include_nulls' => true]` to keep nulls; default excludes) |
| numeric range | `RangeFilter` | `RangeFilter` |
| sort | `SortFilter` | `OrderFilter` |
| OR | `OrFilter` | `OrFilter` (decorator: `new OrFilter(new EqualsFilter())`) |

```php
use ApiPlatform\Laravel\Eloquent\Filter\EqualsFilter;
use ApiPlatform\Laravel\Eloquent\Filter\OrderFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(key: 'name', filter: EqualsFilter::class)]
#[QueryParameter(key: 'sort[:property]', filter: OrderFilter::class, properties: ['name', 'id'])]
class Book extends Model {}
```

Constraints are **Laravel validation rules** (a string/array), not Symfony
constraints: `new QueryParameter(key: 'name', filter: PartialSearchFilter::class, constraints: 'min:2')`.
Custom filters implement `ApiPlatform\Laravel\Eloquent\Filter\FilterInterface`
(`apply(Builder $builder, mixed $values, Parameter $parameter, array $context = []): Builder`);
scaffold with `php artisan make:filter`. There is no `ExistsFilter`, `ComparisonFilter`,
`IriFilter` or `FreeTextQueryFilter` in the Eloquent set — `nativeType` on a parameter
is also unused (use `BooleanFilter` for booleans). `#[ApiFilter]` is not the Laravel
idiom; models declare `#[QueryParameter]` directly.
