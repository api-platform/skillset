---
name: mercure
description: Pushes real-time resource updates from API Platform to clients over the Mercure protocol (Server-Sent Events). Use whenever the user mentions real-time updates, live data, push notifications, websockets-vs-SSE, Mercure, subscribing to changes, "update the UI when data changes", broadcasting create/update/delete, private/authorized updates, or custom Mercure topics.
---

# Mercure: Real-Time Updates

> **Symfony-only.** The Mercure integration is built on Doctrine's lifecycle events
> and `symfony/mercure-bundle`; the API Platform Laravel package ships no Mercure
> support. On Laravel use Laravel's own broadcasting (Reverb/Echo/Pusher).

Mercure pushes the new version of a resource to all subscribed clients whenever it
is created, updated or deleted. API Platform hooks Doctrine's lifecycle: after a
write it serializes the resource and publishes it to the Mercure hub, which fans it
out to connected clients over Server-Sent Events. Clients subscribe with the native
`EventSource` API — no extra client library required.

Use this for dashboards, collaborative editing, notifications. It is push-only and
one-directional (server → client); for bidirectional messaging you still need
something else.

## Enabling on a resource

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(mercure: true)]
class Book {}
```

Now every create/update/delete on a `Book` publishes to the hub. On delete, only
the (now-gone) IRI is sent so clients can drop it. API Platform also adds a `Link`
header advertising the hub URL on every response for this resource, so smart
clients auto-discover where to subscribe.

> `mercure` is intentionally untyped — it accepts `true`, an **options array**, or
> an **ExpressionLanguage string**. There is no `MercureOptions` attribute class in
> core; pass a plain array. (Verified against `ApiResource::$mercure`, declared
> `mixed`.)

## Symfony hub configuration

Mercure is preconfigured in the API Platform distribution. With a manual install,
add the MercureBundle and point it at a hub:

```yaml
# config/packages/mercure.yaml
mercure:
    hubs:
        default:
            url: '%env(MERCURE_URL)%'          # internal URL API Platform publishes to
            public_url: '%env(MERCURE_PUBLIC_URL)%'  # URL the browser subscribes to
            jwt:
                secret: '%env(MERCURE_JWT_SECRET)%'
                publish: ['*']                 # API Platform must be allowed to publish
```

The hub itself runs as a separate process/container (the `mercure.rocks` binary or
the Caddy Mercure module).

## Subscribing from a browser

Read the hub URL from the `Link` header, then open an `EventSource` on the topic
(the resource IRI):

```js
const url = new URL('https://example.com/.well-known/mercure');
url.searchParams.append('topic', 'https://example.com/books/1');

const es = new EventSource(url);
es.onmessage = ({ data }) => {
    const book = JSON.parse(data);   // the serialized resource, or {"@id": "..."} on delete
};
```

Use a topic **template** like `https://example.com/books/{id}` to receive updates
for every book in one subscription.

## Available options

Pass an array instead of `true`:

```php
#[ApiResource(mercure: [
    'private' => true,                          // authorized subscribers only (see below)
    'topics' => ['https://example.com/books'],  // override the default (resource IRI)
    'normalization_context' => ['groups' => ['book:push']], // serialize differently for the push
    'type' => 'book-updated',                   // SSE event type
    'retry' => 3000,                            // SSE retry hint (ms)
])]
class Book {}
```

`data` (override the published payload) and `id` (SSE event id) are also accepted.
A common reason to set `normalization_context` is to push a leaner shape than the
HTTP response, or to avoid pushing fields a subscriber shouldn't compute on.

## Private (authorized) updates

A public update reaches anyone subscribed to the topic. For per-user data, mark the
update **private** so only clients whose JWT carries a matching `subscribe` target
selector receive it:

```php
#[ApiResource(mercure: ['private' => true])]
class Notification {}
```

The subscriber's Mercure JWT must include a `mercure.subscribe` claim matching the
topic. This is the mechanism that prevents one user's real-time updates leaking to
another — pair it with HTTP-level `security` (see **operations**) so the resource is
both fetch-protected and push-protected.

## Dynamic options via expression

When the options depend on the resource instance (e.g. the topic must include the
owner), use an ExpressionLanguage string resolved per object. `iri()` builds an IRI,
`escape()` is `rawurlencode`:

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(mercure: 'object.getMercureOptions()')]
class Book
{
    public function getMercureOptions(): array
    {
        return [
            'private' => true,
            // '@=' marks each topic value as an expression
            'topics' => [
                '@=iri(object)',
                '@=iri(object.getOwner()) ~ "/?topic=" ~ escape(iri(object))',
            ],
        ];
    }
}
```

The alternate `owner/?topic=...` form is the "restrictive update" pattern: a
subscriber authorized only for `https://example.com/users/foo/{?topic}` receives the
update via the alternate topic without being able to subscribe to all books. This is
how you scope a shared resource type to per-owner visibility.

## GraphQL subscriptions

When GraphQL is enabled (see **graphql**), `mercure: true` also powers GraphQL
`subscription` operations — clients subscribe to a query and receive pushes through
the same hub via a generated subscription IRI.

## Checklist

- [ ] Hub configured with separate internal `url` and public `public_url`
- [ ] API Platform's publisher JWT allows `publish`
- [ ] Per-user resources use `'private' => true`, not public topics
- [ ] Private pushes paired with HTTP `security` on the same resource
- [ ] `normalization_context` set when the push shape differs from the HTTP response
- [ ] Clients discover the hub from the `Link` header, not a hardcoded URL
