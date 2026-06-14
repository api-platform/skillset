---
name: operations
description: "Configures API Platform operations — security expressions, validation groups, denormalization error collection, parameter validation and parameter-level security, deprecation headers, and nested PATCH. Use whenever the user wants to restrict who can call an endpoint, vary validation between create and update, validate query/header parameters, deprecate an endpoint, or debug merge-patch on nested resources — including plain 'protect this endpoint' or 'only admins can X' requests."
---

# Operations: Security, Validation & Lifecycle

Operation-level configuration that sits between the resource shape (see
**api-resource**) and the read/write logic (see **state-provider** /
**state-processor**).

## Read & write phases (the provider/processor lifecycle)

Every operation runs two phases, each toggled by a boolean flag:

| Phase | Flag | Runs | Maps to (CQRS) |
|---|---|---|---|
| **Read** | `read` | a **provider** fetches the data | Query |
| **Write** | `write` | a **processor** persists / acts on the data | Command |

A full request is `read → deserialize → validate → write → serialize`. The provider
supplies the object the rest of the chain operates on; the processor performs the
side effect (persist, delete, send mail, build a file). This is API Platform's CQRS
split: **GET-style reads go through a provider, state changes go through a
processor.**

### The flags default from the request, not the verb

You leave `read`/`write` unset and API Platform resolves them at runtime from the
HTTP request (see `MainController` / `WriteListener` in core):

- **`write` defaults to "the method is not safe"** — `false` for `GET`/`HEAD`,
  `true` for `POST`, `PUT`, `PATCH`, `DELETE`.
- **`read` defaults to "the operation has URI variables, or the method is safe"** —
  so item operations (which carry `{id}`) and all `GET`s read, but a collection
  `POST` (no URI variables, unsafe) does **not**.

| Operation | `read` | `write` |
|---|---|---|
| `Get` (item) | `true` *(has `{id}`)* | **`false`** |
| `GetCollection` | `true` *(safe)* | **`false`** |
| `Post` (collection) | **`false`** *(no URI vars, unsafe)* | `true` |
| `Put`, `Patch` (item) | `true` *(has `{id}`)* | `true` |
| `Delete` (item) | `true` *(has `{id}`)* | `true` |

So a `Get` skips the processor by default, and a collection `Post` runs a processor
without a provider. Override either flag to decouple the phase from this default:

```php
// Run a processor on a GET (file download, report generation, counter bump):
new Get(
    uriTemplate: '/orders/{id}/download',
    write: true,                 // turn the write phase ON — processor now fires
    processor: OrderDownloadProcessor::class,
)

// Skip the built-in fetch on an item write (upsert: processor handles a missing row):
new Put(
    read: false,                 // no provider runs; $data comes from deserialization only
    processor: UpsertProcessor::class,
)
```

`write: true` is the common case: a `Get` whose `write` defaults to `false` would
**never invoke its `processor`** — flipping the flag is what enables it. Conversely
`read: false` stops the built-in provider on an item operation from a needless fetch.

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

Each parameter carries its own `security` expression, where the parameter name
becomes a variable bound to the submitted value. Declare them in `parameters` as
`QueryParameter` or `HeaderParameter`:

```php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\HeaderParameter;
use ApiPlatform\Metadata\QueryParameter;
use Symfony\Component\TypeInfo\Type\BuiltinType;
use Symfony\Component\TypeInfo\TypeIdentifier;

#[GetCollection(
    parameters: [
        'name' => new QueryParameter(security: 'is_granted("ROLE_ADMIN")'),
        'auth' => new HeaderParameter(security: '"secured" == auth', nativeType: new BuiltinType(TypeIdentifier::STRING)),
        'secret' => new QueryParameter(security: '"secured" == secret', nativeType: new BuiltinType(TypeIdentifier::STRING)),
    ],
)]
```

`?name=foo` evaluates `is_granted("ROLE_ADMIN")` (403 otherwise); the `auth` header
or `?secret=…` must equal `secured` or the request is rejected. The parameter is
only checked when present — an absent or value-less parameter passes.

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

Each malformed field becomes a `violations` entry with a `propertyPath`, a
`This value should be of type …` message, and a `hint` explaining the failure.

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

`deprecationReason` emits a `Deprecation` header and `sunset` a `Sunset` header. To
also advertise a migration doc, add an explicit operation `links:` entry —
`new Link('deprecation', 'https://…')` renders `Link: <…>; rel="deprecation"`.

## Nested PATCH gotcha

With `application/merge-patch+json`, to **update** an existing nested resource you
must include its identifier. Omitting it makes the serializer treat the nested
object as new (and attempt to create it):

```json
{ "shippingAddress": { "id": 12, "city": "Lyon" } }
```
Without `"id"`, API Platform tries to create a new `Address` rather than patch #12.

## Laravel

Per-operation metadata (`security`, `validationContext` groups, `collect_denormalization_errors`,
`deprecationReason`/`sunset`, parameter validation/security) is mostly shared, but
auth and validation wiring differ:

- **Authorization** integrates with Laravel **policies**, not Symfony voters. Once a
  policy exists, API Platform auto-maps operations to methods: GET collection →
  `viewAny`, GET → `view`, POST → `create`, PATCH/PUT → `update` (PUT → `create` if
  absent), DELETE → `delete`. Override the mapping with a `policy:` property on the
  operation: `new Patch(policy: 'myCustomPolicy')`. The `security` ExpressionLanguage
  string also works.
- **Authentication / middleware** is attached with the Laravel `middleware:` property
  per operation (`new Patch(middleware: 'auth:sanctum')`) or globally under
  `defaults.middleware` in `config/api-platform.php`.
- **Validation** uses Laravel `rules` (array / closure / `FormRequest`) per resource
  or operation rather than Symfony `validationContext` groups — see the
  **custom-validator** Laravel section. `AuthenticationException` → 401 and
  `AuthorizationException` → 403 are mapped by default in the config's
  `exception_to_status`.
- Query/header `Parameter` `constraints` are Laravel validation rule strings (e.g.
  `'min:2'`), not Symfony constraints.

## Checklist

- [ ] Write operations restricted with `security` / `securityPostDenormalize`
- [ ] Collection isolation handled by an extension, not `security` (see securing-collections)
- [ ] `validationContext` groups split create vs update constraints
- [ ] `collect_denormalization_errors` enabled where clients need full error lists
- [ ] Query parameters carry validation constraints
- [ ] Deprecated endpoints set `deprecationReason` + `sunset`
