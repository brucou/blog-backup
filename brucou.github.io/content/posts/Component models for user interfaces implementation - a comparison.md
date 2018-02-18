---
title: "Component models for user interfaces implementation - a comparison"
date: 2018-01-07
lastmod: 2018-01-07
draft: false
tags: ["functional programming", "reactive programming", "components"]
categories: ["programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# Abstract
 In the previous articles, we have presented our proposed component model for user interfaces 
 implementation, which relies on a cyclejs architecture, and is based on streams and reactive 
 programming techniques.
 
 We will now compare our proposed component model to `Angular2` and `React` component models. We will
  not compare it to `cyclejs` component model as it does not have a proper one : while it is 
  possible to breakdown a component into smaller components, there is no generic or structured 
  mechanism to glue them together - it has to be done manually each time. In addition 
  standard `cyclejs` conflates the interfaces with the external systems (`sources`) with the 
  parameterization of components (for instance adding miscellaneous `prop$` source). It is 
  precisely for that reason that we have come to propose our componentization model.

# Angular2 component model
## Description
A component controls a patch of screen called a view. You define a component's application logic—what it does to support the view—inside a class.  You define a component's view with its companion template. A template is a form of HTML that tells Angular how to render the component. A template looks like regular HTML, except for a few differences. The class interacts with the view through an API of properties and methods.  To tell Angular that a class is a component, you must attach metadata to the class. In TypeScript, you attach metadata by using a decorator. 

An Angular2 template, in addition to HTML syntax, includes, among other things, the following 
features:

- [data binding](https://angular.io/guide/template-syntax#binding-syntax-an-overview) — includes 
(property|attribute|class|style) binding, event binding, and two-way binding
- [template expressions](https://angular.io/guide/template-syntax#template-expressions) — expressions evaluated in the [component instance's context](https://angular.io/guide/template-syntax#expression-context), which produce a value that is assigned to a binding target. Template expressions are written in a language that looks like JavaScript.
- [template statements](https://angular.io/guide/template-syntax#template-statements) — evaluated in the [component instance's context](https://angular.io/guide/template-syntax#statement-context), responds to an event raised by a binding target such as an element, component, or directive.
- Structural directives (`ngFor`, `ngIf`, `ngSwitch`, etc.) — change the DOM layout by adding and
 removing DOM elements
- Attribute directives — change the appearance or behavior of an element, component, or another 
directive (`ngStyle`, etc.)

The key methods/decorators for a component's class are the following :

- inputs admitted by the component
- output produced by the component
- lifecycle methods : `constructor`, `ngOnDestroy`, `ngOnInit`
- event handlers

The key metadata to attach to make a class into a component are : 

- the selector which will hold the view
- the template which specifies the view
- the providers which are services (read : functions tackling the concerns not related to displaying the view). Services can be injected
- component directives, which includes other components the declared component will require

Better than words, here are some sample code from the Angular2 implementation of the project 
management [sample application](https://github.com/PacktPublishing/Mastering-Angular-2-Components) we dealt with in a [previous article](/posts/applying-componentization-to-reactive-systems---sample-application/).

**Example of component's class**

```javascript
export class App {
  // We use the data provider to obtain a data change observer
  constructor(@Inject(ProjectService) projectService) {
    this.projectService = projectService;
    this.projects = [];

    // Setting up our functional reactive subscription to receive project changes from the database
    this.projectsSubscription = projectService.change
      // We subscribe to the change observer of our service and deal with project changes in the function parameter
      .subscribe((projects) => {
        this.projects = projects;
        // We create new navigation items for our projects
        this.projectNavigationItems = this.projects
          .filter((project) => !project.deleted)
          .map((project) => {
            return {
              title: project.title,
              link: ['/projects', project._id]
            };
          });
        // Uses functional reduce to get a count over open tasks across all projects
        this.openTasksCount = this.projects
          .reduce((count, project) => count + project.tasks.filter((task) => !task.done).length, 0);
      });
  }

  // If this component gets destroyed, we need to remember to clean up the project subscription
  ngOnDestroy() {
    this.projectsSubscription.unsubscribe();
  }
}
```

**Example of component declaration**

```javascript
@Component({
  selector: 'ngc-app',
  template,
  directives: [Project, Navigation, NavigationSection, NavigationItem, ROUTER_DIRECTIVES],
  providers: [ProjectService, UserService, ActivityService, TagsService]
})
```

The interaction between the view defined in the template, and the class is performed through an 
extensive and complex API, which includes among other things binding inputs, outputs, 
and events.

**Example of template with bindings**

```html
<div class="app">
<div class="app__l-side">
  <ngc-navigation [openTasksCount]="openTaskCount">
    <ngc-navigation-section title="Main">
      <ngc-navigation-item title="Dashboard" [link]="['/dashboard']"></ngc-navigation-item>
    </ngc-navigation-section>
    <ngc-navigation-section title="Projects"
                            [items]="projectNavigationItems">
    </ngc-navigation-section>
    <ngc-navigation-section title="Admin">
      <ngc-navigation-item title="Manage Plugins" [link]="['/plugins']"></ngc-navigation-item>
    </ngc-navigation-section>
  </ngc-navigation>
</div>
<div class="app__l-main">
  <router-outlet></router-outlet>
</div>
</div>
```

Note how the template binds together the view, the class and the component, into a cohesive whole, 
and relate to each other's entities :

- standard HTML elements describe the view (`div`, etc.)
- `ngc-navigation` relates to the `Navigation` component, passed as a directive to the `App` 
component
- `openTasksCount` from the `App` class is passed as *live* input to `ngc-navigation` (data binding with the `[]` syntax). This means that whenever `openTasksCount` will change, the `ngc-navigation` will adjust to that new value. This is by the way a typical case of parent-child communication via inputs

The corresponding extract of the code for the `Navigation` component should illustrate the use of
 the `@Input` decorator to bind the parameters passed in the template to properties of the 
 corresponding class :
 
 ```javascript
@Component({
  selector: 'ngc-navigation',
  template,
  directives: [NavigationSection, UserArea]
})
export class Navigation {
  @Input() openTasksCount;
}
```

Because of the coupling between template, class and component, none of those entities can be, a priori, reasoned about individually. As a result, in all what follows when we will refer to a component, unless otherwise specified, we will mean the triple (template, class, @component).

The ease of reasoning, or understanding, i.e. the readability of an Angular2 application will largely be a function of the component breakdown adopted, and the complexity of the interaction between components, and the complexity of the component definition itself.

## Application architecture
The primary responsibility of a Angular2 component is to display a view described by a template. 

Templates bind events to event handler, which are located in the component's class. Change 
propagation and state management is performed by binding between template syntactic elements and 
the class properties. As such a class has three key responsibilities : state management, event 
handling, and lifecycle management.

The component declaration concerns itself with linking the component to the Angular2 framework, 
so it can be handled and integrated with the rest of the components.

While being highly prescriptive about syntax (and there is a lot of new syntax), the Angular2 
component architecture is relatively flexible in terms of the paradigms that it supports. In its 
simplest expression, events are associated to action handlers, which directly modify local 
component state, which in turn updates the view. It is however possible to follow functional reactive programming principles to [some extent](https://vsavkin.com/managing-state-in-angular-2-applications-caf78d123d02), to [a larger extent](https://github.com/ngrx/platform) or simply adopt a redux-like [one-way data flow](https://blog.angular-university.io/angular-2-application-architecture-building-applications-using-rxjs-and-functional-reactive-programming-vs-redux/).

## Component interaction
The [Angular2 documentation](https://angular.io/guide/component-interaction) lists 4 
communication methods between components : 

- [Pass data from parent to child with input binding](https://angular.io/guide/component-interaction#pass-data-from-parent-to-child-with-input-binding)
- [Parent listens for child event](https://angular.io/guide/component-interaction#parent-listens-for-child-event)
  - the implicit pub/sub mechanism associated (linked to the `@Output` decorator) can also be 
  used to communication between components in different hierarchies, i.e components which are not
   in a ancestor/child relationship
- [Parent and children communicate via a service](https://angular.io/guide/component-interaction#parent-and-children-communicate-via-a-service)
  - this is equivalent to communication by shared state, except that the service forms a facade 
  which handles the state update logic
- Parent owns an instrumentable reference to its child
  - [interacts with child via local variable](https://angular.io/guide/component-interaction#parent-interacts-with-child-via-local-variable)
     - a reference variable is created at the child level
  - [Parent calls an `@ViewChild`](https://angular.io/guide/component-interaction#parent-calls-an-viewchild)
     - this allows the parent to own a reference of its children and directly instrument the 
  children, for instance to modify its state

## Routing
The Angular Router enables navigation from one view to the next as users perform application tasks. The router configuration has its own dedicated syntax (directives, decorators, etc.) and [recommended pattern](https://angular.io/guide/router#a-crisis-center-with-child-routes). Learning to use routing hence requires learning new syntax and building a specific mental model. This leads to a fair amount of syntax to understand but also to produce, and a significant lead time to getting up-to-speed. On the bright side, the router comes already packaged with most of the features one will ever need in a web application.

## Lifecycle
Angular creates, updates, and destroys components automatically as the user moves through the application. The app's programmer can take action at each moment in this lifecycle through optional lifecycle hooks, like `ngOnInit`. 

## Testing
Isolated unit tests examine an instance of a class all by itself without any dependence on Angular or any injected values. The tester creates a test instance of the class with new, supplying test doubles for the constructor parameters as needed, and then probes the test instance API surface.

Isolated unit tests should be written for pipes and services.

Components may be tested in isolation as well. However, isolated unit tests don't reveal how components interact with Angular. In particular, they can't reveal how a component class interacts with its own template or with other components.

Such tests require the Angular testing utilities. The Angular testing utilities include the `TestBed` class and several helper functions from `@angular/core/testing`. Writing Angular2 tests again requires learning (and producing) a great deal of new syntax and constructing yet another mental model.

## Conclusion
Angular2 as a framework has a very large scope, and covers the large majority of the needs of
 the web application programmer. However, this plethora of features comes with the cost of 
 a large incidental complexity. **There are many parts** (angular2 framework architecture, 
 component model, routing, templating, change detection, etc.), **every part is complex** on its 
 own, and then that complexity is **compound by the inter-relation** or dependencies between the parts. That complexity means a significant time is required to construct the required mental models to understand and produce Angular2 code. That complexity also multiplies the possibility of errors in the system's implementation and may significantly increase debugging time. All those factors negatively impact productivity.
  
A good example of complexity generated by inter-related parts is the templating mechanism, **a 
double-edged sword.** On the one hand, it makes the view contents more readable, specially to people with little coding experience (designers, etc.). On the other hand, the templating language precisely excels at expressing the view content, not at expressing logic and control-flow (loops, branching, etc.). The resulting DSL has a peculiar syntax, is not
 Turing-complete, which forces to complement it with a general purpose language. That makes two languages to master, the peculiarities of the interface between the two to understand, two tightly coupled files (`.html` for the view and `.ts` for the logic) themselves coupled to the router, with what can be significant back-and-forth between the two to fully grasp the view's behaviour. This impacts negatively both readability and productivity.

Another source of complexity is the inter-relation between components. For one thing, parent component can directly access and 
instrument their children, which can make it a nightmare to reason about state in a complex 
application. Then, depending on the state management strategy adopted, reconstructing the flow of data can be challenging in large component trees. Every component may have its own state, its own bindings and eventing to 
parent component or any level in the component tree. They can also communicate with the parent via a service (i.e. shared state at the closest common ancestor level). State is then scattered among the application in 
miscellaneous places, in miscellaneous programming artefacts, with both hidden and explicited 
dependencies between pieces of state to be aware of, and it may be a arduous process to link all the pieces together. Once again, we are in a situation with **many inter-related parts**, which drives the complexity.
This can be mitigated by following a strong discipline about state management (one-way dataflow, 
single store, reactive programming, etc.), and **documentation**, but this is on the programmer's onus. A lot of the complexity is inherent to the framework and has be to assumed.

In summary, Angular2 is without a doubt a **superb piece of technology**. It often addresses 
efficiently most of the issues a developer must resolve when implementing a user interface. By doing so, it removes a source of incidental complexity through its abstractions (browser quirks, synchronizing state and view, 
routing, animation, performance, multi-platform targetting, etc.). However it also adds new 
complexity linked to the number of inter-related concepts/parts which have to be mastered, and the complexity of each of those parts. Taming that complexity will require **considerable discipline** on the side of the developer(s), **good 
architecting** on the side of the technical leadership, and the same level of **supporting tools** 
(CLI, visualizing component trees, router trees, inspecting component's properties, etc.) available for the DSL than for the core language. There will be projects where resorting to Angular2 does reduce the complexity more than it increases it. Large-scale, enterprise projects might fall in that category --- few user-interface tricks, mostly workflow driven, complexity resides in the domain, more than in the user interface. There will also be projects where Angular2 will bring more problems than it 
 will solve.

# React component model
## Description
Components let you split the UI into independent, reusable pieces, and think about each piece in isolation.
Conceptually, components are like JavaScript functions. They accept arbitrary inputs (called 
“props”), a context (kind of global variable), and return React elements describing what should 
appear on the screen. Components can be defined either as a function (termed functional components) or as a class.

**functional component**

```javascript
function Welcome(props, context) {
  return <h1>Hello, {props.name}</h1>;
}
```

TODO : hugo shortcode for adding title to code

**class-based component**

```javascript
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

Components can be composed simply by referring to other components in their output. With the help
 of the `JSX` DSL, we for instance can write :
 
 ```javascript
function App() {
  return (
    <div>
      <Welcome name="Sara" />
      <Welcome name="Cahal" />
      <Welcome name="Edite" />
    </div>
  );
}
``` 

The `Welcome` component itself could be broken into several other components. This gives rise to 
a component tree, which JSX allows to express in a readable form (additional benefits are better 
error and warning messages). The same principle guiding a good component breakdown applies here :
 loosely-coupled, reusable components, assembled into cohesive parts (cf. [extracting components](https://reactjs.org/docs/components-and-props.html#extracting-components) from the documentation).

Generic components, which may admit an unspecified number or type of components, may also be 
drafted with the special property `props.children` which is akin to Angular2's  
`<ng-content></ng-content>`, or the slot mechanism (`<slot></slot>`) of [web components](https://developers.google.com/web/fundamentals/web-components/shadowdom#slots). This is equivalent to passing the 
enclosed content of a component as a parameter like any other. An example can be found in the 
React's [documentation](https://reactjs.org/docs/composition-vs-inheritance.html#containment).

The class-based version of components allow to express components which encapsulate state. State 
management is then handled through life-cycle hooks (`componentDidMount`, `componentWillUnmount`,
 etc.) and ad-hoc methods possibly calling `setState` to update the local state. This is for 
 instance the code for a `Clock`  component  updating every second.

```javascript
class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

  componentDidMount() {
    this.timerID = setInterval(
      () => this.tick(),
      1000
    );
  }

  componentWillUnmount() {
    clearInterval(this.timerID);
  }

  tick() {
    this.setState({
      date: new Date()
    });
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
``` 

Note as the `tick` method updates local state, which automatically triggers the (re-)rendering of 
the clock. 

There are peculiarities to state update, but they are few : [state update may be asynchronous](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous), and
 [state updates are merged](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-are-merged). 

## Application architecture
Stateful components, i.e. components which manage their state, can be defined through classes. 
That class may hold event handlers, and other methods which directly and imperatively update the 
component's local state, triggering an update of the user interface. The `setState` accept a 
`prevState` parameter which allows to incrementally update the local state (vs. building the 
full state each time).

Additionally, control flow may be expressed in regular `javascript`, in order to [conditionally 
display](https://reactjs.org/docs/conditional-rendering.html) components, or iterate over them. `JSX` also provides some specific syntax to that purpose. Displaying lists requires assigning a key to each item, key which must 
be [unique among siblings](https://reactjs.org/docs/lists-and-keys.html#keys-must-only-be-unique-among-siblings).

`JSX`, while close to regular HTML, has some specific syntax ([forms](https://reactjs.org/docs/forms.html), [event binding](https://reactjs.org/docs/handling-events.html)). There are however few specificities, the 
full syntax can be relatively quickly be acquired, and JSX can be [entirely discarded](https://reactjs.org/docs/react-without-jsx.html) if need be in favor of standard javascript.

React strongly encourages [composition over inheritance](https://reactjs.org/docs/composition-vs-inheritance.html), 
providing mechanisms to compose components which allow to avoid using class inheritance. 

There a few advanced React concepts ([type checking](https://reactjs.org/docs/typechecking-with-proptypes.html), [Refs](https://reactjs.org/docs/refs-and-the-dom.html), [Performance](https://reactjs.org/docs/optimizing-performance.html), [Fragments](https://reactjs.org/docs/fragments.html), [Portals](https://reactjs.org/docs/portals.html), 
[Error management](https://reactjs.org/docs/error-boundaries.html)). Each of them has however a 
small surface, combine simply with the key concepts, resulting in an overall manageable complexity.

## Component interaction
The [React documentation](https://reactjs.org/docs/) lists 3 communication 
methods between components : 

- [Pass data from parent to child with input binding](https://angular.io/guide/component-interaction#pass-data-from-parent-to-child-with-input-binding)
- [Children with parent via ''protected'' shared state and handlers](https://reactjs.org/docs/lifting-state-up.html)
  - this is akin to Angular's communication via services. The shared state is lifted up to the 
  closest common ancestor. The common ancestor passes both the shared state, and a handle to its 
  children through `props`, handle by which the shared state can be modified (has a closure over 
  the state, or access to the state via `this`). The passing of handlers reminisces of event 
  binding. However the mechanism is more private, as only the parent which passes the handler can
   be affected by that 'event'.
- components communicate between each other via global state
  - this may mean using the [`context` feature](https://reactjs.org/docs/context.html) of React, 
  and is [strongly discouraged](https://medium.com/react-ecosystem/how-to-handle-react-context-a7592dfdcbc).
  - another possibility is have a global store at the app level. This is where additional 
  libraries like `Redux` or `Mobx` come into play.

In all cases, React strongly encourages [one-way dataflow](https://reactjs.org/docs/state-and-lifecycle.html#the-data-flows-down) communication between components.

## Routing
Unlike `Angular2`, `React` routing does not require a pre-made list of routes to later integrate 
with the component tree. Routing is expressed by [regular components](https://reacttraining.com/react-router/web/guides/philosophy), and nested routing is just nesting components, like one would nest html's `div`. For instance :

```javascript
const App = () => (
  <BrowserRouter>
    {/* here's a div */}
    <div>
      {/* here's a Route */}
      <Route path="/tacos" component={Tacos}/>
    </div>
  </BrowserRouter>
)

// when the url matches `/tacos` this component renders
const Tacos  = ({ match }) => (
  // here's a nested div
  <div>
    {/* here's a nested Route,
        match.url helps us make a relative path */}
    <Route
      path={match.url + '/carnitas'}
      component={Carnitas}
    />
  </div>
)
```  

This is a very powerful mechanism, which only increases marginally complexity.

## Lifecycle
In the context of stateful components, classed must have lifecycle methods. This means lifecycle 
of the local state has to be handled by hand. However, the library automatically handles the 
lifecycle of the component itself, using the hooks to complement its task. 

## Testing
UI testing is a complex subject. It encompasses testing :

* UI structure : the view should have an expected structure as a function of its state (i.e. 
$view = f(state)$, where $f$ is a pure function)
* UI behaviour : some user action should lead to some system actions (remember the reactive 
equation $actions = f(state,events)$ where $f$ is a pure function) 
* UI visual appearance (style, look, etc.) (this can be done by directly comparing pixel by pixel
 the rendered DOM and the target DOM)

I will focus on structure and behaviour testing. React has two main available tools to that 
purpose : `Jest` and `Enzyme`. Both allow to render a component with or without a DOM, and test 
its content for the expected structure. A `simulate` method allow to emit events 
and observe the resulting change in the UI. Mocks and spies can also be used. A summary of the 
techniques, mostly up-to-date, even if the article is old, can be found [here](http://reactkungfu.com/2015/07/approaches-to-testing-react-components-an-overview/). The testing of actions can be
 simplified with the use of `redux` and its derivated libraries.

## Conclusion
React is significantly less complex than Angular2 for two reasons. First of all, as a library, it 
takes on less responsibilities than a full-fledged framework like Angular2. Second, there are 
less parts to handle to build a react application, and each of those parts is relatively easy and
 quick to understand on its own, with clear and simple interaction with the other parts. 

For instance, the pseudo-templating technology used (`JSX`) is optional, and embeds nicely into 
 regular javascript code. All the power of the Turing-complete javascript logic can be used to 
 render a component while still enjoying the benefits of a HTML-like syntax when necessary. `JSX` 
 is easy (very few syntactic constructs above HTML) and cohesively located exactly where used, 
 instead of the file separation imposed by `Angular2`.

Furthermore, routing involves just regular components. There is very little new to learn to 
handle even nested routing. One-way dataflow simplifies state tracking. Similarly to Angular2, 
there are excellent DevTools which can be built for productivity gains.

The most complex aspect is probably testing. `Angular2` requires what seems like some fairly 
complex set-up in comparison with `React`, but does offer dependency injection. React is silent 
on the latter. As a result, it is possible that complex applications may be a la fine easier to 
test with `Angular2`. 

In fact, adding functionalities to React, means adding parts, and then adding 
complexity sources. The most common extra parts (`Redux`, `redux-thunk`, `redux-saga`), 
are in increasing order of complexity, and have to be integrated manually, together with their 
respective workflow and tooling. Other possible parts (animations, etc.) have to be evaluated. At equal perimeter of features, `React` still compares favorably to  `Angular2`, but the distance is lower. There will be teams and projects for which `Angular2` 
 will generate a higher value, precisely because it imposes uniform choices, and its set of technologies is tightly integrated.

# Our component model
## Description
Our component model is directly inspired by our functional approach to implement reactive 
systems, understood as a reactive function linking stream of events to stream of actions, and an 
interface with external systems to receive events and perform actions. A component in our model 
is a function with two arguments : `sources` and `settings`. The first argument is the interface to external 
systems' events and state. The second argument represents the component parametrization concern, 
and will allow to have generic components which can be specialized or parameterized through those 
settings. Components in our model compute the relevant system actions to perform, and pass the 
result of that computation to so-called **drivers**, which interface with the external systems to
 execute the relevant actions.  The computed actions are passed as an object whose every property
  (termed sink) represents a sequence of actions to be handled by the driver for that property.

Going back to our functional equations, a component implements a reactive function $f$ such that 
$actions = f(state, events)$. The  view update action can be shortened to $view = f(state)$, given that the only event which updates the  view is precisely a change of state, whose information is already included in the equation. In 
  short, we have :

$$\begin{cases}
   view & = f_v(state) \\\\\
   actions & = f_a(state, events)
\end{cases} $$

Here is an example of component illustrating all the previous points : 

```javascript
function NavigationItem(sources, settings) {
  const { url$ } = sources;
  const { project: { title, link } } = settings;
  const linkSanitized = link.replace(/\//i, '_');

  const events = {
    // NOTE : we avoid having to isolate by using the link which MUST be unique over the whole
    // application (unicity of a route)
    click : sources.DOM.select(`.navigation-section__link.${linkSanitized}`).events('click')
  };
  const state$ = url$
    .map(url => url.indexOf(link) > -1)
    .shareReplay(1);

  const actions = {
    domUpdate : state$.map(isLinkActive => {
      const isLinkActiveClass = isLinkActive ? '.navigation-section__link--active' : '';

      return a(
        `${isLinkActiveClass}.navigation-item.navigation-section__link.${linkSanitized}`,
        { attrs: { href: link }, slot: 'navigation-item' },
        title)
    }),
    router : events.click
      .do(preventDefault)
      .map(always('/' + link + '/'))
  }

  return {
    [DOM_SINK]: actions.domUpdate,
    router: actions.router
  }
}
```

Note the 2 sinks, one for the DOM update driver, one for the router driver. The DOM update driver
 receives the new DOM, computed only from `state$`.  The router driver receives the router 
 actions computed only from the events (reduced to `click` here). Last, note how fixed parameters 
 `{ project: { title, link } }` are passed to the `NavigationItem` component through `settings`.

## Application architecture
For DOM updates, a [`snabbdom`](https://github.com/snabbdom/snabbdom) virtual DOM syntax is generally used. A virtual node is used for performance reasons. Writing the DOM as a function of state, means that we compute the full DOM 
every time. Because it would be inefficient to directly display the new DOM, without reusing the 
already displayed fragments of the current DOM, a virtual node mechanism allows to do a delta 
between virtual DOM objects, and use that diff to compute and only update the portions of the DOM 
that needs updating. Note that another syntax in which the dom updates are a function of the 
state delta (i.e. $ view\_{\Delta} = f(state\_{\Delta}) $ ) is also possible (that is the model for instance 
used by [`Ractive`](https://ractive.js.org/)), but has not been investigated for the moment.

The actual syntax recognized for actions will vary for each driver, and as such should come as 
part of the documentation for each driver. Drivers perform whatever action on external systems 
which is part of their specification, and may produce a response, which is passed back into the 
application via a property of the `sources` object named the same as the driver's attributed sink.

Streams are commonly used when writing the application, allowing to write equations once, with 
the runtime making sure that they keep in sync, by handling change propagation. This means in 
particular that there is no need for lifecycle methods, and no need for imperative update of state.

The reactive functional programming core concepts are built on top of (here `rxjs`) streams. 
Events are streams which are `shared`, behaviours are streams which are `shareReplayed(1)`. 
Events modelize... events occuring in the system. Behaviours modelize pieces of state. 
The adopted granularity of state will depend on the programmer's design decision and 
domain at hand. At one extreme, one single behaviour may keep all state in one place. At the other
 extreme, dividing the application state in pieces may lead to having one behaviour for each piece.

As such, application architecture is rather simple (streams, events, behaviours, reactive 
function, drivers are all the parts), though one could argue that the complexity of manipulating 
state and events has been transferred to that of understanding and manipulating streams. However, it is my opinion that the stream abstraction benefits are such that it pays out in the end. Because we write equations, it is 
easy to reason about our component and application. 

Furthermore, we propose a component model which allows to write an application as the aggregation
 of smaller, easier to reason about applications. The reactive system under implementation is 
 deconstructed into a component tree, which is assembled by component combinators to give back 
 the target application. 

Component combinators follow a syntax conceptually similar to `JSX`. For instance :

**TODO** title SidePanel
```javascript
export const SidePanel =
  m({}, {}, [Div('.app__l-side'), [
    Navigation({}, [
      NavigationSection({ title: 'Main' }, [
        m({}, { project: { title: 'Dashboard', link: 'dashboard' } }, [NavigationItem])
      ]),
      NavigationSection({ title: 'Projects' }, [
        InSlot('navigation-item', [ListOfItemsComponent])
      ]),
      NavigationSection({ title: 'Admin' }, [
        m({}, { project: { title: 'Manage Plugins', link: 'plugins' } }, [NavigationItem])
      ]),
    ])
  ]]);
```

could be written : 

```javascript
export const SidePanel = 
  <Div class='app__l-side'>
    <Navigation>      
      <NavigationSection title='Main'>
        <NavigationItem project={ title: 'Dashboard', link: 'dashboard' }\>
      </NavigationSection>
      <NavigationSection title='Projects'>
        <Slot name='navigation-item'>
          ...
        </Slot>
      </NavigationSection>
      <NavigationSection title='Admin'>
        <NavigationItem project={ title: 'Manage Plugins', link: 'plugins' }\>
      </NavigationSection>
    </Navigation>
  </Div>
```

This opens the door to a `JSX`-like DSL and many other optimizations in the future.

## Component interaction
There are 4 ways of interaction between components :

- Pass static (i.e. constants) parameterization data from parent to child with `settings`
  - this would be similar to the `props` passing in `React`
- parent can inject state into children
  - this fulfills a similar purpose to the `[]` binding syntax in `Angular2`
- components communicate between each other via global state
  - this means that a in-memory store driver could be used (à la `Redux`)
- components communicate between each other via events
  - a component may send an event to a pub/sub driver, and another component might listen to it. 
  Note that while this pattern is possible, it is not often used in practice. Communication 
  through shared state is likely preferable, due to the equational reasoning it facilitates.   

## Routing
Routing can be realized through the `OnRoute` combinator and is inserted naturally at the 
position of the component tree when the routing needs to occur. Routing can also be achieved 
directly without the combinator though there will necessary be more boilerplate. As a matter of 
fact, routing is matching a route change event to predefined component(s) actions, and as such 
remains within the conceptual framework without modification.  

## Lifecycle
There is no need for lifecycle management component-wise. Component are just functions. Events 
and behaviours exists as long as they are necessary, and picked up by the runtime when there is 
no more reference of them.
  
## Testing
Component can be tested via mocking of the interfaces with the external systems (`sources` and 
`drivers`).

## Conclusion

# Conclusion
