---
name: api-platform-mcp
description: "Exposes API Platform resources to AI agents (LLMs) via the Model Context Protocol — declaring MCP tools with `#[McpTool]` / `McpToolCollection`, read-only `#[McpResource]`, the `mcp:` array on `#[ApiResource]`, input/output DTOs, structured vs custom (CallToolResult) responses, JSON-Schema overrides via `#[ApiProperty]`, and MCP-specific validation. Use whenever the user wants an LLM/AI agent to discover or call their API, mentions MCP, Model Context Protocol, tools/resources for an agent, `tools/call`, `tools/list`, or asks to make a resource 'callable by an AI' — even if they don't name MCP explicitly."
---

# MCP: Exposing Your API to AI Agents

API Platform turns PHP classes into [Model Context Protocol](https://modelcontextprotocol.io/)
tools and resources. It reuses the existing metadata stack — state processors/providers,
validation, serialization, JSON Schema — so an MCP tool is just an operation with no HTTP route.

> **`@experimental`.** API may change between minor releases.

## Install & enable

Symfony — `composer require api-platform/mcp symfony/mcp-bundle` (bundle namespace is
`Symfony\AI\McpBundle`, composer name `symfony/mcp-bundle`):

```yaml
# config/packages/mcp.yaml
mcp:
    client_transports:
        http: true
        stdio: false
    http:
        path: "/mcp"
        session: { store: "file", directory: "%kernel.cache_dir%/mcp", ttl: 3600 }
```

```yaml
# config/packages/api_platform.yaml
api_platform:
    mcp:
        enabled: true     # default: true
        format: jsonld    # default; must be a registered api_platform.format (json, jsonapi, …)
```

`format` sets the serialization format for structured content. `jsonld` emits `@context`,
`@id`, `@type`. Laravel — enabled by default, endpoint auto-registered at `/mcp`:

```php
// config/api-platform.php
return [ /* … */ 'mcp' => ['enabled' => true] ];
```

API Platform registers its own SDK `Handler`; `supports()` returns `true` only for tools/resources
in its registry, otherwise the SDK's own handler chain runs. API Platform tools and native
`mcp/sdk` tools coexist on the same server.

## Two ways to declare a tool

**Standalone `#[McpTool]` class attribute** — convenient for a one-off tool. Class properties
become the `inputSchema`; a [state processor](../state-processor/SKILL.md) handles the command
(CQRS-style — the agent's input is processed through your logic):

```php
use ApiPlatform\Metadata\McpTool;

#[McpTool(
    name: 'process_message',
    description: 'Process a message with priority',
    processor: [self::class, 'process'],
)]
class ProcessMessage
{
    public function __construct(
        private string $message,
        private int $priority = 1,
    ) {}

    // getters + setters drive (de)serialization

    public static function process($data): mixed
    {
        $data->setMessage('Processed: '.$data->getMessage());
        return $data;
    }
}
```

The agent's arguments are deserialized into a `ProcessMessage`, passed to the processor; the
returned object is serialized back as structured content. `processor` is any callable or a
`ProcessorInterface` service.

**`mcp:` array on `#[ApiResource]`** — use when the class exposes **no** HTTP endpoints
(`operations: []`), combines several MCP operations, or needs resource-level control. Keys are
the tool names, so **omit `name:`** inside each `new McpTool(...)`:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpTool;
use App\State\ReadHydraResourceProcessor;

#[ApiResource(
    operations: [],
    mcp: [
        'read_hydra_resource' => new McpTool(
            description: 'Navigate to a Hydra API resource by URI.',
            processor: ReadHydraResourceProcessor::class,
            structuredContent: false,
        ),
    ],
)]
class ReadHydraResource
{
    public string $uri;
}
```

## Separate input DTO

When the input schema must differ from the result class, point `input:` at a DTO. That DTO drives
`inputSchema`; the host class describes the output:

```php
#[McpTool(
    name: 'search_books',
    description: 'Search books by keyword',
    input: SearchQuery::class,        // what the agent sends
    processor: SearchBooksProcessor::class,
)]
class BookSearchResult
{
    public int $id;
    public string $title;
    public string $isbn;
}
```

The processor receives a `SearchQuery` instance and returns the result(s).

## Collections — `McpToolCollection`

Use `McpToolCollection` (extends `McpTool`, implements `CollectionOperationInterface`) when a tool
returns a list — the serializer and schema factory emit collection-shaped output:

```php
use ApiPlatform\Metadata\McpToolCollection;

#[ApiResource(operations: [], mcp: [
    'list_books' => new McpToolCollection(
        description: 'List Books',
        input: SearchQuery::class,
        processor: SearchBooksProcessor::class,
        structuredContent: true,
    ),
])]
class Book { public ?int $id = null; public ?string $title = null; public ?string $isbn = null; }
```

With `structuredContent: true` and the default `jsonld` format, the response carries `@context`,
`hydra:totalItems`, and a `hydra:member` array of serialized items; the published `outputSchema`
reflects that shape. Output schemas are **flattened** for MCP compliance — no `$ref`, `allOf`,
`$schema`, or `definitions`.

## Custom responses — `CallToolResult`

Default: results serialize through API Platform's [serialization](../serialization-groups/SKILL.md)
as structured JSON. For full control, set `structuredContent: false` and return a `CallToolResult`:

```php
use Mcp\Schema\Content\TextContent;
use Mcp\Schema\Result\CallToolResult;

#[McpTool(name: 'generate_report', description: 'Generate a markdown report',
    processor: [self::class, 'process'], structuredContent: false)]
class Report
{
    public function __construct(private string $title, private string $content) {}
    // getters/setters …

