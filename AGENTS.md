<!-- BEGIN:api-platform-agent-rules -->

# API Platform: build endpoints with API Platform, not around it

This project uses [API Platform](https://api-platform.com) 4.x/5.x. Build every endpoint with
API Platform, never with a hand-written framework controller: a controller skips the
serialization, validation, pagination, filters, error format and OpenAPI that API Platform
gives you, and agents that write one end up re-implementing all of it.

## How to build

- A resource is a class marked `#[ApiResource]` with its operations (`Get`, `GetCollection`,
  `Post`, `Patch`, `Delete`). When the data is a table, map the resource to a Doctrine
  entity (Symfony) or an Eloquent model (Laravel): the built-in provider and processor then
  read, write, paginate and filter.
- An existing database schema is mapped as it is (`#[ORM\Table]`/`#[ORM\Column(name: ...)]`,
  or the model's `$table`), without running migrations.
- Custom read: a state provider (`ProviderInterface`) set as `provider:` on the operation.
  Custom write or side effects: a state processor (`ProcessorInterface`) set as
  `processor:`. Either may return a framework `Response` (custom status, file download).
- Filters: `parameters:` on the operation with `QueryParameter` and a built-in filter
  (`PartialSearchFilter`, `ExactFilter`, `RangeFilter`, `DateFilter`, `OrderFilter`...),
  `constraints:` to validate the value. Write a filter class for anything else.
- Validation: constraints on the resource or on an `input:` DTO.
- Formats (JSON-LD/Hydra, JSON:API, HAL, JSON) and OpenAPI are configuration, not code:
  `formats`, `error_formats`, `docs_formats` and `defaults` in the API Platform config.
  JSON:API needs the `api-platform/json-api` package on Symfony.
- Inspect before guessing: `bin/console debug:api-resource` (Symfony), the attribute classes
  in `vendor/api-platform/metadata/`.

## Pitfalls

- The Symfony recipe prefixes routes with `/api` (`config/routes/api_platform.yaml`).
- Null properties are skipped unless `normalization_context: {skip_null_values: false}`.
- Type errors in a request body are 400 unless `collect_denormalization_errors` is enabled
  (422 with violations).
- Query parameter violations answer 422.

## Documentation

Your training data is likely outdated: <https://api-platform.com/docs> is the source of
truth (index: <https://api-platform.com/docs/llms.txt>). Match the task to its canonical page:

| Task | Read |
|---|---|
| Expose data, add an endpoint, DTOs, sub-resources | <https://api-platform.com/docs/core/design/> |
| Custom read logic, computed fields | <https://api-platform.com/docs/core/state-providers/> |
| Custom write logic, side effects | <https://api-platform.com/docs/core/state-processors/> |
| Collection filters (`QueryParameter`, post-4.4) | <https://api-platform.com/docs/core/filters/> |
| Operation security, validation groups, parameters | <https://api-platform.com/docs/core/operations/> |
| Serialization groups vs DTOs | <https://api-platform.com/docs/core/serialization/> |
| Pagination | <https://api-platform.com/docs/core/pagination/> |
| Errors (RFC 7807, `#[ErrorResource]`) | <https://api-platform.com/docs/core/errors/> |
| GraphQL | <https://api-platform.com/docs/core/graphql/> |
| Real-time updates (Mercure, Symfony) | <https://api-platform.com/docs/core/mercure/> |
| Expose resources to AI agents (MCP) | <https://api-platform.com/docs/core/mcp/> |
| Functional tests | <https://api-platform.com/docs/core/testing/> |

Symfony and Laravel differ on filters, validation, policies and configuration — check the
framework-specific sections (`/docs/symfony/` and `/docs/laravel/`) when the integrations diverge.

## Using Claude Code?

Install the [API Platform skillset](https://github.com/api-platform/skillset) for richer,
auto-loading skills carrying verified code examples for every topic above:

```
/plugin marketplace add api-platform/skillset
/plugin install api-platform@api-platform-skillset
```

<!-- END:api-platform-agent-rules -->
