---
layout: post
title:  "Thoughts on Angular Controllers"
date:   2015-06-07
header-img: img/angular_backdrop.svg
author: Cayle Sharrock
categories: angular
---

# Things that ocurred to me while developing my first Angular-based front end

## Controllers are constructors

I know it says this in the docs, but it's easy to forget whilst still in the giddy euphoria after experiencing two-way data binding for the first time.

*So two-way data binding is awesome but,* you can't naiively think it will still work everywhere.

    function MyController() {
       this.datathingy = getDynamicResult(); //Only called once, yo
    }

Just remember, *Controllers are constructors*, and you'll evade many of the issues where you expect variable to be updated, but they're not.

## Calling functions to create arrays in `ng-repeat` loops.

    <ng-repeat "item in calcItems()">
    ...

Uh huh. Don't do this. Angular puts a watch on the arrays in its loops. Everytime `calcItems()` is called it thinks the whole variable has changed (because it
has) and calls for a recalc, which calls the function again, which... leads to an infinite loop. This is not a subtle bug -- Angular crashes and tells you
exactly how naughty you've been, so you'll learn this lesson pretty smartly.

## But I have an expensive array calculation function...

You have an expensive function that returns an array. As we've established, you can't assign it in the controller's constructor (it will only ever be called
once) and you *really* can't put it in the `<ng-repeat>` directive.

So how do we make the array dynamic (by dynamic, I mean 'act like it has two-way data binding')?

We have two options:

 * Put a `$watch` on the variable
 * Broadcast/emit an event message whenever the variable value change.
 
{:note: .note}

<div class="note" markdown='1'>
Somewhere on StackOverflow someone suggested implementing the Observer pattern within your controller. This sounds too much like reinventing the wheel to me,
and it seems won't even work out the gate (properly) due to some more subtle issues around the specifics of the Angular object lifecycle
</div>

{:note: .note .warning}

<div class="note warning" markdown='1'>
**Update:** I've spent quite a bit more time digging into this, and I renounce (almost) everything in this post. Go to the next post for more enlightenment.
</div>

From what I've been able to gather, both options place some burden on the processing loop, so it probably needs some thought as to how to approach this for a
given use case.

As I understand it, `$watch`es are evaluated on every `$digest` cycle (i.e. pretty often), so this is probably more of a computational burden that event
broadcasts; unless the controller hierarchy is very big.

But, if the underlying value of the array is actually changing all the time, then why block up the event chain -- you've got no real choice but to bite the
bullet and pay for the CPU cycles. You may as well calculate the value on every `$digest`.

On the other hand, if the values are not being updated often (like once per session, or even every few minutes), then I see no reason to go near a `$watch` and
a broadcast is surely the way to go, even if the controller hierarchy is large.


{:note: .note}

<div class="note" markdown='1'>
*IF* you go the events route, a nice tip I picked up is to define the event names as a `constant` service in Angular. Then the event name mappings are available
everywhere you need them via Dependency Injection
</div>

{:note: .note .warning}

<div class="note warning" markdown='1'>
**Update:** I've completely changed my mind on this last point. The only real benefit of keeping event names in a map is to prevent typos
during coding -- i.e. relying on the IDE's static analysis to present you with the map fields during code completion. The benefits
of using enums, or a static maps of constants in e.g. Java don't apply here:

  * The map certainly isn't immutable; so it's not a rock-solid reference object.
  * It reduces portability of your components because they all depend on this global `EventNames` module.
  * The AngularJS source uses hardcoded strings, so... ok smart guys @Google. You win.
  
  Since static compiler support is average for JavaScript (actually, it's pretty impressive given that JS is a completely 
  dynamic language, so getting anything in your IDE is a fair achievement), I reckon, the best approach is just to
  maintain a README-like file of your event names and copy-and-paste them into code when required.
</div>