---
name: pagination
description: Configures collection pagination in API Platform — page-based (default), items-per-page, client-controlled page size, maximum items, disabling, partial pagination, and cursor-based pagination. Use whenever the user mentions pagination, page size, itemsPerPage, paginating a collection, "too many results", slow COUNT queries on large tables, infinite scroll, cursor pagination, or returning a Paginator from a custom provider — even if they don't say "pagination" explicitly.
---

# Pagination

Pagination is **enabled by default** on every collection, 30 items per page, page
number in the `page` query parameter. The Hydra response carries a
`PartialCollectionView` with `first`/`last`/`next`/`previous` links and
`totalItems`. Most tuning is done with `pagination*` attributes on `#[ApiResource]`
or a single operation — operation-level wins over resource-level wins over global
config.

## Items per page

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(paginationItemsPerPage: 20)]            // resource default
#[GetCollection(paginationItemsPerPage: 100)]         // override one operation
class Book {}
```

Global default:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        pagination_items_per_page: 30
```

## Letting the client choose the page size

Off by default (a client could otherwise ask for a million rows). Opt in, then cap
it with `paginationMaximumItemsPerPage` so the client can't DoS the database:

```php
#[ApiResource(
    paginationClientItemsPerPage: true,
    paginationMaximumItemsPerPage: 100,
)]
class Book {}
```

`GET /books?itemsPerPage=50` now works (clamped to 100). The parameter name is
`itemsPerPage` by default; change it under `collection.pagination.items_per_page_parameter_name`.

## Letting the client toggle pagination on/off

```php
#[ApiResource(paginationClientEnabled: true)]
class Book {}
```

`GET /books?pagination=false` returns the full collection. The value goes through
PHP's `FILTER_VALIDATE_BOOLEAN`, so `false`/`0`/`no` all disable it.

## Disabling pagination

Per resource or per operation:

```php
#[ApiResource(paginationEnabled: false)]
#[GetCollection(paginationEnabled: false)]
class Book {}
```

Globally under `defaults.pagination_enabled: false`. Only do this for collections
you know stay small — an unbounded collection is an availability risk.

## Partial pagination

Default pagination issues a `COUNT` query to compute `totalItems` and the `last`
page. On huge tables that COUNT dominates the request. Partial pagination skips it:
you lose `totalItems` and `last`, but keep a `next` link (computed by fetching one
extra row). This is the right mode for "infinite scroll" UIs that only need "is
there more?".

```php
#[ApiResource(paginationPartial: true)]
#[GetCollection(paginationPartial: true)]
class Book {}
```

Client-controlled variant: `paginationClientPartial: true` enables
`GET /books?partial=true`.

## Cursor-based pagination

Page-based pagination drifts when rows are inserted/deleted between requests (an
item can appear twice or be skipped). Cursor-based pagination anchors on a unique,
ordered field instead. It requires partial pagination plus a `RangeFilter` and an
`OrderFilter` on the cursor field:

```php
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;

#[ApiResource(
    paginationPartial: true,
    paginationViaCursor: [
        ['field' => 'id', 'direction' => 'DESC'],
    ],
)]
#[ApiFilter(RangeFilter::class, properties: ['id'])]
#[ApiFilter(OrderFilter::class, properties: ['id' => 'DESC'])]
class Book {}
```

The response's `view` then contains cursor links (`id[lt]=…`) instead of `page=N`.
Use the ODM filter namespace (`ApiPlatform\Doctrine\Odm\Filter\…`) for MongoDB.

## Doctrine ORM paginator tuning

The ORM `PaginationExtension` inspects the `QueryBuilder` to decide two Doctrine
Paginator settings. Override the guesses when the query is unusual:

- `paginationFetchJoinCollection` — set `true` when the query joins a
  collection-valued association (Doctrine then runs an extra query to count
  distinct roots correctly). API Platform usually detects this; force it when a
  `to-many` join produces a wrong `totalItems`.
