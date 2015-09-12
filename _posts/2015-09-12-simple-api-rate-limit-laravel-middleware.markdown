---
layout: post
title:  "Simple API Rate Limiter Middleware"
description: "Laravel Middleware"
date:   2015-09-12
tags: [api, laravel, middleware]
comments: true
---

This post shows how to quickly build a very simple middleware to add a rate limit to selected routes in a
Laravel application. It makes use of Laravel's build in caching system to store the users request count, the time
to reset and returns all the necessary information to the users via http response headers, as well as handling
exceeded rate limit messages.

#### Parameters

The middleware code below makes use of three different parameters which will be factors of the rate limit.

- **Time Period** - set the time period, the period of time in which the rate limit it enforced.
- **Limit Number of Requests** - set the maximum number of requests, this is the total number of requests which 
can be made during the given time period.
- **Cost of the Request** - set a cost value relating to the api request, for example you may have an API endpoint
which is process heavy and you may wish to place an addition weight to the requests above the default value.

#### Response Headers
We need to provide API consumers information of their current rate limits, one common practice is to provide
additional HTTP headers in the response and depending on the application and personal preference there are a number
of different HTTP headers which can be returned to assist API consumers.

There are lots of blog posts, examples and suggested standards. Below is a list some of the commonly implemented
responses headers, at the end of the day it's your decision as to how you inform your uses just remember to
document it clearly within your API documentation.

Header Name	| Description
------------|------------
X-RateLimit-Cost | The cost of the request used as part of the request.
X-RateLimit-Limit | The maximum number of requests that the consumer is permitted to make per hour.
X-RateLimit-Remaining | The number of requests remaining in the current rate limit window.
X-RateLimit-Reset | The time at which the current rate limit window resets in UTC epoch seconds.
X-RateLimit-Reset-Ttl | The time-to-live until the current rate limit window resets in seconds.

For more information and real world examples take a look at someone else's API for inspiration.

- GitHub [https://developer.github.com/v3/#rate-limiting](https://developer.github.com/v3/#rate-limiting)
- Twitter [https://dev.twitter.com/rest/public/rate-limiting](https://dev.twitter.com/rest/public/rate-limiting)

#### Caching Drivers
Laravel has built-in support for many caching drivers which we can make use of, by default Laravel is setup to use
the file system as its cache storage which is fine for small applications. However this isn't really practical for a
real applications for many reasons, if you want to understand more about the benefits and disadvantages to each
caching driver just do some searching around I'm sure there are lots of opinions but in the end it comes down
to your application and the potential scale of your project.

If your using Laravel homestead as your local development environment you have both redis and memcached installed
and ready to use, just set the cache driver in your '.env' file and your good to go. Check out the documentation for
configuring laravel's [caching drivers](http://laravel.com/docs/5.1/cache) for more information.

{% highlight php %}
CACHE_DRIVER=file
{% endhighlight %}

#### Building the Rate-Limit Middleware

There are two options for setting up the new middleware we can use the artisan generator to generate the
initial template or you can simply copy the below code and add the file to the middleware folder in your project
directory.

{% highlight php %}
    php artisan make:middleware RateLimiter
{% endhighlight %}

Here is the full source for the middleware class which provides the rate limiting and sets up the response headers
to be return, depending on your given preferences you may wish to change the error messages and/or the types of
headers returned in the response.

{% highlight php %}
<?php

namespace Acme\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\Cache;

class RateLimiter
{
    /**
     * Default rate limit, maximum number of requests.
     */
    const DEFAULT_LIMIT = 1000;

    /**
     * Default request cost per request.
     */
    const DEFAULT_COST = 1;

    /**
     * Default period which the requests are limited (minutes).
     */
    const DEFAULT_PERIOD = 60;

    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request $request
     * @param  \Closure                 $next
     * @param int                       $limit   The overall request limit (defaults to 1000)
     * @param int                       $cost    The cost of the request (defaults to 1)
     *
     * @return mixed
     */
    public function handle(
        $request,
        Closure $next,
        $limit = self::DEFAULT_LIMIT,
        $cost = self::DEFAULT_COST)
    {
        // Rate limit by IP address
        $count = sprintf('api:count:%s', $request->getClientIp());
        $reset = sprintf('api:reset:%s', $request->getClientIp());

        // Add if doesn't exist, remember for the limit period
        Cache::add($reset, time() + (self::DEFAULT_PERIOD * 60), self::DEFAULT_PERIOD);
        Cache::add($count, 0, self::DEFAULT_PERIOD);

        $reset_time = Cache::get($reset, time());
        $remaining = $limit - Cache::increment($count, $cost);

        $response = $next($request);

        // Break out and return an error message
        if ($remaining <= 0) {
            $response->setContent([
                "status" => 429,
                "message" => "Rate limit exceeded"
            ])->setStatusCode(429);
        }
        
        // Set rate limit headers
        $response
            ->header('X-RateLimit-Cost', $cost)
            ->header('X-RateLimit-Limit', $limit)
            ->header('X-RateLimit-Remaining', max($remaining, 0))
            ->header('X-RateLimit-Reset', $reset_time)
            ->header('X-RateLimit-Reset-Ttl', max($reset_time - time(), 0));

        return $response;
    }
}
{% endhighlight %}

Next update **/app/Http/Kernel.php** adding the following instruction which will make Laravel aware of the new
custom middleware:

{% highlight php %}
protected $routeMiddleware = [
    'auth' => \Acme\Http\Middleware\Authenticate::class,
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
    'guest' => \Acme\Http\Middleware\RedirectIfAuthenticated::class,
    'rate' => \Acme\Http\Middleware\RateLimiter::class, // <<< add this line
];
{% endhighlight %}

To wrap things up, just apply the middleware to your routes as required:

{% highlight php %}
Route::group(['middleware' => 'rate', 'prefix' => 'api'], function () {
  // <<< add your rate limited routes
});
{% endhighlight %}

Optionally add custom limit and cost values, they can be specified as
[middleware parameters](http://laravel.com/docs/5.1/middleware#middleware-parameters).

{% highlight php %}
// limit requests to 500 and each request has a cost of 25
Route::group(['middleware' => 'rate:500,25', 'prefix' => 'api'], function () {
  // <<< add your rate limited routes
});
{% endhighlight %}