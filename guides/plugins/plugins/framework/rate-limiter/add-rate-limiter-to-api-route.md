# Add Rate Limiter to API Route

## Overview

In this guide you'll learn how to secure API routes with a rate limit to reduce the risk against bruteforce.

## Prerequisites

This guide is built upon both the [Plugin base guide](../../plugin-base-guide.md) as well as the [Dependency injection](../../plugin-fundamentals/dependency-injection.md) guide.

Furthermore you need an existing API route, to create a new one, head over to our [Add store API route](../store-api/add-store-api-route.md) guide.

## Configuring rate limit

{% hint style="danger" %}
Do not remove the default rate limits, otherwise it will throw an exception due to they are used in the core.

If you want to disable them, just set `enabled` to `false` in 
`<platform root>/src/Core/Framework/Resources/config/packages/shopware.yaml` under `shopware.api.rate_limiter`.
{% endhint %}

First of all, we have to create a new configuration file for our rate limit. In this example we named it `rate_limiter.yaml` located in `<plugin root>/src/Ressources/config/`.
The root key of the configuration is the name which has to be a unique key. In this example we named it `example_route`.

Each rate limit configuration needs the following keys:
- `enabled`: Enables / Disables the rate limit for the specific route (default value: true).
- `policy`: Possible policies are `fixed_window`, `sliding_window`, `token_bucket`, `time_backoff`. For more information check the [Symfony documentation](https://symfony.com/doc/current/rate_limiter.html#rate-limiting-policies).

If you plan to configure the `time_backoff` policy, head over to [the next part](#configuring-time-backoff-policy).
Otherwise, check the [Symfony documentation](https://symfony.com/doc/current/rate_limiter.html#configuration) for the other keys you need for each policy.

{% code title="<plugin root>/src/Resources/config/rate_limiter.yaml" %}
```yaml
example_route:
    enabled: true
    policy: 'time_backoff'
```
{% endcode %}

### Configuring time backoff policy

The `time_backoff` policy is build by Shopware itself. It enables you to throttle the request in multiple steps with different waiting times.
In this example we're going to configure a limit, which throttles the user for 10 seconds after 3 requests and starting from 5 requests it always
throttles the request for 60 seconds.

{% code title="<plugin root>/src/Resources/config/rate_limiter.yaml" %}
```yaml
example_route:
    enabled: true
    policy: 'time_backoff'
    reset: '24 hours'
    limits:
        - limit: 3
          interval: '10 seconds'
        - limit: 5
          interval: '60 seconds'
```
{% endcode %}

## Extending rate limit configuration in the DI-container

In this section we will create a small compiler pass called `RateLimiterCompilerPass`. If you are not very familiar with compiler passes,
head over to the [Symfony documentation](https://symfony.com/doc/current/service_container/compiler_passes.html).

### Creating compiler pass

{% code title="<plugin root>/src/CompilerPass/RateLimiterCompilerPass.php" %}
```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\CompilerPass;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\Yaml\Yaml;

class RateLimiterCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        /** @var array<string, array<string, string>> $rateLimiterConfig */
        $rateLimiterConfig = $container->getParameter('shopware.api.rate_limiter');

        $rateLimiterConfig += Yaml::parseFile(__DIR__ . '/../Resources/config/rate_limiter.yaml');

        $container->setParameter('shopware.api.rate_limiter', $rateLimiterConfig);
    }
}

```
{% endcode %}

As you can see, we're getting the current configuration of the rate limit from the DI-container and extend it by our `rate_limiter.yaml`
and reassign it with the merged configuration.

### Adding compiler pass to the container

Now, we have to add our compiler pass to the container. This will be done by overriding the `build()` method of
our `SwagBasicExample` plugin class. Important here is to use `Symfony\Component\DependencyInjection\Compiler\PassConfig::TYPE_BEFORE_OPTIMIZATION`
with a higher priority, otherwise it will be built to late.

{% code title="<plugin root>/src/SwagBasicExample.php" %}
```php
<?php declare(strict_types=1);

namespace Swag\BasicExample;

use Swag\BasicExample\CompilerPass\RateLimiterCompilerPass;
use Shopware\Core\Framework\Plugin;
use Shopware\Core\Framework\Plugin\Context\InstallContext;
use Symfony\Component\DependencyInjection\Compiler\PassConfig;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class SwagBasicExample extends Plugin
{
    public function install(InstallContext $installContext): void
    {
        parent::install($installContext);
    }

    public function build(ContainerBuilder $container): void
    {
        parent::build($container);

        $container->addCompilerPass(new RateLimiterCompilerPass(), PassConfig::TYPE_BEFORE_OPTIMIZATION, 500);
    }
}
```
{% endcode %}

## Implementing rate limit in API route

### Inject service

After we've configured our rate limit, we want to use it in our API route.
For this we need to inject the `Shopware\Core\Framework\RateLimiter\RateLimiter` service.

{% code title="<plugin root>/src/Core/Content/Example/SalesChannel/ExampleRoute.php" %}
```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Core\Content\Example\SalesChannel;

use Shopware\Core\Framework\RateLimiter\RateLimiter;
...

/**
 * @RouteScope(scopes={"store-api"})
 */
class ExampleRoute extends AbstractExampleRoute
{
    private RateLimiter $rateLimiter;

    public function __construct(RateLimiter $rateLimiter)
    {
        $this->rateLimiter = $rateLimiter;
    }

    ...
}
```
{% endcode %}

### Call the rate limiter

After we've injected the service into our API route, we can call the limiter in our route method.

To do this, we call the method `ensureAccepted` of the rate limiter which accepts the following arguments:
- `route`: Unique name of the rate limit, we defined in the [configuration](#configuring-rate-limit).
- `key`: Key we want to use to limit the request e.g., the client IP.

When calling the `ensureAccepted` method it counts the request for the key in the defined cache.
If the limit has been exceeded, it throws `Shopware\Core\Framework\RateLimiter\Exception\RateLimitExceededException`.

{% code title="<plugin root>/src/Core/Content/Example/SalesChannel/ExampleRoute.php" %}
```php
/**
 * @Route("/store-api/example", name="store-api.example.search", methods={"GET", "POST"})
*/
public function load(Request $request, SalesChannelContext $context): ExampleRouteResponse
{
    // Limit ip address
    $this->rateLimiter->ensureAccepted('example_route', $request->getClientIp());
    
    ...
}
```
{% endcode %}

### Reset the rate limit

Once we've made a successful request, we want to reset the rate limit for the client. 
We just have to call the `reset` method as you can see below.

{% code title="<plugin root>/src/Core/Content/Example/SalesChannel/ExampleRoute.php" %}
```php
/**
 * @Route("/store-api/example", name="store-api.example.search", methods={"GET", "POST"})
*/
public function load(Request $request, SalesChannelContext $context): ExampleRouteResponse
{
    // Limit ip address for example
    $this->rateLimiter->ensureAccepted('example_route', $request->getClientIp());
    
    // if action was successfully, reset limit 
    if ($this->doAction() === true) {
        $this->rateLimiter->reset('example_route');
    }
    
    ...
}
```
{% endcode %}