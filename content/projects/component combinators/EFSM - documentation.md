---
title: "EFSM combinator - documentation"
date: 2018-02-17
lastmod: 2018-02-17
draft: false
tags: ["functional programming", "reactive programming", "user interface"]
categories: ["documentation", "programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# Motivation
In a [previous article](/projects/component-combinators/efsm---the-case-for-ui-programming/), we introduced the rationale behind using state machines in the 
context of UI programming. We introduce here our state machine library in the context of our 
component combinator library for cyclejs. We finish with a roadmap including new features for the
 next iteration of the library.

# Extended finite state machine library

We propose a state machine library which features :

- streams to represent a possibly infinite sequence of data which can be obtained asynchronously
- an extended finite state machine with :
    - transitions associated to asynchronous actions with run-to-completion semantics
	- state entry actions associated to dataflow components
- state machine component is a dataflow component following `cyclejs` conventions

With this library, it is possible to orchestrate a sequence of screens according to the control logic enclosed in the state machine. The implementation of each screen takes the form of a `cyclejs` dataflow component which has: access to the state machine model; a predefined interface to the external world; and can pass actions back to the external world (for instance an action interpreter in charge of executing the actions).
This makes this library a good choice to implement wireframe flows such as the ones included for the sample application.

We will make all these assertions more clear by :

- describing the `cyclejs` approach to reactive programming, and the associated terminology
- presenting the APIs

## Reactive programming the `cyclejs` way

We have seen previously how `actions` can be related to `events` via a pure function. `cyclejs` follows a similar approach. But first some terminology :

- A **dataflow component** is a **function** which takes as only input an object, and returns another object. The input object must allow to connect and listen to event sources of interest, the output object is a representation of the actions that needs to be processed by the action interpreter. 
- In the context of `cyclejs`, the output object is a map (action type -> stream of action data). Its type is called `Sinks` (`:: HashMap ActionName (Stream ActionData)`)
- In the context of `cyclejs`, the input object shape can vary, but invariably offers an interface to listen on specific events. Its type is called `Sources`.

So we have `actions = f(sources)`, where `f` is generally not a pure function, as the mechanics of event subscriptions may feature effectful operations (read effects or else). However :

-  the `cyclejs` dataflow component formulation is sufficiently close from the purely functional reactive `actions = f(events)` to be useful in practical cases.
- the presence of a polymorphic `sources` object and read effects can be worked around at testing time by the usual techniques : mocking, stubbing, etc.
- `cyclejs` comes bundled with mocking facilities, and a few action interpreters, to effectively test and run the implemented reactive application

We will express our state machine component as a `cyclejs` dataflow component, with an extra `settings` parameter for easy parameterization : `Component :: Sources -> Settings -> Sinks`

## APIs
The APIs is composed of the state machine factory function, and a few utilities.

### `makeFsm :: Events -> Transitions -> EntryComponents -> FsmSettings -> Component`

Creates and returns a state machine component whose behaviour is defined by :

- the events handled by the state machines
- the control states of the state machine 
- the associated components to be started upon entry in the control states
- the transitions between states
- the state machine settings

It generally goes like this:

- an initial event is fired upon initialization of the state machine which leads to an initial state
- the state machine changes state as a function of the incoming events it is configured to pay attention to, and the guards and transitions defined for the current state it is in
- on entering a new state, the corresponding entry dataflow component is started
	- those entry dataflow components can have their private state, as usual, that no other component will have access to
	- however, entry dataflow components can only have a read access to their enclosing context (i.e. the model associated to the state machine)
	- entry dataflow components are not aware of and do not control their lifecycle. That is done by the state machine component
- entry dataflow components are terminated when a transition is started (**TO DO : CHECK - or when a transition is completed?**)

Going back to the volunteer application, the example implementation uses a state machine which sequences the application into 5 steps (states) to which correspond 5 screens (entry dataflow components), and controls the transition from one state to another according to configured application-specific rules.

### Events
An event has a name and is mapped to an event factory. An event factory is the function which actually creates the stream of event objects for that event name, from the `sources` and `settings` passed in arguments to the state machine data flow component. `settings` can hence be used to parameterize to some extent the event stream creation.

Event names can be any valid identifier, except the reserved identifiers used for the good behaviour of the state machine (cf. contracts section).

#### Types
- `Events :: (HashMap EventName Event) | {}`
- `Event :: Sources -> Settings -> Stream EventData`
- `EventName :: String`
- `EventData :: *`

#### Contracts
- The `EV_INIT` event is reserved and **MUST NOT** be used in user-defined event configuration (FOOTNOTE : the corresponding event name string is found in the declaration of `INIT_EVENT_NAME`)
- For the `EV_INIT` event, there **MUST** be defined at least one transition 

#### Description
Events to be processed by the state machine are defined through event factories. The `events` parameter maps an event name to an event factory which once executed will return a stream of events associated to that event name.

The event factory receives the same `sources` and `settings` arguments than the state machine component, hence is injected the same dependencies than the state machine component.

**NOTE** : the stream of events returned by the factory **MAY** have an immediate starting event. However, there should only ever be one of such (`FETCH` event for instance), as all starting events for all stream of events will be executed in order of definition, right after the reserved initial event - that might not be the desired order. Also, the library does not offer the possibility for now to have immediate events triggered automatically by entering a state (no transient state, no immediate preemption).


### Transitions
The `Transitions` data structure is the most complex object and encodes most of the behaviour of the state machine. It is a hashmap whose each entry encodes the logic for a given transition between an origin state and a target state.

Every such entry has a name and is mapping to an object holding :

- the origin state for the transition
- the event that triggers the transition
- a data structure which contains the set of possible target states and the conditions necessary to select a target state to transition to.

In that data structure, for each entry, we have :

- the guard that applies to the triggering event.
- an action request to execute prior to transitioning to a target state (optional)
- a data structure which aims at defining the logic to apply according to the execution of the action request. This includes 
	- a guard applied to the action response
	- the target state to transition to if the guard is fulfilled
	- a model update function which gives the updates to apply to the state machine encapsulated model

**Event guards** : Predicate function which takes the event data for the triggering event, the current value of the state machine's encapsulated model, and returns a boolean which enables or invalidates the associated transition.

If no event guards are satisfied, then the state machine does not transition and remains in the same state.

**Action guards** : Predicate function which takes the action response for the triggered action request, the current value of the state machine's encapsulated model, and returns a boolean which enables or invalidates the associated transition.

Once an action request has been emitted, the state machine waits for a response. Once an action response is received, the state machine must transition to some state, otherwise it would remain stuck forever in an intermediary state. This means that at least one action guard must be satisfied, otherwise an exception will be raised. 

Action guards allow to have a treatment differentiated per the action response. A common use case is to deal with action requests which have failed. One guard can test for correct execution of the action request, and associate with a success state, another guard can test for the opposite case, and associate with an error state. 

**Model update functions** : Model update functions compute the set of update operations to perform on the model, from the triggered event data, the executed action response (if any), the current value of the model, and the `settings` passed through the state machine dataflow component.

As such, model update functions do not modify the model, but rather computes a list of update operations to perform. These are the typical CRUD operations and take inspiration from `JSON Patch` (RFC 6902) which is a format for describing changes to a JSON document. It is hence considered safer to have a model which is a valid JSON document (i.e. avoid functions and other constructs which are not JSON-serializable). This is not a hard requirement at the present, but it might be in future versions.

The rationale for this is two-pronged:
- we treat the encapsulated model as an externally inmutable object
- we can store (for debugging or else) the update operations, and replay them at will to rewind to a previous version of the model (**not implemented yet**).

**Action requests** : Action requests are specified via a data structure containing the driver responsible for interpreting the action request, and a request factory function which takes the triggered event data and the current value of the state machine model to produce the actual request. That request has three properties: `context`, `command`, `payload`. The `context` property serves to parameterize/refine the `command`, and the `payload` holds parameters for the command. 

**NOTE** : This is an arbitrary choice, we could have included `context` in the `payload`. We chose to extract it out to make the refinement meaning more apparent. For instance, we have an `update` command in the sample application, with a `userApplication` context, which means the command is `update userApplication`, and the parameters are `payload`.

**Action responses** : The state machine will be reading responses for requests on the same source name than the driver name. That is if the driver for the request is `domainAction$`, then the response will be expected on `sources.domainAction$`. That response has a specific form : it must include the original request object, so the response can be reconciliated with its originating request. The rest of the properties of the request is arbitrarily set to `response` and `err` to indicate success or failure of the request.

**NOTE** : In the current version of the library, an action request **MUST** be associated to a response. The state machine will **WAIT** for that response, hence it will block if no such response occurs.

#### Types
- `Transitions :: HashMap TransitionName TransitionOptions`
- `TransitionOptions :: Record {`
  - `      origin_state :: State // Must exist in StateEntryComponents,`
  - `      event :: EventName // Must exist in Events,`
  - `     target_states :: [Transition]`
  - `}`
- `Transition :: Record {`
  -  `event_guard :: EventGuard  | Null,`
  -  `re_entry :: Boolean | Null,`
  -  `action_request :: ActionRequest | Null,`
  -  `transition_evaluation :: [TransEval]`
  - `}`
- `TransEval :: Record {`
  -  `action_guard :: ActionGuard | Null,`
  -  `target_state :: State // Must exist in StateEntryComponents,`
  -  `model_update :: FSM_Model -> EventData -> ActionResponse -> Settings -> [UpdateOperation]`
  - `}`
- `ActionGuard :: Model -> ActionResponse -> Boolean`
- `EventGuard :: Model -> EventData -> Boolean`
- `ActionRequest :: Record {`
  - `driver :: SinkName | Null,`
  - `request :: (FSM_Model -> EventData) -> Request`
  - `} `
- `Request : Record {
    context :: *,
    command :: Command,
    payload :: Payload | Null,
  }`
- `ActionResponse :: Record {
    request : ActionRequest,
    response : * | Null,
    err : Error | Null,
  }`
- `UpdateOperation :: JSON_Patch`
- `JSON_Patch :: Op_Add | Op_Remove | Op_Replace | Op_Move | Op_Copy | Op_Test | Op_None`
- `Op_Add :: Record { op: "add", path: JSON_Pointer, value : *}`
- `Op_Remove :: Record { op: "remove", path: JSON_Pointer}`
- `Op_Replace :: Record { op: "replace", path: JSON_Pointer, value: *}`
- `Op_Move :: Record { op: "move", from: JSON_Pointer, path: JSON_Pointer}`
- `Op_Copy :: Record { op: "copy", from: JSON_Pointer, path: JSON_Pointer}`
- `Op_Test :: Record { op: "test", path: JSON_Pointer, value: *}`
- `Op_None :: {} | Null`

#### Contracts
- Any state defined in the `transitions` parameter **MUST** be corresponded with an entry in the `stateEntryComponents` parameter
- if a target state can be the same as an origin state, the `re_entry` property **MUST**be configured
- if a target state differs from an origin state, the `re_entry` property **MUST NOT** be configured
- A transition **MAY** have an action response guard
- A transition **MAY** have an event guard
- If a transition between two states have one or several action response guards, then for any incoming events, **at least ONE** of the guards **MUST** pass.
- (events and actions) guard predicates **MUST** be pure functions - or in a more relaxed manner **MAY** have side-effects which do not affect the evaluation of other guards for the same event
- model update functions MUST be pure functions
- To ensure event guards, action guards, model update functions do not update the state machine model, a clone of that model is passed to those functions (**IMPLEMENTED??**)
- All functions part of the transitions configuration must be synchronous functions (event guards, action guards, model updates)
- Run-To-Completion (RTC) semantics : a state machine completes processing of each event before it can start processing the next event
	- Additional events occurring while the state machine is busy processing an event are dropped
- FSM configured functions **MUST NOT** modify the model object parameter that they receive
- FSM configured functions **MUST** be synchronous and immediately compute and return their value
- FSM configured functions **MAY** throw, and that results in the state machine re-throwing the exception
- if `action_request` property has no driver associated, then the request field **MUST** be null (**implemented??**)
- if `action_request` property is none then `transition_evaluation.action_guard` **MUST** be Null  (**implemented??**)
- an action request **MUST** have an action response associated. Otherwise, the state machine will block without end, waiting for a response.
- if an action request features a driver name, that driver name **MUST** be found in the sources.
Otherwise, the state machine will block without end, waiting for a response. (**implemented??**)

#### Description
- The state machine starts in the `S_INIT` state, and automatically fires the special reserved `EV_INIT` event. From then onwards, the state machine behavior is determined by its user-defined configuration
- Given that the state machine is in a origin state, and given an incoming event :
  - when that incoming event does not have a configured transition associated, a warning is issued and the event is discarded. That is the case for not configured user-event but also for responses to requests (arriving for instance while the state machine is not expecting them).
  - when that incoming event has a configured transition associated, then :
     - when there are no event guards, the program behaves as if there was an event guard which is always satisfied (`always(true)` predicate)
     - when there are events guards, they are evaluated in order of declaration (i.e. array order)
         - when no event guards are satisfied, a warning is issued, and the incoming event is discarded
         - when an event guard is satisfied :
             - when there is an action request, that action request is sent, and the statemachine **WAITS** for a response
                 - when a user event arrives while waiting, a warning is issued, and the user event is discarded
                 - when a response arrives AND matches the request sent, then
                     - when there is no action guard, the program behaves as if there was an action guard which is always satisfied (`always(true)` predicate)
                     - the action guards are executed in order of declaration (i.e. array order)
                         - if no action guard is satisfied an exception is raised!
                         - if an action guard is satisfied, the state machine's model is updated and transitions to the configured target state
                 - when a response arrives AND DOES NOT match the request sent, then a warning is issued and the response is discarded
               - when there is no action request, the state machine's model is updated and transitions to the configured target state

### States
Control states are associated to state entry dataflow component factories. Those are functions which takes the current value of the encapsulated model of the state machine and return a regular dataflow component, which can be further parameterized by a `settings` parameter (optional).
Upon entry of a state, the mapped dataflow component will be executed, and its sinks (i.e. actions) will be connected to the sinks of the same name of the state machine dataflow component.

**entry component initialization** : performed when a target state has been evaluated and the target state is entered. If there is no entry component configured, then no action is taken.

**entry component termination** : performed when an action request is emitted, and the control state is left. Note that while the action is being executed, the state machine is in limbo, it is neither in the origin state, nor in the target state.

**state re-entry**: when the target state is the origin state, it is possible to configure whether the state entry component must be re-initialized (`re_entry : true`) or left as is (`re_entry : false`). I haven't found a case where one would not want to re-execute the entry dataflow component (specially given that it is terminated on leaving the state), but the configuration option will remain there for some time, should that case appear.

#### Types
- `StateEntryComponents :: HashMap State StateEntryComponent`
- `StateEntryComponent :: FSM_Model -> Component | Null`
- `FSM_Model :: *`
- `Component :: Sources -> Settings -> Sinks`
- `Sinks :: HashMap SinkName (Stream *)`

#### Contracts
- The starting state is `S_INIT` and is reserved and cannot be used in user-defined state configuration, other than to represent the starting state.
- States MAY be associated to entry actions which is the instantiation of a component (i.e. in`stateEntryComponents`)
- the `S_INIT` state MUST be associated ONLY with transitions featuring `EV_INIT` event (**NOT implemented??**).

**NOTE OF CAUTION** :: an state machine configured with an event guard such that the init event fails will remain in the initial state : the init event will not be repeated, hence the state machine will never transition.


#### Description
The states that the state machines can navigate to are necessarily the keys of the `stateEntryComponents` parameters. To each such state/key, a component is associated. That component will be initialized upon entering the state, and stopped upon exiting that state.

It should be noted that depending on the configuration of the state machine, in particular in line with the `re_entry` configuration parameter, the component may or may not be re-initialized in the special case of a state machine exiting one state and re-entering the same state.

The returned component is taking the same `sources` and `settings` parameters than the state machine component, hence is injected the same dependencies than the state machine component.

Its `sinks` output is merged with the state machine component output and should then logically be passed downstream to the sinks interpreter.


**NOTE**: to prevent unwanted modification of the internal state machine model, the `model` parameter passed to all configured state machines' functions is a clone of the internal model. This might have some effect on performance, we might consider relaxing this constraint in future versions if need be.

**NOTE**: *NOT IMPLEMENTED* :
The state machine keeps a journal of the modification of its model (history of update operations together with the transitions taken) for debugging purposes or else.


### State machine settings
#### Types
 ```
 FSM_Settings :: Record {
     initial_model :: FSM_Model
     init_event_data :: EventData
     sinkNames :: [SinkName] | []
     debug: Boolean | Null
   }
   ```

#### Description
- `initial_model` : initial value of the state machine model
- `init_event_data` : data to be included in the initial event upon initialization of the state machine
- `sinkNames` : names of the sink which will be output by the state machine. This is parameter is of paramount importance, as no sinks which are not included here will be output and processed by the state machine.
- `debug` : if set to `true`, automatically adds logging and tracing aspects to all functions used in the state machine configuration. This includes, event factory, event streams, actions, actions guards, event guards, and state entry components. 

**NOTE** : the debug is best used with functions which have meaningful names (i.e. avoid anonymous functions). This gives for more readable logs.


# Roadmap
- the state machine keeps a journal of the modification of its model (history 
of update operations together with the transitions taken) for debugging 
purposes or else
- allow action request without waiting for response
	- this allows for example to implement `forkJoin` with FSM
	- will also help implementing optimistic update (EHFSM will be great for that as we will still have to cope with the pessimistic case, so there will be error flows for any state, hence will be nice to group them hierarchically not to repeat transtiions)
- implement exit actions for cleanup purpose 
	- will have to be specified in sync with the entry action (which probably creates the resource to clean up) 
- specify event queuing, back pressure and interruption policies
- automatic events (i.e. immediate preemption or transient state)
- isolate out the output of the state machine (command passed through sinks)
- isolate out further the asynchronous actions to reuse a synchronous hierarchical state machine
	- input -> EHFSM -> command request => command manager
	- command manager orchestrates the command requests
		- drop command request if there is a command currently executing and not finished
		- queue command request to be executed after current command is done executing
		- abort current command execution and execute incoming command request
		- abort current command execution if it takes too long to return a response
		- this would require a controlling input (channel) linking the command manager to the EHFSM.
			- the EHFSM would hence have a BUSY state for all async. command which are pending execution
			- the BUSY state would transition on controlling input received from command manager : BUSY, ABORT -> return to state before BUSY, SUCCESS -> transition to next state, ERROR -> same thing
			- queuing events would have to be done at EHFSM level, but that is tricky...
				- if event 1 succeed, the FSM changes state so the queued events would be interpreted in the new state context
				- if event1 fails, the FSM might transition to an error state, so the queued events would be interpreted in the new error state context
				- etc. so it is tricky - best is to drop the events, but not always possible?
				- so keep the policy, but queue the events and execute them after the currently executed event is finished. It will be up to the machine designer to choose wisely his policies

- old policy thinking
- - policies when a state machine is in between states (expecting action response)
    - CANCEL AND FIRE : the action in course is cancelled and the event is processed
    - DROP : the events are dropped and not processed, a warning is issued (default)
    - QUEUE AND FIRE : events for that state are queued. If the transition is towards the same state, then events are dequeued and processed in order of arrival. If to another state, warning is issued and events are dropped
    - QUEUE AND DROP : events for that state are queued. If the transition is towards the same state, then queued events who happened while the action was executed are dropped, the queued events who happened after action has ended (and hence state is re-entered) are processed in order of arrival. If transition is to another state, warning is issued and events are dropped
- policy should be specified at the transition level
  - the event triggering the transition should define its queuing policy
