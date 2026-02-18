---
name: custom-validator
description: Creates custom validation constraints for API Platform resources. Use when implementing business rule validation like rate limits, uniqueness checks, domain-specific format validation, or cross-field validation that Symfony's built-in constraints cannot express.
---

# Custom Validation Constraints

Create custom validators when built-in Symfony constraints are insufficient.

## Constraint Class

```php
<?php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;

#[\Attribute(\Attribute::TARGET_PROPERTY | \Attribute::TARGET_METHOD | \Attribute::IS_REPEATABLE)]
class IsValidAccountLimit extends Constraint
{
    public string $message = 'Account limit reached for your current plan.';
    public int $limit = 50;
}
```

## Validator Class

Convention: `{ConstraintName}Validator` in the same namespace.

```php
<?php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

class IsValidAccountLimitValidator extends ConstraintValidator
{
    public function __construct(
        private readonly UserHelper $userHelper,
        private readonly AccountRepository $accountRepository,
    ) {}

    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$constraint instanceof IsValidAccountLimit) {
            throw new \InvalidArgumentException('Unexpected constraint type');
        }

        if (null === $value || '' === $value) {
            return;
        }

        $user = $this->userHelper->getUser();
        $count = $this->accountRepository->countByUser($user->getId());

        if ($count >= $constraint->limit) {
            $this->context->buildViolation($constraint->message)
                ->addViolation();
        }
    }
}
```

## Usage on Properties

```php
use App\Validator\IsValidAccountLimit;
use Symfony\Component\Validator\Constraints as Assert;

class Account
{
    #[Assert\NotBlank(groups: ['account:write'])]
    #[Assert\Email(groups: ['account:write'])]
    #[IsValidAccountLimit(groups: ['account:write'])]
    public ?string $address;
}
```

## Class-Level Constraint (UniqueEntity)

For constraints that validate across multiple fields:

```php
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::IS_REPEATABLE)]
class MongoDBUnique extends Constraint
{
    public array $fields = [];
    public string $message = 'This value is already used.';

    public function getTargets(): string
    {
        return self::CLASS_CONSTRAINT;
    }
}
```

Usage:

```php
#[MongoDBUnique(fields: ['address'], groups: ['account:write', 'account:patch'])]
class Account {}
```

## Validation Groups with Operations

Target validation to specific operations:

```php
#[ApiResource(
    operations: [
        new Post(validationContext: ['groups' => ['Default', 'account:write']]),
        new Patch(validationContext: ['groups' => ['Default', 'account:patch']]),
    ]
)]
```

Apply constraints only in specific groups:

```php
#[Assert\NotBlank(groups: ['account:write'])]    // Required on create only
#[Assert\Email(groups: ['account:write', 'account:patch'])]  // Validated on both
public ?string $address;
```

## Callback Validation

For quick one-off validation on DTOs:

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

class SendMessage
{
    public ?string $text = null;
    public ?string $html = null;

    #[Assert\Callback]
    public function validateContent(ExecutionContextInterface $context): void
    {
        if (null === $this->text && null === $this->html) {
            $context->buildViolation('Either text or html content is required.')
                ->atPath('text')
                ->addViolation();
        }
    }
}
```