- `paginationUseOutputWalkers` — output walkers are required for some queries
  (e.g. `HAVING`); they cost performance. Set `false` to disable when you know the
  query is simple, `true` when a complex query throws.

```php
#[ApiResource(paginationFetchJoinCollection: false)]
#[GetCollection(name: 'with_join', paginationFetchJoinCollection: true)]
class Book {}
```

## Pagination in custom state providers

The Doctrine/Eloquent providers paginate for you. A **custom** provider (see
**state-provider**) must return a paginator itself, or the collection comes back
unpaginated with no Hydra `view`. Return one of:

- `ApiPlatform\State\Pagination\ArrayPaginator` — you already have all results in
  memory and want a page sliced out.
- `ApiPlatform\State\Pagination\TraversablePaginator` — you fetched exactly the
  current page (e.g. an external API that paginates server-side) and know the
  totals.

`ArrayPaginator` takes the full array plus offset/limit:

```php
use ApiPlatform\State\Pagination\ArrayPaginator;
use ApiPlatform\State\Pagination\Pagination;
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Metadata\Operation;

final class BookProvider implements ProviderInterface
{
    public function __construct(private Pagination $pagination) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): ArrayPaginator
    {
        $results = $this->fetchEverything();
        [$page, $offset, $limit] = $this->pagination->getPagination($operation, $context);

        return new ArrayPaginator($results, $offset, $limit);
    }
}
```

`TraversablePaginator` is for when you only hold one page and know the totals — its
constructor is `(\Traversable $items, float $currentPage, float $itemsPerPage, float $totalItems)`:

```php
use ApiPlatform\State\Pagination\TraversablePaginator;

return new TraversablePaginator(
    new \ArrayIterator($pageItems),
    currentPage: $page,
    itemsPerPage: $limit,
    totalItems: $remoteTotalCount,
);
```

Inject `ApiPlatform\State\Pagination\Pagination` to resolve the effective page,
offset and limit from the operation + request — don't read `?page` off the request
yourself, or you'll bypass the resource's `paginationItemsPerPage`/client toggles.

Both classes also implement `HasNextPagePaginatorInterface` (`hasNextPage()`), so a
partial-style `next` link works without a total. Implement
`PartialPaginatorInterface` directly only if you genuinely can't compute totals.

## Laravel

The `pagination*` attributes (`paginationItemsPerPage`, `paginationClientItemsPerPage`,
`paginationMaximumItemsPerPage`, `paginationEnabled`, `paginationPartial`,
`paginationClientEnabled`/`…Partial`) and the Hydra view are the same. Differences:

- Global defaults live in `config/api-platform.php` under `defaults` and `pagination`
  (`pagination_items_per_page`, `page_parameter_name`, `items_per_page_parameter_name`,
  etc.) — not YAML.
- The Eloquent `CollectionProvider` paginates with `paginate()` (full) or
  `simplePaginate()` (partial), returning `ApiPlatform\Laravel\Eloquent\Paginator` /
  `PartialPaginator`. Custom Eloquent providers can still return the framework-neutral
  `ArrayPaginator` / `TraversablePaginator`.
- **Doctrine-only, no Eloquent equivalent:** `paginationViaCursor` (cursor pagination)
  and the ORM paginator tuning (`paginationFetchJoinCollection`,
  `paginationUseOutputWalkers`).

## Checklist

- [ ] Default pagination left on for unbounded collections
- [ ] `paginationClientItemsPerPage` paired with `paginationMaximumItemsPerPage`
- [ ] `paginationPartial` used where the `COUNT` query is the bottleneck
- [ ] Cursor pagination has both `RangeFilter` and `OrderFilter` on the cursor field
- [ ] Custom providers return `ArrayPaginator`/`TraversablePaginator`, not a bare array
- [ ] Page/offset/limit read via injected `Pagination`, not the raw request
