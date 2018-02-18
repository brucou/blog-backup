---
title: "A componentization model for cyclejs"
date: 2017-07-07
lastmod: 2018-01-07
draft: false
tags: ["functional programming", "reactive programming", "components"]
categories: ["programming"]
author: "brucou"
toc: true

# you can close something for this content if you open it in config.toml.
comment: false
mathjax: true
---

# Background
A component framework or platform provides both a systematic method to construct components, possibly from other components (namely dealing with interfacing, binding and interactions between components), and a systematic interface between component and the component framework, by which components can be introspected, instantiated, executed, destroyed (namely dealing with component lifecycle)[^classification].

[^classification]: Cf. [A classification framework for software component models](https://pdfs.semanticscholar.org/04ab/304cd8102fdbecd6a41cde9a934e1567b1b3.pdf)

The first figure shows a classification framework for software component models, emphasizing the miscellaneous responsibilities of a component model. We will not examine here the third point(extra-functional properties).

![Classification framework for software component models](https://i.imgur.com/9yziyvB.png)

The second figure illustrates construction mechanisms linking components to each other and to the component framework.

![Component framework](https://i.imgur.com/mLozPJ4.png)

The third figure illustrates the two key ways to interface components : operational-based interface, or port-based interface.

![Interface specification](https://i.imgur.com/t84fq4c.png)

We will thereafter expose our proposal for a componentization model for implementing reactive systems, based on `rxjs` as a streaming technology, and `cyclejs` as an architecture and framework.

# Proposed component framework
We have seen [previously](/posts/user-interfaces-as-reactive-systems/) that to implement a reactive system we need :

- a reactive function $f$ linking actions to events, with $actions = f(events)$, where actions 
and events are represented as streams
- an interface with the systems through which actions intended by the user must be performed
- an interface by which the reactive system receives its events

Our component framework will be inspired by `cyclejs` framework. A component will be a procedure `f :: Sources -> Settings -> Sinks`, where :

- `Sources` contains any necessary accessor/factory methods to the internal state of the framework and relevant events
- `Settings` represents the parameterization concern of the component behaviour
- `Sinks` holds the computed action representations in response to incoming events

Note that $f$ here is not in general a pure function, as typically $f$ is accessing the external world to read events from it. We will still call $f$ the reactive function, by abuse of language. It would be easy to write `f = compose(g, h)` where `h :: Sources -> Settings -> Events` and `g :: Events -> Settings -> Sinks`, so that `g` is a pure function. We however follow `cyclejs` choice here and accept an impure reactive function, for reasons we will detail thereafter.

The component framework exposes an interface to its components by which it receives the actions to be performed : `Sinks :: HashMap<SinkName, Sink>`, is to be matched with `Drivers :: HashMap<SinkName, Driver>`, where a `Driver :: Sink -> Source` takes responsibility for the **execution** of actions passed through the **matching** sink, and passing up the eventual results of such actions as an event into the reactive system.

The component framework also exposes the interface which the reactive system receives its events : `Sources :: HashMap<SourceName, Source>`, where `Source` is anything presenting an interface to access **events** and **state** from/of external systems of interest. `Source` can, for instance, be a parameterizable event factory (events obtained from DOM listeners, etc.). `SourceName` is a moniker uniquely referencing the corresponding source of events.

The component framework connects together identical `SourceName` and `SinkName`, so that a driver corresponding to a given `SinkName` will output the result of the actions it executes as events with the same moniker as a `SourceName` (in figure thereafter, represented with same colour).

![cyclejs framework](/img/graphs/cycle_component_framework.png)

In the context of this component framework, component composition is simply function composition, with message-passing/dataflow through streams and a port-based interface emulated by the monikers `SinkName`, `SourceName`.

We will review in what follows parallel composition, sequential composition, parametricity and genericity, all of which being concerns incorporated into what we term component combinators.

## Parallel composition
Parallel composition in our context is based on expressing the reactive function $f$ as the combination of functions, each of which captures a smaller and ideally isolated part of the overall $f$'s complexity.

The bet is that :

- the smaller functions will lead to lower complexity,
- that complexity will be low enough to be addressed satisfactorily at the smaller function level,
- $f$ can be recombined in a systematic way without loss in specification from the smaller functions
- it will be possible to encapsulate a large class of reactive subsystems into reusable generic components, which can then be parameterized to reflect the targeted reactive subsystem at hand.

In short, we want a `combine :: Array<Component> -> Component`, where :

- $f$, the reactive function is a `Component`
- $f$ can be obtained by applying `combine` to other reactive functions
	- $f = combine([f_1, f_2, f_3...])$

Note that :

- the `combine` function can take any extra arguments, in which case, by uncurrying, it is always possible to come back the canonical `combine` form shown previously.
- As any component used to derive another component can itself have been derived, parallel composition naturally leads to the manipulation of component trees

There are usually many ways to perform that decomposition. The idea in every case is to reach functions $f\_{m.n...}$ whose complexity is easily manageable. If we understand that part of complexity of such functions emanates from the top-level $f$, while another part stems from the interaction of $f\_{m.n...}$s with the larger reactive system, we see that there is a sweet spot where the function is 'small' enough to be manageable but not too small so it has to be coupled with many other functions to achieve a given functionality (coupling increases complexity).

## Sequential composition
We seek `combine :: Array<Component> -> Component`, where :

- $combine([f]) = f$
- $combine([f, g])$
  - only defined when f and g have at least one matching output/input
  - connect input of $g$ to output of $f$

As a side effect, sequential composition can be used for interface adaptation purposes, like in 
$combine([adaptInput, f, adaptOutput])$.  

## Genericity, parametricity and reuse
At the core of reusability of components is the ability to design components implementing a behaviour which is generic enough to cover a large set of contexts, and parameterizable enough to be customized at design-time or run-time without modification.

In this effort, as previously introduced, we will address the parameterization concern with a specific parameter (`:: Settings`) passed to the reusable component factories or combinators. For instance a `CheckBox` component implementing the generic reactive system made of a checkbox which when clicked emits an action including its checked/unchecked state,  could be written to be parameterized in appearance/style (allowing to customize the checkbox background for example) :

```javascript
export function CheckBox(sources, settings) {
  const { checkBox: { label:_label, namespace, isChecked } } = settings;
  const checkBoxSelector = '.' + [defaultTo(defaultNamespace, namespace), ++counter].join('-');
  const __label = defaultTo('', _label);

  assertContract(isCheckBoxSettings, [settings.checkBox], `CheckBox : Invalid check box settings! : ${format(settings.checkBox)}`)

  const events = {
    'change' : sources[DOM_SINK].select(checkBoxSelector).events('change')
      .map(ev => ev.target.checked)
  };

  return {
    [DOM_SINK]: $.of(div('.checkbox', [
      label(labelSelector, [
                input([inputSelector, checkBoxSelector].join(''), {
                  "attrs": {
                    "type": "checkbox",
                    "checked": isChecked,
                  }
                }),
                // NOTE : !! snabbdom overload selection algorithm fails if last input is undefined
                span(checkBoxTextSelector, __label)
              ])
    ])),
    isChecked$: events.change
  }
}

```

## Component combinators
A component combinator is a function of the type `Combinator :: Settings -> Array<Component> -> Component`. The signature is basically the formerly presented `combine` signature to which `Settings` have been added to allow to parameterize the behaviour of the combinator.

Those combinators are themselves specialization of a generic combinator, here called `m`, with `m :: CombinatorSpecs -> Settings -> Array<Component> -> Component`, which can also be written `m :: CombinatorSpecs -> Combinator`. That is, one can see `m` both as a combinator factory, or a component factory.

We have so far implemented a list of combinators which allow to realize the following composition :

- merge reactive functions
- switch reactive functions according to events
- recursively switching reactive functions according to route (i.e. nested routing)
- merge a list of reactive function based on a template and incoming array
- extend a reactive function to process additional events or add extra parameterization
- build a set of reactive functions into a state machine
- sequential composition of ''reactive'' functions (in quotes here, because the 
sequentially composed functions are not necessarily reactive functions in the sense that some may not compute actions but intermediary results which serve to compute actions)

## Example
We will take as target application an example application from the book `Mastering Angular2 components`. 

![Example application's main screen](/img/screens/main_screen_ang2_example.png)

That screen is broken down in non-overlapping regions. For each region, a component is assigned the responsibility to implement the corresponding reactive system. For instance, the menu on the left is taken care of by `SidePanel`, while the right part is handled by `MainPanel`. At a high-level, we have the following component tree, where we can see a few component combinators (injecting remote data to be used in that branch of the tree, and adding the application container) :

![First level of decomposition](/img/graphs/App_Component_Tree_Direct_Lowest_Detail.png)

Iterating on that process, we reach a more detailed component tree, with components catering to more and more specific concerns. The following picture shows how the main panel has been further broken down into smaller components, linked through combinators :

![Detailed component tree with combinators](/img/graphs/App_Component_Tree_ProjectTaskList_with_leaves_out_flat.png)

As apparent from the figure, the `MainPanel` handles three routes, one of which corresponds to 
the `Project` component, which, among other things, handles a `ProjectsTaskList`.

That `ProjectsTaskList` itself is implemented as follows:

![Detailed component tree with combinators](/img/graphs/App_Component_Tree_ProjectTaskList_with_leaves_flat.png)

<img src="/img/graphs/legend.jpg" style="width:30%"></img>

From the breakdown of the components, one can easily gather that the `ProjectsTaskList` is a 
`TaskList`, which is a list (`ListOf`) of `Task`, read remotely (through `InjectSourcesAndSettings`) and updated for each change of that list (`ForEach`), and displayed within a `TaskListContainer`. Additionally, 
it displays a `ToggleButton` next to a `EnterTask` component, all enclosed in a 
`ProjectTaskListContainer`.

# Characteristics of a good decomposition
A good decomposition should :

- decide on the ideal granularity of the decomposition
- identify generic reusable components
- exhibit high cohesion and loose coupling

## Granularity of decomposition
As previously seen, breaking down the system in loosely coupled parts results in the individual parts having lower complexity. The right level of granularity comes from the tradeoff between fine-grain componentization (large number of small and relatively simple components) and coarse grain componentization (small number of large and relatively complex components). On the one hand, finer grain components are simpler to implement. On the other hand, decomposition into a large number of fine-grain components makes the interactions among components voluminous. In that case, the corresponding increase in complexity may overweight the decrease in complexity experienced in the individual components, nullifying the benefits of componentization.

Additionally there might be non-functional costs to decomposition which becomes non-neglectable at a fine-grain level (performance, resource consumption, etc.). Hence the optimal granularity of a decomposition must be assessed on a case-by-case basis.

Hierarchical decomposition (where components are themselves composed from other components, termed as children components) helps alleviate somewhat issues from fine-grain decomposition, by containing the increase in interaction between components (interaction is restricted to the scope of the parent component). As we saw before, this hierarchical decomposition gives birth to a component tree, where each parent component (node) is connected to its children (subtrees).

A system with hierarchical modularity can be viewed at different granularities, from coarser grain at the top of the hierarchy to a finer grain at the bottom. The goal is to constrain the interacting modules at each level to an understandable number while avoiding constraints linked to the total number of components.

System design is top-down: first the coarse-grain modularity is established, and at each successive phase the next level of hierarchy is established by decomposition of the modules at the next higher level. System implementation, on the other hand, is bottom-up: Only the modules at the leaves of a hierarchy are actually implemented, while each module above is integrated from existing modules below, starting at the bottom.

The [example](#example) provided allows to illustrate the additional point :
 
 - implementation can be performed by successive or parallel refinements. In the example, one can 
 find in green components which are still big and pending breakdown. While focusing on the rest 
 of the implementation, a developer can use a placeholder component (`DummyComponent`). 
 Alternatively, the component in question can be safely delegated to another developer or another 
 team

## Generic components vs. ad-hoc components
In the frame of reactive systems, two main drivers lead decomposition :

- reusing already existing components
  - in the context of reactive system, this mostly involves reusing common presentational/behavioural functionalities : tabbed groups, cards, breadcrumbs, etc. An existing library of reusable components can be leveraged to that effect. Those components however have to be customizable per the actions triggered by user events, or offer an interface decoupled from such actions, i.e. decoupled from the domain at hand.
- separating concerns
  - typically a graphical application will have events belonging to a specific region of the screen. An obvious divide-and-conquer strategy is to break down the application in such independent regions and assign a component to each of those. A component could for instance handle the navigation concern of the application, while another might handle displaying a domain object and possible interactions with that object, with yet another handling notifications from external systems to the application.

It is in addition worthy to identify domain-specific components, should they arise from the application implementation. This might be for instance in the case of a trading application, components which display value of a stock in real-time.

The [example](#example) provided allows to illustrate those points by emphasizing and 
distinguishing generic UI component from ad-hoc components from container components. 

# Cross-cutting concerns
## State management
We distinguish 3 types of state :

1. Locally persisted state
      - copy of remote data
      - locally generated data
        - browser state
          - route
        - app state
        - session state
        - page state
          - ui (scrollbar position, checkbox state, etc.)
          - user scripts
2. Remotely persisted state
      - domain data
3. Transient state
  - Existence is dependent and connected to the lifecycle of the component which originates it

### Locally persisted state
#### Fetching remote data
Remote data is fetched from its remote source via the corresponding driver, and becomes locally 
persisted data. The fetch can be a snapshot (no synchronisation with the source), or act like a 
live query (updating itself whenever the source updates itself). Using streams makes the second 
option easy to handle in the program. 

In our example, we use a `Query` driver to fetch remote data from a repository. That query driver
 needs a context from which it knows (having been configured prior to its use) how to retrieve 
 the remote data. The `Query` driver is meant to be used in connection with the application 
 domain. The context here can be anything, but with a complex domain, it could be a bounded 
 context of the domain. For more details about usage of `Query` driver, please refer to the 
 documentation and related tests.  

Depending on the underlying repository, `Query` can return just a snapshot (normal case), or a 
live query (updating with the remote repository). For instance, with `Firebase`, `Query` could 
return a live query. For database which do not support live updates, it is however possible to 
simulate it, by using a `Command` or `Action` driver, which notifies of the new value every time 
there is an update on a dependency of the query. Please refer to the [examples](/posts/applying-componentization-to-reactive-systems---sample-application/#step-4-projecttasklist-component) to see that 
technique in application.

#### Local state
Local state comes from different sources, originating in different local systems with which the 
user is interacting. Of particular interest are the application, the page, the browser.

Page and browser state can be retrieved by the appropriate interfaces defined by their 
specifications. We will focus here on explaining how to manage local state related to the 
application.

To display a view, we often need pieces of application state that are relevant to that view. 
Because all inputs to our function are to be taken from `Sources`, it is necessary for the 
framework to initialize `Sources` so that any piece of application state can be generated from it. 
Typically, this involves copying locally some remotely stored domain data, in addition to 
initializing some application-specific properties at start-up time. State, evolving over time, is 
conveniently modelized as a behaviour.

We recommend to generate the necessary pieces of state as close as possible of where it is needed
 in the component hierarchy. The figure reproduced thereafter serves to illustrate that point. 
 The top-level state is passed down the hierarchy, where it is massaged into new pieces of state 
 by the `InjectSourcesAndSettings` combinator. As components `A` and `B` needs the same piece of 
 state, that piece of state is injected at the closest common ancestor level. However, only 
 component `D` requires an additional piece of state, hence that piece of state is injected only 
 for it. 

![state management](/img/graphs/state_injection.png)

Note that the new pieces of state computed by the state injection combinator are **added** to 
existing pieces of state, and can be generated from already generated pieces of state, or through 
some factory in `Sources` (`Query` driver for instance - see yellow pills for component `D`).

### Remotely persisted state
This category includes whatever sits in remote databases, but also in any system of interest (for
 instance banking information accessible through a gateway interface). Remotely persisted state 
 must be queried, in our architecture, through drivers. We recommend here a custom or customized 
 driver which reflects the peculiarity of the domain at hand. See the [`Action`]
 (/projects/component-combinators/actiondriver) and [`Query`](/projects/component-combinators/querydriver) 
 driver for an example.

### Transient state
This is a piece of state which last only as long as the component that created it in the first 
place. Transient state has no need to kept around and is recreated when needed. As an example, a 
search-as-you-type component might keep track of the length of the search key, and perform search
 API calls only when that length has reach a minimum value. The length of the entered search key 
 would be transient state. Transient state is generally created through the stream `scan` operator.

## Concurrency management
User interfaces are inherently concurrent due to their interfacing with distributed systems. This 
often leads to a host of concurrency problems :

- consistency
  - optimistic updates means that a user interface can fall out of synchronization vs. external 
  systems of interest. Possible resynchronization actions may go against already performed 
  actions, if the user interface is not carefully designed or implemented
- race conditions
  - when user can input actions faster than the system can execute them, and those actions have 
  an effect on the user interface, the user can be led to realize inputs based on stale data. 
  This is a cas of read-write concurrency : a value can be read through the user interface, and 
  triggered actions can update that value remotely.
  - lost update : two user-triggered actions, fired in close sequence, write to the same piece of 
  data (write-write concurrency). Under some conditions, one of the two writes can be lost. 
- order violation
  - the user enters a sequence of inputs, triggering a sequence of actions. However those actions
   are not executed in the same order than they were generated
- missed signal
  - an action is supposed to return a result, and the user interface awaits for such. However, 
  that action result never occurs

This is further compounded by modern applications straying away from javascript's single-threaded 
model, with the use of web workers. Some applications are also strongly concurrent (collaborative
 applications, multi-windows applications, etc.).

For simple synchronization cases, keeping track of concurrent actions execution in some local 
variable(s) may do. For more complex control-flow cases, we offer the [`FSM` combinator](http://brucou.github.io/projects/component-combinators/efsm/), which implements a state machine, which add control flow capabilities to shared memory. While state machines allow to manage an intermediate level of complexity, it is still a rather low-level tool for concurrency control.  
  A more powerful option would be to make use of a specific DSL for concurrency (like [Reo]
  (http://reo.project.cwi.nl/reo/) or [Oz](https://mozart.github.io/mozart-v1/doc-1.4.0/tutorial/node8.html#chapter.concurrency)) to express the concurrency requirements separately for the sequential computations. The latter option would however require a different architecture than the one we are presently discussing, and as such, we do not pursue this option.

Also worth mentioning, `cyclejs` matches each action handler (driver) to a moniker (sink name). 
This brings the following concurrency issue : if two, somehow dependent actions, have to be executed, they must be collected into a single new action. Imagine a user action which leads to some remote data updates, and then a 
change of route. The change of route might happen even if the data update has failed, or before the data 
update is successfully performed. There is no mechanism to differentiate the processing in either
 case. In short, sinks must represent **independent** actions.
 
## Error management
Leaving aside unexpected errors, or exception, there are expected errors which are part of the 
specification of a reactive system. For instance, a user tries to book a trip but the 
reservation system is not responding, or the seat that was available while browser is no longer 
available when booking. Those errors generally prevent the system to move from one state (`booking`)
 to another (`booked`). They can hence use simple state variables in the simplest cases, or a 
 full-fledge state machine (`FSM` combinator) for complex control flows. The [example](https://github.com/brucou/component-combinators/tree/master/examples/volunteerApplication) for `FSM` 
 illustrates a multi-step application process, where at every step the back-up of application 
 data may fail.

# Conclusion
In summary, reactive systems can be specified by means of a reactive function associating inputs to actions. That reactive function can be obtained by composition of reactive functions from smaller reactive systems.

A good decomposition or factoring is one :

- which ensures for each subsystem a reduction in complexity
(simpler specifications, smaller size of the reactive system, few interactions with other subsystems i.e. intercomponent dependency)
- which can be reassembled in a way that is easy to reason about and trace
- can be parameterized without modification (open-closed principle), so futures changes in the overall reactive system have a higher change to result mainly in changes in parameterization of subsystems.
- highly cohesive, loosely coupled to ensure adaptability[^adaptability]
 
[^adaptability]: 80% of software engineering deals with maintaining or releasing new versions. The cost of redesigning each of such adoptable components (or replacing by a better component) must be minimized.

In our reactive system, `cyclejs` context: for component construction and interfacing, functions are used, exposing a fixed interface, differentiating parameterization concern and dataflow concern in their inputs; a port metaphor is used for interfacing components; combinators are used for parallel and sequential composition; component lifecycle is handled by the framework which by means of <em>start</em> and <em>stop</em> functionalities.

A range of generic combinators have been defined, which allows to handle a series of concerns, 
including sequential composition, data injection, control flow, routing, iteration, change 
propagation, etc. They are tools which can be used to for concurrency control, state management, 
and error management for the application at hand.

# Bibliography

- [Software Component Models - Component Life Cycle](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.450.9230&rep=rep1&type=pdf)
- [Component Based Systems - 2011](http://www.win.tue.nl/~johanl/educ/2II45/ADS.09.CBSE.pdf)
- [Concurrency Control Problems](http://www.myreadingroom.co.in/notes-and-studymaterial/65-dbms/532-concurrency-problems.html)
- [Concurrency Issues Motivation, Problems, Directions](http://courses.cs.vt.edu/cs5204/fall10-kafura-BB/Presentations/Concurrency-Issues.pdf)
