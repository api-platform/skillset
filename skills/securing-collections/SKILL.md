---
name: securing-collections
description: "Secures API Platform collections with Doctrine extensions and link handlers. Use whenever the user mentions multi-tenant isolation, 'users must only see their own data', soft-delete filtering, scoping queries by organization or user, or validating parent ownership in nested URIs — even if they frame it as a security or privacy bug rather than a query concern."
---

# Securing Collections

Two mechanisms restrict what data users can access at the query level.

## Doctrine Extensions (Global Query Filters)

Extensions automatically modify every query for a resource class. Use them for:
- Multi-tenant isolation (filter by organization/user)
- Hiding soft-deleted records
- Any filter that must apply to ALL queries (item + collection)

### ORM Extension

```php
<?php
namespace App\Extension;

use ApiPlatform\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
use ApiPlatform\Doctrine\Orm\Extension\QueryItemExtensionInterface;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;

final class AccountExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
{
    public function __construct(private UserHelper $userHelper) {}

    public function applyToCollection(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        $this->addWhere($qb, $resourceClass);
    }

    public function applyToItem(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, ?Operation $operation = null, array $context = []): void
    {
        $this->addWhere($qb, $resourceClass);
    }

    private function addWhere(QueryBuilder $qb, string $resourceClass): void
    {
        if (Account::class !== $resourceClass) {
            return;
        }

        $user = $this->userHelper->getUser();
        if (!$user) {
            return;
        }

        $rootAlias = $qb->getRootAliases()[0];
        $qb->andWhere(sprintf('%s.organization = :org', $rootAlias))
           ->andWhere(sprintf('%s.isDeleted = :deleted', $rootAlias))
           ->setParameter('org', $user->getCurrentOrganization())
           ->setParameter('deleted', false);
    }
}
```

Extensions are auto-registered by Symfony's autowiring. No service tag needed.

### MongoDB ODM Extension

Same structure as the ORM version; only the interfaces, the apply-method
signatures, and the query API differ. Implement
`AggregationCollectionExtensionInterface` / `AggregationItemExtensionInterface`
(from `ApiPlatform\Doctrine\Odm\Extension`), and build the filter on a
`Doctrine\ODM\MongoDB\Aggregation\Builder` instead of a `QueryBuilder`:

```php
public function applyToCollection(Builder $aggregationBuilder, string $resourceClass, ?Operation $operation = null, array &$context = []): void
{
    $this->applyFilters($aggregationBuilder, $resourceClass);
}

// applyToItem adds `array $identifiers` after $resourceClass; both delegate here:
private function applyFilters(Builder $aggregationBuilder, string $resourceClass): void
{
    if (Account::class !== $resourceClass) {
        return;
    }
    $user = $this->userHelper->getUser();
    if (!$user) {
        return;
    }

    $aggregationBuilder->match()
        ->field('isDeleted')->equals(false)
        ->field('organization')->equals($user->getCurrentOrganization()->getId());
}
```

The ODM apply methods take `string $resourceClass` directly (no
`QueryNameGenerator`) and receive `$context` by reference.

## Link Handlers (Nested Resource Security)

Link handlers validate parent resource ownership for nested URIs like `/accounts/{accountId}/mailboxes/{mailboxId}/messages`.

### MongoDB ODM Link Handler

