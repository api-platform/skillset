---
name: api-docs
description: Customizes OpenAPI documentation for API Platform resources. Use when adding descriptions, examples, custom responses, hiding operations from documentation, or decorating the OpenAPI factory for global customization.
---

# Customizing API Documentation

API Platform generates OpenAPI v3 documentation automatically. Customize it using attributes.

## Global Info & Security Schemes (YAML)

Set the API-wide title, version, description and auth schemes in
`config/packages/api_platform.yaml`:

```yaml
api_platform:
    openapi:
        info:
            title: 'My API'
            version: '1.0.0'
            description: 'What this API does.'
        components:
            securitySchemes:
                Bearer:
                    type: http
                    scheme: bearer
                    bearerFormat: JWT
```
*Pattern: `OpenApiTest.php`.*

## Operation-Level Customization

```php
use ApiPlatform\Metadata\Post;
use ApiPlatform\OpenApi\Model\Operation;
use ApiPlatform\OpenApi\Model\Response as OpenApiResponse;

#[Post(
    openapi: new Operation(
        summary: 'Create a new order',
        description: 'Creates an order and sends confirmation email.',
        responses: [
            '201' => new OpenApiResponse(description: 'Order created successfully'),
            '422' => new OpenApiResponse(description: 'Validation failed'),
        ]
    )
)]
class Order {}
```

## Property Documentation

```php
use ApiPlatform\Metadata\ApiProperty;

class Order
{
    #[ApiProperty(description: 'The unique identifier', example: 1)]
    public int $id;

    #[ApiProperty(
        description: 'Current status',
        example: 'pending',
        openapiContext: ['enum' => ['pending', 'shipped', 'delivered']]
    )]
    public string $status;

    #[ApiProperty(
        genId: false,
        types: ['https://schema.org/sender'],
        openapiContext: [
            'example' => ['address' => 'user@example.com', 'name' => 'John'],
        ],
    )]
    public Recipient $from;
}
```

Use `genId: false` on embedded objects (non-IRI properties) to suppress `@id` generation.

## Hiding from Documentation

```php
// Hide entire resource
#[ApiResource(openapi: false)]

// Hide specific operation
#[Get(openapi: false)]

// Hide from Hydra entrypoint only (keep in OpenAPI)
#[Get(hydra: false)]
```

## Custom Parameters

```php
use ApiPlatform\Metadata\HeaderParameter;

#[Post(
    parameters: [
        'X-Idempotency-Key' => new HeaderParameter(
            description: 'Unique key to prevent duplicate processing',
            required: true,
        ),
    ],
)]
```

## OpenApiFactory Decorator (Global Customization)

Decorate the built-in factory for global changes like custom server URLs:

```php
<?php
namespace App\OpenApi;

use ApiPlatform\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\OpenApi\Model;
use ApiPlatform\OpenApi\OpenApi;

final class OpenApiFactory implements OpenApiFactoryInterface
{
    public function __construct(
        private readonly OpenApiFactoryInterface $decorated,
        private readonly string $openapiUrl,
    ) {}

    public function __invoke(array $context = []): OpenApi
    {
        $openApi = $this->decorated->__invoke($context);

        return $openApi->withServers([
            new Model\Server($this->openapiUrl),
        ]);
    }
}
```

Register as a decorator in `services.yaml`:

```yaml
App\OpenApi\OpenApiFactory:
    decorates: 'api_platform.openapi.factory'
    arguments:
        $openapiUrl: '%env(OPENAPI_URL)%'
```
