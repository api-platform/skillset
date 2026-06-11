---
name: api-test
description: "Writes functional tests for API Platform endpoints with ApiTestCase. Use whenever the user asks to test an API resource, write or fix a functional test, verify validation, security or multi-tenant isolation, set up authentication or database fixtures for tests, or reproduce an endpoint bug with a test — including plain 'add tests for X' requests."
---

# Testing API Platform Endpoints

Use `ApiPlatform\Symfony\Bundle\Test\ApiTestCase` for functional API tests.

## Base Test Class

Create a base class that handles database setup and authentication:

```php
<?php
namespace App\Tests;

use ApiPlatform\Symfony\Bundle\Test\ApiTestCase as SymfonyApiTestCase;
use ApiPlatform\Symfony\Bundle\Test\Client;

class ApiTestCase extends SymfonyApiTestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        // Reset database state for each test
        $manager = static::getContainer()->get('doctrine')->getManager();
        // For MongoDB: drop and recreate
        // For ORM: use fixtures or transactions
    }

    /**
     * Creates an authenticated client with test fixtures.
     */
    public function createLoggedInClient(): Client
    {
        $manager = static::getContainer()->get('doctrine')->getManager();

        // Create user, token, and required fixtures
        $user = new User();
        // ... set up user
        $manager->persist($user);
        $manager->flush();

        // Return client with auth headers
        return static::createClient([], [
            'headers' => ['x-api-key' => 'test_token'],
        ]);
    }
}
```

## Basic CRUD Tests

```php
class OrderTest extends ApiTestCase
{
    public function testGetCollection(): void
    {
        $client = $this->createLoggedInClient();
        $client->request('GET', '/orders');

        $this->assertResponseIsSuccessful();
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
    }

    public function testCreate(): void
    {
        $client = $this->createLoggedInClient();
        $client->request('POST', '/orders', [
            'headers' => ['Content-Type' => 'application/ld+json'],
            'json' => ['name' => 'Test Order', 'total' => 99.99],
        ]);

        $this->assertResponseStatusCodeSame(201);
        $data = $client->getResponse()->toArray();
        $this->assertSame('Test Order', $data['name']);
    }

    public function testUpdate(): void
    {
        $client = $this->createLoggedInClient();

        // Create then update
        $client->request('POST', '/orders', [
            'headers' => ['Content-Type' => 'application/ld+json'],
            'json' => ['name' => 'Original'],
        ]);
        $order = $client->getResponse()->toArray();

        $client->request('PATCH', '/orders/' . $order['id'], [
            'headers' => ['Content-Type' => 'application/merge-patch+json'],
            'json' => ['name' => 'Updated'],
        ]);

        $this->assertResponseIsSuccessful();
        $this->assertSame('Updated', $client->getResponse()->toArray()['name']);
    }

    public function testDelete(): void
    {
        $client = $this->createLoggedInClient();
        // ... create resource first
        $client->request('DELETE', '/orders/' . $id);
        $this->assertResponseStatusCodeSame(204);
    }
}
```

## Testing Validation

```php
public function testCreateWithInvalidData(): void
{
    $client = $this->createLoggedInClient();
    $client->request('POST', '/orders', [
        'json' => ['email' => 'not-valid'],
    ]);

    $this->assertResponseStatusCodeSame(422);
    $this->assertJsonContains([
        'violations' => [
            ['propertyPath' => 'email', 'message' => 'This value is not a valid email address.'],
        ],
    ]);
}
```

## Testing Authentication

```php
public function testRequiresAuthentication(): void
{
    // No auth headers — use raw createClient
    $client = static::createClient();

    $client->request('GET', '/orders');
    $this->assertResponseStatusCodeSame(401);
}
```

## Multi-Tenant Isolation Tests

Test that users can only access their own resources:

