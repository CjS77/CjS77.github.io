---
layout: post
title:  "Clean data binding in Angular JS"
date:   2015-06-26
header-img: img/angular_backdrop.svg
author: Cayle Sharrock
categories: angular 
---

# Clean Data binding in AngularJS - the wishlist

Databinding in Angular is easy -- every basic tutorial in angular has cool two-way binding up and running in seconds.
But if you're a polarised [ENTP](https://en.wikipedia.org/wiki/ENTP) like me, then you feel very antsy about how the tutorials
implement data binding. What I want (and took me some time to figure out) is:

* Keep all logic out of controllers and encapsulate them in services (see [this blog][1]). This helps with testing and 
  keeps your code clean.
* Use the `ControllerAs` syntax as per [John Papa's style guide][2]. The benefits are manifold, see the guide.
* [Concision](http://www.thefreedictionary.com/concision) and elegance. No boilerplate, *si vouz plait*

# What's the issue

There are two reason it took me so long to figure out how to meet all these goals. The first is that there wasn't a 
 hell of a lot on the web about this; mainly it seem because the Angular API is changing all the time (and Angular 2.0 
 looks like a completely different beast, so there's that).
 
Secondly, I either misread the Angular docs on how collections are watched (quite possible); I still have some misconceptions
  about how object referenced are managed in JS (possible), or the docs are just misleading.

I only really got to the bottom of the issue after posting a question on 
[Stack Overflow](http://stackoverflow.com/questions/31050534/whats-the-best-practice-for-binding-service-data-to-views-in-angular-1-3).
If nothing else, writing up a question really forces you to articulate your ideas on an issue which can (and in this case, did) 
lead you to more insight than many of the answers provide.

# The wrong way. Part 1

Ok, so all my logic is nicely bundled up in services and I've unit tested the hell out of them. Nice. Sexy. Easy.
 Now to bind the data to views. My first naiive approach to data binding, was like what you see in all the tutorials:
 
            /**
            * Don't do it this way - it doesn't work
            */
            angular.module('app', [])
                .service('listManager', function($log) {
                    var self = this;
                    this.items = {a: 'one', b: 'two', c: 'three'};
                    this.private = 'you should not have access to this'
                    this.counter = 1;
                    this.fetchItems = function() {
                        //e.g. from http
                        self.counter++;
                        self.items = {a: 'a' + self.counter, b: 'b' + self.counter, c: 'c' + self.counter};
                        $log.info(self.items);
                    }
            })
            .controller('listController1', ['listManager', function(listManager) {
                this.items = listManager.items;
                this.refresh = listManager.fetchItems;
            }])

The view looks like:

                <h1>Service watcher - Approach 1</h1>
                <p>This one will not update beyond the initial value</p>
                <div ng-controller="listController1 as list">
                    <button ng-click='list.refresh()'>Refresh</button>
                    <ul>
                      <li ng-repeat="item in list.items">  ({item}} </li>
                    </ul>
                </div>

You can try this out in [Plunker][3]. What you'll notice is that the view data is not refreshed. Why? Because Angular
*only sets up automagic watches on expressions written in views*. i.e. `({list.items}}`. So what happens?

 1. Ok, so Angular watches `list.items` for changes. 
 2. In the controller, this was made equal to `listManager.items` -- the service's data structure. Let's say this `items`
    object is stored in memory at address `@1000`.
 3. Groovy, the binding is set up and at first, the data is actually displayed correctly.
 4. Then the service updates it data structure. The achilles heel is the way it is done:
                   
                   self.items = {...};
 
 See the problem? `listManager.items` is now pointing to a new object at say `@2000`. That old object? `@1000`? It's
 still there, and the controller is still happily watching it. It'll watch it until Microsoft open-sources Excel. But it
  ain't ever gonna change. So the view never updates.


# It's actually a threesome and as usual, someone is left out

Part of the confusion is that the `ControllerAs` syntax moves you away from having explicit `$scope` objects. `$scopes`
were our models? Now they're gone. So now what? `ControllerAs` obfuscates the *model* in the MVC philosophy. There was 
a passing comment to this effect in this [blog post][1], but I didn't see any real conclusion on the matter.

The Angular Docs state that the model is "the single point of truth" for data. Perhaps I read to much into this 
(is it just the "truth" of values between the V and C?) but this does suggest that, if you're using `ControllerAs` syntax, 
that the controller properties are the model -- and the "single point of truth". 

So the controller and model become intertwined; though now it's really more of a View-Model (or a Facade, or a Presenter?), 
and by moving logic and state management to services, we use Angular as a V-VM/C-M (VMMC) framework.

As I discussed above, Angular handles binding between the View and ViewModel for you:
  
        View  <---- Angular manages ----> View Model / Controller

But our data lives in services (what some people call the Model. Yeah it's confusing I know, so I'll just refer to Services
from here on out). So we need to manage the binding between the View-Model/Presenter (I'll just call it the Controller, ok?
As in, the angular language construct `controller`) and our Services.

        View  <---- Angular manages ----> Controller <---- ??? ----> Services 

There are a few ways to do this.
  
# The wrong way, Part II

You can just expose the service to the view. Like this:
 
        angular.module('app', [])
            .controller('listController2', ['listManager', function(listManager) {
                //This does work, but it comes with a funny smell
                this.data = listManager;
                this.refresh = listManager.fetchItems;
            }])

The service is identical to the one above.

The corresponding view looks like

        <h1>Service watcher - Approach 2</h1>
            <p>This works, but is a bit meh. Why should I have to expose everything?</p>
            <div ng-controller="listController2 as list">
                <ul>
                  <li ng-repeat="item in list.data.items"> ({item}} </li>
                </ul>
                <h2>Private data</h2>
                <p class="private">({list.data.private}}</p>
            </div>
            
This works, but is pretty gross, because all the service data is exposed to the view. You might not want the view to
have access to your entire service. 

What this really is, is making the controller a pass-through device. Or to look at it another way, the service has now 
 become the *de facto* 'fat' controller, full of business logic.

        View  <---- Angular manages ----> Services  //Cut out the middle man 
        
# The wrong way, part III        

Alternatively, expose only the elements from the service that you want the view to access, (like we did in part 1)
 but put `$watch` expressions on each of them. Again, the service is unchanged, but the controller becomes

            .controller('listController3', function(listManager, $scope) {
                this.items = listManager.items;
                this.refresh = listManager.fetchItems;
                var self = this;
                //So much boilerplate! and performance?
                //String expressions don't seem to work very well with this / self
                $scope.$watch(function() {
                    return listManager.items
                }, function(val) {
                    self.items = val;
                })
            }) 
  
This also works, but really doesn't feel right because  

  - You add a lot of boilerplate to your controllers
  - You add significantly (asymptotically approaching double) to the number of watch expressions in your code,
     possibly presenting performance issues for large projects.
  - But mostly the boilerplate

        View  <---- Angular manages ----> Controller <---- manually added watches ----> Services
  
# A better way. (Thanks for hanging in there)

Based on the discussions in Part I, you might be thinking "well, let's change the service so that the same object is 
used all the time". 

Sounded good until I read in the Angular Docs that watch expressions on collections only fire if the reference changes,
not the contents:

<https://docs.angularjs.org/api/ng/type/$rootScope.Scope#$watch>:

>The listener is called only when the value from the current watchExpression and 
>the previous call to watchExpression are not equal (with the exception of the 
>initial run, see below). Inequality is determined according to reference inequality, 
>strict comparison via the !== Javascript operator, unless objectEquality == true 

So *it seems* like this approach is dead in the water. That is until you try it (see the [Plunker][3]). It works like a charm.
For fuck's sakes. I don't know what I'm missing, or whether the Angular docs are just wrong, or what, but this code is the
kernel of what you're after. 

Note how the `angular.copy` function can be used to change the contents of an object:

            .service('listManager2', function($log) {
                var self = this;
                this.items = {a: 'one', b: 'two', c: 'three'};
                this.private = 'you should not have access to this'
                this.counter = 1;
                this.fetchItems = function() {
                    //e.g. from http
                    self.counter++;
                    //THIS LINE IS THE KEY:
                    angular.copy({a: 'a' + self.counter, b: 'b' + self.counter, c: 'c' + self.counter}, self.items);
                    $log.info(self.items);
                }
            })

Now the original controller works just fine:

            .controller('listController4', function(listManager2, $scope) {
                this.items = listManager2.items;
                this.refresh = listManager2.fetchItems;
            })
            
We've only exposed the service features we want to, without any boilerplate or extra watches:
            
            View  <---- Angular manages ----> Controller <---- Angular manages  ----> Services
                                                                 via reference

## Mountain climbers not welcome

*because they're scalers*

The `angular.copy` approach doesn't work with scalar variables (strings, booleans, numbers etc).

However, and easy workaround is to wrap them in an object. So instead of

        service.username = returnsString();
        
use

        service.username = {
            value: returnsString(); 
        }

And carry on as before. Just remember to update your views to reference the expressions properly, e.g.: 
`({ctrl.username.value}}`.
        
# Summary

It can take a while to open up the black boxes of framework that promise a lot of magic for developers. Alas, I don't think
developers can ever get away from having to open up those boxes to figure out what is actually happening before they
become really productive. I'm not there yet w.r.t. AngularJS, but I'm getting there.
                
[1]: http://toddmotto.com/rethinking-angular-js-controllers/
[2]: https://github.com/johnpapa/angular-styleguide
[3]: http://plnkr.co/pSGZp4  

