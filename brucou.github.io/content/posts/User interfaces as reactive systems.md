---
title: "User interfaces as reactive systems"
date: 2017-06-11
lastmod: 2018-01-07
draft: false
tags: ["functional programming", "reactive programming", "user interface"]
categories: ["programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# Background
As the name suggests, user interfaces allow a user to interface with other systems, with the idea that this interface will present some sought-for advantages vs. direct interaction with the mentioned systems.  
For instance, when the interaction between systems becomes too complex (too many commands or sequences of commands, too many parameters, outputs difficult to exploit in their raw form, etc.), a user interface can help to reduce the cognitive load and the risk of errors on the user side. As a trivial example, while it is possible for a travel agent to interact directly with the reservation system, doing so through a well-designed user interface allows to have agents with little knowledge of the intricacies of the underlying systems. They can focus instead on mastery of their specific domain (travel packages, customer segments, etc.) and the corresponding user stories, reflected in the user interface (book a flight, cancel a hotel, etc.).

The user expresses an intent through some input means (key presses, vocal entry, etc.) with a view to realize pre-existing actions on the interfaced systems. Hence user interfaces are reactive systems almost by nature.

They are additionally concurrent systems, as the user can continue to send orders through the user interface without waiting for the currently processed order to be completed. As such, user interface programming has the added complication of handling concurrency, both at a system and user interface level.

Any specification and implementation techniques for user interfaces must then specify a correspondence between user inputs and system actions, and the concurrent behaviour of the system. We will review thereafter specification and implementation techniques.

# Specification and implementation techniques
In all what follows, it will be implicit that `event` and `action` are representations of the corresponding entities, i.e. not the actual events or actions, whatever that be, but symbols for those.

Functional and technical specifications for reactive systems must allow to extract :

1. an interface by which the reactive system receives its events
2. a relation between events and actions such that $event \sim action$, where
	- $\sim$ is called here the reactive relation
	- event is an event received through the user interface and triggering an action. Events can be
		- user-initiated
		- system-initiated i.e. generated by the environment or external world
3. an interface with the systems through which actions intended by the user must be performed

Because most reactive systems are stateful, the relation $\sim$ is not in general a **deterministic** (the same output associated for the same input, as opposed to probabilistic functions) **function** (which can only associate **ONE** output to an input).

In addition to this, many frameworks for implementing user interfaces (`Angular2`, `Ember`, 
`Ractive`, `React`, etc.) make use of callback **procedures**[^defProcedure], or event handlers, which, when triggered by an event, perform the corresponding action. A few important issues stem from that choice : 

- having the procedure directly handling in the same bundle different concerns such as input validation, performing triggered actions, error management, updating local state, makes it harder to trace, debug and reason about the program execution. The concurrent nature of user interfaces magnifies this issue. 
- procedures also do not compose as easily as pure functions, which limits the use of componentization techniques to reduce complexity and increase reuse. 
- while generally optimized and efficient, the low-level description such frameworks offer is very far from the specification of the systems, making implementation error prone and certification activities very difficult.

However, when the reactive relation can be made into a deterministic function, by separating events and actions from their representation, and making local state explicit, miscellaneous implementation methods borrowing at various degrees from functional programming techniques, have some nice characteristics. We will focus in what follows on such implementation techniques.

To that purpose, we identify three models, which will be detailed thereafter :

- a reactive function gives for each event both the local state update, and the action to perform (`Elm` model)
- a reactive function expressed as the transition function for an automata (state machine model)
- a reactive function specified as a set of concurrent equations (the formal model)

In any case, a reactive system run leads to traces which are the sequence of `(events, actions)` 
which have occurred during the period of the run. To a correct behaviour of a reactive system 
corresponds a set of admissible traces. Conversely, testing a reactive system consists in 
invalidating actual traces vs. the set of admissible traces. We will finish with addressing the 
visualization of such traces.

[^defProcedure]: In the present context, a procedure is a routine, i.e. a sequence of statements, written against a runtime system, which executes them. Such procedures may or may not take arguments, may or may not return values, and may or may not perform side-effects.

## Reactive system as a joint specification of local state updates and actions
Most of the time, it is possible to formalize a state for the reactive system such that : $(action, new\ state) = f(state, event)$ where :

- $f$ is a pure function
- $state$ subsumes all the variability resulting from the environment and the reactive system's specifications, so that $f$ is pure
- $f$ will be termed here as the reactive function

If we index time chronologically by a natural integer, so that the index $n$ correspond to the nth event occurring, we have :

- $(action\_n,\ state\_{n+1}) = f(state\_n,\ event\_n) $ where :
	- $n$ is the $n$th event processed by the reactive system
	- $state_n$ is the state of the reactive system **when the $n$th event is processed**
	- we hence have an implicit temporal relation here between the event occurrence and the state used to compute the reaction of the system.

Those observations give rise to implementation techniques relying on a reactive function $f$ which **explicitly** computes and exposes for each event the new state of the reactive system, and the action to execute. Notable examples are :

- **Elm** : The `update :: Msg -> Model -> (Model, Cmd Msg)` function corresponds closely to the reactive function $f$, `Msg` to `events`, `Model` to `states`, `Cmd Msg` to `actions`.
- **Purescript/pux** : `foldp :: ∀ fx. Event -> State -> EffModel State Event fx` function is the equivalent formulation within [purescript/pux](http://purescript-pux.org/docs/architecture/) framework

### Example
Here is for example a run for the user story `User can book a trip`, for a reservation system :

| `n` | Event | State (update) | Action |
|------|-----------------------------------------|-------------------------------------------------|-----------------------------------------------------------|
| `0` | Init | is_round_trip : false today_date = '13.12.2017' | Fetch recent searches, origin list, destination list |
|  |  |  | Display initial screen |
| `1` | Click on `Return ticket` | is_round_trip : true | Update screen with `Return Departure` enabled |
| `2` | Click on `From` dropdown |  | Update screen to display `Stations` dropdown |
| `3` | Select `Vienna` |  | Update screen to display chosen trip's origin |
| `4` | Click on `To` input field |  |  |
| `5` | Type `p` |  |  |
| `6` | Type `r` |  |  |
| `7` | Type `a` |  | Update screen with display of filtered origin list |
| `8` | Select Prague |  | Update screen to display chosen trip's destination |
| `9` | Click on `Return Departure` input field |  | Update screen with calendar initialized with `today_date` |
| `10` | Select `14.12.2017` as departure date |  | Update screen with chosen departure date |
| `11` | Click on `Search` button |  | Search API call to reservation database |
| `12` | API call completed successfully |  | Update screen with search results |

## Reactive systems as automata
By using streams to represent sequences over time, the equation is rewrote as $(actions, next(states,\ events)) = f(states,\ events)$ where :

- $f$ is a pure function, taking a stream of events to a stream of actions
- $next$ is derived from the reactive system's specification
	- i.e. specifying how the state of the reactive system changes in response to an event
- $states$ is a sequence where each value is the corresponding state of the reactive system at the time when an event is triggered, and subject to the equation :
  - $states_0$ is the initial state of the system
	- $next(states,\ events)$ is such that :
		- $states\_n = next(states\_{n-1},\ events\_{n-1})$ (1)

The $states$ stream hence has a recursive definition. Lazily-evaluated languages will allow for expressing $states$ quite naturally as a fixed point for $x \rightarrow next(x, events)$, while imperative languages will directly use equation (1).

Assuming the function $next$ have been determined, the reactive system behaviour is then entirely described by the following equations, which are those corresponding to a class of automata called state transducer[^transducer] :

[^transducer]: Finite-State Machines, http://www.cse.chalmers.se/~coquand/AUTOMATA/book.pdf, cf. p.479

- $states = next(states, events)$
	- $next$ will be termed here as the state transition function
- $actions = f(states, events)$

State transducers are state machines, which have an internal state, and on receiving an input, **may** produce an output and update their internal state. This opens the path to implementing reactive systems with state machines. This further opens the door to a well-studied set of techniques used in that area : tracing, visualization, automatic code generation, model-based testing, formal reasoning.

Knowing that state transducers are translators between input symbols and output symbols, it seems logical that they can be used for specifying and implementing reactive systems. As a matter of fact, a reactive system translates the actions of the user on the user interface to actions on the interfaced system.

A major reference for specification and implementation techniques based on the state machine formalism can be found in <em>Constructing the User Interface With State charts</em> by Ian Horrocks[^Horrocks]. 
Alternatively, an introduction to the technique can be found in this [medium article](https://medium.com/@asolove/pure-ui-control-ac8d1be97a8d)

[^Harel]: Programming reactive systems with state charts, David Harel
[^Horrocks]: [Constructing the User Interface With State charts, Ian Horrocks](https://www.amazon.com/Constructing-User-Interface-Statecharts-Horrocks/dp/0201342782) 

**TODO** : add my example of the calculator instead

### Example
Here is, extracted from Ian Horrocks' seminal book, an example of state charts specification for 
a CD player :

![CD player statechart specification](http://i.imgur.com/ygsOVi9.jpg)

## Reactive systems as a system of equations
The $states$ stream can often conveniently be broken down in several streams, each carrying a specific portion of the reactive system's state. If that split is well done, one can write :

- $states_i = next_i(states, events)$
- $actions_i = f_i(states_i, events)$

Dataflow languages, such as Lucid Synchrone, use such an approach both for specification and implementation of reactive systems[^synchronous]. The approach used by Lucid Synchrone and the like is particularly interesting for user interfaces in safety-critical domain (avionics, automotive, medicine, nuclear plants...) with stringent certification requirements. By keeping the implementation close to the specification, restricting the target implementation language and defining formal semantics for it, it is possible to guarantee liveliness and safety properties of the implemented system.

[^synchronous]: https://www.lri.fr/~sebag/Slides/Pouzet.pdf, Functional Synchronous Programming of Reactive Systems ; https://www.di.ens.fr/~pouzet/bib/chap_lucid_synchrone_english_iste08.pdf, Synchronous Functional Programming: The Lucid Synchrone Experiment

### Example
We reproduce here the coffee machine example taken from [lucid synchrone manual](https://www.di.ens.fr/~pouzet/lucid-synchrone/lucid-synchrone-3.0-manual.pdf). The description is the following. The machine may serve coffee or tea. A tea costs ten cents whereas a coffee costs five. The user may enter dimes or nickels. He can select a tea, a coffee or ask for his money back.

We see thereafter how the streams `drink` and `devolution` are equationally derived from streams 
`coin` and `button`, which are themselves derived from external systems (here the keyboard)
[^lucid]. 

[^lucid]: Note the use of recursivity to equationally define the operator `coffee` : use of `rec` and `last`. Also, the keyword `node` is used to indicate <em>combinatorial</em> streams (vs. sequential streams), which are streams whose current value depends on past values. 

```
type coin = Dime | Nickel
type drinks = Coffee | Tea
type buttons = BCoffee | BTea | BCancel

(* producing events from the keyboard *)
let node input key = (coin, button) where
match key with
    "N" -> do emit coin = Nickel done
  | "D" -> do emit coin = Dime done
  | "C" -> do emit button = BCoffee done
  | "T" -> do emit button = BTea done
  | "A" -> do emit button = BCancel done
  | _ -> do done
  end

(* Now we define a function which outputs a drink and returns some money when necessary. *)
let node coffee coin button = (drink, devolution) where
  rec last v = 0
  and present
      coin(Nickel) ->
        do v = last v + 5 done
      | coin(Dime) ->
        do v = last v + 10 done
      | button(BCoffee) ->
        do (drink, v) = vend Coffee 10 (last v)
        done
      | button(BTea) ->
        do (drink, v) = vend Tea 5 (last v)
        done
      | button(BCancel) ->
        do v = 0
        and emit devolution = last v
        done
  end

(* auxiliary : emits a drink if the accumulated value [v] is greater than [cost] *)
let node vend drink cost v = (o1, o2) where
    match v >= cost with
      true ->
        do emit o1 = drink
        and o2 = v - cost
        done
    | false ->
        do o2 = v done
    end  
```

### An imperative approach
A similar, though more relaxed approach, is taken by `cyclejs` which further collapses the $states$ stream into the $actions$ streams, by considering that state change is itself an action over the internal state of the system. The previous equations are then reduced to $actions_i = f_i(states, events)$[^cycle]. To keep the equivalence to the original formulation, this approach however requires to clearly and consistently define the timing of the internal state update action vs. actions on the interfaced external systems : the internal state update must be executed before the next event is processed[^instantStateUpdate]. In the cycle framework, `App :: Sources -> Sinks`, where `Sources` contains any necessary accessor/factory methods for the state of the system and events admitted by the system, and `Sinks` holds the actions triggered by the events.

[^cycle]: cycle however gives up on the purity of $f$ in exchange for a familiar interface for componentization - components are simply functions ; but keeps discriminating between action and action representation.
[^instantStateUpdate]: so that we maintain $states\_n = next(states\_{n-1}, events\_{n-1})$. If the $n$th event would be processed before $state_n$ is set through the equation, the $next$ function could be computing an invalid state for the reactive system.

## Visualization
A run from the aforementioned [coffee machine example](#example) may be represented more succinctly 
on a timelined diagram :

```csv
events : iccsctttscscS
state  : ++-----------
actions: fuuu---uuuuSu
         u

c : click event
s : select event
t : type event
S : search API call event/action
u : update screen event
f : fetch remote data
+ : state update
```

Implementation of reactive systems which allows to obtain the trace of a run as a sequence of `
(event, action)` (or any equivalent format) allow to reduce testing of such systems to checking an 
actual trace vs. a set of expected traces.

# Conclusion
User interfaces are reactive systems and as such can be specified by means of its events/actions interface with the external systems of interest, and a pure reactive function mapping the actions of the user on the user interface to actions on the interfaced system.  
Implementation techniques leveraging functional programming techniques, can lead to an implementation closer to the specification, easier to reason about and maintain. In the best case, implementations may be automatically generated from the specifications and amenable to formal verification.

# Recommended bibliography
- [**Synchronous Functional Programming: The Lucid Synchrone Experiment**](https://www.di.ens.fr/~pouzet/bib/chap_lucid_synchrone_english_iste08.pdf)
Chapter 1.1 is an amazing introduction to the drivers behind the rise of synchronous languages vs. regular languages for critical software(avionics etc.) : the closeness between specification and implementation allows to formally guarantee properties.

> Synchronous languages are based on the synchronous hypothesis. This hypothesis comes from the idea of separating the functional description of a system from the constraints of the architecture on which it will be executed.

- [**Functional Synchronous Programming of Reactive Systems**](https://www.lri.fr/~sebag/Slides/Pouzet.pdf)  
Rationale behind functional synchronous programming for reactive systems, domains of application, state of the art.

> Concurrency and determinism are absent but they are fundamental! Design specific languages with a limited expressive power, a formal semantics, well adapted to the culture of the field

- [markdown tables helper](https://www.tablesgenerator.com/markdown_tables)