---
layout: post
title: "Jess, ants, modules, and conflicts"
comments: true
category: blog
date: 2012-03-01
tags: [jess, rule engine]
share: true
---

After attending the last two editions of [Rules Fest](http://rulesfest.org), I was unable to make it to San Francisco this year. The conference is a chance to meet the people behind the algorithms and technologies being used in expert systems. At least the last two years, it was a small enough conference where sitting down and chatting with the expert systems experts was possible. During a panel session last year, the discussion drifted into the differences between [Jess](http://www.jessrules.com/) and [Drools](http://www.jboss.org/drools) and it was mentioned that Drools did not have support for what Jess called modules. Besides a few cursory projects, I haven't really used Drools, so I may be mistaken, and with Drools development advancing at such a furious pace it may have already added that functionality. Anyway, having never used modules in any of my Jess projects, I fired up Jess and decided to take a gander at modules.

<!-- more -->

I remembered trying out an [Salient Ant Simulator](http://www.jessrules.com/jesswiki/view?SalientAntSimulator) Jess program which was part of a series of posts by [Jason Morris](http://zen-of-jess.blogspot.com/) on the proper way to use salience. He mentions three cases where salience can safely be used:

 1. Stratifying the rule base into different classes of rules that are intended for specific tasks.
 2. Making a particular rule have priority over all other rules.
 3. Preventing a utility rule from firing until the rest of the program has finished running.

The simulator example code shows how to use salience to partition rules into groups. Since that is what Jess modules do, I thought it a great example to tweak and gain a better understanding modules.

The basic premise of the simulator is that all ants exhibit the same behaviors. A full explanation can be found on the [Jess Wiki][good salience], I'll just point out the details relevant to this exercise. An ant is stimulated by food, therefore an ant will gather food if there is any available. If there is a lack of food, an ant will do chores, like take out the garbage. If a hostile ant approaches the ant colony, the ant drops everything and goes off to fight for glory. Food appears at any time in the ant colony while no new garbage is ever produced.

With this definition in place, we begin to see how to organize the rules into modules for specific tasks. We'll have a work module for food gathering, a chore module for taking out the trash and a threat module to go off and fight for the colony.

The threat module is the simplest to implement. When there is an enemy sighting, the ant should go into threat mode. This is accomplished by declaring the rule with the ```auto-focus``` property set to true.

{% highlight cl %}
(defmodule THREAT)
(defrule attack-enemy-ant
    (declare (auto-focus TRUE))
    ?ant <-(enemy-ant ?)
    =>
    (printout t "Enemy appeared... Will attack." crlf)
    (change-ant-environment)
    (attack-ant ?ant))
{% endhighlight %}

When the rule is activated, the threat module is pushed onto the focus stack. Once the rule has fired, if there are no activations in the threat module agenda, i.e. no more enemy ants have appeared, the module pops off the stack and the ant continues with its previous activity before it was so rudely interrupted.

When in work mode, our ant will gather food until food sources are exhausted.

{% highlight cl %}
(defmodule WORK)
(defrule gather-food
    ?food <- (food-source ?)
    =>
    (change-ant-environment)
    (gather-food ?food))
{% endhighlight %}

When in chore mode, an ant will dutifully take out the garbage. After taking out the garbage the ant will check and see if more food has appeared via the ```there-is-food``` rule. If no food has appeared, the ant keeps doing its chores.

{% highlight cl %}
(defmodule CHORE)
(defrule take-out-garbage
    ?item <-(garbage-source ?)
    =>
    (change-ant-environment)
    (take-out-garbage ?item))

;; Return to gathering food.
(defrule there-is-food
    (exists (food-source ?))
    =>
    (printout t "Still food silly.")
    (focus WORK))
{% endhighlight %}

To start the simulation we create a few food sources, add some garbage sources, and push the chore and work module onto the focus stack.

{% highlight cl %}
(deffacts ant-environment
    (food-source 1)
    (garbage-source 1)
    (food-source 2)
    (garbage-source 2)
    (food-source 3)
    (food-source 4)
    (food-source 5))

;; Run our little ant world
(reset)
(focus WORK CHORE)
(run-until-halt)
{% endhighlight %}

Let's see what happens if we run a simple simulation with a static environment, no new food sources and no enemies.
First, the ant happily gathers all the food sources. With no food left, the food module pops off the focus stack and the chore module gets focus. The ant takes out the garbage until there is no more trash. Once the ant finishes its chores, the chore module pops off the stack and the main module get focus. At this point there are no activations and the simulation ends.

Let's make the simulation little more interesting and in the process catch a glimpse of the power of expert systems in action. The ```(change-ant-environment)``` function randomly changes the environment. A new food source or enemy ant can appear at any moment.

{% highlight cl%}
;; Have nature disturb the environment in a few ways
(deffunction change-ant-environment()
    (if (>= (/ (random) ?*max*) 0.25) then
        (assert (food-source (gensym*))))
    (if (>= (/ (random) ?*max*) 0.9) then
        (assert (enemy-ant (gensym*)))
        (printout t "Enemy Ant has appeared." crlf)))
{% endhighlight %}

Now if we run our simulation we should expect our ant to happily gather food until either an enemy ant appears or it runs out of food.
If an enemy ant appears, the ant will go off to fight. If the ant survives it goes back to gathering food.
An interesting scenario happens when the ant is taking out the trash.
One of two things can happen to change its plans; An enemy ant appears or a new food source appears.
If a new food source appears, the work module is pushed onto the focus stack and the ant begins gathering food.
If an enemy ant appears at the same time as a new food source, or a new food source appears while the ant is off fighting, according to our ant world, the ant should gather food on its return. Let's see what exactly is happening under the hood under this scenario. Once the ant is done fighting and the work module gets focus, the ```there-is-food``` rule will fire and push the work module into focus. The ant leaves the remaining garbage and begins gathering food. Or does it?

Due to the undefined order of activations firing in Jess and the Rete algorithm in general, it's possible that our ant will return from battle and first take out the garbage before gathering available food. While our hero is off fighting for the colony, the chore module has added an activation of ```there-is-food``` to the agenda. If there are also activations of ```take-out-garbage``` on the agenda we can't be sure what activation will fire first. Our ant could get stuck taking out the trash while there is food available. We need a mechanism to assure us that ```there-is-food``` fires before ```take-out-garbage```.

Determining the order in which activations are fired is called conflict resolution. Jess first looks at rule priorities, called salience.  Unless explicitly set, each rule has a default salience of 0. In the case where activations have the same salience, by default Jess will fire the most recently activated rule. Jess comes with two conflict resolution strategies, depth and breadth. In the depth strategy, the most recently activated rules will fire (LIFO). If using breadth strategy, the rules are fired in the order they were activated (FIFO).
In our example, since no new garbage sources are ever asserted, ```there-is-food``` is guaranteed to fire before any ```take-out-garbage``` activations, since ```there-is-food``` will always be the most recently activated rule in the chore module.

Let's think about why this is true in our example. If we are in the chore module, it means there are no food sources available. Since by definition of our environment no new garbage sources appear, all the ```take-out-garbage``` activations are added to the agenda once the chore module gets focus. If a new food source appears, ```there-is-food``` will be added to the agenda. Since Jess uses a LIFO agenda by default, ```there-is-food``` will always fire before any of the existing ```take-out-garbage``` activations. This immediately pushes the work module onto the stack, leaving the chore module agenda with the ```take-out-garbage``` activations still in the queue, waiting to be fired until the module regains focus. In our example, because of the peculiarities of the world we defined and existing default settings, rules will always fire in the correct order and our ant will always behave correctly.

In large production systems, assumptions on the firing of rules become dangerous as the system evolves and the rules grow more complex. Say we left our simulator as is, everything works perfectly until one day a [myrmecologist] comes along and tears our super ant simulator apart. He says "Hey, what kind of environment does this ant live in anyway. No garbage is ever produced? That's not right, ants create tons of garbage. You need to model that phenomenon." Everyone agrees on the oversight, so we dig into the code we haven't touched in months and change the ```change-ant-environment``` function to model the phenomenon of garbage appearing in the ant world. The new and improved ant simulator is released to the world.

All of a sudden the ants are behaving all weird. They're picking up garbage even when food is available. What happened? None of the important ant behavioral rules were changed. The answer is, there is no problem. That is the nature of expert systems. Remember the undefined nature of activations firing in the conflict resolution strategy, that's where our strange behavior is manifesting itself. Since new garbage sources now appear in our ant world, ```there-is-food``` is not always guaranteed to be activated before ```take-out-garbage```. The seemingly random ant behavior is just a function of the order in which food and garbage sources appear in the world.

In our ant simulator, we really do need to be certain ```there-is-food``` fires before ```take-out-garbage``` since it's part of the definition of our world. In fact, we need ```there-is-food``` to have priority over all other rules in order to simulate ant behavior correctly. This corresponds directly to the second point on good uses of salience made by Jason, _Making a particular rule have priority over all other rules_. We should use salience to achieve this. By giving ```there-is-food``` a higher salience than ```take-out-garbage``` we make sure that the correct order is preserved [^1] and all is right in our ant world. Salience can be used as _a legitimate mechanism for guiding the behavior of rules within individual modules -- a powerful concept_ [^2].

To conclude, this exercise helped me better understand modules, Jess conflict resolution strategies, and salience. In the future I really want to dig deeper into the inner workings of Jess (and Rete in general), so hopefully I'll find the time to experiment with custom conflict resolution strategies. Another lesson learned is to be careful when using modules and explicitly changing the focus stack. It's possible to forget about activations that are in the module you're leaving. If the module never regains focus, the activations will be lost and the rules will never fire. In our ant simulator that's the exact behavior we want. We want the chore module agenda to act as a sort of stash which stores all ```take-out-garbage``` activations until we're ready to use them. The way the program is structured guarantees that the chore module will gain focus before the program ends. Take a look at the [full source code][simulator source].

[good salience]: http://www.jessrules.com/jesswiki/view?GoodSalience "Jess Wiki: Good Salience"
[myrmecologist]: http://en.wikipedia.org/wiki/Myrmecology
[simulator source]: https://gist.github.com/2212515
[^1]: Unless you changed the conflict resolution strategy.
[^2]:[Jess Wiki: Good Salience][good salience]
