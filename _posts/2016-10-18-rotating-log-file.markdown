---
layout: post
title:  "Log Rotation in Lumen"
description: "Logging provider for rotating debug file"
date:   2016-10-18
tags: [lumen]
comments: true
---

Quickly and simply add a rotating log file in the Lumen framework.

### Why Rotate?

> Logs are useful when you want to track usage or troubleshoot an application. As more information
> gets logged, however, log files use more disk space. Over time a log file can grow to unwieldy
> size. Running out of disk space because of a large log file is a problem, but a large log file
> can also slow down the process of resizing or backing up your virtual server. Additionally, it’s
> hard to look for a particular event if you have a million log entries to skim through. So it’s a
> good idea to keep log files down to a manageable size, and to prune them when they get too old to
> be of much use.
>
> [https://support.rackspace.com/how-to/understanding-logrotate-utility/](https://support.rackspace.com/how-to/understanding-logrotate-utility/)

To add the log rotation functionality we will use a custom service provider to configure the log instance and specify
the number of log files we should be keeping.

N.B. this method only manages the number of log files being retained, and doesn't take in to account the
file size. The other option is to use a utility tool like [logrotate](http://www.linuxcommand.org/man_pages/logrotate8.html)
to do the log rotation for you, that way you could have it mail, compress or remove the file depending on
the file size amongst other options.

{% highlight php %}
<?php

namespace Acme\Providers;

use Illuminate\Support\ServiceProvider;
use Monolog\Formatter\LineFormatter;
use Monolog\Handler\RotatingFileHandler;

class LogServiceProvider extends ServiceProvider
{
    /**
     * Configure logging on boot.
     *
     * @return void
     */
    public function boot()
    {
        $maxFiles = env('LOG_RETENTION', 5);

        // Allow the log path to be configurable
        $logPath = env('LOG_PATH', storage_path("logs"));

        $handlers[] = (new RotatingFileHandler($logPath . "/debug.log", $maxFiles))
            ->setFormatter(new LineFormatter(null, null, true, true));

        $this->app['log']->setHandlers($handlers);
    }

    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
    }
}
{% endhighlight %}

Now lets just register the provider in the `bootstrap/app.php` file and we're done.

{% highlight php %}
/*
|--------------------------------------------------------------------------
| Register Service Providers
|--------------------------------------------------------------------------
|
| Here we will register all of the application's service providers which
| are used to bind services into the container. Service providers are
| totally optional, so you are not required to uncomment this line.
|
*/

$app->register(\Acme\Providers\LogServiceProvider::class);
{% endhighlight %}

And we're all done.
