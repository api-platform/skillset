---
name: errors
description: "Handles errors in API Platform — RFC 7807 Problem Details responses, mapping exceptions to HTTP status (exception_to_status, per-operation exceptionToStatus), #[ErrorResource] domain error resources, the 422 validation violations shape, and safely exposing/hiding exception messages. Use when the user mentions error responses, problem+json, HTTP status codes for exceptions, custom error formats, domain exceptions from a provider/processor, 404/409/422 mapping, validation error output, or leaking internal error messages."
---

# Error Handling

API Platform converts every thrown exception into a structured error response. With
`rfc_7807_compliant_errors` (the default since 3.1) errors are emitted as
**RFC 7807 Problem Details** — `application/problem+json` for JSON clients, the
Hydra error shape for JSON-LD. You rarely build error bodies by hand; you control
the **status code** and how much of the **message** is exposed.

## How the status code is chosen

For a thrown exception, API Platform resolves the status in this order (Symfony):

1. `exception_to_status` mapping (global or per-resource/operation) — first match wins
2. Symfony `HttpExceptionInterface` → its status
3. `ApiPlatform\Metadata\Exception\ProblemExceptionInterface` with a status → use it
4. `ApiPlatform\Metadata\Exception\HttpExceptionInterface` → its status
5. Defaults: `RequestExceptionInterface` → 400, `ValidationException` → 422
6. The status declared on an `#[ErrorResource]`
7. Fallback → 500

## Mapping exceptions to status

Throw a plain domain exception from your business layer, then map it. Global config:

```yaml
# config/packages/api_platform.yaml
api_platform:
    exception_to_status:
        # keep the defaults — removing them changes built-in behavior
        Symfony\Component\Serializer\Exception\ExceptionInterface: 400
        ApiPlatform\Validator\Exception\ValidationException: 422
        # your mappings
        App\Exception\ProductNotFoundException: 404
        Doctrine\ORM\OptimisticLockException: 409
```

Per resource / per operation with `exceptionToStatus` — operation overrides
resource overrides global:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use App\Exception\ProductNotFoundException;
use App\Exception\ProductWasRemovedException;

#[ApiResource(
    exceptionToStatus: [ProductNotFoundException::class => 404],
    operations: [
        new Get(exceptionToStatus: [ProductWasRemovedException::class => 410]),
    ],
)]
class Product {}
```

> A plain exception mapped this way is **flattened** — API Platform turns it into an
> `HttpException` and loses any custom properties. To keep structured fields, use an
> `#[ErrorResource]` (below) instead of a bare exception + mapping.

## Domain exceptions thrown from providers/processors

The natural place to raise these is inside a state provider or processor (see
**state-provider** / **state-processor**) — e.g. a processor that rejects a write
because a business invariant fails:

```php
use ApiPlatform\State\ProcessorInterface;
use ApiPlatform\Metadata\Operation;
use App\Exception\InsufficientStockException;

final class OrderProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        if ($data->quantity > $this->stock->available($data->sku)) {
            throw new InsufficientStockException();  // mapped to 409 via exceptionToStatus
        }
        // ... persist
    }
}
```

## Message scope: don't leak internals

The message handling is status-dependent and is a real security control:

- status **>= 500**: the message is shown only in dev/test. In production a generic
  message replaces it — so a `RuntimeException("DB host db-prod-3 refused")` never
  reaches clients.
- status **< 500**: your message **is** sent to the client.

So a domain `404`/`409`/`422` message is client-visible by design — write those
messages for end users, and never put internal detail in a sub-500 exception.

## #[ErrorResource]: structured domain errors

For full control over the body (extra fields, a stable `type` URI), make the
exception an Error resource. It must implement
`ApiPlatform\Metadata\Exception\ProblemExceptionInterface` and carry `#[ErrorResource]`:

