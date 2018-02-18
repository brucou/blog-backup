---
title: "The case for state machines in UI programming"
date: 2017-06-15
lastmod: 2018-02-17
draft: false
tags: ["functional programming", "reactive programming", "user interface"]
categories: ["documentation", "programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# Introduction
A web application's user interface can be understood as a reactive system which displays a sequence of screens and perform a sequence of commands which are determined unequivocally by user actions, the application domain logic, and contextual data from the application's environment (miscellaneous databases, sensor inputs, etc.). We aim to show that, under some restrictive conditions, using a state machine to control that sequence, allow for better modelization, specification, testing and implementation of such user interface.

We will do so in the course of three articles :

- the present article takes some distance to reflect on the pain points of human-computer user interface design and implementation. We present patterns which have been used to alleviate some of those pain points. We then show how an asynchronous extended state machine complements these approaches 
- the [second article](/projects/component-combinators/efsm---documentation/) details our proposed asynchronous extended state machine library
- the [third article](/projects/component-combinators/efsm---example-application/) illustrates the previous two articles with the implementation of the 
sample application, using our state machine library, and exemplifying how such a state machine 
implementation faithfully follows the specifications, and reacts well to changes in those 
specifications. We finish that article by exposing the restrictive conditions under which a  
state machine is a good fit for user interface modelization, and discuss various ways to create those restrictive conditions in some circumstances when they do not naturally appear.

This is the first of the three articles.

# Challenges of human-computer interface design and implementation

In addition to the difficulties associated with designing any complex software system, user 
interfaces add the following problems :

- user interfaces are reactive systems, they are hard to specify **precisely** in their entirety
- they often require managing asynchrony, and concurrency
- user interfaces design (hence implementation) is an iterative process. An initial design is refined through feedback from the relevant stakeholders
- robustness is paramount, as the user interface will frequently receive unexpected/malformed inputs
 
## Reactive systems design is hard to specify and communicate 
A reactive system, in contrast with a transformational system, is characterized by being event driven, continuously having to react to external and internal stimuli. The problem in specifying reactive systems is rooted in the difficulty of describing reactive behavior in ways that are clear and realistic, and at the same time formal and rigorous, in order to be amenable to precise computerized analysis.

The behavior of a reactive system is really the set of allowed sequences of input and output events, conditions, and actions, perhaps with some additional information such as timing constraints. What makes the problem especially acute is the fact that a set of sequences (usually a large and complex one) does not seem to lend itself naturally to 'friendly' gradual, level-by-level descriptions, that would fit nicely into a human being’s frame of 
mind[^DavidHarel].

[^DavidHarel]: Cf. [STATECHARTS: A VISUAL FORMALISM FOR COMPLEX SYSTEMS](www.inf.ed.ac.uk/teaching/courses/seoc/2005_2006/resources/statecharts.pdf), p232.

This results in discrepancies both between the intended design and the actual design (design bug), and between the actual design and the implementation (implementation bug).

## User interface design is a creative, iterative process, which means implementation is constantly changing
The most important function that user interface designers do for their clients is the iterative extraction and refinement of the user needs and their corresponding translation in the interface.

This means that implementation is also an iterative process. Ideally a small change in the design should correspond to a small change in the implementation. Costly implementation updates leads to the temptation to lower the number of iterations with end users, leading to a user interface which incorporates less feedback from the user. 

Of major value are rapid prototyping techniques, but also implementation techniques which allow for modularity, consistent patterns, and decoupling. The single responsibility principle here is key ('only one reason to change').


## Asynchrony is tractable, but concurrency is hard
The user interface software must be structured so that it can accept input events at all times (low latency), even while executing commands. Consequently, any operations that may take a long time is problematic and must be handled in separate ways. 
At the same time, processing events triggering actions while another action is executing can lead to the well-known concurrency issues : **synchronization**, **consistency**, **deadlocks**, and **race conditions**.[^accidental-concurrency]

[^accidental-concurrency]: Concurrency can also be by design instead of accidental. Take the case of a user interface where a user may be involved in multiple ongoing dialogs with the application, for example, in different windows. These dialogs will each need to retain state about what the user has done, and will also interact with each other. 

## User interface must handle the expected and the unexpected gracefully
Naturally, all software has robustness requirements. However, the software that handles the users’ inputs has especially stringent requirements because all inputs must be gracefully handled. Whereas a programmer might define the interface to an internal procedure to only work when passed a certain type of value, the user interface **must always accept any possible input, and continue to operate**.

Furthermore, unlike internal routines that might abort to a debugger when an erroneous input is discovered, user interface software must respond with a helpful error message, and allow the user to start over or repair the error and continue. To make the task even more difficult, user interfaces should allow the user to abort and undo operations. Therefore, the programmer should implement most actions in a way that will allow them to be aborted while executing and reversed after completion.


# The functional reactive programming pattern matches well the reactive nature of the system

A reactive system is described as the sequence of its reactions to inputs, that is a relation `(re)action = f(event)`, where `event` is taken from a set of events that the GUI is responsive to, and `action` is the associated reaction intended by the user/actor who triggered the event.

A functional reactive program which uses streams to abstract out asynchrony, describes the behaviour of the reactive system as `actions = f(events)` where :

- `events` is a (push)stream (i.e. a possibly unbounded sequence of data) of incoming events that the GUI responds to,
- `actions` a (push)stream of the corresponding reactions
- `f` is a pure function

For reactive systems which can be specified by their set of traces over the space of expected events,  i.e. `{(events, actions) | events = [event | Events]}`, the functional reactive paradigm allow to check  the correct behaviour of a reactive system for a given pair `(events, actions)`, by simply simulating the events sequence, and checking that the `actions` stream conforms to the specified trace.

For practical purposes, the set of traces being generally infinite, gray-box testing techniques can be used to reduce considerably the set of traces under test.

## ... but stream-based implementations often obfuscate the user interface design
However, the functional reactive programming with streams approach suffers from two key issues:

- linear, one-way dataflows are expressed easily through combining higher-order stream operators (`map`, `flatMap`), simple control flow is made possible by specific operators (`filter`, `switch`, `fold`), but complex control flow often requires ad-hoc solutions(jumping, interrupts, conditional branching, looping, etc.)
- streams operators are generally pure functions, hence any desired state must be passed explicitly throughout all the relevant part of the operator chain, and changes to that state must be propagated explicitly at a given point of the operator chain

In short, in the case of wireframe flows featuring complex control flow, the design cannot be easily and automatically reconstructed from the reactive implementation, as the control flow cannot be easily separated from other implementation concerns (state passing 
and manipulation, stream breakdown and wiring, etc.).

The loss in readability is compounded by a lower maintainability : a small change in the control flow can lead to mistakes in identifying the impacted section of the code(a.k.a. [program slicing](https://en.wikipedia.org/wiki/Program_slicing)), or 
modifying a comparatively large portion of the code, all of which increases the likelihood of errors.

The ideal implementation which supports well an iterative design process is to associate a small implementation cost to a small change in design, and minimize the risks of errors while doing so (or to sound like a mathematician, the ideal implementation is a continuous function of design).

# Extended state machines are great to specify control-driven reactive systems

## Quid est
A finite state machine (FSM) is a machine specified by a finite set of conditions of existence (called states) and a likewise finite set of transitions among states triggered by events. 

Finite state machines model behavior where responses to future events depend upon previous events. There is a rich body of academic literature in this field, but a useful working definition is straightforward. Finite state machines are computer programs that consist of:

- *Events* that the program responds to
- *States* where the program waits between events
- *Transitions* between states in response to events
- *Actions* taken during transitions

An extended state machines adds : 
- *Guards* which are predicates which enable a transition when satisfied
- *Variables* that hold values needed by actions and guards between events. The set of those variables is referred to as *model*.

The events that drive finite state machines can be external to the computer, originating from a keyboard, mouse, timer, or network activity, or they can be internal to the computer, originating from other parts of the application program, or other applications.

## From event-action paradigm to event-state-action paradigm
We already mentioned that a GUI follows an event-action paradigm. The user initiates events by interacting with the user interface, and those events are meant to produce the corresponding actions intended by the designer of the user interface. In the simplest case, we have `action = f(event)`, and action depends only of the event.

In practice, this is not always the case. The action to be performed often depends not only the triggering event but on the history of events and actions which have taken place before ; or on the context in which the event takes place. For instance:

 - a toggle button click can have two actions associated, according to its current state
 - in a cd player, the play button triggers `close the cd tray and play`, if the tray is opened, `play` if the cd player is not playing or is paused, `pause` if the cd player is currently playing
 - in a video game AI, a game agent might have a state `Patrol` wherein it wanders the map looking for enemies. Once it detects an enemy, it transitions into the `Attack` state. If the target starts taking shots at our agent, it might momentarily transition into the `TakeCover` state (a "blip") before transitioning back to `Attack`
 - A large number of enterprise applications are workflow-driven systems in which documents/files/processes transition from state to state

What these have in common is that the same event is associated to different actions depending on the context in which the event occurs. We can then rewrite our equation as `action = f(event, context)` where context modelizes somehow the relevant and necessary contextual data. In the case where we can discriminate discrete logical branching points which determine the triggered action (`isToggled`, or `carriesWeapon`, etc.), we can write this equation `(action_n+1, model_n+1) = f(event_n, state_n, model_n, transitions)`, where :

- `f` is a pure function (to have `f` be a pure function, we had to expose a `model` variable as `f` computes an action but ALSO produces updated contextual data),
- `state` is to be understood in the state machine sense (`isToggled` etc., we will call that `control state`, or sometimes `qualitative state`),
 - `model` is the contextual data necessary to parameterize the action to be triggered (we will call that `quantitative state` or simply just `model` - quite the polysemic term, but keep in mind that this is whatever extra information you need to write your reactive function as a pure function).
- `transitions` is an object which contains all transition data on the aforementioned logical branching points (origin state, target state, event trigger, guards, output action), i.e. the encoded control flow of the program

Written with streams, the equation becomes `actions = f(events, states, initialModel, initialEvent, transitions)`, where :

- `f` is a pure function,
- `transitions` is an object which contains all transition data (origin state, target state, event trigger, guards, output action), i.e. the encoded control flow of the program
- `events` is a stream of incoming events accepted by the state machine
- `states` is an object which contains all relevant information necessary to describe control states for the state machine (name, entry actions, etc.)
- `initialModel` is the initial value of the model for the state machine (`model_0`)
- `initialEvent` is the initial event to kick-off the execution of the state machine (`event_0`). The initial state (`state_0`) is fixed by default, and a transition must exist from that default initial state and the initial event so the state machine can actually be non-trivial.

**NOTE** : the model disappears from the streams formulation, as use of the model is solely inward. We have then successfully encapsulated the state (control state and quantitative state) of our state machine.

## Benefits of an extended state machine
Let's see how extended state machines help alleviate the previously mentioned pain points of reactive systems design and implementation.

### Design, implementation, test and communication
We have seen that transitions between states hence allow to express complex control flow with a simple vocabulary : state, transitions, guards. State machines are in fact akin to domain-specific languages, where inputs are events, (reactive) sub-routines are transitions, variables are the model, assignments are model updates, branching constructs are control states and guards, the actions being the output of the program.
The combination of dataflow (streams) and DSL-controlled flow allow to express a reactive design into an implementation with reasonable ease.

More importantly, the control flow is encoded in a regular object which can be automatically parsed in many ways of interest.

Among key examples, the state machine definition (`:: Record {transitions, eventNames, states, initialModel, initialEvent}`) can be :

- used by `f` to compute the actions from the events, i.e to produce an executable version of the state machine
- automatically parsed into a graphical representation of all or a slice of the behaviour (removing error handling flows for instance) of the state machine
- used to generate automatic tests for the state machine (note that this requires additional formalization (refinement of the DSL), more on this in a future section)

Going back to the [sample application](https://github.com/brucou/component-combinators/tree/master/examples/volunteerApplication), here is an example of automatizable flow graph, in which the error flows have been segregated :

![(Automatically generated flow graph)](https://i.imgur.com/dkbSwEw.png "Automatically generated flow graph")

It becomes apparent that the state machine representation closely matches and refines the wireframe 
flows obtained from the design phase. This shows how the systematic way to translate a reactive design into an implementation that is a state machine minimizes implementation bugs, by sticking to the design.

Design bugs themselves can be reduced (discrepancy between the produced design and the desired specification), as we mentioned before, via rapid prototyping, user feedback and iterating on the design.

Generative testing can be used to increase the test process automatization, reduce test implementation time, and increase confidence in the behaviour of the system.

Last but not least, [readability](https://en.wikipedia.org/wiki/Computer_programming#Readability_of_source_code) refers to the ease with which a human reader can 
comprehend the purpose, control flow, and operation of source code. A state-machine-based DSL goes a long way in communicating the intent and meaning of a reactive program.

### Asynchrony and concurrency
All state machine formalisms, universally assume that a state machine completes processing of each event before it can start processing the next event. This model of execution is called run to completion, or RTC.
In the RTC model, the system processes events in discrete, indivisible RTC steps. New incoming events cannot interrupt the processing of the current event and must be stored (typically in an event queue) until the state machine becomes idle again. The RTC model also gets around the conceptual problem of processing actions associated with transitions, where the state machine is not in a well-defined state(is between two states) for the duration of the action. During event processing, the system is unresponsive (unobservable), so the ill-defined state during that time has no practical significance.

These semantics completely avoid any internal concurrency issues within a single state machine. This eliminates a hard-to-deal-with class of bugs.

Note : The proposed implementation at this moment does not queue events. Hence, events which occurs while an action is being executed, i.e. in between states will be ignored.

### Behaviour with respect to incremental changes in design
As we discussed before, a reactive system whose behaviour is encoded in a state machine definition can be executed by a function `f` such that `actions = f(events, stateMachineDefinition)` where`stateMachineDefinition` is of type `Record {transitions, eventNames, states, initialModel, initialEvent}`.

incremental changes in the design are then translated into changes in events, states, model, and transitions. 

#### Transitions changes
If we suppose that the incremental change does not affect the model datatype and the set of control states, then that change can be expressed as a modification of transitions, i.e. origin-target mapping, or guards. The key point is that only an easily identifiable subset of the implementation is changing. Because the change is incremental, that subset is small. Because that subset is small, the surface area to re-test is also small, and the corresponding impact on implementation is small and contained.

#### Control state changes
If the incremental change in design results in a change in the set of control states, then the corresponding change in implementation cannot often be considered small. Removing a control state, for example, means removing all transitions to and from that control state, and the associated guards. That is a lot of meaning which disappears! This puts the onus on the state machine designer to come up with a set of control states such that incremental design change does not result in a change in control states. This may be achieved by domain-specific knowledge, and anticipation of the design changes.
Conversely, adding a control state leads to evaluate the possibility of transitions from any previously existing control states to the new control states. The complexity of that is obviously linked to the number of control states. Hence if that number of control states is small, then the impact from adding a new control state is likely to be small.

#### Model changes
Incremental design changes which result in changes in the model (for example adding a property to the model) are tricky, as the worse case require touching all model updates functions, guards and entry/exit actions and the retesting of the whole reactive system. However, in practice, in many cases, changes in the existing implementation can be small and a significant portion of the existing code can be reused.

#### Multiple type of changes
One can argue that incremental design changes which affect the whole state (control state and model) are not incremental. They often reflect either a poor state machine design (which does not stick closely enough to the reactive system design) or a significant change in the reactive system behaviour. 

#### Conclusion
Those issues are naturally compounded by the number of states and transitions of the state machines, i.e. the complexity of the control flow that is implemented. 

I however empirically found that carefully designing in a context of low-complexity control flow 
(around 5-7 states, and 20-30 transitions as my personal rule of thumb) generally results in a small change in reactive system design being corresponded to a small change in implementation. 

The point by and large here, is that when reactive systems are **specified** as state machines (and such is the case of the presented wireframe flows, and of business processes' user interface in general), a state machine implementation obviously may have nicer properties than alternative implementations. Because in the end, one of the best things one can do for maintainability is to keep a strong correspondence between the specifications and the code.

### Robustness
[Robustness](https://en.wikipedia.org/wiki/Robustness_(computer_science)) is the ability of a computer system to cope with errors during execution and cope 
with erroneous input. 

#### Handling errors during execution
Error handling is the poor cousin of programming. Programmers often focus on the main path of their program, because :

- error handling is precisely a disruption of the program control flow that is pretty hard to incorporate while keeping the program readable. 
- errors when they occur often do not have access to the necessary contextual data to take a good decision about their handling. That contextual data is dispersed in miscellaneous parts of the program and often cannot be gathered easily (relevant variables not in closure, unexploitable stack traces, etc.).

With state machines, errors related to executed actions (exceptions or returned error codes) can be included in the state machine definition as accepted events and associated to an (error) transition. This allows the system to return to a stable state in case of errors. The occurrence of an error while in a given control state also associate valuable context to an error (the control state but also the quantitative state), which facilitates error recovery. The same action returning error can lead to differentiated treatments, according to where (control state) it occurs.

For instance, in the sample application's implementation, errors occurring while saving the volunteer form data leads to a transition back to the origin state, and an update of the model which encodes the display of a control-state-specific message error in the user interface.

#### Erroneous inputs
State machines resists well to erroneous, invalid or unexpected inputs due to the following properties:

- Events which are not specified for a given state are not processed when in that state
- Transitions are deterministic[^non-deterministic-possible], hence the system is always in a predictable and expected state, independently of its inputs

  [^non-deterministic-possible]: In the present context they are, it is however possible to define non-deterministic state machines.
  



# References
- [**Challenges of HCI Design and Implementation**, Brad A. Myers, 1994](https://www.researchgate.net/publication/220383184_Challenges_of_HCI_Design_and_Implementation) 
- [A_Crash_Course_in_UML_State_Machines (Quantum Leaps)](https://classes.soe.ucsc.edu/cmpe013/Spring11/LectureNotes/A_Crash_Course_in_UML_State_Machines.pdf)
- Design Patterns for Embedded Systems in C: An Embedded Software Engineering Toolkit (Bruce Powel
 Douglass, 2003)
- [Hierarchical state machine, World Heritage Encyclopedia](http://self.gutenberg.org/articles/Hierarchical_state_machine)
- [Readability of source code, Wikipedia](https://en.wikipedia.org/wiki/Computer_programming#Readability_of_source_code)
- [STATECHARTS: A VISUAL FORMALISM FOR COMPLEX SYSTEMS](www.inf.ed.ac.uk/teaching/courses/seoc/2005_2006/resources/statecharts.pdf)
