---
title: "Applying componentization to reactive systems : sample application"
date: 2017-10-16
lastmod: 2018-01-06
draft: false
tags: ["functional programming", "reactive programming", "components"]
categories: ["programming"]
author: "brucou"
toc: true

# you can close something for this content if you open it in config.toml.
comment: false
toc: true
mathjax: true
---

# Introduction
As discussed in a [former post](/posts/user-interfaces-as-reactive-systems), user interfaces are 
reactive systems, and can be specified through a reactive function which associate events 
(originating from the user, or the interfaced systems) to actions to be executed. Expressing events, actions, and local state as streams, and expressing local state update also as an action, we have  : $actions = f(state, events)$, where $f$ is a pure function. In addition to that function, which expresses the logic of the 
application, interfaces with the relevant systems must be defined. Typically this means 
interfaces to receive events, and execute actions.

Our proposed approach is based on cyclejs framework, in which actions are executed through 
a dedicated interface called `driver` and events streams are created where and when necessary 
through streams and streams factories. Those streams factories themselves are passed as parameters
 to the reactive function. As such the reactive function, in a `cyclejs` context is not pure. 
 Adjusting for this caveat, our reactive equation still holds.

We enrich the `cyclejs` architectural choices, with a [componentization model](/posts/a-componentization-framework-for-cyclejs), which allows to 
build a larger application from a number of small components with the help of component 
combinators. A combinator library is provided to cover the most generic needs occurring while 
building a componentized application, including, but not limited to, sequential composition, data 
injection, control flow, routing, iteration, change propagation.

The present document aims at showing how to translate a reactive system specification into an 
implementation with our proposed componentization model. It is hence :

- part documentation, addressing the question *How to leverage the proposed component model to write
 an application*
- part showcase, in which the advantages of the componentization approach appear in connection with
 an iterative software development process

In the first section, we will quickly describe the application under development. We will then 
proceed with setting up the structure (shell) in which the application logic will be contained. 
We will then in subsequent sections, iteratively extend our code to implement more and more of 
our target application. For each section, we will provide a link to the corresponding source 
code, contained in a dedicated branch of the demo repository. We will however not develop the 
whole application here, but the minimum subset necessary for us to reach our documentation and 
showcasing goals.

