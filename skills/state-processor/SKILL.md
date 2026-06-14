---
name: state-processor
description: "Creates state processors for write operations in API Platform — the write/Command side of its CQRS-style provider/processor split. Use whenever a POST/PUT/PATCH/DELETE needs custom behavior — persistence logic, soft-delete, file downloads, side effects like emails or events, creating related entities, hashing passwords — or any 'when X is created/updated/deleted, do Y' request, or to understand why a GET processor doesn't fire without write: true (the write vs read phase), even if the user doesn't say 'processor'."
---

# Creating State Processors

State Processors handle persistence and side effects for write operations (Post,
Patch, Put, Delete) — the **write phase** of an operation, the **Command** side of
API Platform's CQRS-style split (a **state-provider** handles the **Query**/read
side).

A processor runs when the operation's `write` flag is `true`. You normally leave it
unset and API Platform resolves it from the request: `write` defaults to *"the HTTP
method is not safe"*, and `read` (the provider) to *"the operation has URI
variables, or the method is safe"*.

| Operation | processor runs (`write`) | provider runs (`read`) |
|---|---|---|
| `Get` (item) / `GetCollection` | **no** | yes |
| `Post` (collection) | yes | no *(no URI vars, unsafe)* |
| `Put`, `Patch`, `Delete` (item) | yes | yes |

A `Get` defaults to `write: false`, so its `processor` **never fires** until you set
`write: true` (file downloads, report generation — see *Returning HTTP Responses*
below). See **operations** → *Read & write phases* for the full `read → deserialize
→ validate → write → serialize` lifecycle.

## Basic Structure

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

/**
 * @implements ProcessorInterface<YourResource, YourResource>
 */
final class YourProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Business logic + persistence
        return $data;
    }
}
```

## Decorating Built-in Processors

Wrap the default processor to add side effects before/after persistence:

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class OrderCreateProcessor implements ProcessorInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine.orm.state.persist_processor')]
        private ProcessorInterface $persistProcessor,
        private MailerInterface $mailer,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Pre-persistence: set relationships, hash passwords, etc.
        $data->setCreatedBy($this->getUser());

        $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);

        // Post-persistence: send notifications, trigger events
        $this->mailer->send(new OrderConfirmation($result));

        return $result;
    }
}
```

### Doctrine service names

| Persistence | Persist Processor | Remove Processor |
|---|---|---|
| **ORM** | `api_platform.doctrine.orm.state.persist_processor` | `api_platform.doctrine.orm.state.remove_processor` |
| **MongoDB ODM** | `api_platform.doctrine_mongodb.odm.state.persist_processor` | `api_platform.doctrine_mongodb.odm.state.remove_processor` |

> **Laravel (Eloquent):** `ProcessorInterface` is identical and resources reference a
> processor by class-string (`processor: YourProcessor::class`). The built-in Eloquent
> processors are bound in the container by class name —
> `ApiPlatform\Laravel\Eloquent\State\PersistProcessor` (handles create/update,
> including BelongsTo/HasMany relations and `standard_put`) and
> `…\State\RemoveProcessor`. To decorate, type-hint that concrete class in your
> constructor instead of using a `#[Autowire(service: 'api_platform.doctrine.*')]`
> attribute. Side-effect patterns (mailers, events) work the same via Laravel's
> container. Returning a Symfony `Response` from a processor is also supported.

## Creating Related Entities in a Processor

Create child resources alongside the main resource:

```php
final class AccountCreateProcessor implements ProcessorInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine_mongodb.odm.state.persist_processor')]
        private ProcessorInterface $persistProcessor,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        $this->hashPassword($data);

        // Create related entities before persisting
        foreach (['INBOX', 'Sent', 'Trash', 'Drafts'] as $name) {
            $mailbox = new Mailbox();
            $mailbox->path = $name;
            $data->addMailbox($mailbox); // cascade: ['persist']
        }

        // Single persist call saves parent + children
        return $this->persistProcessor->process($data, $operation, $uriVariables, $context);
    }
}
```

## Soft-Delete Processor

Reusable processor for soft-deleting resources:

```php
/**
 * @implements ProcessorInterface<Account|Domain, Account|Domain>
 */
readonly class SoftDeleteProcessor implements ProcessorInterface
{
    public function __construct(private DocumentManager $dm) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        $data->isDeleted = true;
        $this->dm->persist($data);
        $this->dm->flush();

        return $data;
    }
}
```

Assign to Delete operations:

```php
new Delete(processor: SoftDeleteProcessor::class)
```

Combine with a collection extension that filters `isDeleted = false` to hide soft-deleted records from queries (see **securing-collections** skill).

## Returning HTTP Responses (File Downloads)

Processors can return a Symfony `Response` for binary content:

```php
use Symfony\Component\HttpFoundation\HeaderUtils;
use Symfony\Component\HttpFoundation\Response;

/**
 * @implements ProcessorInterface<Message, Response>
 */
final class DownloadProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): Response
    {
        $content = $this->buildContent($data);
        $response = new Response($content);
        $response->headers->set('Content-Type', 'application/octet-stream');
        $response->headers->set('Content-Disposition',
            HeaderUtils::makeDisposition(HeaderUtils::DISPOSITION_ATTACHMENT, $data->getId() . '.bin')
        );

        return $response;
    }
}
```

Use with a Get operation that has `write: true`:

```php
new Get(
    uriTemplate: '/orders/{id}/download',
    write: true,
    processor: DownloadProcessor::class,
    name: '_api_order_download',
)
```

## Handling Both Persist and Delete

When a single processor handles Create/Update AND Delete, inject both services:

```php
public function __construct(
    #[Autowire(service: 'api_platform.doctrine.orm.state.persist_processor')]
    private ProcessorInterface $persistProcessor,
    #[Autowire(service: 'api_platform.doctrine.orm.state.remove_processor')]
    private ProcessorInterface $removeProcessor,
) {}

public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
{
    if ($operation instanceof DeleteOperationInterface) {
        return $this->removeProcessor->process($data, $operation, $uriVariables, $context);
    }

    return $this->persistProcessor->process($data, $operation, $uriVariables, $context);
}
```

## Processor Parameters

- **$data**: The deserialized/validated object
- **$operation**: Operation metadata (Post, Patch, Delete, etc.)
- **$uriVariables**: URI path variables
- **$context**: Includes `previous_data` (before modification), `request`, `resource_class`

## Best Practices

1. **Always return data** from processors — omitting breaks serialization
2. Decorate built-in processors rather than replacing them
3. Keep processors focused: one processor per operation when logic differs significantly
4. Use `$context['previous_data']` in Patch/Put to detect what changed
5. Throw `HttpException` subclasses for domain errors (`BadRequestHttpException`, `UnprocessableEntityHttpException`)
