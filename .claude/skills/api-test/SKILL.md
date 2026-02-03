---
name: api-test
description: Write functional tests for API Platform endpoints. Use when creating tests for API resources, testing validation, security, or verifying response formats.
---

# Testing API Platform Endpoints

Use `ApiPlatform\Symfony\Bundle\Test\ApiTestCase` for functional API tests.

## Basic Test Structure

```php
<?php
namespace App\Tests\Api;

use ApiPlatform\Symfony\Bundle\Test\ApiTestCase;
use App\Entity\YourEntity;

class YourResourceTest extends ApiTestCase
{
    public function testGetCollection(): void
    {
        $response = static::createClient()->request('GET', '/your_resources');

        $this->assertResponseIsSuccessful();
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
    }

    public function testGetItem(): void
    {
        // Load fixtures first
        $this->loadFixtures();

        $response = static::createClient()->request('GET', '/your_resources/1');

        $this->assertResponseIsSuccessful();
        $this->assertJsonContains([
            '@type' => 'YourResource',
            'id' => 1,
        ]);
    }
}
```

## Making Requests

```php
// GET collection
static::createClient()->request('GET', '/resources');

// GET item
static::createClient()->request('GET', '/resources/1');

// POST (create)
static::createClient()->request('POST', '/resources', [
    'json' => [
        'name' => 'New Resource',
        'email' => 'test@example.com',
    ],
]);

// PATCH (update)
static::createClient()->request('PATCH', '/resources/1', [
    'headers' => ['Content-Type' => 'application/merge-patch+json'],
    'json' => [
        'name' => 'Updated Name',
    ],
]);

// DELETE
static::createClient()->request('DELETE', '/resources/1');
```

## Common Assertions

```php
// Status assertions
$this->assertResponseIsSuccessful();
$this->assertResponseStatusCodeSame(201);
$this->assertResponseStatusCodeSame(404);
$this->assertResponseStatusCodeSame(422); // Validation error

// Content assertions
$this->assertJsonContains(['name' => 'Expected Name']);
$this->assertJsonEquals([/* exact expected response */]);

// Schema validation
$this->assertMatchesResourceCollectionJsonSchema(YourResource::class);
$this->assertMatchesResourceItemJsonSchema(YourResource::class);

// Header assertions
$this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
```

## Testing Validation Errors

```php
public function testCreateWithInvalidData(): void
{
    static::createClient()->request('POST', '/resources', [
        'json' => [
            'email' => 'invalid-email',
        ],
    ]);

    $this->assertResponseStatusCodeSame(422);
    $this->assertJsonContains([
        'violations' => [
            ['propertyPath' => 'email', 'message' => 'This value is not a valid email address.'],
        ],
    ]);
}
```

## Testing Security

```php
public function testUnauthorizedAccess(): void
{
    static::createClient()->request('POST', '/admin/resources', [
        'json' => ['name' => 'Test'],
    ]);

    $this->assertResponseStatusCodeSame(403);
}

public function testAuthorizedAccess(): void
{
    $client = static::createClient();
    // Authenticate somehow (JWT, session, etc.)

    $client->request('POST', '/admin/resources', [
        'json' => ['name' => 'Test'],
    ]);

    $this->assertResponseIsSuccessful();
}
```

## Loading Test Fixtures

```php
private function loadFixtures(): void
{
    $manager = static::getContainer()->get('doctrine')->getManager();

    $entity = new YourEntity();
    $entity->setName('Test Fixture');
    $manager->persist($entity);
    $manager->flush();
}
```

## Testing Different Formats

```php
// JSON:API format
$response = static::createClient()->request('GET', '/resources', [
    'headers' => ['Accept' => 'application/vnd.api+json'],
]);

$this->assertJsonContains([
    'data' => [
        ['type' => 'YourResource'],
    ],
]);
```

## Test File Location

Place tests in `tests/Api/` following the naming convention `{ResourceName}Test.php`.

## Reference

For detailed patterns, see:
- [Testing Guide](../../../skills/testing/AGENTS.md)
