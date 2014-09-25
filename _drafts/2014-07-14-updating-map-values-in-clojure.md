---
layout: post
title: "Updating map values in Clojure"
comments: true
date: 2014-7-14 7:05
category: blog
tags: [clojure]
share: true
---

The other day I needed to update certain values in a map.

{% highlight clojure %}
(def m {:a 1 :b 2 :c 3 :d 4})
{% endhighlight %}

I saw post by Jay Fields, but that did not do exactly what I needed.
The clojure function update-in seem to be a good fit.
Let's try it out.

{% highlight clojure %}
(update-in m [:a :b] inc)
NullPointerException   clojure.lang.Numbers.ops (Numbers.java:961)
{% endhighlight %}

That didn't quite work. Let's take a look at the docs.

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

This updates a neste associative array, not exactly what I need.

{% highlight clojure %}
user=> (update-in m [:a] inc)
{:c 3, :b 2, :d 4, :a 2}
{% endhighlight %}

Hmm. That won't do.
Maybe we can create a vector of updated key-values
and `assoc` them onto the existing map.
Let's take all the keys we want to update the values of,
create a vector of [key updated-value] for each
and `assoc` each onto the original map.

{% highlight clojure %}
(apply assoc m (mapcat #(vector % (inc (% m))) [:a :b]))
{% endhighlight %}

This works, but there has to be a better way.

Let's create a function for updating key-value pairs.

{% highlight clojure %}
(defn map-keys
  [f & ks]
        (for [k ks] [k (f k)]))
user=> (apply assoc m (apply map-keys (comp inc m) [:a :b]))
{[:a 2] [:b 3], :c 3, :b 2, :d 4, :a 1}
{% endhighlight %}

Thats not quite what we want.

{% highlight clojure %}
user=> (apply assoc m (flatten (apply map-keys (comp inc m) [:a :b])))
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

That gives us what we want, but is even more complicated than the original.
We gain nothing by eliminating the mapcat vector calls.

Next try:
Instead of creating a vector of key-values, let's have map-keys creata a map and merge
with the existing map.

{% highlight clojure %}
(defn map-keys
  [f & ks]
  (into {}
        (for [k ks] [k (f k)])))

user=> (merge m (apply map-keys (comp inc m) [:a :b]))
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

This gives us the right answer, but we can do better.
Why don't we add the merge into the function to make it cleaner.

{% highlight clojure %}
(defn map-keys
  [m ks f]
  (merge m
    (into {}
      (for [k ks] [k (f (k m))]))))

user=> (map-keys m [:a :b] inc)
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

Let's add optional args to function.

{% highlight clojure %}
(defn map-keys
  [m ks f & args]
  (merge m
    (into {}
      (for [k ks] [k (apply f (k m) args)]))))

user=> (map-keys m [:a :b] * 5)
{:c 3, :b 10, :d 4, :a 5}
{% endhighlight %}

Perfect. I know this is perfectly good Clojure code.
We are iterating over the keys and only applying the function to those keys.

So let's go back to `update-in`.
In Jay's update-values version, I got hung up on the fact that we were iterating
over the whole map.

From looking at the definition of `map-keys`, what we are actually
doing is iterating over keys.
If we think about it, what we really want is to call `update-in`
on each key and accumulate the results.

Ding. Ding. Ding. What we wanted all along was `reduce`.

{% highlight clojure %}
user=> (defn uv
  #_=>   [m keys f & args]
  #_=>   (reduce #(apply update-in %1 [%2] f args) m keys))
#'user/uv
{% endhighlight %}

{% highlight clojure %}
user=> (uv m [:a :b] inc)
{:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

Perfect.

Now let's clean it up a little to accept functions with parameters.

{% highlight clojure %}
(defn uv
  [m keys f & args]
  (reduce #(apply update-in %1 [%2] f args) m keys))
{% endhighlight %}

Let's make sure we haven't broken anything.

{% highlight clojure %}
(uv m [:a :b] inc)
; => {:c 3, :b 3, :d 4, :a 2}
{% endhighlight %}

And a function with args.

{% highlight clojure %}
user=> (uv m [:a :b] * 5)
{:c 3, :b 10, :d 4, :a 5}
{% endhighlight %}

There are many ways to implement updating keys, there is no right way.
I prefer the reduce variant as I believe it is more concise.