```php
<?php
namespace App\Extension;

use ApiPlatform\Doctrine\Odm\State\LinksHandlerInterface;
use Doctrine\ODM\MongoDB\Aggregation\Builder;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

class MessageLinkHandler implements LinksHandlerInterface
{
    public function __construct(
        private readonly UserHelper $userHelper,
        private readonly MailboxRepository $mailboxRepository,
    ) {}

    public function handleLinks(Builder $aggregationBuilder, array $uriVariables, array $context): void
    {
        $user = $this->userHelper->getUser() ?? throw new AccessDeniedHttpException();

        // Validate parent resource exists
        $mailbox = $this->mailboxRepository->find($uriVariables['mailboxId'])
            ?? throw new NotFoundHttpException();

        // Validate parent belongs to correct account
        if ($mailbox->account->getId() !== ($uriVariables['accountId'] ?? null)) {
            throw new NotFoundHttpException();
        }

        // Validate user has access to this resource tree
        if ($mailbox->organization?->getId() !== $user->getCurrentOrganization()?->getId()) {
            throw new AccessDeniedHttpException();
        }

        // Apply query filters
        $aggregationBuilder->match()
            ->field('isDeleted')->equals(false)
            ->field('mailbox')->equals($mailbox);

        // Filter individual item if ID provided
        if (isset($uriVariables['id'])) {
            $aggregationBuilder->match()
                ->field('id')->equals($uriVariables['id']);
        }
    }
}
```

### Assigning to Resource

```php
use ApiPlatform\Doctrine\Odm\State\Options;

#[ApiResource(
    operations: [
        new Get(
            uriTemplate: '/accounts/{accountId}/mailboxes/{mailboxId}/messages/{id}',
            uriVariables: ['accountId', 'mailboxId', 'id'],
            stateOptions: new Options(
                handleLinks: MessageLinkHandler::class,
                documentClass: Message::class,
            ),
        ),
    ]
)]
```

## When to Use Which

| Pattern | Use Case |
|---|---|
| **Extension** | Global filters: multi-tenant isolation, soft-delete, applies to ALL queries for a resource |
| **Link Handler** | Nested resources: validate parent ownership, scope child queries to parent |
| **`security` attribute** | Per-operation checks on already-fetched objects: `security: 'object.user == user'` |

These mechanisms stack: an extension filters the query, a link handler scopes it to the parent, and `security` validates the final object.

## Laravel (Eloquent)

The two mechanisms exist on Laravel too, but there are **no Doctrine extensions**.
The global-query-filter equivalent is `ApiPlatform\Laravel\Eloquent\Extension\QueryExtensionInterface`
— one `apply()` method covering both item and collection (the Eloquent providers run
it for every query):

```php
<?php
namespace App\Extension;

use ApiPlatform\Laravel\Eloquent\Extension\QueryExtensionInterface;
use ApiPlatform\Metadata\Operation;
use Illuminate\Database\Eloquent\Builder;

final class AccountExtension implements QueryExtensionInterface
{
    public function __construct(private readonly UserHelper $userHelper) {}

    public function apply(Builder $builder, array $uriVariables, Operation $operation, $context = []): Builder
    {
        if (Account::class !== $operation->getClass() || !($user = $this->userHelper->getUser())) {
            return $builder;
        }

        return $builder
            ->where('organization_id', $user->getCurrentOrganization()->id)
            ->where('is_deleted', false);
    }
}
```

Tag it so the providers pick it up by binding to the `QueryExtensionInterface` tag in
a service provider (`$this->app->tag([AccountExtension::class], QueryExtensionInterface::class)`);
there is no Symfony autowiring.

For nested-resource ownership, implement
`ApiPlatform\Laravel\Eloquent\State\LinksHandlerInterface` —
`handleLinks(Builder $builder, array $uriVariables, array $context): Builder` —
returning the scoped builder (it operates on an Eloquent `Builder`, not an aggregation
pipeline). Assign it with the Eloquent `Options`:

```php
use ApiPlatform\Laravel\Eloquent\State\Options;

new Get(
    uriTemplate: '/accounts/{accountId}/messages/{id}',
    uriVariables: ['accountId', 'id'],
    stateOptions: new Options(handleLinks: MessageLinkHandler::class, modelClass: Message::class),
)
```

Note Eloquent `Options` uses `modelClass:` (not `documentClass:`/`entityClass:`).
Per-operation `security` expressions and Laravel **policies** (`viewAny`/`view`/
`create`/`update`/`delete`, or an explicit `policy:` on an operation) layer on top —
see the **operations** skill's Laravel section.