```php
use ApiPlatform\Metadata\ErrorResource;
use ApiPlatform\Metadata\Exception\ProblemExceptionInterface;

#[ErrorResource]
class InsufficientStockException extends \Exception implements ProblemExceptionInterface
{
    public function getType(): string    { return '/errors/insufficient-stock'; }
    public function getTitle(): ?string  { return 'Insufficient stock'; }
    public function getStatus(): ?int    { return 409; }
    public function getDetail(): ?string { return $this->getMessage(); }
    public function getInstance(): ?string { return null; }

    public int $availableQuantity = 0;   // extra field, serialized into the body
}
```

By default the normalization context strips `trace`, `file`, `line`, `code`,
`message`, `traceAsString` so a stack trace can't leak through an Error resource.
Override that context only if you deliberately want one of those fields.

## Documenting domain errors in OpenAPI

Link an Error resource to an operation with `errors:` so it appears as a documented
response in the OpenAPI/Swagger UI (the `errors` param is typed
`class-string<ProblemExceptionInterface>[]`):

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use App\Exception\InsufficientStockException;

#[ApiResource(operations: [
    new GetCollection(errors: [InsufficientStockException::class]),
])]
class Order {}
```

See **api-docs** for broader OpenAPI customization.

## Validation errors (422)

A failed Symfony validation throws `ApiPlatform\Validator\Exception\ValidationException`,
mapped to **422** with a `violations` array — each entry pairs the offending
`propertyPath` with a human `message`:

```json
{
    "@context": "/contexts/ConstraintViolationList",
    "@type": "ConstraintViolationList",
    "status": 422,
    "violations": [
        { "propertyPath": "email", "message": "This value is not a valid email address." }
    ]
}
```

To report *every* malformed field in one 422 instead of failing on the first type
mismatch, enable `collect_denormalization_errors` (see **operations**). Validation
constraints and groups themselves are covered by **custom-validator** and
**operations**.

## Customizing the generic error provider

To reshape *all* non-validation errors, decorate `api_platform.state.error_provider`
and build the body from `ApiPlatform\State\ApiResource\Error`:

```php
use ApiPlatform\Metadata\HttpOperation;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ApiResource\Error;
use ApiPlatform\State\ProviderInterface;
use Symfony\Component\DependencyInjection\Attribute\AsAlias;

#[AsAlias('api_platform.state.error_provider')]
final class ErrorProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object
    {
        $exception = $context['request']->attributes->get('exception');
        $status = $operation instanceof HttpOperation ? ($operation->getStatus() ?? 500) : 500;

        $error = Error::createFromException($exception, $status);
        if ($status >= 500) {
            $error->setDetail('Something went wrong');   // belt-and-suspenders against leaks
        }

        return $error;
    }
}
```

## Laravel

The RFC 7807 output, the 422 `violations` shape, `#[ErrorResource]` +
`ProblemExceptionInterface`, and per-resource/operation `exceptionToStatus` are all
framework-neutral. The differences are configuration and the global error provider:

- The **global** `exception_to_status` map lives in `config/api-platform.php` (PHP
  array), not YAML. It ships with `Illuminate\Auth\AuthenticationException => 401` and
  `Illuminate\Auth\Access\AuthorizationException => 403` — keep these when adding your
  own.
- `'error_handler' => ['extend_laravel_handler' => true]` lets API Platform format
  errors raised through Laravel's own handler.
- To reshape all non-validation errors, bind your own provider for the error operation
  in a service provider instead of the Symfony `#[AsAlias('api_platform.state.error_provider')]`
  decoration shown above; the `ApiPlatform\State\ApiResource\Error` body is the same.

## Checklist

- [ ] Domain exceptions mapped via `exceptionToStatus` (operation > resource > global)
- [ ] Default `exception_to_status` entries kept when adding custom ones
- [ ] Structured errors use `#[ErrorResource]` + `ProblemExceptionInterface`, not flattened exceptions
- [ ] No internal detail in sub-500 exception messages (those reach clients)
- [ ] Domain errors documented on operations via `errors:`
- [ ] Validation relies on the built-in 422 `violations` shape, not a custom one