```php
public function testIsolationBetweenUsers(): void
{
    $client1 = $this->createLoggedInClient();       // User 1, Org 1
    $client2 = $this->createSecondUserClient();      // User 2, Org 2

    // User 1 creates a resource
    $client1->request('POST', '/accounts', [
        'headers' => ['Content-Type' => 'application/ld+json'],
        'json' => ['address' => 'user1@example.com', 'password' => 'Pass123!'],
    ]);
    $account = $client1->getResponse()->toArray();

    // User 2 cannot read it
    $client2->request('GET', '/accounts/' . $account['id']);
    $this->assertResponseStatusCodeSame(404);

    // User 2 cannot update it
    $client2->request('PATCH', '/accounts/' . $account['id'], [
        'headers' => ['Content-Type' => 'application/merge-patch+json'],
        'json' => ['isActive' => false],
    ]);
    $this->assertResponseStatusCodeSame(404);

    // User 2 cannot delete it
    $client2->request('DELETE', '/accounts/' . $account['id']);
    $this->assertResponseStatusCodeSame(404);

    // Collections are isolated
    $client2->request('GET', '/accounts');
    $collection = $client2->getResponse()->toArray();
    $ids = array_column($collection['member'], 'id');
    $this->assertNotContains($account['id'], $ids);
}
```

## Testing Filters

```php
public function testFilterByStatus(): void
{
    $client = $this->createLoggedInClient();
    // ... create test data with different statuses

    $client->request('GET', '/orders?isActive=true');
    $this->assertResponseIsSuccessful();
    $data = $client->getResponse()->toArray();

    foreach ($data['member'] as $order) {
        $this->assertTrue($order['isActive']);
    }
}
```

## Common Assertions

```php
$this->assertResponseIsSuccessful();
$this->assertResponseStatusCodeSame(201);
$this->assertJsonContains(['name' => 'Expected']);
$this->assertMatchesResourceItemJsonSchema(Order::class);
$this->assertMatchesResourceCollectionJsonSchema(Order::class);
```

## Resetting the Schema Between Tests (ORM)

For a clean database per test, drop and recreate the schema in `setUp()`. API
Platform's own suite uses an internal `ApiPlatform\Tests\RecreateSchemaTrait` —
that namespace is **not** autoloaded in your app, so copy the mechanism rather than
importing it:

```php
use Doctrine\ORM\Tools\SchemaTool;

protected function setUp(): void
{
    parent::setUp();

    $manager = static::getContainer()->get('doctrine')->getManager();
    $metadata = $manager->getMetadataFactory()->getAllMetadata();

    $schemaTool = new SchemaTool($manager);
    $schemaTool->dropDatabase();
    $schemaTool->createSchema($metadata);
}
```

For MongoDB ODM, drop the collections via
`$manager->getSchemaManager()->dropDocumentCollection(...)` instead.

## Test File Location

Place tests in `tests/Functional/` following the naming convention `{ResourceName}Test.php`.

## Laravel

Laravel uses its own HTTP test client (PHPUnit `Tests\TestCase` or Pest), **not**
`ApiTestCase`. API Platform ships `ApiPlatform\Laravel\Test\ApiTestAssertionsTrait`
(adds `assertJsonContains()`, `assertArraySubset()`, `getIriFromResource()`). Use
`RefreshDatabase` for isolation and model **factories** for fixtures (not Doctrine
`SchemaTool`/fixtures).

```php
<?php
namespace Tests\Feature;

use ApiPlatform\Laravel\Test\ApiTestAssertionsTrait;
use App\Models\Book;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class BooksTest extends TestCase
{
    use RefreshDatabase, ApiTestAssertionsTrait;

    public function testCreate(): void
    {
        $response = $this->postJson('/api/books', ['isbn' => '0099740915', 'title' => 'X']);
        $response->assertStatus(201);
        $this->assertJsonContains(['title' => 'X'], $response->json());

        $book = Book::factory()->create();
        $this->getJson($this->getIriFromResource($book))->assertStatus(200);
    }
}
```

Key deltas: request helpers are `getJson`/`postJson`/`patchJson`/`deleteJson`;
assertions are `$response->assertStatus(...)`, `$response->assertHeader(...)`,
`$this->assertDatabaseMissing(...)`. `assertJsonContains()` takes the decoded body
(`$response->json()`). Isolation/validation(422)/auth tests use the same status-code
assertions; auth uses Sanctum/`actingAs()`, not custom headers. Pest is the default
and wraps the same trait via `uses(RefreshDatabase::class, ApiTestAssertionsTrait::class)`.
