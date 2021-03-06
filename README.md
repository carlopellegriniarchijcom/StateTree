# StateTree: simple javascript statechart implemenation.

# About


## What is a state chart?

A statechart is similar to a FSM (finite state machine), but extends the concept with hierarchy, concurrency, and communication.

## Why statecharts?

An application needs to determine how it should respond to user interaction based on the current state of the application.
Most applications have a lot of implicit and ad-hoc state mutation that is difficult to understand and leads to bugs.
In a simple app it is easy enough to manage by reducing state and paying attention to detail.
However, as applications become complex this gives every new feature the potential to break existing features.
Statecharts were originally created to wrangle the complexity of jet fighter software, but I think they can scale down nicely to even simple applications.
Rather than having implicit state mutation, Statecharts allow us to be very explicit about state and how it can be changed.
This leads to fewer defects, lets us reason about how our application operates, and even explain it to non-programmers.

Every feature in this library was created to satisfy a need in an [application](https://apps.facebook.com/yaptvguide/).

## Why not FSMs?

FSMs lack nested and concurrent states


## Why not routing?

Routing is often used to describe application state. However, routing has difficulty handling concurrent states, handling nested state transitions, and maintaining history of different branches.
Ember.js has taken a great approach of combining routing with a statechart. However, their code, including the underlying statemachine library does not work outside of the Ember framework.


## StateTree

StateTree is a simplification of the statechart concept. 
Our statechart is modeled as a tree (hierarchy) with multiple active branches (concurrency).
There are no library features for the broadcast communication aspect of statecharts.

StateTree was developed for managing the state of a typical client-side UI where the user is able to navigate around the entire UI.
There are few illegal state transitions in such a scenario.
Rather than have the user wire an event for every transition, you simply goto the state you would like with state.goTo()
There is one function to restrict state transitions: `onlyEnterThrough`.

As a result, StateTree is smaller and simpler to use than most alternatives.
You may find its features to be lacking.
However, switching to using a more featureful statechart library should be straightforward because StateTree should be a subset.

* Stativus is a fully-featured statechart library.
* If you are using Sproutcore or Ember, they have their own statechart library with a dependency on the frameworks.
* [statechart](https://github.com/DavidDurman/statechart), a fully-featured statechart library.

I passed the latter statechart library over because it was originally in somewhat of an abandoned state. It was undocumented and GPL licensed, but it has since been switched to an MIT license and documentation has been added.

StateTree is MIT licensed and does not require any frameworks (but does currently have a lodash/underscore dependency).

I have reviewed other libraries, but Stativus is the only one I have used.
Stativus requires you to define state transtition events, which is required for certain use cases, but burdensome for mine.
I ran into buggy or undefined behavior with Stativus where no error was thrown, but things just didn't work in the way I thought they would, and I had great difficulty tracking the issue down in the source code and decided it would be better just to roll my own solution.
In contrast, StateTree is designed to check for error conditions whenever possible and immediately throw an exception.
If you use TypeScript (or always call the library functions using the correct types), the goal is for there to be no way to use the library that leads to undefined behavior.



## Safety and TypeScript

Most other statechart libraries ask you to give a large JSON structure to describe your state chart.
JSON hierarchy looks nice, but these structures normally rely on strings that are a typo away from silent error.
StateTree instead uses setter methods because they will always fail immediately at runtime if mistyped.
StateTree leverages TypeScript to reduce bugs in the implementation.
You can also use TypeScript in your usage of this library to move some errors from runtime to compile time and also to have better autocompletion.


# Dependencies

lodash/underscore (this dependency can be removed in the future)


# Development Requirements

TypeScript


# Usage

Setup a state chart with `enter` and `exit` callbacks for different transitions.
Changing the state is done with `state.goTo()`
You do not indicate what the originating state is.
The state tree:

 * determines which concurrent substate the new state is in and moves within that concurrent substate
 * moves up the tree as necessary, exiting (invoke exit callback for) each state and setting history states.
 * moves down the tree and enters (invoke enter callback for) each state


## Example: Application & UI state

~~~~~~~~~~~~~~ {.js}
// login service
function login(){ return {onApplicationStart:function(cb){setTimeout(cb)}} }

var authenticate = makeStateTree().root.subState("authenticate")
 .enter(function(){ login.onApplicationStart(function(){ loggedin.goTo()}})

// state references to export
var tab2 = null
var openPopup = null

var loggedin = authenticate.subState("loggedin", function(loggedin){
  loggedin.concurrentSubStates()

  // can't enter loggedin by calling main.goTo(),
  // must be loggedin.goTo() while in authenticate
  .onlyEnterThrough(authenticate)

  // You might be tempted to write something like:
  //
  //   .enter(function(){ main.goTo() })
  //
  // But generally avoid goTo in an enter/exit callback.
  // Just 1 is allowed during all transitions
  // for this case we can use defaultSubState to automatically enter "main"

  .subState("main", function(main){
    main.defaultSubState()
    .subState('tab1')
      // defaultSubState will choose betweent tab1 & tab2 when we call main.goTo()
      .defaultSubState()
      .enter(function(){ UI.activateTab('tab1')})
     
    tab2 = main.subState('tab2')
     .enter(function(){ UI.activateTab('tab2')})
  }) 
  
  loggedin.subState("popup", function(popup){
    openPopup = popup.subState("open")
      .enter(function() { popupService.popup() })
    
    popup.subState("closed")
      .enter(function(){ UI.unmask() })
  })
})

// start up the application
authenticate.goTo()
// user clicks a tab
tab2.goTo()
// popup a dialog
openPopup.goTo()

~~~~~~~~~~~~~~


* loggedin has concurrent substates (main and popup)
* tab1 is the default substate of main
* When the user enters the loggedin state, then enter callback will start the main state.
* The main state will enter the tab1 state because it is the default sub state.
* The popup can be opened and closed without effecting the main state.


## History states

History states let us know the previous substate so we can easily restore previous application state.

~~~~~~~~~~~~~~ {.js}
// access the previous sub-state
state.history

// if there is a history state always go to it instead of the default state
tree.defaultToHistoryState()
~~~~~~~~~~~~~~


## Dynamic state lookup

~~~~~~~~~~~~~~ {.js}
tree.stateFromName('loggedIn')
~~~~~~~~~~~~~~


## Running code for every transition

Callbacks for all transitions can be registered with `tree.enter` and `tree.exit`


## Events

There is no event sytem.
If you want to send data with a transition, just create a wrapper function (and tie it to an event if you want).

~~~~~~~~~~~~~~ {.js}
 // wrapper function
 function goToStateA(arg1) {
   // do something with arg1
   stateA.goTo()
 } 

 // event hook: use your own event system
 myEventSystem.on('stateAEvent', goToStateA)
~~~~~~~~~~~~~~


## Limiting/Enforcing transitions

This library provides one function: `onlyEnterThrough`. See usage in the main example.

If you want to limit access, just export a new object rather than all of the states.
Instead of exporting states, you can export functions.
These can handle data and limit usage to only valid state transitions.

~~~~~~~~~~~~~~ {.js}
 // statechart definition from the main example above of using statechart ...

 // don't export authenticate or loggedin, those need to be locked down
 return {
   goToTab2: function(){ tab2.goTo() } // private, but easy to use
 , openPopup: openPopup // public
 }
~~~~~~~~~~~~~~


## Flexible state lookup

Programming in strings is not safe, but sometimes gets the job done quickly.
You can lookup a state from its name with `tree.stateFromName(name:String):State`.


## Using with AngularJS

First make sure to require the statetree.js file.

You may want to make this library a module value

~~~~~~~~~~~~~~ {.js}
angular.module('StateTree', []).value('statetree', window.makeStateTree)
~~~~~~~~~~~~~~

Then your own state service for your own application.

~~~~~~~~~~~~~~ {.js}
angular.module('myApp', ['StateTree'])
.factory('appState', ['statetree', function(statetree){
  // code from examples above goes here
}])
~~~~~~~~~~~~~~


Now you can use this service in your controller

~~~~~~~~~~~~~~ {.js}
.controller('PopupController', ['$scope', 'appState', function(){
  $scope.closePopup = function() { appState.closed.goTo() }
}]
~~~~~~~~~~~~~~


## Integrated with routing

There are already a lot of routing libraries out there, so right now I want to take the approach of writing adapters.

There is an [AngularJS adapter available](
https://github.com/yaptv/StateTree/blob/master/statetree.routes.angular.js.ts
).
See the comments at the top of the file on how to set it up and also the API.
This is all a work in progress, but it works well for me.

Here is some usage:

~~~~~~~~~~~~~~ {.js}
var mkRoute = routeGenerator(routeProvider, $location, $q, $rootScope)

showPopupState.subState("open-data-detail", function(openDataDetail) {
    var setDataId = (dataId) => dataViewer.setDataId(showId)
    var getDataId = () => [showViewer.getShowId()]
    mkRoute(openDataDetail, ['data', Number], getDataId, setDataId)
})
~~~~~~~~~~~~~~

`dataViewer` is a service that retrieves data from a service and returns a promise.
This links the state with a url `/data/:dataId`.
If the user navigates to the url, the state will be entered after waiting for the promise from the `setDataId` function.
If the state is entered programatically, it will use the `getDataId` function to change the url.


Routing now becomes a way to move around the statchart, so think carefully.
`onlyEnterThrough` will stop transitions in their tracks, but we may instead want to wait until the transition is valid.
For example, if the user must first log in, one way of dealing with that is to create a promise for the loggedin state.

~~~~~~~~~~~~~~ {.js}
var loginDefer = $q.defer()
loggedin.enter(() => { loginDefer.resolve() })

// change the definition of setDataId to require the loggedin state
var setDataId = (dataId) => $q.all([loginDefer.promise, dataViewer.setDataId(dataId)])

// or instead require every single route to wait for the login promise
var mkRoute = routeGenerator(routeProvider, $location, $q, $rootScope, loginDefer.promise)
~~~~~~~~~~~~~~

Another feature I added is watching for changes within a state by setting the watch flag to true for mkRoute.

    mkRoute(openDataDetail, ['data', Number], getParameters, setParameter, true)

Now getParameters will be called in the watch cycle and the url will be automatically updated when getParameters changes

