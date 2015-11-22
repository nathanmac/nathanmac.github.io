---
layout: post
title:  "Adding header and footer banners"
description: "Laravel Elixir"
date:   2015-11-21
tags: [laravel, elixir]
comments: true
---

This post demonstrates how to add banners to the generated css/js asserts as part of the laravel elixir pipeline.
For the purpose of this demonstration the following comment will be added either as a header or a footer
to the css/js assets generated as part of the elixir pipeline.

{% highlight php %}
/**
 * ABC Solutions - Copyright (c) 2015
 */
{% endhighlight %}


In order to add this as an additional step in the elixir pipeline we need to use one of the following laravel
elixir extensions. Install the laravel elixir extension using the following commands:

{% highlight bash %}
npm install --save-dev laravel-elixir-header
{% endhighlight %}

{% highlight bash %}
npm install --save-dev laravel-elixir-footer
{% endhighlight %}

Both these extension libraries for elixir are simple wrappers and rely on the underlying functionality
of the following libraries:

- laravel-elixir-header - [gulp-header](https://www.npmjs.com/package/gulp-header)
- laravel-elixir-footer - [gulp-footer](https://www.npmjs.com/package/gulp-footer)

Change the gulp file in the laravel project, first add the library as a **require** statement, now the header
or the footer function call can be added to the elixir pipeline, the order is important as it works on the generated
files from the previous stages.

### Adding a header banner (gulpfile.js)

{% highlight javascript %}

var elixir = require('laravel-elixir');

require('laravel-elixir-header');

/*...*/

elixir(function(mix) {
    mix.sass('app.scss')
        .header('/**\n * ABC Solutions - Copyright (c) <%= new Date().getFullYear() %>\n */\n');
});
{% endhighlight %}

##### Output

{% highlight css %}
/**
 * ABC Solutions - Copyright (c) 2015
 */
body{background:red}
{% endhighlight %}

### Adding a footer banner (gulpfile.js)

{% highlight javascript %}

var elixir = require('laravel-elixir');

require('laravel-elixir-footer');

/*...*/

elixir(function(mix) {
    mix.sass('app.scss')
        .footer('\n/**\n * ABC Solutions - Copyright (c) <%= new Date().getFullYear() %>\n */\n');
});
{% endhighlight %}

##### Output

{% highlight css %}
body{background:red}
/**
 * ABC Solutions - Copyright (c) 2015
 */
{% endhighlight %}





