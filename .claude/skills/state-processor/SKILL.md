---
name: state-processor
description: Creates state processors for writing data in API Platform. Use when implementing custom persistence, soft-delete, file downloads, side effects like sending emails, creating related entities, or handling business logic on write operations.
---

# Creating State Processors

State Processors handle persistence and side effects for write operations (Post, Patch, Put, Delete).

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
