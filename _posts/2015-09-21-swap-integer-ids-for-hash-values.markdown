---
layout: post
title:  "Swap Integer IDs for Hash Values"
description: "Eloquent"
date:   2015-09-21
tags: [laravel, eloquent]
comments: true
---

By default, Eloquent uses an auto-incrementing integer as the primary key for its tables. For the most part this is
exactly what is required however for some applications it would be better for primary keys to be less predictable.
 
{% highlight bash %}
/api/v1/application/1
{% endhighlight %}

To this end here we describe how to change your primary key integer ID values to hash equivalent values without
having to alter your database structure and with minimal performance degradation or additional overhead.

{% highlight bash %}
/api/v1/application/s3BeSH52
{% endhighlight %}

In order to do this we are going to make use of a third party package which provides a method of translating the integer
values to hash values and vise versa. The package can be found here ([hashids.org](http://hashids.org/php/)) and best of
all there is a Laravel package: [github.com/vinkla/hashids](https://github.com/vinkla/hashids). Just pull in using composer:

{% highlight bash %}
$ composer require vinkla/hashids
{% endhighlight %}

Add the service provider to **config/app.php** in the providers array and you're good to go.

{% highlight php %}
'providers' => [
    ...
    Vinkla\Hashids\HashidsServiceProvider::class
]
{% endhighlight %}

Laravel Hashids requires configuration, publish the vendor assets. This will create a **config/hashids.php**
file in your app that you can modify to set your configuration.

{% highlight bash %}
php artisan vendor:publish
{% endhighlight %}

### Adding the Logic (Service Class)

For ease of maintenance we add a service class which encapsulates all the functionality and logic required to do the
encoding and decoding of the ID values. This will allow us to change the logic at a later date without having to make
modifications across our code base.

{% highlight php %}
<?php

namespace Acme\Utils;

use Vinkla\Hashids\Facades\Hashids;

class ID
{
    /**
     * Encodes an ID value to its hash value
     *
     * @param int    $value
     * @param string $config
     *
     * @return string
     */
    public static function encode($value, $config = 'main')
    {
        return Hashids::connection($config)
            ->encode($value);
    }

    /**
     * Decodes an ID value to the numeric string value
     *
     * @param string $value
     * @param string $config
     *
     * @return int
     */
    public static function decode($value, $config = 'main')
    {
        $id = Hashids::connection($config)
            ->decode($value);

        return isset($id[0]) ? $id[0] : 0;
    }
}
{% endhighlight %}

### Changing ID Values

One option is to add an accessor method to the Eloquent model, this will change the value to a hash whenever the ID value
is accessed. Below is getter method for the ID attribute which makes use of the service class defined above.
 
{% highlight php %}
/**
 * Returns and encode hash id value
 *
 * @return string
 */
public function getIdAttribute()
{
    return \Acme\Utils\ID::encode(
        $this->attributes['id']
    );
}
{% endhighlight %}

However this can cause issues, particularly when using foreign key relationships. In any case this only solves the display
issue, to get this to work correctly we need to allow users to pass hashed ID values into the system and translate
the values back to integers before database lookups occur.

The better option is to transform the ID values on input and output, to do this we add a helper function which will
handle both the transformation from a hash value and from the integer value. First we will add a helper file in our 
application where we can add helper functions. Adding the file in the autoloader will allow access to the functions
across the application.

{% highlight bash%}
"autoload": {
    ...
    "files": [
        "app/helpers.php"
    ]
},
{% endhighlight %}

And now for the helper function, adding a simple function which takes an input value and transforms the value to and from
the hash value. If the value is numeric we know it needs encoding into a hash and if not we assume that the value is a already a
hash value which needs to be translating to the numerical equivalent.

{% highlight php %}
<?php

if (! function_exists('id'))
{
    /**
     * Transforms ID value
     *
     * @param int|string $value
     * @param string     $config
     *
     * @return int|string
     */
    function id($value, $config = 'main')
    {
        if (is_numeric($value))
            return \Acme\Utils\ID::encode($value, $config);

        return \Acme\Utils\ID::decode($value, $config);
    }
}
{% endhighlight %}

That's it we're done, just use the helper function whenever an ID value is passed into the application or displayed to the
users of the application.

{% highlight php %}
// Input
$id = id(Request::get('id'));
//Output
return id($id);
{% endhighlight %}

#### Other Observations
You may notice that the service class and helper method (shown above) provides the option of setting an additional config
parameter. If you take a look at the documentation for the [github.com/vinkla/hashids](https://github.com/vinkla/hashids) you
will notice that you have the ability to set the configuration for generating the hash values. This includes setting a salt
value, hash length and the alphabet.

{% highlight php %}
'main' => [
    'salt' => 'saltstring',
    'length' => 8,
    'alphabet' => 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVZXYZ01234567890',
]
{% endhighlight %}

This is great for some applications where you have multiple ID values for different eloquent models because it allows set
different configuration settings for different models ensuring your hash values don't clash.
