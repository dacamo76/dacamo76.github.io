---
layout: post
title: "Jess, ants, modules, and conflicts: Revisited"
comments: true
category: blog
date: 2012-03-12
tags: [jess, rule engine]
share: true
---

In my [previous post][last post] I explored the Jess defmodule construct. Experimenting more with defmodules I ran the [simulator source] and got the following result:

{% highlight console %}
Jess, the Rule Engine for the Java Platform
Copyright (C) 2008 Sandia Corporation
Jess Version 7.1p2 11/5/2008

Module Ant Simulator
---------------------
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Garbage emptied OK
Still food silly.Enemy Ant has appeared.
Food item gathered OK
Enemy appeared... Will attack.
Enemy Ant has appeared.
Attacking enemy ant...
Ant killed by enemy :-( <Fact-17>
{% endhighlight %}

I thought I found a mistake. On the surface it looks like the ant is gathering food before going off to fight the enemy. This doesn't make any sense given that the [Jess documentation](http://www.jessrules.com/jess/docs/71/rules.html) states _when an auto-focus rule is activated, the module it appears in is automatically pushed onto the focus stack and becomes the focus module._ So when an enemy ant appears, the threat module is immediately pushed onto the stack and gains focus. Once the threat module has focus, the ```attack-enemy-ant``` rule fires. To understand why, we must understand that the rule is already on the agenda in the threat module, given that its activation is what caused the activation of the```auto-focus``` property in the first place.

<!-- more -->

So the problem must have been in the code. Well, sort of. The problem turned out to be not a problem at all, but a function of the print statements and the order in which certain functions were called. Below is the output of another simulation with similar results, this time with ```(watch all)```.

{% highlight console %}
FIRE 7 WORK::gather-food f-5
 <== f-5 (MAIN::food-source 3)
Food item gathered OK
FIRE 8 WORK::gather-food f-3
 ==> f-12 (MAIN::food-source gen16)
==> Activation: WORK::gather-food :  f-12
 ==> f-13 (MAIN::enemy-ant gen17)
==> Activation: THREAT::attack-enemy-ant :  f-13
 <== Focus WORK
 ==> Focus THREAT
Enemy Ant has appeared.
 <== f-3 (MAIN::food-source 2)
Food item gathered OK
FIRE 9 THREAT::attack-enemy-ant f-13
Enemy appeared... Will attack.
Attacking enemy ant...
Enemy ant killed :-D <Fact-13>
 <== f-13 (MAIN::enemy-ant gen17)
 <== Focus THREAT
 ==> Focus WORK
FIRE 10 WORK::gather-food f-12
 <== f-12 (MAIN::food-source gen16)
Food item gathered OK
{% endhighlight %}

From this output we can see exactly what is happening and why our output seems to be incorrect when in reality everything is functioning just as it should. In this run of the simulation the ant is in the work module gathering food. In line 4, the ```gather-food``` rule is fired, in the right hand side of the rule the ```change-ant-environment``` function is called which asserts a new food source and an enemy ant.
Line 5 shows the new food source being asserted and the rule ```gather-food``` being activated on line 6. Line 7 shows the enemy ant asserted and the rule ```attack-enemy-ant``` being activated in the threat module, immediately followed by the threat module gaining focus and conversely the work module losing focus, lines 9 and 10. Then the text "Enemy Ant has appeared." is printed and we fall out of ```change-ant-environment```.

So now we have two new activations, ```WORK::gather-food``` and ```THREAT::attack-enemy-ant``` and we are in the threat module. Since there is an activation in the threat module, that rule will be the next to fire. But not quite yet, we still haven't finished running the right hand side of the ```gather-food``` rule which fired in line 4. That rule must finish before any new rules are fired. All that's left in the ```gather-food``` right hand side is to call the ```gather-food``` function. This function is what prints out "Food item gathered OK" on line 13. After the rule finishes firing, Jess checks the agenda , finds ```attack-enemy-ant```, and fires it. This is exactly what's happening on line 14. From there you can follow the progression as the ant engages and kills the enemy. The enemy ant is retracted in line 18, and since there are no more activations in the threat module, it falls off the stack, the work module regains focus, and the activations in the work module are fired.

What I had considered a mistake is a consequence of rules calling ```change-ant-environment``` before actually doing the heavy lifting. See the ```gather-food``` rule as and example.

{% highlight cl %}
(defmodule WORK)
(defrule gather-food
    ?food <- (food-source ?)
    =>
    (change-ant-environment)
    ;(assert (food-work (gensym*)))
    (gather-food ?food))
{% endhighlight %}

The rule calls ```change-ant-environment``` before calling ```(gather-food ?food)```, this causes the new food sources and enemy ants to appear before our ant gathers the food, giving us the impression that our ant gathers food after an enemy ant appears and before going to fight for the colony.

Now we can follow what was happening in our original output. I will post it here again as reference:

{% highlight console %}
Jess, the Rule Engine for the Java Platform
Copyright (C) 2008 Sandia Corporation
Jess Version 7.1p2 11/5/2008

Module Ant Simulator
---------------------
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Food item gathered OK
Garbage emptied OK
Still food silly.Enemy Ant has appeared.
Food item gathered OK
Enemy appeared... Will attack.
Enemy Ant has appeared.
Attacking enemy ant...
Ant killed by enemy :-( <Fact-17>
{% endhighlight %}

We can see that the ant finished collecting food and was taking out the garbage. While taking out the garbage a new food source appeared and the work module was popped onto the stack, this is evidenced by line 20 text "Still food silly." which is printed in the ```there-is-food``` rule when a new food source appears while in the chore module. (Notice I forgot to add "crlf" in the print statement in the [version](https://gist.github.com/2212515/c3eb0638c72a4b6cc79a2b7553df77b7118674cb#file_there_is_food+rule) I was running at the time.)
The work module gained focus and the ```gather-food``` rule fired. While in the ```change-ant-environment``` call in the right hand side of the rule, an enemy ant appeared, the output corresponds to the remaining text on line 20. Remember, this entails the threat module receiving focus and the ```attack-enemy-ant``` rule activated in the threat module agenda. The ```gather-food``` runs finishes, outputting line 21, and immediately the ```attack-enemy-ant``` rule is fired, which is seen on line 22.

To conclude, there are a few ways to fix this seemingly strange behavior. One way is to include print statements for everything, for example, adding a print statement when a new food source is added would have cleared up the confusion. The most sensible way is make the call to ```change-ant-environment``` the last instruction in the right hand side of the rule it is called in. This way all the environment randomness happens after the rule has already completed its main objective. This experience helped me learn to debug Jess programs and more specifically, come up with useful tips on how to structure Jess programs to ease the pain of debugging.

[last post]: {% post_url 2012-03-01-jess-modules-and-ants %} "Jess, Ants, Modules, and Conflicts"
[simulator source]: https://gist.github.com/2212515 "Gist for ant simulator"
