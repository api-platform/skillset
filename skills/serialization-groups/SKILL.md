---
name: serialization-groups
description: "Configures serialization with normalizationContext/denormalizationContext and #[Groups] in API Platform — per-operation contexts, #[Context] per-property overrides, and read/write group naming. Use when the user mentions serialization groups, #[Groups], normalizationContext, denormalizationContext, hiding/exposing fields per operation, write-only or read-only properties, controlling embedded relation depth, or 'why is this field showing up / missing' in the JSON output."
---

# Serialization Groups

> **Prefer one resource class per representation over groups for new designs.**
> Before reaching for groups, consider a DTO split: one class per output shape and
> per input shape, each with a flat, explicit set of properties (see
> **api-resource**). Reasons groups are a poor default:
>
> - **The public shape becomes implicit and scattered.** To know what
>   `GET /books` returns you must mentally union every `#[Groups]` tag across the
>   class hierarchy and cross-reference each operation's `normalizationContext`.
>   A dedicated `BookOutput` DTO states the shape in one place.
> - **They complicate the metadata.** Per-operation contexts, per-property
>   `#[Context]`, and group inheritance interact in ways that are hard to predict
>   and hard to test.
> - **They block streaming serialization.** API Platform's JsonStreamer path needs
>   one static shape per class; a class whose output varies by group can't be
>   streamed and falls back to the slower object normalizer.
>
> Use groups when: you're working in an **existing codebase already built on
> them** (stay consistent), or a DTO split is genuinely overkill (one tiny
> read/write asymmetry on an internal resource). Otherwise, split the class.

## The two contexts

`normalizationContext` governs **reading** (entity → JSON). `denormalizationContext`
governs **writing** (JSON → entity). A property is only serialized/deserialized if
one of its `#[Groups]` is listed in the active context.

```php
use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Attribute\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['book:read']],
    denormalizationContext: ['groups' => ['book:write']],
)]
class Book
{
    #[Groups(['book:read'])]                 // appears in output, ignored on input
    public ?int $id = null;

    #[Groups(['book:read', 'book:write'])]   // read and write
    public string $title = '';

    #[Groups(['book:write'])]                // write-only, e.g. a password
    public ?string $secret = null;
}
```

A property with no group in the active context is invisible: a field present on the
entity but absent from the response is almost always a missing read group.

## Naming convention

Use `resource:operation`-style group names — `book:read`, `book:write`,
`book:item:read`. This keeps groups namespaced per resource so a `read` group on
one class never accidentally pulls properties on another. Pairing a broad
`book:read` with a narrower `book:item:read` lets item operations expose more than
collections (see next section).

## Per-operation contexts

Override the resource-level context on a single operation. Common pattern: a lean
collection, a richer item view.

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Patch;

#[ApiResource(normalizationContext: ['groups' => ['book:read']])]
#[GetCollection]                                                  // book:read only
#[Get(normalizationContext: ['groups' => ['book:read', 'book:item:read']])]
#[Patch(denormalizationContext: ['groups' => ['book:write']])]
class Book
{
    #[Groups(['book:read'])]
    public string $title = '';

    #[Groups(['book:item:read'])]            // only on the single-item GET
    public ?string $description = null;
}
```

## Controlling embedded relations

A relation is embedded as a full object when the related class has a property in a
group shared with the parent's context; otherwise it serializes as an IRI/string.
This is how you choose embed-vs-reference:

```php
#[ApiResource(normalizationContext: ['groups' => ['book:read']])]
class Book
{
    #[Groups(['book:read'])]
    public ?Author $author = null;          // embedded only if Author has a 'book:read' property
}

class Author
{
    #[Groups(['book:read'])]
    public string $name = '';               // this property pulls Author inline
}
```

Drop the `book:read` group from `Author::$name` and `author` collapses back to an
IRI. This implicit coupling is exactly the kind of thing a DTO split makes explicit.

## Per-property context override with #[Context]

`#[Context]` overrides serializer options for one property without a new group —
useful for date formats or switching the normalization group on a single relation
(embed the same related type at two depths):

```php
use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Attribute\Groups;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

class Book
{
    #[Groups(['book:read'])]
    #[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]
    public ?\DateTimeInterface $publishedAt = null;

    #[Groups(['book:read'])]
    #[Context(['normalization' => ['groups' => ['author:summary']]])]
    public ?Author $author = null;          // serialized with author:summary, not book:read
}
```

The `normalization`/`denormalization` keys scope the override to one direction. See
the parallel example in **state-provider** (Per-Property Serialization Context).

## Interaction with other features

- Validation groups are **separate** from serialization groups — don't reuse the
  same names expecting them to align. See **operations** for `validationContext`.
- `security` expressions on properties (`security: "is_granted(...)"`) gate fields
  independently of groups; both must pass.

## Laravel (Eloquent)

API Platform uses the Symfony Serializer on Laravel too, so `normalizationContext` /
`denormalizationContext`, the two-context model, per-operation contexts and embed-vs-IRI
behaviour are identical. The one difference: **Eloquent models have no typed
properties**, so `#[Groups]` can't be attached per property. Declare groups at the
class level via `#[ApiProperty(serialize: …, property: …)]`:

```php
use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Attribute\Groups;

#[ApiResource(normalizationContext: ['groups' => ['book:read']])]
#[ApiProperty(serialize: new Groups(['book:read']), property: 'title')]
#[ApiProperty(serialize: new Groups(['book:read']), property: 'author')]
class Book extends Model {}
```

DTO/`ApiResource` classes (not extending `Model`) use property-level `#[Groups]` as
shown above. The `name_converter` (default `SnakeCaseToCamelCaseNameConverter`) is set
in `config/api-platform.php`.

## Checklist

- [ ] Considered a DTO-per-shape split before adding groups (see api-resource)
- [ ] Group names namespaced per resource (`book:read`, not bare `read`)
- [ ] Write-only secrets carry only the write group
- [ ] Embedded-vs-IRI relations verified against actual JSON, not assumed
- [ ] `#[Context]` used for one-off format/group tweaks instead of a whole new group
