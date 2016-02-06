---
layout: post
title:  "Helper function in Laravel for Fractal"
description: "API Responses"
date:   2016-02-06
tags: [laravel, api]
comments: true
---

### What is Fractal?

> Fractal provides a presentation and transformation layer for complex data output, the like found in RESTful APIs,
 and works really well with JSON. Think of this as a view layer for your JSON/YAML/etc.

> When building an API it is common for people to just grab stuff from the database and pass it to `json_encode()`.
This might be passable for “trivial” APIs but if they are in use by the public, or used by mobile applications
then this will quickly lead to inconsistent output.

For more on implementing the Fractal library check out the website, [http://fractal.thephpleague.com/](http://fractal.thephpleague.com/)

# Building on Fractal for Laravel

This post shows how by adding two additional [Laravel Helper Functions](https://laravel.com/docs/5.2/helpers) it is possible
to abstract all of the functionality of the Fractal library out of your controllers into two reusable helper functions. This
ensures your controllers remain clean and readable.

Below is an example of the end goal for the controller, all the `fractal` implementation has been refactor out of the controller 
behind the two helper functions:

- `collection()` is used to return paginated collections of transformed objects.
- `item()` is used for returning the single transformed objects.

Both helper functions accept a collection or single model object as their first parameter and have an addition
second optional parameter which can accept either a closure containing the transformation or a custom class transformer,
checkout the Fractal documentation for more information as to how the transformation classes and closures are implemented.

If no transformation closure or class is specified then a default transformation is applied to the object which simply returns 
the object with no transformation applied.

{% highlight php %}
/**
 * Display a listing of the resource.
 *
 * @param Request $request
 *
 * @return Response
 */
public function index(Request $request)
{
    $limit = $request->get('limit', 15);

    $projects = Project::paginate($limit)
        ->appends([
            'limit' => $limit
        ]);

    return collection($projects, function ($model) {
        return [
            'id' => $model->id,
            'name' => $model->name
        ];
    });
}

/**
 * Display the specified resource.
 *
 * @param  int  $id
 *
 * @return Response
 */
public function show($id)
{
    $project = Project::findOrFail(id($id));

    return item($project, ProjectTransformer::class);
}
{% endhighlight %}

### The Helper File 

First we will add a helper file in our application where we can add helper functions. Adding the file in the autoloader will
allow access to the functions across the application.

{% highlight bash%}
"autoload": {
    ...
    "files": [
        "app/helpers.php"
    ]
},
{% endhighlight %}

And now for the helper functions...

{% highlight php %}
<?php

if (! function_exists('collection'))
{
    /**
     * Transform Collection
     *
     * @param $model
     * @param $transformer
     *
     * @return array
     */
    function collection($model, $transformer = false)
    {
        if (false === $transformer) {
            $transformer = function($model) { return $model; };
        }

        if (! ($transformer instanceof Closure)) {
            $transformer = app()->make($transformer);
        }

        $resource = (new \League\Fractal\Resource\Collection($model->getCollection(), $transformer))
            ->setPaginator(new \League\Fractal\Pagination\IlluminatePaginatorAdapter($model));

        return (new \League\Fractal\Manager())
            ->createData($resource)
            ->toArray();
    }
}

if (! function_exists('item'))
{
    /**
     * Transform Item
     *
     * @param $model
     * @param $transformer
     *
     * @return array
     */
    function item($model, $transformer = false)
    {
        if (false === $transformer) {
            $transformer = function($model) { return $model; };
        }

        if (! ($transformer instanceof Closure)) {
            $transformer = app()->make($transformer);
        }

        return (new \League\Fractal\Manager())
            ->createData(new \League\Fractal\Resource\Item($model, $transformer))
            ->toArray();
    }
}
{% endhighlight %}