# Target application
The target application is a task management application, taken from the [Mastering Angular2 
components](https://github.com/PacktPublishing/Mastering-Angular-2-Components) book. Users handle projects, projects have tasks, which can be in status finalized or pending. The user can review and update existing tasks, create new tasks, and delete existing ones. Tasks can be filtered by status. The user can add comments to a project which are presented chronologically. A summary of activities can be consulted in a dedicated section. A dashboard provides a summary view of the projects and tasks. Finally, plugins can be administered and integrated into the application to extend its functionality.

This is an application with a reasonable size, and we will not seek to implement the full 
functionality previously described, in one go. In the present version of this document, we will 
focus on a first batch of features, which includes only the project browsing and task management
. We will also detail only portions of our implementation, selecting and detailing those portions which 
illustrates a use of the component model not previously illustrated. This aims at covering as 
large a percentage possible of the available component combinators, and illustrating as many 
different techniques as possible.

Rather than producing a detailed written specification, we will provide the following 
screenshot, from which a feature list is easy to guess. For supplementary details, we refer the 
reader to the book.

![Angular2 project application](/img/screens/main_screen_ang2_example.png)

In addition to this, the following routes are specified :

- `/dashboard` : handles the dashboard functionality
- `/projects/:projectId` : shows the project description, together with a tab bar (`Tasks`, 
`Comments`, `Activities`)
  - `/projects/:projectId/tasks` : additionally shows the tasks belonging a project with the given 
  `projectId`, together with a button group to filter tasks, and a input field to enter new tasks
  - `/projects/:projectId/task/:nr` : additionally shows a specific task details, with the ability
   to modify those details 
  - `/projects/:projectId/comments` : additionally shows the comments logged for a given project 
  - `/projects/:projectId/activities` : additionally shows the activities logged for a given 
  project 
- `/plugins` : handles the plugin functionality

# Step 0 : set up
## Domain model
We will work with a domain model with three entities : projects, tasks, and activities. Data is stored remotely in a firebase repository. Thereafter follows an example of the physical data model (dictated only by 
our self-imposed constraint to allow for direct comparison with the Angular2 application - there 
are other ways to design that physical data model):

```javascript
projects: {
    _id: 'project-1',
    type: 'project',
    deleted: false,
    title: 'Your first project',
    description: 'This is your first project in the task management system you\'re building within the context of the Angular 2 Components book.',
    tasks: [{
      type: 'task',
      nr: 1,
      position: 0,
      title: 'Task 1',
      done: null,
      created: +Moment(now),
      efforts: {
        estimated: 86400000,
        effective: 0
      }
    }]
  }

activities: [
    {
    type: 'activity',
    user: {
      name: 'You',
      pictureDataUri: 'data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0idXRmLTgiPz4NCjwhLS0gR2VuZXJhdG9yOiBBZG9iZSBJbGx1c3RyYXRvciAxOS4yLjAsIFNWRyBFeHBvcnQgUGx1Zy1JbiAuIFNWRyBWZXJzaW9uOiA2LjAwIEJ1aWxkIDApICAtLT4NCjxzdmcgdmVyc2lvbj0iMS4xIiBpZD0iQ2FwYV8xIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiB4PSIwcHgiIHk9IjBweCINCgkgdmlld0JveD0iMCAwIDMxMS41IDMxMS41IiBzdHlsZT0iZW5hYmxlLWJhY2tncm91bmQ6bmV3IDAgMCAzMTEuNSAzMTEuNTsiIHhtbDpzcGFjZT0icHJlc2VydmUiPg0KPHN0eWxlIHR5cGU9InRleHQvY3NzIj4NCgkuc3Qwe2ZpbGw6IzMzMzMzMzt9DQo8L3N0eWxlPg0KPGc+DQoJPGc+DQoJCTxwYXRoIGNsYXNzPSJzdDAiIGQ9Ik0xNTUuOCwwQzY5LjcsMCwwLDY5LjcsMCwxNTUuOGMwLDM3LjUsMTMuMyw3MS45LDM1LjMsOTguOGMzLjQtMjcuMywzMC42LTUwLjMsNjguOC02MS4yDQoJCQljMTMuOSwxMywzMiwyMC45LDUxLjcsMjAuOWMxOS4yLDAsMzYuOS03LjUsNTAuNy0xOS45YzM4LjUsMTEuOSw2NS4xLDM2LjMsNjYsNjQuNmMyNC4zLTI3LjUsMzkuMS02My42LDM5LjEtMTAzLjENCgkJCUMzMTEuNSw2OS43LDI0MS44LDAsMTU1LjgsMHogTTE1NS44LDE5NS43Yy05LjksMC0xOS4zLTIuNy0yNy42LTcuNWMtMjAuMS0xMS40LTMzLjktMzQuOC0zMy45LTYxLjdjMC0zOC4xLDI3LjYtNjkuMiw2MS41LTY5LjINCgkJCWMzMy45LDAsNjEuNSwzMSw2MS41LDY5LjJjMCwyNy40LTE0LjIsNTEtMzQuOCw2Mi4yQzE3NC40LDE5My4yLDE2NS4zLDE5NS43LDE1NS44LDE5NS43eiIvPg0KCTwvZz4NCjwvZz4NCjwvc3ZnPg0K'
    },
    time: +Moment(now),
    subject: 'project-1',
    category: 'tasks',
    title: 'A task was updated',
    message: 'The task \'New task created\' was updated on #project-1.',
    _id: 'ECEF8127-C237-9612-924B-2A087D6FACA4'
  },
  ]
``` 

## Interfaced systems' drivers
Actions that are undertaken as a response to events are : domain data update, DOM update, route 
change, and of course updating local state (persisted and non persisted). 
As per events to be handled by the system, we have : user events (i.e. DOM events), local state 
update notification, route change notification.

This leads to the following drivers :

- write drivers : DOM driver, router driver, domain update driver, local state update driver 
- read drivers : DOM driver, router driver, domain query driver, local state query driver

We will use :

- router driver : for read and write, we use the same [history driver](https://github.com/cyclejs/cyclejs/tree/master/history) from `cycle`
- DOM write driver : standard snabbdom default DOM `cycle` driver
- DOM read driver : document driver which injects the `document` dependency to read from the DOM
- domain update and query drivers : we use a domain [Query](/projects/component-combinators/querydriver) 
driver and domain [Action](/projects/component-combinators/actiondriver) driver, where entities 
and the methods applicable to them are defined 
- local state drivers : we use a in-memory store to query and update data in the local domain

This leads us to the following set up code :

```javascript
    const { sources, sinks } = run(App, {
      [DOM_SINK]: filterNull(makeDOMDriver('#app', {
        transposition: false,
        modules: defaultModules
      })),
      document: documentDriver,
      domainQuery: makeDomainQueryDriver(repository, domainObjectsQueryMap),
      domainAction$: makeDomainActionDriver(repository, domainActionsConfig),
      storeAccess: makeDomainQueryDriver(inMemoryStore, inMemoryStoreQueryMap),
      storeUpdate$: makeDomainActionDriver(inMemoryStore, inMemoryStoreActionsConfig),
      router: makeHistoryDriver(createHistory(), { capture: true }),
    });
```

For the full code and running demo, see [here](https://github.com/brucou/component-combinators/tree/project-management-app-step-0/examples/AllInDemo). 

# Step 1 : SidePanel (navigation) and MainPanel
We divide the application in two independent components, one handling the side navigation 
(`SidePanel`) and the other one handling functionalities depending on the current route 
(`MainPanel`). 

Our starting code for the `App` is then :

```javascript
App = Combine({}, [Div('.app'), [
                    SidePanel,
                    MainPanel
                  ]])
```

This deserves a few words of explanation. `Combine` is a combinator which takes some components (let's call them children components), and returns a combined component. That combined component will pass its sources and settings (here `{}`) to all 
children components to compute their sinks, merge[^merge] the resulting non-DOM sinks from the children components, and combine the DOM sinks [à la web components](/projects/component-combinators/m-component---merge-default-functions/#slotted-dom-merge), by dispatching children DOM content into its respective slots (the container component holds the slot definition -- here `Div('.app')` does not define any
 slot). The full details of how `Combine`'s merge mechanism works can be found in the [corresponding documentation](/projects/component-combinators/m-component---merge-default-functions/). Note that it is important to understand how `Combine` merges children sinks, as most of the combinators in the library, merge sinks in the same way.

[^merge]: Simple merge is used for non-DOM sinks (as in `Rx.Observable.merge`). Another example of such simple merge can be found in the [`cyclejs-utils`](https://github.com/cyclejs-community/cyclejs-utils) utility library for cycle.

We are however going to do two things : inject pieces of state that will be used by both 
components, and massage the route input arriving through the router driver in a convenient form 
(basically removing a leading `/` if any). This can be done at any relevant level. However, as 
those sources of events and pieces of state will be used by the whole application, it is more DRY
 to inject them at the top level, so that all downstream components inherit them unless otherwise
  configured. For the same reason, we will configure the settings of the router driver once for all, and
  declare the list of sinks that the application handles at this level. `sinkNames` is mandatory 
  for several components combinators to filter the sinks returned from children components, so we
   put it at the top level for the same DRY reasons. 
   
   The `settings` and `sources` inheritance mechanism is described in the documentation for the 
   `m` combinator and the [`InjectSourcesAndSettings` combinator](/projects/component-combinators/injectsourcesandsettings/). 

The corresponding code includes :

```javascript
export const App = InjectSourcesAndSettings({
    sourceFactory: function (sources, settings) {
      // NOTE : we need the current route which is a behaviour
      const { router, domainQuery } = sources;
      const currentRouteBehaviour = router
        .map(location => {
          const route = location.pathname;
          return (route && route[0] === '/') ? route.substring(1) : route
        })
        // starts with home route
        .startWith('')
        .shareReplay(1);
      // NOTE : we need the route change event
      // Now it was important to do this in that order, because we want currentRouteBehaviour to
      // be subscribed before (no route change before having a current route)
      // A former implementation url$ = incomingRouteEvents$.shareReplay(1) failed as url$ was not
      // subscribed till after the route had changed, and by then the new route value was already
      // emitted, so url$ would not emit anything... One has to be very careful dealing with
      // streams and ordering
      const incomingRouteEvents$ = currentRouteBehaviour.share();
      const projects$ = domainQuery.getCurrent(PROJECTS);
      const user$ = domainQuery.getCurrent(USER);

      return {
        // router
        url$: currentRouteBehaviour,
        [ROUTE_SOURCE]: incomingRouteEvents$,
        // NOTE : domain driver always send behaviour observables (i.e. sharedReplayed already)
        user$,
        // NOTE : `values` to get the actual array because firebase wraps it around indices
        projects$: projects$.map(values),
        projectsFb$: projects$
      }
    },
    settings: {
      sinkNames: ['domainAction$', 'storeUpdate$', DOM_SINK, 'router', 'focus'],
      routeSource: ROUTE_SOURCE
    }
  }, [Div('.app'), [
    SidePanel,
    MainPanel
  ]]
);
```

**Pay attention to** :

- the querying for domain entities related to the projects and the user through the query 
driver (for instance `domainQuery.getCurrent(PROJECTS)`)
- the use of `Div('.app')` as a container for the list of components (main and side panels). Full explanation of the feature can be found at the [corresponding documentation page](/projects/component-combinators/m-component---merge-default-functions/#regular-dom-merge) (the container here acts as parent component). In short, all DOM trees returned by the enclosed components will be inserted within the DOM tree of the container. 
- the distinction between the route change event, and the route state which is a behaviour[^behaviour]. The 
general, and very important rule, is to **systematically** decide whether a reactive entity of 
interest is modelized better by an event or a behaviour, and to apply the corresponding marker 
(`share()` for events, `shareReplay(1)` for behaviours)[^marker]. Failing to do this is the origin 
of a **large portion of bugs** encountered when manipulating streams.

[^behaviour]: The reason why we need both the route change event and the route state is that sometimes we want to get the current route without reacting to a route change, and the best way to achieve this is to use `shareReplay(1)` in connection with `sample` or `withLatestFrom`. 

[^marker]: A good way to enforce this is to build on top of the stream library constructors for events and behaviours, and enforce their usage, effectively prohibiting direct manipulation of the underlying streams. We do not follow this option however, to avoid adding extra syntax. We might reconsider later on, based on feedback.

For the full code and running demo, see [here](https://github.com/brucou/component-combinators/tree/project-management-app-step-1/examples/AllInDemo). 

This example allows us to illustrate an important componentization tip. 

## TIP : What makes a good breakdown
What makes this breakdown a good one? Independence or loose coupling is the key here :

- `MainPanel` can be implemented fairly independently from `SidePanel`
- the only reason for change of `SidePanel` that would affect `MainPanel` is a change in the
route associated to the projects[^routechange]
- both components share no common events or actions or logic. We took our initial reactive system, split it in two, smaller, largely independent subsystems, whose complexity is strictly lower than the original system, and
 easier to write

[^routechange]: The corresponding change could be anticipated by designing the main panel to allow for parameterization of these routes. This would ensure that a change in `SidePanel` does not entail a change in `MainPanel` implementation, but rather a change in parameterization (we have decided not to refactor our sample application in that direction though, to not distract the learner with implementation details)

More generally, it is desirable to build a complex system by assembling, in a cohesive way, loosely 
coupled components, so that the cost of redesigning each of such adoptable components (or replacing 
by a better component) can be minimized.

# Step 2 : SidePanel
Let's implement our `SidePanel` component. Here are the specifications we can extract from the 
overall application's specifications :

| Events               | Actions                                             |
|----------------------|-----------------------------------------------------|
| INIT                 | DOM : display welcome message and task summary      |
|                      | DOM : display 3 sections with  possibly subsections |
| click on subsections | navigate to the corresponding route                 |

We define the routes to be navigated to as per specification :

- `/projects/:projectId`
- `/dashboard`
- `/plugins`

To compute the DOM view (in particular the task summary and the projects section), we will need a
 local copy of the remotely persisted following domain entities:

  - `projects`
  - `user`

As explained in the previous step, we have already injected those sources at the top level of 
the application. This means that those sources will be available for every component under 
`sources.user$`, and `sources.projects$`. We will then seek to write `SidePanel` as :

```javascript
SidePanel =
  Combine({}, [Div('.app__l-side'), [
    Navigation({}, [
      NavigationSection({ title: 'Main' }, [
        NavigationItem({ project: { title: 'Dashboard', link: 'dashboard' } }, [])
      ]),
      NavigationSection({ title: 'Projects' }, [
        InSlot('navigation-item', [ListOfItemsComponent])
      ]),
      NavigationSection({ title: 'Admin' }, [
        NavigationItem({ project: { title: 'Manage Plugins', link: 'plugins' } }, [])
      ]),
    ])
  ]]);
```

where :

- `Navigation` is a ad-hoc component combinator which
  - accepts children components as an array of components
  - wraps all `navigation-section` slot content from its children components into a `nav` tag
  - passes up unmodified all non-DOM actions (carried by the non-DOM sinks)
  - adds some style specific to its navigation concern
- `NavigationSection` is a component combinator which
  - can (must) be parametrized by a `title` property
  - accepts children components as an array of components
  - wraps all `navigation-item` slot content from its children components into a list item (`li`)
  - passes up unmodified all non-DOM actions from its children component (carried by the non-DOM sinks)
  - adds some style specific to its navigation concern
- `NavigationItem` is a component combinator which
  - can (must) be parametrized by the `title` and `link` properties
  - does not accept children components
  - displays the title, and a click on that title triggers a routing action passed through the router sink
  - emphasizes the current project selection as determined by the route `/projects/:projectId`
- `ListOfItemsComponent` is a component which, from `projects` :
      - get the current list of projects,
      - computes a title (project name) and a link (`/project/projectId`),
      - and for each project in that list, accumulates it in the shape of a `NavigationItem` parameterized with the computed title and link

Let's go in detail about the implementation, and by doing so learn about the slot mechanism, how 
to write ad-hoc combinators, and components which displays a list of items.

## Slots and container components
Combinators accept either a list of components (`Array<Component>`), or a container and a list of 
components (`[ContainerComponent, Array<ChildrenComponent>]`). The container component is used to apply some processing to the children components' sinks. Unless otherwise specified, the default processing is to merge all non-DOM actions from the children components' sinks, and to merge DOM sinks from the children components' sinks following a slot mechanism à la web components. 

The container component may define a slot or location, where children DOM sinks will be inserted.
 The container component hence provides a template, and the children DOM sinks provide the 
 content filling that template. For more details of the slot mechanism, see the [documentation](/projects/component-combinators/m-component---merge-default-functions/#slotted-dom-merge). A 
 simple way to set a slot for the DOM content for children components is to use the [`InSlot`]
 (/projects/components-combinators/inslot) combinator.

For our `Navigation` combinator, we want to put the DOM sinks corresponding to the navigation 
sections after displaying the welcome message, and within a styling `div`.

We define first the container component, which displays the welcome message and task count(termed 
`TaskSummary` here), and declares the slot for the navigation sections :

```javascript
function NavigationContainerComponent(sources, settings) {
  const { user$, projects$ } = sources;
  // combineLatest allows to construct a behaviour from other behaviours
  const state$ = $.combineLatest(user$, projects$, (user, projects) => ({ user, projects }))

  return {
    [DOM_SINK]: state$.map(state => {
      return div('.navigation', [
        renderTasksSummary(state),
        nav({ slot: 'navigation-section' }, [])
      ])
    })
  }
}
```

## Navigation combinator
After that, we are ready to define the `Navigation` combinator :

```javascript
function Navigation(navigationSettings, componentArray){
  return Combine(navigationSettings, [NavigationContainerComponent, componentArray])
}
```

## NavigationSection combinator
In the same way, we can write the `NavigationSection` as :

```javascript
function NavigationSectionContainerComponent(sources, settings) {
  const { title } = settings;

  return {
    [DOM_SINK]: $.of(
      div('.navigation-section', { slot: 'navigation-section' }, [
        h2('.navigation-section__title', title),
        ul('.navigation-section__list', { slot: 'navigation-item' }, [])
      ])
    )
  }
}

function NavigationSection(navigationSectionSettings, componentArray){
  return Combine(navigationSectionSettings, [NavigationSectionContainerComponent, componentArray])
}
```

Pay attention to how :

- the `div` vNode combinator has been extended with a slot module. That slot module marks 
the corresponding vTree with the name of the slot it belongs to. Note also how we set the slot 
for the `NavigationItem` components (reminder : `NavigationItem` are the expected children 
components for `NavigationSection`)
- For the `navigation-section` slot, content is provided. For the `navigation-item`, content is not provided, as it will be filled in by children components.
- `navigation-section` slot is at the top level of the `vNode` tree for the 
`NavigationSectionContainerComponent`. Similarly, `NavigationItem` components will also have to 
have their slot set at top level, because slot merging only looks at the children's DOM top vNode
 for slots.

## ListOfItemsComponent component
Last, let's see how to write a component which displays a list of items. We have the [`ListOf`](/projects/component-combinators/listof) combinator which does just that, but we have to call it 
with the right inputs. To do so, we will first inject the relevant piece of state (the list to 
project titles to be displayed, and the links to navigate to) with `InjectSourcesAndSettings`. We
 also want the side panel updated whenever the project titles change in the source of 
 truth (remote repository). The [`ForEach`](/projects/component-combinators/foreach) allows to activate a component tree whenever the value of a behaviour changes, or an event is emitted ; and will cover our requirements nicely. This leads to the following code :
 
 ```javascript
function getProjectNavigationItems$(sources, settings) {
   return sources.projects$
     .map(filter(project => !project.deleted))
     .map(map(project => ({
       title: project.title,
       link: ['projects', project._id].join('/')
     })))
     .distinctUntilChanged()
     // NOTE : this is a behaviour
     .shareReplay(1)
     ;
}

const ListOfItemsComponent =
  InjectSources({ projectNavigationItems$: getProjectNavigationItems$ }, [
    ForEach({ from: 'projectNavigationItems$', as: 'projectList' }, [
      ListOf({ list: 'projectList', as: 'project' }, [
        EmptyComponent,
        NavigationItem
      ])
    ])
  ]);
```   

For each new value of `projectNavigationItems$`, the setting property `projectList` will be 
updated to that value, and a list of `NavigationItem` will be activated. Each of the 
`NavigationItem` will receive a `project` setting property which holds the value of one element of
 the `projectList` array, together with the index of that particular element. For more 
 information, see the corresponding documentation of both operators. 

For the full code and running demo, see [here](https://github.com/brucou/component-combinators/tree/project-management-app-step-2/examples/AllInDemo). 

# Step 3 : Main panel
As per specifications, the main panel is url-driven. This gives us the following breakdown :

```javascript
export const MainPanel =
  Combine({}, [Div(`.app__l-main`), [
    OnRoute({ route: 'dashboard' }, [ProjectsDashboard]),
    OnRoute({ route: 'projects/:projectId' }, [Project]),
    OnRoute({ route: 'plugins' }, [ManagePlugins]),
  ]]);
```

where :

- `ProjectsDashboard` handles the dashboard functionality
- `ManagePlugins` handles the plugin functionality
- `Project` handles the display of information about the projects and the tab bar (`Task`, `Comment`,
 `Activities`)

As it is immediately visible from the example, configuring components to be activated on a given 
route is pretty straight forward. Not immediately visible here, in the case of a route with 
parameter (`projects/:projectId`), the parameters (`projectId`) will be passed to downstream 
components through `settings`. For more details on the `OnRoute` combinator, see the 
[documentation](/projects/component-combinators/router/). 

We will not go into detail about all three components, but rather focus on detailing the 
`Project` component.

## Project component
The project component has the following specifications :

| Events                 | Actions                                             |
|----------------------  |-----------------------------------------------------|
| INIT  | DOM : display project header and tab bar with no tab selected  |
| route change to `tasks` | DOM : display project task list              |
| route change to `task/:nr` | DOM : display task detail for task `nr`   |
| route change to `comments` | DOM : display project's comments          |
| route change to `activities` | DOM : display project's activities      |

The relevant piece of state here (reminder : the route here is `projects/:projectId`) is 
information about the specific project pointed at by the route. This is the object from which we 
will get the project's comments, activities, etc.

The state we need is retrieved as follows : 

```javascript
export function projectsStateFactory(sources, settings) {
  const { [ROUTE_PARAMS]: { projectId } } = settings;

  return sources.projectsFb$
    .map(projectsFb => {
      const fbKeys = keys(projectsFb);
      const _values = values(projectsFb);
      const index = _values.findIndex(project => project._id === projectId);
      const fbIndex = fbKeys[index];
      const project = _values[index];

      return {
        fbIndex,
        project
      }
    })
    .shareReplay(1)
}
```

Not how `projectId` from the route is passed through `settings`. Then `Project` can be written as :
 
```javascript
export const Project =
  InjectSources({ projectFb$: projectsStateFactory }, [Div('.project'), [
    ProjectHeader,
    Combine({ tabItems }, [TabContainer, [
      OnRoute({ route: 'tasks' }, [ProjectTaskList]),
      OnRoute({ route: 'task/:nr' }, [Div('.task-details', { slot: 'tab' }), [ProjectTaskDetails]]),
      OnRoute({ route: 'comments' }, [Div('.comments', { slot: 'tab' }), [ProjectComments]]),
      OnRoute({ route: 'activities' }, [Div('.activities', { slot: 'tab' }), [ProjectActivities]])
    ]])
  ]]);
```

where :

- `ProjectHeader` displays the title of the project together with some description
- `TabContainer` is a container component, also handling the tab logic (i.e. emphasizing the tab 
which is active)
- `ProjectTaskList` handles displaying the panel which displays a list of tasks, allows to enter a 
new task, and/or filter the list of tasks. 
- `ProjectTaskDetail` handles displaying the details for a task, and allows the user to modify 
those details
- `ProjectComments` handles displaying the comments for a project 
- `ProjectActivities` handles displaying the activities related to a project 

Note how we have nested routes (`projects/:projectId/tasks`) as naturally as we have flat routes. 

For more details on how the slot logic interacts with the `TabContainer` component, see the 
`TabContainer` code. For the full code and running demo, see [here](https://github.com/brucou/component-combinators/tree/project-management-app-step-3/examples/AllInDemo). 

# Step 4 : `ProjectTaskList` component
We will seek to write the `ProjectTaskList` component as follows :

```javascript
const ProjectTaskListContainer = vLift(
  div('.task-list.task-list__l-container', { slot: 'tab' }, [
    div('.task-list__l-box-a', { slot: 'toggle' }, []),
    div('.task-list__l-box-b', { slot: 'enter-task' }, []),
    div('.task-list__l-box-c', [
      div('.task-list__tasks', { slot: 'tasks' }, [])
    ])
  ]));

export const ProjectTaskList =
  Combine({}, [ProjectTaskListContainer, [
    InSlot('toggle', [ToggleButton]),
    InSlot('enter-task', [EnterTask]),
    InSlot('tasks', [TaskList])
  ]]);
```

where :

- `ProjectTaskListContainer` is a container component which allocates children content into their
 slots and applies some styling
- `ToggleButton` handles the tasks filter (values to be picked within `All`, `Open`, `Done`)
- `EnterTask` allows the user to enter a new task for the active project
- `TaskList` displays a list of project tasks, taking into account the task filter set by the user

Note again how the slot mechanism allows to separate a template (fixed part of the target 
`vTree`), from the contents filling that template (variable part). It is important to become 
familiar with the mechanism, as it is fundamental to divide the user interface in ever smaller 
and isolated pieces. Pay attention also to how slots are declared : either directly through the 
slot module, or with the `InSlot` combinator. The latter is useful in combination with other 
components which do not declare slots for their DOM content, for instance generic components. 
`InSlot` can also be used to override existing slots for a given component's DOM content. 

Note the use of the utility function [`vLift`](https://github.com/brucou/component-combinators/blob/master/src/utils.js#L1118), which takes a `Vtree` and lifts it into a component (whose only sink is a DOM sink emitting that `vTree`).

Let's go in further details about the `EnterTask` component (illustrating domain driver write 
actions), and the `ToggleButton` component (illustrating local state driver actions).
 
## Adding a task
The `EnterTask` has obvious specifications :

| Events                 | Actions                                             |
|----------------------  |-----------------------------------------------------|
| INIT  | DOM : display input field and `Add` button |
| click on `Add` button| domainAction$ : add a new task to the project's tasks entity  |
| | domainAction$ : add a new activity to the project's activity entity  |

The relevant state is constituted of :

- relevant information about the project under review
- relevant information about the user (activities are linked to users)
- task description as found in the input field

We will write the `EnterTask` component as follows :

```javascript
export function EnterTask(sources, settings) {
  let key = 0;
  const { projectFb$, user$, document } = sources;
  const { [ROUTE_PARAMS]: { projectId } } = settings;
  const taskEnterButtonClick$ = sources[DOM_SINK].select(taskEnterButtonSelector).events('click')
  // NOTE : is event -> share
    .share();

  // NOTE :: we use a key here which changes all the time to force snabbdom to always render the
  // input vNodes. Because we read from the actual DOM, the input vNodes are no longer the
  // soruce of truth for the input state. From a snabbdom point of view, without that key, we
  // render two exact same vNodes and hence it does not do anything. So we have to force the 
  // update by making the successive vNodes differ.
  return {
    [DOM_SINK]: taskEnterButtonClick$
      .map(_ => renderTaskEntryArea(++key))
      .startWith(renderTaskEntryArea(++key)),
    domainAction$: taskEnterButtonClick$
      .do(preventDefault)
      .withLatestFrom(projectFb$, user$, (ev, projectFb, user) => {
        const {fbIndex : projectFbIndex, project} = projectFb;
        const tasks = project.tasks;
        const newTaskPosition = tasks.length;
        const nr = tasks.reduce((maxNr, task) => task.nr > maxNr ? task.nr : maxNr, 0) + 1;
        // NOTE : has to be computed just before it is used, otherwise might not get the current
        // value
        const _taskEnterDescription = getInputValue(document, taskEnterInputSelector);
        const taskEnterDescription = _taskEnterDescription ? _taskEnterDescription : 'Task';

        // NOTE : We have two domain actions to perform here
        return $.from([{
          context: TASKS,
          command: ADD_NEW_TASK,
          payload: {
            projectFbIndex,
            newTask: taskFactory(taskEnterDescription, newTaskPosition, nr),
            tasks
          }
        }, {
          context: ACTIVITIES,
          command: LOG_NEW_ACTIVITY,
          payload: activityFactory({
            user,
            time: +Date.now(),
            subject: projectId,
            category: 'tasks',
            title: 'A task was added',
            message: `A new task "'A task was added'" was added to #${projectId}.`
          })
        }
        ])
      })
      .switch()
  }
}
```

Note how we use the domain update driver, and request methods execution on domain entities. 

## Setting the task filter
Specifications for the `ToggleButton` component are as follows :

| Events                 | State | Actions                                             |
|----------------------  |-------|----------------------------------------------|
| INIT  | task filter | DOM : display all tabs, emphasizing the tab corresponding to the task filter (i.e. active tab)|
| click on a tab | task filter | local state : update the task filter|

We will use the in-memory store driver to keep track of the task filter. As the
 task filter will also be used by the `TaskList` component to filter out the project's tasks, 
 it needs to be accessible by several components which are not in a parent/child relationship. 
 There are basically two ways to handle this :
  
  - The first one is to create and inject that piece of state at the closest ancestor level (this would be here at `ProjectTaskList` level). 
 - The second one is to create that piece of state at the global level (i.e. cutting across the component 
 hierarchy - that is what the in-memory store is), and access it anywhere necessary. 
 
 We chose the second solution for didactic purposes (injecting `tasksFilter$`), as an example of use of the in-memory store driver. Concretely, the breakdown is as follows :
 
```javascript
export const ToggleButton =
  InjectSourcesAndSettings({
      sourceFactory: tasksFilter$,
      settings: tasksButtonGroupSettings
    }, tasksButtonGroupSettings.buttonGroup.labels.map((buttonLabel, index) => {
      return function (sources, settings) {
        return ButtonFromButtonGroup(sources, merge(settings, {
          buttonLabel,
          index
        }))
      }
    })
  );
```

where the injected state is : 

```javascript
export function getStateInStore(context, sources, settings) {
  const {storeAccess, storeUpdate$} = sources;

  return  storeAccess.getCurrent(context)
    .concat(storeUpdate$.getResponse(context).map(path(['response'])))
    // NOTE : this is a behaviour
    .shareReplay(1)
}
export function tasksFilter$(sources, settings) {
  return {
    taskFilter$: getStateInStore(TASKS_FILTER, sources, settings)
      .map(prop('filter'))
      // In case the `TASKS_FILTER` entity has changed but the tasks filter property has not
      .distinctUntilChanged()
  }
}
```

Note how we combine the store read and write drivers (`storeAccess`, `storeUpdate$`) to get  
a *live* `taskFilter$` which emits the current value of the task filter and then the new values of 
the task filter, every time that value changes. To ensure this, we listen on responses to 
in-memory entity update requests. Such responses are objects which hold the updated entity in the 
`response` property. For extra details, refer to the [configuration](https://github.com/brucou/component-combinators/blob/master/examples/AllInDemo/src/inMemoryStore/index.js#L21)
 of the in-memory driver, and at the documentation for the [domain `Action` driver](/projects/component-combinators/actiondriver/).

The `ButtonFromButtonGroup` is a component which will display a button with a label passed 
through `settings` and update the task filter when the button is clicked :

```javascript
const updateTaskTabButtonGroupStateAction = label => ({
    context: TASKS_FILTER,
    command: PATCH,
    payload: [
      { op: "add", path: '/filter', value: label },
    ]
  });

function ButtonFromButtonGroup(sources, settings) {
  const { taskFilter$ } = sources;
  const { buttonLabel, index, buttonGroup: { labels, namespace, buttonClasses } } = settings;
  const buttonGroupSelector = makeButtonGroupSelector({ label: buttonLabel, index, namespace });

  const events = {
    click: sources[DOM_SINK].select(buttonGroupSelector).events('click')
      .map(always(buttonLabel))
  };
  const state$ = taskFilter$;
  
  return {
    [DOM_SINK]: state$.map(taskFilter => {
      const classes = ['']
        .concat(buttonClasses(taskFilter, buttonLabel))
        .join('.') + buttonGroupSelector;

      return button(classes, buttonLabel)
    }),
    storeUpdate$: events.click
      .withLatestFrom(state$, (label, taskFilter) => ({ label, taskFilter }))
      // no need to do anything if clicking on a button already active
      .filter(x => x.label !== x.taskFilter)
      .map(prop('label'))
      .map(updateTaskTabButtonGroupStateAction)
  }
}
```

Note that [in-memory store driver](https://github.com/brucou/component-combinators/blob/master/examples/AllInDemo/src/inMemoryStore/index.js#L29) uses [json patch](http://jsonpatch.com/) to describe the updates to the in-memory entity.

For the full code and running demo, see [here](https://github.com/brucou/component-combinators/tree/project-management-app-step-4/examples/AllInDemo).

## Displaying a list of tasks
A `TaskList` is a list of tasks. We inject the corresponding state, compute the component with 
`ListOf` combinator, and affect the DOM content to the corresponding slot :

```javascript
export const TaskList = InjectSourcesAndSettings({ sourceFactory: taskListStateFactory }, [TaskListContainer, [
  InSlot('task', [
    ForEach({ from: 'filteredTasks$', as: 'filteredTasks' }, [
      ListOf({ list: 'filteredTasks', as: 'filteredTask' }, [
        EmptyComponent,
        Task
      ])
    ])
  ])
]]);
```

a `Task` is made of a checkbox to change its status, a button to allow deletion, another button 
to allow task edition, and the task information. Clicking on a task detail button should route 
the user to another screen where he can modify the task information. The breakdown is hence 
as follows : 

```javascript
function TaskContainer(sources, settings) {
  const { filteredTask: { done, title }, listIndex } = settings;
  const coreVnodes = div('.task', [
    div(".task__l-box-a", { slot: 'checkbox' }, []),
    div(".task__l-box-b", [
      div(".task__title", { slot: 'editor' }, []),
      button(taskDeleteSelector(listIndex)),
      div('.task-infos', { slot: 'task-infos' }, []),
      div({ slot: 'task-link' }, []),
    ])
  ]);

  return {
    [DOM_SINK]: $.of(
      done
        ? div(`.task--done`, [coreVnodes])
        : coreVnodes
    )
  }
}
const Task = InjectSourcesAndSettings({
  settings: function (settings) {
    const { filteredTask: { done, title }, listIndex } = settings;

    return {
      checkBox: { isChecked: !!done, namespace: [TASKS, listIndex].join('_'), label: undefined },
      editor: { showControls: true, initialEditMode: false, initialContent: title }
    }
  }
}, [TaskContainer, [
  InSlot('checkbox', [
    Pipe({}, [
      CheckBox,
      ComputeCheckBoxActions
    ]),
  ]),
  InSlot('editor', [
    Pipe({}, [
      Editor,
      ComputeEditorActions
    ]),
  ]),
  InSlot('task-infos', [
    TaskInfo,
  ]),
  TaskDelete,
  InSlot('task-link', [
    TaskLink
  ]),
]]);
```

Using `CheckBox` and `Editor` generic UI components requires us to do some adaption of input, 
and output to fit their APIs. This allows to illustrate two interesting techniques :

- Inputs (here `settings`) are adapted through `InjectSourcesAndSettings` ahead of using the 
generic UI components
- Outputs are adapted through the use of the `Pipe` combinator. For instance, `Editor` returns a 
`save$` sink which needs to be mapped to a `domainAction` sink to update the corresponding domain
 entity. As a matter of fact, the [`Pipe` combinator](/projects/component-combinators/pipe) allows to pass sinks of a component in the 
 pipe as sources for the next component in the pipe. In the mentioned case, for instance the 
 output adaptation is performed by `ComputeEditorActions`.

# TIP : How to write a component
Within the chosen architecture (cycle), writing a *leaf* component, i.e. a component 
which is not derived from other components, but only from `sources` and `settings`, means :

- having clear specifications as per the reactive behaviour to implement
  - from the equation `actions = f(state, events)` :
      - identify the relevant events the component reacts to
      - identify the actions to trigger in response to events
      - identify the necessary pieces of state to compute the actions from the events
- decide on the parameterization of the component
  - this means deciding which parts, if any, of the component's behaviour will be parameterizable
   through the component's `settings` property
- compute the events
  - while doing so, ensure in particular that events are coupled to unique selectors : this is 
  particularly important when operating within the cycle architecture where events are decoupled 
  from the elements that originate them.
- compute the necessary pieces of state
- compute the actions, as a function of the events and pieces of state

## Example
We want to implement a `NavigationItem` component with the following specifications :

- displays a project
- parameterized by a project title and a route corresponding to that project (termed project link)
- if the current route corresponds to the project link, emphasize visually that project 

This gives us, the following reactive function :

| Events                 | Actions                                             |
|----------------------  |-----------------------------------------------------|
| INIT (route == project link) | DOM : display project title |
| INIT (route != project link) | DOM : display emphasized project title |
| click on project title | router : navigate to the project link                 |

This helps us identifying the pieces of state part of the reactive function :

| Events                 | State | Actions                                             |
|----------------------  |-------|----------------------------------------------|
| INIT  | isLinkActive : true (i.e. `route == project link`) | DOM : display project title |
| INIT  | isLinkActive : false (i.e. `route != project link`) | DOM : display emphasized project title |
| click on project title | -- | router : navigate to the project link                 |

We assure the unicity of the selector for the click event, by coupling the corresponding element 
to the project link, which is unique as per the specification. This is a tradeoff of `cyclejs`'s 
architectural choice to separate the event creation from the declaration of the view structure. 
As a matter of fact, the view is conceptually tightly coupled to the event handlers, and forcing 
the decoupling of the two results in having to define unequivocally, for each event handler, the DOM element to 
which it relates. On the positive side of that tradeoff, we are able to test components, by 
mocking their inputs. For readability and DRY reasons, we recommend to isolate the `css` selector by 
which event handler and element are coupled into a separate variable. 

This leads to the implementation below : 

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

# TIP : How to write a component combinator
We have already given two examples of component combinators ([here](/posts/applying-componentization-to-reactive-system---sample-application/#navigation-combinator), and [here](/posts/applying-componentization-to-reactive-system---sample-application/#navigationsection-combinator)). The general 
process is as follows :

- having clear specifications as per the combining behaviour to implement
  - how is the component combinator to be parameterized ?
  - is a container component necessary ?
  - what are the contracts which settings or component tree must fulfill?
  - how are the components' sinks combined?
    - are default combining functions sufficient ?
    - are specific combining functions necessary ?
- implement the target behaviour
  - reuse as much as possible existing combinators
  - select, when necessary, the most appropriate form of the `m` combinator (out of the three 
  reducing patterns)
      - reminder : `= m(componentCombinatorSpec, componentCombinatorSettings,
        childrenComponents | componentTree)`
      - understanding `m` default reducing functions is paramount : often times, they have the 
    behaviour that is sought for

# Generic, reusable components
Components developed for a specific application usually have to be generalized to make them
reusable. The generalized component can then be adapted, specialized or parameterized to be used
for the specific application use case.

For instance, in our application's UI, we have a checkbox on which a click leads to miscellaneous
actions on the domain model. Abstracting out the actions specific to the domain model, we can
build a reusable checkbox UI component, where the clicks will emit a dummy action passing on the
status of the checkbox (checked/unchecked).

That generic UI checkbox can then be reused in different contexts, within the application, or in
other applications, by specifying how the clicks translates into actions on the given domain
model : the UI checkbox component is **adapted** to the application under development.

In other cases, the generalized component will be specialized (the `m` combinator is such a case,
 where the programmer can specialize the reduction of the component tree to one of three patterns).

In yet other cases, the generalized component behaviour will be configured by parameterization
through the component settings.

In our application, we have identified the following reusable UI components :

- `CheckBox` component
  - for each click on the checkbox, passes the state of that checkbox  
- `Editor` component
  - allows to define an editable user content zone where the user can modify, save, delete
  content.

Potential candidates for further refactoring into reusable components are :

- ToggleButton (<em>xor</em> button group component)
- EnterTask (input entry)

A component such as `TaskInfo` is not a fruitful target for a generalization that allows to reuse
it in other domains, as its behaviour seems very much tied (coupled) to the application's domain
model. However, should that component be needed with slight modifications more than twice ([rule of three](https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)) in our application, we would have considered 
writing a generalized version of the component.

Ideally, there is already at hand a component (UI or domain) )library that is already tested, and 
documented. That could be the case for example for UI components, such as those exposed 
previously. This could also be the case if domain experts have succeeded in identifying repeating
 patterns in their domain, and produced a domain-specific component library. In the general case,
  the software designer will have to find and assess the abstraction/generalization opportunities presented to
 him. Those opportunities are generally identified while refactoring. As explained in a [former article](/posts/componentization-against-complexity/#barriers-to-reuse), refactoring for reuse has a 
 cost, and the possible benefits to be derived have to be weighted against that cost.

For illustration purposes, here is part of the source code for the `CheckBox` component:

```javascript
export function CheckBox(sources, settings) {
  const { checkBox: { label: _label, namespace, isChecked } } = settings;
  const checkBoxSelector = '.' + [defaultTo(defaultNamespace, namespace), ++counter].join('-');
  const __label = defaultTo('', _label);

  assertContract(isCheckBoxSettings, [settings.checkBox], `CheckBox : Invalid check box settings! : ${format(settings.checkBox)}`)

  const events = {
    'change' : sources[DOM_SINK]
      .select(checkBoxSelector).events('change')
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
                span(checkBoxTextSelector, __label)
              ])
    ])),
    isChecked$: events.change
  }
}
```

The `isChecked$` sink can later on be used in coordination with the `Pipe` combinator to produce 
the desired sink.

# Conclusion
We have seen while implementing the sample application how to address common issues arising when 
implementing a web application :

- **routing** : a quintessential requirement such as routing is very naturally expressed with 
 the `OnRoute` combinator.
- **state management** : state can be injected at any point of the component tree and becomes 
visible to any component down the injection point. Alternatively, state can also be kept at the
 root level, through the use of in-memory store.
- **change propagation** : at the lowest level, using streams as the corner stone of our 
architecture solves the issue of updating a variable (behaviour) when one of its dependencies 
change. *Live queries* can then be built on top of read and write drivers as exemplified in the 
sample application. Additionally, we offer the `ForEach` combinator, to execute a given logic on 
a every change of a behaviour. 
- **communication between components** : parent-child communication may occur through passing 
settings and sources, child-parent communication and communication between components with no 
direct ascendency relationship in the component tree may occur via shared state. 
- **lists** : list of things are dealt with reactively with the `ListOf` and `ForEach` combinators. 

While these were not encountered in the present sample application, our combinator library also 
helps deal with :

- **control flow** : Two combinators (`Switch` and `FSM`) allow to implement both simple and 
complex control flow logic. A [realistic example](https://github.com/brucou/component-combinators/tree/master/examples/volunteerApplication) for the `FSM` combinator showcases the advantage of state machines to that purpose.

We have showcased our componentization model, and the accompanying combinator library. By 
dividing the target reactive system into smaller components, and expliciting the event-state-action
 relation, it is possible to reach a breakdown where each leaf component is easy to write. At the
  same time, those relatively independent components are glued into the cohesive whole that is 
  the target application, in a way that is made both concise and readable by the use of combinators.
