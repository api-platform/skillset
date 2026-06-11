---
name: operations
description: Configures API Platform operations — security expressions, validation groups, denormalization error collection, query-parameter validation, deprecation headers, and nested PATCH. Use when restricting access to endpoints, controlling validation per operation, deprecating endpoints, or debugging merge-patch on nested resources.
---

# Operations: Security, Validation & Lifecycle

Operation-level configuration that sits between the resource shape (see
**api-resource**) and the read/write logic (see **state-provider** /
**state-processor**).

## Securing operations

The `security` attribute takes a Symfony ExpressionLanguage string evaluated
before the operation runs. Available variables: `user`, `object` (item operations),
request parameters when explicitly exposed.

### Role-based

```php
#[ApiResource(
    operations: [
        new GetCollection(),
        new Post(security: "is_granted('ROLE_ADMIN')"),
    ]
)]
class Invoice {}
```

### Object-based (item operations)

`object` is the fetched resource — use it for ownership checks on `Get`, `Put`,
`Patch`, `Delete`:

```php
new Get(security: "object.getOwner() == user")
new Patch(security: "object.getOwner() == user or is_granted('ROLE_ADMIN')")
```

`securityPostDenormalize` runs *after* the request body is applied — use it when the
decision depends on incoming values (e.g. preventing privilege escalation on PATCH).

### Parameter-based

Expose a request parameter to the expression with `securityHeader` (header) or by
declaring it in `parameters`:

```php
#[GetCollection(
    securityHeader: 'auth',
    security: "auth == 'secured'",
)]
```
Requires an `auth: secured` request header.

> For query-level data isolation (multi-tenant, soft-delete) that must apply to
> **every** query regardless of operation, use a Doctrine extension or link handler
> instead — see **securing-collections**. `security` only guards an
> already-fetched object; it does not scope collections.

## Validation

API Platform runs Symfony's Validator on the deserialized object before the
processor. Constraints live on the resource/input DTO (see **custom-validator**).

### Validation groups per operation

```php
#[ApiResource(
    operations: [
        new Post(validationContext: ['groups' => ['Default', 'user:create']]),
        new Patch(validationContext: ['groups' => ['Default', 'user:update']]),
    ]
)]
```

```php
#[Assert\NotBlank(groups: ['user:create'])]            // required on create only
#[Assert\Email(groups: ['user:create', 'user:update'])] // checked on both
public ?string $email = null;
```

### Collect denormalization errors

By default a type mismatch in the body throws on the first bad field. Enable
`collect_denormalization_errors` to report every malformed field at once in the 422
`violations` array:

```php
#[Post(validationContext: ['collect_denormalization_errors' => true])]
```
*Pattern: `tests/Functional/.../ValidationTest.php::testPostWithDenormalizationErrorsCollected`.*

### Validating query parameters

Attach constraints to a `QueryParameter` (or `HeaderParameter`); invalid values
yield 422 before the provider runs:

```php
use ApiPlatform\Metadata\QueryParameter;
use Symfony\Component\Validator\Constraints as Assert;

#[GetCollection(
    parameters: [
        'page' => new QueryParameter(constraints: [new Assert\Positive()]),
    ],
)]
```
*Pattern: `tests/Functional/Parameters/ValidationTest.php`.*

## Deprecating endpoints

`deprecationReason` adds a `Deprecation` header; `sunset` adds a `Sunset` header
with the removal date. Apply at resource or operation level.

```php
#[ApiResource(
    deprecationReason: 'Use /v2/invoices instead.',
    sunset: '2026-01-01T00:00:00+00:00',
)]
class Invoice {}
```
*Pattern: `tests/Functional/.../DeprecationHeaderTest.php`.*

## Nested PATCH gotcha

With `application/merge-patch+json`, to **update** an existing nested resource you
must include its identifier. Omitting it makes the serializer treat the nested
object as new (and attempt to create it):

```json
{ "shippingAddress": { "id": 12, "city": "Lyon" } }
```
Without `"id"`, API Platform tries to create a new `Address` rather than patch #12.
*Pattern: `tests/Functional/.../NestedPatchTest.php`.*

## Checklist

- [ ] Write operations restricted with `security` / `securityPostDenormalize`
- [ ] Collection isolation handled by an extension, not `security` (see securing-collections)
- [ ] `validationContext` groups split create vs update constraints
- [ ] `collect_denormalization_errors` enabled where clients need full error lists
- [ ] Query parameters carry validation constraints
- [ ] Deprecated endpoints set `deprecationReason` + `sunset`
