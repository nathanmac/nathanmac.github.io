---
layout: post
title:  "Sortable Trait Eloquent"
description: "Eloquent"
date:   2015-11-12
tags: [laravel, eloquent]
comments: true
---

This post shows how to quickly add sort functionality to your eloquent models, providing your consumers a
simple easy to understand interface for sorting results.

It should allow ascending and descending sorting over multiple fields, the following demonstrates the
simple functionality that can be achieved.

{% highlight shell %}
GET /apps?sort=-name,+created_at
{% endhighlight %}

In this example a list of apps sorted by descending name and ascending created datetime should be returned to the
consumer of the API.

{% highlight php %}
$collection = Apps::sort()->paginate(10);
{% endhighlight %}

If you don't specific the parameter for the sort function the input will be taken from the query
string, by default this takes the value from the sort parameter. Adding the parameter in the
function call overwrites this value.

{% highlight php %}
$sort = "name,-enabled";

$collection = Apps::sort($sort)->paginate(10);
{% endhighlight %}

Adding the SortableTrait *(see below)* to the eloquent model provides the sorting functionality, which
provides the functions seen above.

So, to get started, you should define which model attributes you want to make sortable. You may do this using
the **$sortable** property on the model. For example, let's make the name attribute of our Apps model sortable:

{% highlight php %}
<?php

namespace Acme;

use Illuminate\Database\Eloquent\Model;

class Apps extends Model
{
    use SortableTrait;
    
    /**
     * The sort parameter used in the query string
     *
     * @var array
     */
    protected $sortParameterName = 'sortBy';
    
    /**
     * The attributes that can be ordered on
     *
     * @var array
     */
    protected $sortable = ['name'];
}
{% endhighlight %}

If you are using **sort** request parameter for other purpose, you can change the name of the parameter that
will be interpreted as sorting criteria by setting a **$sortParameterName** property in your model.

And here is the source code for the sortable trait...

{% highlight php %}
<?php

namespace Acme;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Facades\Input;

trait SortableTrait
{
    /**
     * Get the sortable parameter used in the query string.
     *
     * @return array
     */
    public function getSortParameterName()
    {
        return property_exists($this, 'sortParameterName') ? $this->sortParameterName : 'sort';
    }

    /**
     * Get the sortable attributes for the model.
     *
     * @return array
     */
    public function getSortable()
    {
        return property_exists($this, 'sortable') ? $this->sortable : array();
    }

    /**
     *  Determine if the given attribute may be sorted on.
     *
     * @param string $key
     *
     * @return bool
     */
    public function isSortable($key)
    {
        return (bool) in_array($key, $this->getSortable());
    }

    /**
     * Sort
     *
     * @param \Illuminate\Database\Eloquent\Builder $builder
     * @param string|null                           $sort    Optional sort string
     *
     * @return \Illuminate\Database\Query\Builder
     */
    public function scopeSort(Builder $builder, $sort = null)
    {
        if ((is_null($sort) || empty($sort)) && Input::has($this->getSortParameterName())) {
            $sort = Input::get($this->getSortParameterName());
        }

        if (! is_null($sort)) {
            $sort = explode(',', $sort);

            foreach ($sort as $field) {
                $field = trim($field);
                $order = 'asc';
                switch ($field[0]) {
                    case '-':
                        $field = substr($field, 1);
                        $order = 'desc';
                        break;
                    case '+':
                        $field = substr($field, 1);
                        break;
                }

                $field = trim($field);

                if (in_array($field, $this->getSortable())) {
                    $builder->orderBy($field, $order);
                }
            }
        }
    }
}
{% endhighlight %}
