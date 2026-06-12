# API Platform Skillset

[Claude Code](https://code.claude.com) plugin providing 15 skills for [API Platform](https://api-platform.com) development. Each skill teaches Claude the current, canonical way to do something in API Platform 4.x — verified against the core source, covering both the Symfony and Laravel integrations.

## Installation

```
/plugin marketplace add api-platform/skillset
/plugin install api-platform@api-platform-skillset
```

Skills load automatically when relevant; they appear namespaced as `api-platform:<skill>`.

## Skills

| Skill | Covers |
|---|---|
| `api-resource` | Resources, DTOs, Object Mapper, nested sub-resources, custom operations |
| `api-filter` | Collection filters with `QueryParameter` — the canonical post-4.4 filter set, legacy `#[ApiFilter]` migration |
| `state-provider` | Custom read logic, decorating Doctrine providers, computed fields |
| `state-processor` | Custom write logic, soft-delete, file downloads, side effects |
| `operations` | Operation security expressions, validation groups, parameter validation, deprecation |
| `securing-collections` | Multi-tenant isolation with Doctrine extensions and link handlers |
| `custom-validator` | Custom validation constraints for business rules |
| `serialization-groups` | Serialization contexts and `#[Groups]` — with guidance on when DTOs are the better choice |
| `pagination` | Page-based, partial, and cursor pagination |
| `errors` | RFC 7807 Problem Details, `#[ErrorResource]`, exception-to-status mapping |
| `graphql` | GraphQL operations, resolvers, Relay pagination |
| `mercure` | Real-time updates over Mercure (Symfony) |
| `mcp` | Exposing resources to AI agents via the Model Context Protocol — `#[McpTool]`, `McpToolCollection`, `#[McpResource]` |
| `api-docs` | OpenAPI customization, hiding operations, factory decoration |
| `api-test` | Functional tests with `ApiTestCase` (Symfony) and HTTP tests (Laravel) |

## Symfony and Laravel

Skills are framework-aware: shared API Platform concepts are presented once, and where the integrations differ (Eloquent filters, Laravel validation rules, policies, `config/api-platform.php`), skills carry dedicated Laravel sections. Mercure is currently Symfony-only.

## Updating

Bump happens through the marketplace:

```
/plugin marketplace update api-platform-skillset
```

## Contributing

Skills live in `skills/<name>/SKILL.md`. Conventions: imperative voice, verified APIs only (checked against [api-platform/core](https://github.com/api-platform/core)), imports at the top of every code block, no bare references to core test paths — inline the relevant extract instead.
