---
name: custom-validator
description: "Creates custom validation constraints for API Platform resources. Use whenever a rule goes beyond built-in Symfony constraints — rate or plan limits, uniqueness checks, domain-specific formats, cross-field or conditional validation, or any 'reject the request when X' business rule on write operations, even if the user doesn't say 'validator'."
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

Target a constraint to specific operations by passing `groups:` (as shown above)
and wiring those groups into each operation's `validationContext` — see
**operations** for the full per-operation setup.

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

## Laravel

Laravel does **not** use Symfony Constraints/`ConstraintValidator`. Validation is
declared with [Laravel validation rules](https://laravel.com/docs/validation) via the
`rules` option on `#[ApiResource]` or per-operation (rules can be an array, a closure,
or a `FormRequest` class-string). The `ValidateProvider` runs them before the
processor and emits the same 422 `ConstraintViolationList` shape.

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Post;

#[ApiResource(rules: ['title' => 'required|min:2'])]
#[Post(rules: ['isbn' => ['required', 'string']])] // per-operation override
class Book extends Model {}
```

For business rules beyond built-in rules, write a Laravel **custom rule** (a class
implementing `Illuminate\Contracts\Validation\ValidationRule`, scaffold with
`php artisan make:rule`) or a closure rule, and reference it in `rules`. A
`FormRequest` class-string is also accepted — its `authorize()`/`rules()` run, and an
`AuthorizationException` maps to 403, `ValidationException` to 422. Validation groups
and Symfony `validationContext` do not apply on Laravel; scope rules per operation
instead. Partial PATCH can relax `required` to `sometimes` via
`partial_patch_validation` in `config/api-platform.php`.
