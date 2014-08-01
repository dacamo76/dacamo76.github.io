---
layout: post
title: "Running LDA on mahout"
comments: true
category: blog
tags: [java, mahout, lda, data mining]
date: 2012-11-11
published: true
modified: 2014-01-28
share: true
---

*(Jan 28 2014): Found this old draft I never got around to polish. It's very rough, but this is a developer notebook after all, so here it goes out into the world.*  

* Install mahout
* Run as core (Create new file)
* Create sequence files from text in directory

{% highlight console %}
$ bin/mahout seqdirectory -c UTF-8 -i data/documents/2012/ -o seqfiles
{% endhighlight %}

* Create document vectors

{% highlight console %}
$ bin/mahout seq2sparse -i seqfiles/ -o normalized-3-gram -ow -a org.apache.lucene.analysis.WhitespaceAnalyzer -chunk 200 -s 5 -md 3 -x 90 -ng 3 -ml 50 -n 2 -seq
$ bin/mahout seq2sparse -i seqfiles/ -o normalized-3-gram -ow -chunk 200 --minSupport 5 --minDF 3 --maxDFPercent 90 -ng 3 -ml 50 -n 2 -seq
{% endhighlight %}

* Get dictionary size (to pass into cvb)

{% highlight console %}
$ bin/mahout seqdumper -i normalized-3-gram/dictionary.file-0 -c
{% endhighlight %}

* Create vectors in format for cvb. Creates two files. ```matrix``` has sparse vector and ```docIndex``` has mapping from numeric key to original key

{% highlight console %}
$ bin/mahout rowid -i normalized-3-gram/tf-vectors -o normalized-3-gram/tf-vectors-cvb
{% endhighlight %}

* run cvb

{% highlight console %}
$ bin/mahout cvb  -i normalized-3-gram/tf-vectors-cvb/matrix -o normalized-3-gram/cvb -k 100 -ow -x 20 -nt 640 -dict normalized-3-gram/dictionary.file-0 -dt normalized-3-gram/cvb-doc-topics-10
{% endhighlight %}

* View index

{% highlight console %}
$ bin/mahout seqdumper -i normalized-3-gram/tf-vectors-cvb/docIndex
{% endhighlight %}

* view results

{% highlight console %}
$ bin/mahout vectordump -i normalized-3-gram/cvb-10 -o prob3 -d normalized-3-gram/dictionary.file-0 -dt sequencefile -p 1 -sort true -vs 5
{% endhighlight %}

* dump docs related to topics

{% highlight console %}
$ bin/mahout vectordump -i normalized-3-gram/cvb-doc-topics-10/ -o prob5 -p 1 
{% endhighlight %}

* doc id mappings

{% highlight console %}
$ bin/mahout seqdumper -i normalized-3-gram/tf-vectors-cvb/docIndex -o doc-id-mappings 
{% endhighlight %}

* JAVA OPTS

{% highlight console %}
-Xmx12000m -server -XX:+UseParallelGC -XX:+UseParallelOldGC -Xms8000m
{% endhighlight %}

## RUN IT
{% highlight console %}
$ nohup /mahout/bin/mahout cvb  -i normalized-3-gram/tf-vectors-cvb/matrix -o normalized-3-gram/cvb-100 -k 100 -ow -nt 237731 -dict normalized-3-gram/dictionary.file-0 -dt normalized-3-gram/cvb-doc-topics-100 -x 20 &
{% endhighlight %}