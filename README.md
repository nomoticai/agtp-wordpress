# AGTP for WordPress

A WordPress plugin that exposes your WordPress site to the Agent
Transfer Protocol (AGTP). Plugin developers write handler classes
using the same `#[AgtpEndpoint]` attribute they would in any other
PHP context.

Pairs with two Composer packages from the [`agtp-php`][agtp-php-repo]
repo:
- [`agtp/agtp-php`][agtp-php] — the language library
- [`agtp/mod-php`][mod-php] — the runtime client (wrapped by the
  `wp agtp serve` WP-CLI command)

AGTP runs on its own port (4480) via `agtpd`. This plugin is the
WordPress-side worker that connects to it. WordPress's HTTP request
pipeline is unaffected — visitors hitting your site over HTTP still
get the normal WordPress experience. The reference `agtpd` daemon
(Python) lives in the [AGTP spec repo][spec-repo].

## Requirements

- WordPress 6.4+
- PHP 8.1+
- WP-CLI
- `agtpd` running locally or on the same host

## Install

```bash
# In your WordPress site root (or wherever Composer lives):
composer require agtp/agtp-wordpress
wp plugin activate agtp-wordpress
```

If your site isn't Composer-managed, install a release ZIP that
bundles `vendor/` from the plugin's release page.

## Writing handlers

Two patterns; pick the one that matches your plugin's style.

### Pattern 1: filter (simplest)

In your own plugin file, hook the `agtp_register_handlers` filter
and return your handler class names:

```php
add_filter('agtp_register_handlers', function (array $classes) {
    $classes[] = \MyPlugin\Agtp\PostHandlers::class;
    return $classes;
});
```

`agtp-wordpress` will instantiate each class with no arguments and
register every method tagged `#[AgtpEndpoint]`.

### Pattern 2: action (when you need DI)

If your handler class takes constructor arguments, listen for
`agtp_init` and call the registry directly:

```php
add_action('agtp_init', function () {
    $service = new \MyPlugin\PostService(get_post_meta_cache());
    \Agtp\HandlerRegistry::default()->registerInstance(
        new \MyPlugin\Agtp\PostHandlers($service)
    );
});
```

### The handler class itself

```php
<?php
namespace MyPlugin\Agtp;

use Agtp\AgtpEndpoint;
use Agtp\EndpointContext;
use Agtp\EndpointError;
use Agtp\EndpointResponse;

final class PostHandlers
{
    #[AgtpEndpoint(
        method: 'QUERY',
        path: '/posts',
        errors: ['post_not_found'],
    )]
    public function listPosts(EndpointContext $ctx): EndpointResponse|EndpointError
    {
        $limit = (int) ($ctx->input['limit'] ?? 10);
        $posts = get_posts(['numberposts' => $limit]);
        if ($posts === []) {
            return new EndpointError('post_not_found', 'No posts.');
        }
        return new EndpointResponse(body: [
            'posts' => array_map(
                fn (\WP_Post $p) => [
                    'id'    => $p->ID,
                    'title' => $p->post_title,
                ],
                $posts,
            ),
        ]);
    }
}
```

## Running the worker

```bash
wp agtp serve --gateway-socket=/var/run/agtpd/gateway.sock
```

Production: wrap this in a systemd unit so it restarts on failure:

```ini
[Service]
Type=simple
User=www-data
ExecStart=/usr/local/bin/wp --path=/var/www/site agtp serve \
    --gateway-socket=/var/run/agtpd/gateway.sock
Restart=on-failure
RestartSec=5s
```

For higher concurrency, run multiple unit copies — `agtpd` accepts
multiple module connections.

## Testing handlers

Use `\Agtp\Testing` to exercise handlers as plain functions.
WordPress's bootstrap is not required for unit tests of pure handler
logic.

```php
public function testListPosts(): void
{
    $ctx = \Agtp\Testing::makeContext(input: ['limit' => 3]);
    $response = \Agtp\Testing::assertOk((new PostHandlers())->listPosts($ctx));
    $this->assertArrayHasKey('posts', $response->body);
}
```

## What this plugin does not do

- Does not route AGTP traffic through WordPress's HTTP pipeline.
- Does not expose handlers to anonymous traffic. Authentication
  happens at `agtpd`.
- Does not provide a settings UI for endpoints — handlers are PHP
  code in your own plugins.

## Related

- [AGTP spec repo][spec-repo] — drafts, `agtpd` reference daemon,
  cross-language conformance tests
- [Server-modules architecture][arch] — daemon / module / library
  layering
- [`agtp-php`][agtp-php-repo] — handler SDK + `mod_php` runtime
- [`agtp-drupal`][drupal], [`agtp-symfony`][symfony],
  [`agtp-laravel`][laravel] — sibling framework integrations

[agtp-php]: https://packagist.org/packages/agtp/agtp-php
[mod-php]: https://packagist.org/packages/agtp/mod-php
[agtp-php-repo]: https://github.com/nomoticai/agtp-php
[spec-repo]: https://github.com/nomoticai/agtp
[arch]: https://github.com/nomoticai/agtp/blob/main/docs/architecture/server-modules.md
[drupal]: https://github.com/nomoticai/agtp-drupal
[symfony]: https://github.com/nomoticai/agtp-symfony
[laravel]: https://github.com/nomoticai/agtp-laravel
