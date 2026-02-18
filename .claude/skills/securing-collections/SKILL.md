---
name: securing-collections
description: Secures API Platform collections with Doctrine extensions and link handlers. Use when implementing multi-tenant data isolation, filtering soft-deleted records, restricting queries by organization/user, or validating parent resource ownership in nested URIs.
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

### MongoDB ODM Extension

```php
<?php
namespace App\Extension;

use ApiPlatform\Doctrine\Odm\Extension\AggregationCollectionExtensionInterface;
use ApiPlatform\Doctrine\Odm\Extension\AggregationItemExtensionInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ODM\MongoDB\Aggregation\Builder;

final class AccountExtension implements AggregationCollectionExtensionInterface, AggregationItemExtensionInterface
{
    public function __construct(private readonly UserHelper $userHelper) {}

    public function applyToCollection(Builder $aggregationBuilder, string $resourceClass, ?Operation $operation = null, array &$context = []): void
    {
        $this->applyFilters($aggregationBuilder, $resourceClass);
    }

    public function applyToItem(Builder $aggregationBuilder, string $resourceClass, array $identifiers, ?Operation $operation = null, array &$context = []): void
    {
        $this->applyFilters($aggregationBuilder, $resourceClass);
    }

    private function applyFilters(Builder $aggregationBuilder, string $resourceClass): void
    {
        if (Account::class !== $resourceClass) {
            return;
        }

        $user = $this->userHelper->getUser();
        if (!$user) {
            return;
        }

        $aggregationBuilder
            ->match()
            ->field('isDeleted')->equals(false)
            ->field('organization')->equals($user->getCurrentOrganization()->getId());
    }
}
```

Extensions are auto-registered by Symfony's autowiring. No service tag needed.

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
