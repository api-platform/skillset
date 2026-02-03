---
name: state-processor
description: Create state processors for writing data in API Platform. Use when implementing custom persistence, sending emails, triggering side effects, or handling business logic on write operations.
---

# Creating State Processors

State Processors handle data persistence and side effects for write operations (Post, Patch, Put, Delete).

## Basic Structure

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

/**
 * @implements ProcessorInterface<YourResource, YourResource|void>
 */
final class YourResourceProcessor implements ProcessorInterface
{
    public function __construct(
        private YourRepository $repository,
    ) {}

    public function process(
        mixed $data,
        Operation $operation,
        array $uriVariables = [],
        array $context = []
    ): mixed {
        // Your persistence/business logic
        $this->repository->save($data);

        return $data;
    }
}
```

## Decorating Built-in Doctrine Processor

Wrap the default processor to add side effects:

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\Mailer\MailerInterface;

final class NotificationProcessor implements ProcessorInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine.orm.state.persist_processor')]
        private ProcessorInterface $persistProcessor,
        private MailerInterface $mailer,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Pre-persistence logic

        $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);

        // Post-persistence side effects
        $this->mailer->send(new ResourceCreatedEmail($result));

        return $result;
    }
}
```

## Processor Parameters

- **$data**: The object to process (after deserialization/validation)
- **$operation**: Metadata about the operation (Post, Patch, Delete, etc.)
- **$uriVariables**: URI path variables
- **$context**: Additional context including `previous_data`

## Common Context Keys

- `request`: The Symfony HTTP request object
- `previous_data`: The data before modifications (useful for Patch/Put)
- `resource_class`: The resource class being operated on

## Built-in Doctrine Services

- `api_platform.doctrine.orm.state.persist_processor` - Persistence
- `api_platform.doctrine.orm.state.remove_processor` - Deletion

## Assigning to Resource

```php
#[ApiResource]
#[Post(processor: YourResourceProcessor::class)]
#[Patch(processor: YourResourceProcessor::class)]
#[Delete(processor: YourResourceProcessor::class)]
class YourResource {}
```

## Best Practices

1. **Always return data** from processors (breaks serialization otherwise)
2. Decorate built-in processors rather than replacing them
3. Keep processors focused on single responsibilities
4. Use event dispatcher for complex side effects
5. Handle errors with appropriate HTTP exceptions

## Error Handling

```php
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;

public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
{
    if (!$this->isValid($data)) {
        throw new BadRequestHttpException('Invalid data');
    }

    return $this->persistProcessor->process($data, $operation, $uriVariables, $context);
}
```

## Reference

For detailed patterns, see:
- [State Management Guide](../../../skills/state.md)
- [State Providers/Processors Guide](../../../skills/state/AGENTS.md)
