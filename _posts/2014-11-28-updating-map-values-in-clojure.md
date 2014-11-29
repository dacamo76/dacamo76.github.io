---
layout: post
title: "Updating map values in Clojure"
comments: true
date: 2014-11-28 7:05
category: blog
tags: [clojure]
share: true
---

This post is an experiment in stream of consciousness programming.
I will attempt to recreate my thought process, even errors.
I believe this may be more useful to my future self than having perfect code and wondering how I ever came up with that. Plus it gives me an excuse to write less than prefect posts and get [back to attaining vision]({% post_url 2014-10-07-back-to-attaining-vision %}).
So here goes.

The other day I needed to update the values of specific keys in a map.
I wanted to apply a function to the values of these keys.


As an example, say we have the following map:
{% highlight clojure %}
user=> (def m {:a 1 :b 2 :c 3 :d 4})
#'user/m
{% endhighlight %}

We want to apply an arbitrary function, say `inc`, to the values of keys `:a` and `:b`, resulting in the following map:

{% highlight clojure %}
{:a 2 :b 3 :c 3 :d 4}
{% endhighlight %}

Googling a bit, I found a post by Jay Fields, [Clojure: Apply a Function To Each Value of a Map](http://blog.jayfields.com/2011/08/clojure-apply-function-to-each-value-of.html), that did almost what I needed, but not quite.
I don't want to apply the function to all values, just a specific subset.

Next I searched [clojure.core](http://clojure.github.io/clojure/clojure.core-api.html).
The function `update-in` seem to be the closest thing to what we're looking for.
Let's try it out.

{% highlight clojure %}
user=> (update-in m [:a :b] inc)

NullPointerException   clojure.lang.Numbers.ops (Numbers.java:961)
{% endhighlight %}

That didn't quite work.
Let's take a look at the docs.

{% highlight clojure %}
user=> (doc update-in)
-------------------------
clojure.core/update-in
([m [k & ks] f & args])
  'Updates' a value in a nested associative structure, where ks is a
  sequence of keys and f is a function that will take the old value
  and any supplied args and return the new value, and returns a new
  nested structure.  If any levels do not exist, hash-maps will be
  created.
{% endhighlight %}

This updates a nested associative array, not exactly what we need.

{% highlight clojure %}
user=> (update-in m [:a] inc)
{:c 3, :b 2, :d 4, :a 2}
{% endhighlight %}

Hmm. That won't do.

Switching gears, maybe we can create a vector of updated key-values and `assoc` them onto the existing map.
Let's take all the keys we want to update the values of, create a vector of `[key updated-value]` for each and `assoc` them onto the original map.

{% highlight clojure %}
user=> (map #(vector % (inc (% m))) [:a :b])
([:a 2] [:b 3])
user=> (mapcat #(vector % (inc (% m))) [:a :b])
(:a 2 :b 3)
{% endhighlight %}

So we have a seq of `kvs`, now let's `assoc` the mappings to our original map.

{% highlight clojure %}
user=> (apply assoc m (mapcat #(vector % (inc (% m))) [:a :b]))
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

This works, but there has to be a better way.

Let's try to get rid of the ugly mapcat-vector calls.
Let's create a function for updating key-value pairs and use a for-comprehension.

{% highlight clojure %}
user=> (defn map-keys [f & ks]
  #_=>   (for [k ks] [k (f k)]))
#'user/map-keys
user=> (map-keys (comp inc m) :a :b)
([:a 2] [:b 3])
{% endhighlight %}

Oops. Let's flatten that out.

{% highlight clojure %}
user=> (defn map-keys [f & ks]
  #_=>   (flatten (for [k ks] [k (f k)])))
#'user/map-keys
user=> (map-keys (comp inc m) :a :b)
(:a 2 :b 3)
user=> (apply assoc m (apply map-keys (comp inc m) [:a :b]))
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

That gives us what we want, but depending on your tastes, may be more complicated than the original.

I'm not sure

{% highlight clojure %}
(defn map-keys [f & ks] (flatten (for [k ks] [k (f k)])))
{% endhighlight %}

is better than

{% highlight clojure %}
(defn map-keys [f & ks] (mapcat #(vector % (f %)) ks))
{% endhighlight %}

Pick your poison.

__Next try:__
Instead of creating a vector of key-values, let's have map-keys create a map and then we can merge it with the existing map.

{% highlight clojure %}
user=> (defn map-keys
  #_=>   [f & ks]
  #_=>   (into {}
  #_=>         (for [k ks] [k (f k)])))
#'user/map-keys
user=> (map-keys (comp inc m) :a :b)
{:a 2, :b 3}
user=> (merge m (apply map-keys (comp inc m) [:a :b]))
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

This gives us the right answer, but we can do better.
Let's add the `merge` into the function to make it cleaner.

{% highlight clojure %}
user=> (defn map-keys
  #_=>   [m ks f]
  #_=>   (merge m
  #_=>     (into {}
  #_=>       (for [k ks] [k (f (k m))]))))
#'user/map-keys

user=> (map-keys m [:a :b] inc)
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

Let's add optional args to function to make it more useful.

{% highlight clojure %}
user=> (defn map-keys
  #_=>   [m ks f & args]
  #_=>   (merge m
  #_=>     (into {}
  #_=>       (for [k ks] [k (apply f (k m) args)]))))
#'user/map-keys
;; Make sure we didn't introduce regressions
user=> (map-keys m [:a :b] inc)
{:c 3, :b 3, :d 4, :a 2}
;; Test function with extra args
user=> (map-keys m [:a :b] * 5)
{:c 3, :b 10, :d 4, :a 5}
{% endhighlight %}

Perfect. I know this is perfectly good Clojure code.
We are iterating over the keys and only applying the function to those keys.

From looking at the definition of `map-keys`, what we are actually doing is iterating over keys.
So let's go back to `update-in`.
If we think about it, what we really want is to call `update-in`
on each key and accumulate the results.

Ding. Ding. Ding. What we wanted all along was `reduce`.

{% highlight clojure %}
user=> (defn map-values
  #_=>   [m keys f]
  #_=>   (reduce #(update-in %1 [%2] f) m keys))
#'user/map-values
user=> (map-values m [:a :b] inc)
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

Perfect.

Now let's clean it up a little to accept functions with parameters.

{% highlight clojure %}
user=> (defn map-values
  #_=>   [m keys f & args]
  #_=>   (reduce #(apply update-in %1 [%2] f args) m keys))
#'user/map-values
;; Let's make sure we haven't broken anything.
user=> (map-values m [:a :b] inc)
{:c 3, :b 3, :d 4, :a 2}
;; And a function with args.
user=> (map-values m [:a :b] * 5)
{:c 3, :b 10, :d 4, :a 5}
{% endhighlight %}

There are many ways to implement updating keys, there is no right way.
I prefer the reduce variant as I believe it is more concise.

### Conclusion

I finally hit upon the `reduce` variant when I stepped back and and saw that what I really wanted was to call `update-in` many times and accumulate the results.

As a rule of thumb, whenever I think, _"Man, that function is almost what I need, I just wish I could apply it to multiple arguments separately and compose the results"_, I start to think the situation may be calling for a `reduce` implementation.
