---
name: api-docs
description: Customize OpenAPI documentation for API Platform resources. Use when adding descriptions, examples, custom responses, or hiding operations from documentation.
---

# Customizing API Documentation

API Platform generates OpenAPI v3 documentation automatically. Customize it using attributes.

## Operation-Level Customization

```php
<?php
use ApiPlatform\Metadata\Post;
use ApiPlatform\OpenApi\Model\Operation;
use ApiPlatform\OpenApi\Model\Response as OpenApiResponse;

#[Post(
    openapi: new Operation(
        summary: 'Create a new resource',
        description: 'Creates a resource with the provided data. Sends notification on success.',
        responses: [
            '201' => new OpenApiResponse(description: 'Resource created successfully'),
            '400' => new OpenApiResponse(description: 'Invalid input data'),
            '422' => new OpenApiResponse(description: 'Validation failed'),
        ]
    )
)]
class YourResource {}
```

## Property Documentation

```php
use ApiPlatform\Metadata\ApiProperty;

class YourResource
{
    #[ApiProperty(
        description: 'The unique identifier',
        example: 1
    )]
    public int $id;

    #[ApiProperty(
        description: 'Email address for the resource',
        example: 'user@example.com'
    )]
    public string $email;

    #[ApiProperty(
        description: 'Current status',
        example: 'pending',
        openapiContext: [
            'enum' => ['pending', 'sent', 'delivered', 'failed']
        ]
    )]
    public string $status;
}
```

## Hiding from Documentation

### Hide entire resource
```php
#[ApiResource(openapi: false)]
class InternalResource {}
```

### Hide specific operation
```php
#[Get(openapi: false)]  // Hidden from OpenAPI
#[Post()]               // Visible in OpenAPI
```

### Hide from Hydra only (keep in OpenAPI)
```php
#[Get(hydra: false)]  // Not in JSON-LD entrypoint, but in OpenAPI
```

## Custom Parameters Documentation

```php
use ApiPlatform\Metadata\HeaderParameter;
use ApiPlatform\OpenApi\Model\Parameter;

#[Post(
    parameters: [
        'X-Custom-Header' => new HeaderParameter(
            description: 'Custom header for authentication',
            required: true,
        ),
    ],
)]
```

## Complete Example

```php
<?php
namespace App\Api\Resource;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;
use ApiPlatform\OpenApi\Model\Operation;
use ApiPlatform\OpenApi\Model\Response as OpenApiResponse;

#[ApiResource(
    shortName: 'Book',
    description: 'A book available in the catalog',
    operations: [
        new Get(
            openapi: new Operation(
                summary: 'Retrieve a book',
                description: 'Returns the details of a book including availability.',
            )
        ),
        new Post(
            openapi: new Operation(
                summary: 'Create a new book',
                description: 'Adds a new book to the catalog.',
                responses: [
                    '201' => new OpenApiResponse(description: 'Book created successfully'),
                    '422' => new OpenApiResponse(description: 'Validation failed'),
                ]
            )
        ),
    ]
)]
class Book
{
    #[ApiProperty(description: 'The unique identifier', example: 1)]
    public int $id;

    #[ApiProperty(description: 'The book title', example: 'Domain-Driven Design')]
    public string $title;

    #[ApiProperty(description: 'The ISBN number', example: '978-0321125217')]
    public string $isbn;

    #[ApiProperty(
        description: 'Availability status',
        example: 'in_stock',
        openapiContext: ['enum' => ['in_stock', 'out_of_stock', 'preorder']]
    )]
    public string $availability;
}
```

## Reference

For detailed patterns, see:
- [Documentation Guide](../../../skills/documentation/AGENTS.md)
