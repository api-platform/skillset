---
name: graphql
description: Exposes API Platform resources over GraphQL — enabling GraphQL, Query/QueryCollection/Mutation/DeleteMutation operations, security expressions, custom resolvers, Relay cursor pagination, and nested relations. Use when the user mentions GraphQL, a GraphQL schema, queries/mutations, Relay connections, GraphQL playground, resolvers, or asks to expose existing REST resources via GraphQL.
---

# GraphQL

> **Default to REST.** GraphQL trades away things API Platform gives you for free
> over REST: HTTP cache semantics (ETag, Cache-Control, invalidation), one URL per
> resource, simple CDN/proxy caching, and predictable per-operation cost. A single
> GraphQL query can fan out into arbitrarily deep/expensive resolution. Reach for
> GraphQL when it is a **hard client requirement** (e.g. a Relay/Apollo frontend, or
> clients that genuinely need to select fields and avoid round-trips) — not as a
> default. The same resource class can serve both; you don't have to choose
> globally.

## Enabling GraphQL

Install `api-platform/graphql` (`composer require api-platform/graphql`), then it's
on. A `/graphql` endpoint and the GraphiQL playground (`/graphql/graphiql`) appear.
Disable globally or per resource as needed:

```yaml
# config/packages/api_platform.yaml
api_platform:
    graphql:
        enabled: true
        graphiql:
            enabled: true
```

## Declaring GraphQL operations

GraphQL operations live in `graphQlOperations` and are **separate classes** from the
REST ones, under `ApiPlatform\Metadata\GraphQl\`. A resource with no
`graphQlOperations` still gets a default set (item query, collection query, create /
update / delete mutations) once GraphQL is enabled.

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\DeleteMutation;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(graphQlOperations: [
    new Query(),
    new QueryCollection(),
    new Mutation(name: 'create'),
    new Mutation(name: 'update'),
    new DeleteMutation(name: 'delete'),
])]
class Book {}
```

`Mutation` and `DeleteMutation` **require** a `name` — it becomes the GraphQL field
name (`createBook`, `updateBook`, `deleteBook`). `Query` and `QueryCollection` only
need a `name` when you declare more than one of the same kind (e.g. a custom query
alongside the default).

## Security on GraphQL operations

Same ExpressionLanguage as REST (see **operations**), set per GraphQl operation:

```php
new Query(security: "is_granted('ROLE_USER')")
new Mutation(name: 'update', security: "object.getOwner() == user")
new Mutation(name: 'update', securityPostDenormalize: "object.getOwner() == user")
```

Because one GraphQL query can traverse relations, securing only the top-level
operation is not enough — guard the related resources' operations too, or a nested
field becomes an unguarded read path.

## Custom resolvers

Use a resolver when a query/mutation needs logic beyond fetch-by-id. Resolvers are
services implementing one of:

- `QueryItemResolverInterface` — `__invoke(?object $item, array $context): object`
- `QueryCollectionResolverInterface` — `__invoke(iterable $collection, array $context): iterable`
- `MutationResolverInterface` — `__invoke(?object $item, array $context): ?object`

Query arguments arrive in `$context['args']`.

```php
use ApiPlatform\GraphQl\Resolver\QueryItemResolverInterface;

final class BookResolver implements QueryItemResolverInterface
{
    public function __invoke(?object $item, array $context): object
    {
        // $item is the fetched Book (or null if read: false); enrich or replace it
        return $item;
    }
}
```

With Symfony autoconfiguration the resolver is wired automatically. Without it, tag
the service `api_platform.graphql.query_resolver` (or `..._mutation_resolver`). Then
reference it by **class name** on the operation, and set `read: false` when the
resolver should fetch the data itself rather than receiving a hydrated item:

```php
new Query(name: 'recommended', resolver: BookResolver::class, read: false)
new Query(
    name: 'search',
    resolver: BookResolver::class,
    args: [
        'query' => ['type' => 'String!', 'description' => 'Full-text search'],
        'limit' => ['type' => 'Int'],
    ],
)
```

`args` overrides the auto-generated argument set — define it when the query takes
parameters that aren't resource fields.

## Relations and the N+1 trap

GraphQL embeds related resources by selecting nested fields:

```graphql
{
  book(id: "/books/1") {
    title
    author { name }
  }
}
```

Relations resolve through the same providers as REST. A deeply nested query can
trigger many small queries (the classic N+1). API Platform mitigates common cases,
but verify against real queries and add Doctrine joins / a custom collection
provider where a hot path fans out. This open-ended cost is the main reason REST is
the safer default for cache-sensitive APIs.

## Pagination: Relay cursor connections

Collection queries return **Relay-style cursor connections** by default
(`edges { node { ... } cursor }`, `pageInfo`, `totalCount`), driven by `first`/
`after`/`last`/`before` arguments:

```graphql
{
  books(first: 10, after: "endCursor") {
    totalCount
    edges { node { title } cursor }
    pageInfo { endCursor hasNextPage }
  }
}
```

To use simple page-based pagination instead, set `paginationType: 'page'` on the
resource or the `QueryCollection`:

```php
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(graphQlOperations: [
    new QueryCollection(paginationType: 'page'),
])]
class Book {}
```

All the `pagination*` controls from **pagination** (items per page, max, partial)
apply to GraphQL collections too.

## Real-time subscriptions

When a resource has `mercure: true`, GraphQL `subscription` operations push updates
through the Mercure hub — see **mercure**.

## Laravel

GraphQL is supported on Laravel. The operation classes (`Query`, `QueryCollection`,
`Mutation`, `DeleteMutation`), `security` expressions, Relay connections,
`paginationType`, custom resolvers and the N+1 caveats are all the same. Differences:

- Install `composer require api-platform/graphql`, then enable it in
  `config/api-platform.php` under `graphql` (`'enabled' => true`) — not YAML. Depth/
  complexity limits and `graphiql` live in the same config block.
- Resolvers implement the same `QueryItemResolverInterface` /
  `QueryCollectionResolverInterface` / `MutationResolverInterface` and are referenced
  by class-string on the operation; Laravel's container resolves them — there are no
  `api_platform.graphql.*_resolver` tags to apply.
- Real-time `subscription` operations depend on Mercure, which has no Laravel
  integration (see **mercure**), so GraphQL subscriptions are effectively Symfony-only.

## Checklist

- [ ] GraphQL chosen for a real client requirement, not as a REST default
- [ ] Every `Mutation`/`DeleteMutation` has a `name`
- [ ] `security` set on nested resources, not just the entry-point operation
- [ ] Custom resolvers tagged (or autoconfigured) and referenced by class name
- [ ] `read: false` set when the resolver fetches its own data
- [ ] Deep/nested queries checked for N+1; joins added on hot paths
- [ ] `paginationType: 'page'` set only if the client doesn't want Relay connections