    public static function process($data): CallToolResult
    {
        return new CallToolResult(
            [new TextContent("# {$data->getTitle()}\n\n{$data->getContent()}")],
            false,               // isError
            // 3rd arg: optional _meta array
        );
    }
}
```

`structuredContent: false` disables automatic JSON serialization; the `CallToolResult` is sent
as-is, with no `structuredContent` / `_meta` key in the response.

## Schema overrides — the nullable-union gotcha

PHP nullable types generate union schemas like `{"type": ["array", "null"]}`. Some LLM providers
reject these. Override per property with `#[ApiProperty(schema: [...])]`:

```php
use ApiPlatform\Metadata\ApiProperty;

class InvokeHydraOperation
{
    public string $uri;
    public string $method;
    #[ApiProperty(schema: ['type' => 'object', 'description' => 'JSON payload for the request'])]
    public ?array $payload = null;   // without override: {"type": ["array","null"]}
}
```

## Validation — off by default

The MCP SDK validates inputs against the JSON Schema at transport level (types, required fields).
API Platform's **own** validation pipeline is **disabled** for MCP tools. Set `validate: true` to
run business constraints (email format, length, custom rules):

```php
use Symfony\Component\Validator\Constraints as Assert;

#[McpTool(name: 'submit_contact', description: 'Submit a contact form',
    processor: [self::class, 'process'], validate: true)]
class ContactForm
{
    #[Assert\NotBlank] #[Assert\Length(min: 3, max: 50)]
    private ?string $name = null;
    #[Assert\NotNull] #[Assert\Email]
    private ?string $email = null;
}
```

Laravel — same `validate: true`, plus `rules:` instead of constraints:

```php
#[McpTool(name: 'submit_contact', description: '…', processor: [self::class, 'process'],
    validate: true, rules: ['name' => 'required|min:3|max:50', 'email' => 'required|email'])]
```

## Read-only resources — `#[McpResource]`

Expose retrievable content (docs, config, reference data) backed by a
[state provider](../state-provider/SKILL.md). `uri` must be unique and use the `resource://` scheme:

```php
use ApiPlatform\Metadata\McpResource;

#[ApiResource(operations: [], mcp: [
    'api_docs' => new McpResource(
        uri: 'resource://my-app/documentation',
        name: 'App-Documentation',
        description: 'Application API documentation',
        mimeType: 'text/markdown',
        provider: [self::class, 'provide'],
    ),
])]
class Documentation
{
    public function __construct(private string $content, private string $uri) {}
    public static function provide(): self
    {
        return new self('# My API Documentation\n\nWelcome.', 'resource://my-app/documentation');
    }
}
```

## Options

`McpTool` and `McpResource` extend `HttpOperation`, so they accept all standard
[operation options](../operations/SKILL.md). MCP-specific / notable ones:

| Option              | Tool | Res | Notes                                                              |
| ------------------- | :--: | :-: | ------------------------------------------------------------------ |
| `name`              |  ✓   |  ✓  | exposed to agents; defaults to class short name. Omit in `mcp:` array (key wins) |
| `description`       |  ✓   |  ✓  | defaults to class DocBlock                                         |
| `structuredContent` |  ✓   |  ✓  | include JSON structured content (default `true`)                  |
| `input` / `output`  |  ✓   |     | separate DTO for input schema / output representation             |
| `inputFormats` / `outputFormats` | ✓ | | (de)serialization formats, e.g. `['json']`, `['jsonld']`     |
| `contentNegotiation`|  ✓   |  ✓  | default `false` for MCP                                            |
| `validate`          |  ✓   |     | run validation pipeline (default `false` for MCP)                 |
| `rules`             |  ✓   |     | Laravel validation rules (Laravel only)                           |
| `annotations`/`icons`/`meta` | ✓ | ✓ | behavior hints / icon URLs / arbitrary metadata               |
| `uri`               |      |  ✓  | **required**, `resource://` scheme, unique across server          |
| `mimeType` / `size` |      |  ✓  | content MIME type / byte size if known                            |

## Wire protocol (for testing)

Everything is JSON-RPC 2.0 POSTed to `/mcp` with
`Accept: application/json, text/event-stream`. `initialize` returns an `mcp-session-id` header;
send it back on every subsequent call. `tools/list` / `resources/list` enumerate; `tools/call`
(`params: {name, arguments}`) and `resources/read` (`params: {uri}`) invoke. The handler overrides
`Accept` to the operation's output format, so a `jsonld` tool always responds JSON-LD regardless
of `use_symfony_listeners`.

## Laravel

- Config under `mcp` in `config/api-platform.php`; endpoint auto-registered at `/mcp`.
- Validation uses `rules:` (string/array/closure/FormRequest), not Symfony constraints.
- MongoDB is **not** supported for MCP.

## Checklist

- [ ] `api-platform/mcp` + `symfony/mcp-bundle` installed, `/mcp` transport configured (Symfony)
- [ ] Tool class has a `processor` (write) or `McpResource` has a `provider` (read)
- [ ] `operations: []` when the class is MCP-only (no HTTP routes)
- [ ] `name:` omitted inside `mcp:` array entries (the array key is the name)
- [ ] `McpToolCollection` (not `McpTool`) for list-returning tools
- [ ] `structuredContent: false` + `CallToolResult` when bypassing JSON serialization
- [ ] Nullable/union properties overridden with `#[ApiProperty(schema: …)]` if the LLM rejects them
- [ ] `validate: true` set explicitly where business constraints must run
- [ ] `McpResource` `uri` is unique and uses `resource://`
