---
title: "EFSM combinator - sample application"
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

# Introduction
We presented in previous articles:
 
 - the case for using [state machines for UI programming](/projects/component-combinators/efsm---the-case-for-ui-programming/), 
 - [our state machine library](/projects/component-combinators/efsm---documentation/), in the 
 larger frame of our combinator library
 
 In the present article, we will illustrate our state machine library by implementing an average-complexity example 
 application. We will first introduce the sample application, in its two iterations, including functional specification, lo-fi wireframes, and user experience flowcharts. We will then implement that application in a stepwise manner (though focusing on 
 relevant parts of the application). We finish with the known limitations to the approach and 
 miscellaneous tips from our experience using state machines.

**Prerequisites**

- understanding of stream concepts
- `cyclejs` knowledge, and Rxjs library knowledge for the implementation section
- `JSON patch` RFC 6902

# Sample application

We will describe a user interface application which allows a volunteer to apply for a festival. That description consists of :

- functional specifications
- wireframes flows

## High-level specifications

### Initial version
Every festival requires prospective volunteers to :

- provide pieces of information about themselves
- answer a motivational question whose answer will be reviewed by the volunteer coordinator
- apply for a list of volunteers' teams by answering a specific motivational question on that team

The designed process to allow a volunteer to apply to several teams with the minimal amount of steps is the following :

- fill-in personal information
- answer the festival's motivational question
- review all teams which can be joined and join the one he/she is interested in
- for each selected team, the volunteer can review some details about that team and answer a 
team-specific motivational question
	- the volunteers cannot continue to the review step if he has applied to no team
- at the end of the process, the volunteer reviews its choices
- from there, the volunteer can either finalize his application, or modify his application

If the volunteer interrupts its application in a given step, it can resume that application at a 
convenient starting point. When reaching the review step, the volunteer can modify his 
application, choosing the section of the application he wants to modify. When he finishes doing so, the next step must be to go back to the review step, where he sees all information about his application.

Other relevant information :

- 90% of students are using the application on low-tech small-screen mobile.
- All data about festival and teams is available and stored on a remote repository.
- When the volunteers' application is finalized, the application data is saved in the remote repository.

### Second iteration (after user feedback)
Before continuing to the next step of the application, user inputs must be checked for validity. When some inputs are invalid, the relevant error messages will be displayed.

## Wireframes 
After discussion with stakeholders, the proposed flow is summarized with the following wireframes :

![(Application process wireframes)](http://i.imgur.com/BFjfgWZ.png "Application process wireframes")

# Sample application implementation - first iteration

## Modelization
The events to process are pretty obvious from the wireframe flows (button clicks, etc.).
We will match each screen with a control state of our state machine. That makes 5 control states. To each of this control state, we will have a dataflow component whose only action request will be to update the DOM with that screen content. 
The quantitative state will have to hold all necessary data for each screen to draw itself, and the guards and actions to describe their logic. This includes :

- all field contents for each screen
- the step (`progress indicator` region) of the application process
- an error message field to give feedback to the user if something went wrong
- validation messages representing the hints related to the content for each field

Transitions and guards are partly given by the wireframe flows. They will have to be completed so there is no logic hole in the design (i.e. a logic branch which is not considered).

The resulting target workflow is then established :

![First iteration workflow](/img/graphs/sparks application process with errors flow.png)

## Events
The events processed by the state machine are defined as follows :

```javascript
export const events = {
  [FETCH_EV]: fetchUserApplicationModelData,
  [ABOUT_CONTINUE]: aboutContinueEventFactory,
  [QUESTION_CONTINUE]: questionContinueEventFactory,
  [TEAM_CLICKED]: teamClickedEventFactory,
  [SKIP_TEAM_CLICKED]: skipTeamClickedEventFactory,
  [JOIN_OR_UNJOIN_TEAM_CLICKED]: joinTeamClickedEventFactory,
  [BACK_TEAM_CLICKED]: backTeamClickedEventFactory,
  [TEAM_CONTINUE]: teamContinueEventFactory,
  [CHANGE_ABOUT]: changeAboutEventFactory,
  [CHANGE_QUESTION]: changeQuestionEventFactory,
  [CHANGE_TEAMS]: changeTeamsEventFactory,
  [APPLICATION_COMPLETED]: applicationCompletedEventFactory
};
```

As an example, here is the fetch event factory :

```javascript
export function fetchUserApplicationModelData(sources: any, settings: any) {
  const { user$ } = sources;
  const { opportunityKey, userKey } = settings;
  const userApp$ = fetchUserApplicationData(sources, opportunityKey, userKey);
  const teams$ = sources.query$.query(TEAMS, {});
  const opportunities$: Stream<Opportunity> = sources.query$.query(OPPORTUNITY, { opportunityKey });

  // NOTE : combineArray will produce its first value when all its dependent streams have
  // produced their first value. Hence this is equivalent to a zip for the first value, which
  // is the only one we need anyways (there is no zipArray in most)
  return combineArray<FirebaseUserChange, Opportunity | null, UserApplication | null, Teams | null, any>(
    (user, opportunity, userApplication, teams) =>
      ({
        user,
        opportunity,
        userApplication,
        teams,
        errorMessage: null,
        validationMessages: {}
      }),
    [user$, opportunities$, userApp$, teams$]
  )
    .take(1)
}
```

And here is the factory for the click on the `continue` button at the `About` step.

```javascript
export function aboutContinueEventFactory(sources: any, settings: any) {
  return sources.dom.select('button.c-application__submit--about').events('click')
    .tap(preventDefault)
    .map((x: any) => ({formData : getAboutFormData() }))
}
```


## Transitions

### Initialization
To jump start the state machine, we will fetch the stored application from the server via `FETCH` event, which will be immediately emitted on state machine initialization (after the initial event is emitted). This means that the state machine is initialized with dummy/empty initial model (it will be fetched) and initial event data (unused). 

```javascript
  T_INIT: {
    origin_state: INIT_STATE,
    event: INIT_EVENT_NAME,
    target_states: [
      {
        event_guard: EV_GUARD_NONE,
        re_entry: true, // necessary as INIT is both target and current state in the beginning
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: ACTION_GUARD_NONE,
            target_state: INIT_S,
            model_update: modelUpdateIdentity
          }
        ]
      }
    ]
  },
```

For the fetch event data, we determine in which step the volunteer left the application and transition to the corresponding control state. But before that, we must check if the volunteer has actually already finished its application and apply the corresponding logic. This gives us the following transitions :

```javacript
  dispatch: {
    origin_state: INIT_S,
    event: FETCH_EV,
    target_states: [
      {
        // whatever step the application is in, if the user has applied, we start with the review
        event_guard: hasApplied,
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: ACTION_GUARD_NONE,
            target_state: STATE_REVIEW,
            // Business rule
            // if the user has applied, then he starts the app process route with the review stage
            model_update: initializeModelAndStepReview
          }
        ]
      },
      {
        event_guard: isStep(STEP_ABOUT),
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: ACTION_GUARD_NONE,
            target_state: STATE_ABOUT,
            model_update: initializeModel // with event data which is read from repository
          }
        ]
      },
      {
        event_guard: isStep(STEP_QUESTION),
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: ACTION_GUARD_NONE,
            target_state: STATE_QUESTION,
            model_update: initializeModel // with event data which is read from repository
          }
        ]
      },
      {
        event_guard: isStep(STEP_TEAMS),
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: ACTION_GUARD_NONE,
            target_state: STATE_TEAMS,
            model_update: initializeModel // with event data which is read from repository
          }
        ]
      },
      {
        event_guard: isStep(STEP_REVIEW),
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: ACTION_GUARD_NONE,
            target_state: STATE_REVIEW,
            model_update: initializeModel // with event data which is read from repository
          }
        ]
      }
    ]
  },
```

### Transition away from `About` screen
We only have one event to consider here : the click on the `Continue` button. The application logic as specified is :

- if the user had already reached the review step, then click on `Continue` must : 
  - update the user application remotely with the newly entered form data
  - send the volunteer to the review step
-  otherwise it must : 
  - update the user application remotely with the newly entered form data
  - send the volunteer to the `Question` step

The corresponding transitions are defined here : 

```javascript
  fromAboutScreen: {
    origin_state: STATE_ABOUT,
    event: ABOUT_CONTINUE,
    target_states: [
      {
        // Case form has NOT reached the review stage of the app
        event_guard: complement(hasReachedReviewStep),
        action_request: {
          driver: 'domainAction$',
          request: makeRequestToUpdateUserApplication
        },
        transition_evaluation: makeDefaultActionResponseProcessing({
          success: {
            target_state: STATE_QUESTION,
            model_update: updateModelWithAboutDataAndStepQuestion
          },
          error: {
            target_state: STATE_ABOUT,
            model_update: updateModelWithStepAndError(updateModelWithAboutData, STEP_ABOUT)
          }
        })
      },
      {
        // Case form has reached the review stage of the app
        event_guard: T,
        re_entry: true,
        action_request: {
          driver: 'domainAction$',
          request: makeRequestToUpdateUserApplication
        },
        transition_evaluation: makeDefaultActionResponseProcessing({
          success: {
            target_state: STATE_REVIEW,
            model_update: updateModelWithAboutDataAndStepReview
          },
          error: {
            target_state: STATE_ABOUT,
            model_update: updateModelWithStepAndError(updateModelWithAboutData, STEP_ABOUT)
          }
        })
      },
    ]
  },
```

For information, here are some examples of model update functions :

```
export const aboutYouFields = ['superPower'];
export const personalFields = ['birthday', 'phone', 'preferredName', 'zipCode', 'legalName'];
export const questionFields = ['answer'];

export const updateModelWithAboutDataAndStepQuestion = chainModelUpdates([
  updateModelWithAboutData,
  updateModelWithEmptyErrorMessages,
  updateModelWithStepOnly(STEP_QUESTION)
]);

export function updateModelWithAboutData(model: FSM_Model, eventData: EventData,actionResponse: DomainActionResponse) {
  const formData = eventData.formData;

  return flatten([
    toJsonPatch('/userApplication/about/aboutYou')(pick(aboutYouFields, formData)),
    toJsonPatch('/userApplication/about/personal')(pick(personalFields, formData)),
  ])
}

....etc....
```

and example of action request :

```
///////
// Action requests
export function makeRequestToUpdateUserApplication(model: UserApplicationModelNotNull, eventData: any) {
  const formData = eventData.formData;
  const { userApplication } = model;
  const newUserApplication  = getUserApplicationUpdates(formData, userApplication);
 
  return {
    context: USER_APPLICATION,
    command: UPDATE,
    payload: newUserApplication
  }
}
```

### Other transitions
The full implementation can be found in the [corresponding repo](https://github.com/brucou/component-combinators/tree/master/examples/volunteerApplication)

## State entry components

They will mainly serve to display the view derived from the value of the `model` parameter. For instance, the entry component for the state `About` is :

```javascript
export const entryComponents = {
...
[STATE_ABOUT]: function showViewStateAbout(model: UserApplicationModel) {
    return flip(renderComponent(STATE_ABOUT))({ model })
  },
}
...
```

with `renderComponent` being :

```javascript
function render(model: UserApplicationModelNotNull) {
  const { opportunity, userApplication, errorMessage } = model;
  const { description } = opportunity;
  const { about, questions, teams, progress } = userApplication;
  const { step, hasApplied, latestTeamIndex } = progress;

  return div('#page', [
    div('.c-application', [
      div('.c-application__header', [
        div('.c-application__opportunity-title', description),
        div('.c-application__opportunity-icon', 'icon'),
        div('.c-application__opportunity-location', 'location'),
        div('.c-application__opportunity-date', 'date'),
      ]),
      div('.c-application__title', `Complete your application for ${description}`),
      div('.c-application__progress-bar', flatten([renderApplicationProcessTabs(step)])),
      form('.c-application__form', flatten([renderApplicationProcessStep(step, model)])),
      errorMessage
        ? div('.c-application__error', `An error occurred : ${errorMessage}`)
        : div('.c-application__error', '')
    ]),
  ]);
}

function _renderComponent(state: State, sources: any, settings: any) {
  void sources;
  const model: UserApplicationModelNotNull = settings.model;
  console.info(`entering ${state}`, model);

  return {
    dom: just(render(model))
  }
}
export const renderComponent = curry(_renderComponent);
```

# Sample application implementation - second iteration

## Modelization
We add the following fields to the model :

- validation messages representing the hints related to the content for each field

The previous workflow is updated as follows :

![Second iteration workflow](/img/graphs/sparks%20application%20process%20with%20validation%20and%20errors%20flow.png)

## Events
We have to add the input form data validation functionalities. Control flow is decided at the guard level, and will basically include two branches : go to next step if fields valid ; remain in same step if one field is invalid. 
Validating involves first retrieving the form data, and passing them through some validation function. 
Retrieving form data will be performed in the event factory as it is a read effect, and we want to keep guards as pure functions.
We choose here to have the validation function return `true` if a field passed validation, and a string representing the error in case the field does not pass validation. The validation will also be performed in the event factory, as we want to keep guards are single-concern functions. Furthermore, including the validation at the guard level would imply that the validation is run several times (for each guard), which is a source of inefficiency.

Applying these ideas to the `events` object, leads, for instance, to this factory for the click on the `continue` button at the `About` step becomes as follows:

```javascript
export function aboutContinueEventFactory(sources: any, settings: any) {
  return sources.dom.select('button.c-application__submit--about').events('click')
    .tap(preventDefault)
    .map((x: any) => {
      const formData = getAboutFormData();

      return {
        formData,
        validationData: validateScreenFields(aboutScreenFieldValidationSpecs, formData)
      }
    })
}
```

## Transisition away from `About` screen
As before, we only have one event to consider here : the click on the `Continue` button. The application logic as now specified includes another branching level and becomes (the modifications vs. former specifications are in bold):

- **if the form data is valid then :**
	- if the user had already reached the review step, then click on `Continue` must :
		- update the user application remotely with the newly entered form data
		- send the volunteer to the review step
	- otherwise it must :
		- update the user application remotely with the newly entered form data
		- send the volunteer to the `Question` step
- **if the form data was invalid then :**
	- **the screen must show some error messages and visual feedback**

We hence now have three branches in our control flow (vs. two previously) and the corresponding updated transitions are defined here : 

```javascript
  fromAboutScreen: {
    origin_state: STATE_ABOUT,
    event: ABOUT_CONTINUE,
    target_states: [
      {
        // Case form has only valid fields AND has NOT reached the review stage of the app
        event_guard: both(isFormValid, complement(hasReachedReviewStep)),
        action_request: {
          driver: 'domainAction$',
          request: makeRequestToUpdateUserApplication
        },
        transition_evaluation: makeDefaultActionResponseProcessing({
          success: {
            target_state: STATE_QUESTION,
            model_update: updateModelWithAboutDataAndStepQuestion
          },
          error: {
            target_state: STATE_ABOUT,
            model_update: updateModelWithStepAndError(updateModelWithAboutData, STEP_ABOUT)
          }
        })
      },
      {
        // Case form has only valid fields AND has reached the review stage of the app
        event_guard: both(isFormValid, hasReachedReviewStep),
        re_entry: true,
        action_request: {
          driver: 'domainAction$',
          request: makeRequestToUpdateUserApplication
        },
        transition_evaluation: makeDefaultActionResponseProcessing({
          success: {
            target_state: STATE_REVIEW,
            model_update: updateModelWithAboutDataAndStepReview
          },
          error: {
            target_state: STATE_ABOUT,
            model_update: updateModelWithStepAndError(updateModelWithAboutData, STEP_ABOUT)
          }
        })
      },
      {
        // Case form has invalid fields
        event_guard: T,
        re_entry: true,
        action_request: ACTION_REQUEST_NONE,
        transition_evaluation: [
          {
            action_guard: T,
            target_state: STATE_ABOUT,
            // keep model in sync with repository
            model_update: updateModelWithAboutValidationMessages
          },
        ]
      }
    ]
  },
```

We also have to update the model update functions to reflect the changes in the model (which now includes validation messages). A change in model means revisiting all existing model update functions and possibly create new model update functions. 
Here `updateModelWithAboutValidationMessages` is a new cretaed function. Existing model update functions will be updated to reflect the rule that validation messages are empty when the fields are valid. For instance, the function `updateModelWithAboutDataAndStepQuestion` adds `updateModelWithEmptyErrorMessages` to its updates and becomes :

```javascript
export const updateModelWithAboutDataAndStepQuestion = chainModelUpdates([
  updateModelWithAboutData,
  updateModelWithEmptyErrorMessages,
  updateModelWithStepOnly(STEP_QUESTION)
]);

```

There is no further updates to the actions, and action guards. 

### Other transitions
The full implementation can be found in the [corresponding repo](https://github.com/brucou/component-combinators/tree/master/examples/volunteerApplication).

## State entry components

State entry components have to be updated to display visually the validation messages passed through the `model` parameter.

## Conclusion

In summary, the seemingly simple changes in specifications necessitated :

- changes in the model
	- reviewing all model update functions to reflect the new model dependencies. Here we added a new field to the model, so changes in model update functions were obvious and minimal.
- changes in event factories
	- event now includes new data which are necessary to compute the validation rules
- changes in the control flow
	- no new control state
	- new transition logic implemented through update of event guards
- changes in the state entry component
	- updating the view rendering model to display the validation messages

We have seen that :

- changes in the specifications are easily translated into changes in the implementation
	- most functions are pure functions with one single concern, hence it is easy to decide whether 
	to modify and how to modify such functions
	- event factories have two concerns : capturing the source event, and producing the event data (producing in the way read effects). Given that the source event does not change, changes in the event factories were minimal
- changes in the `model` require reviewing all functions taking `model` as a parameter (event, guards, action requests, entry components). However, because not all functions do not use all parts of the model, we are able to bring down the number of modifications performed on model-related functions.

The lesson to be learnt is that special care have to be given to how the `model` is defined, as it is the key source of complexity left unabridged :

- writing the model in a way that all functions use all parts of the model gives the worse case for maintainability
- the model, by nature, gathers several concerns to be transmitted to different parts of the state machine component. It is good practice to isolate early cross-cutting concerns, and separate the concerns which are orthogonal into different properties. This would help ensure that a modification of the model structure have lower impact on the implementation.

# Known limitations 
## You can still write an unmaintainable *spaghetti* implementation
The need for guards in the extended state machine formalism is the immediate consequence of adding memory (extended state variables) to the state machine formalism. Used sparingly, extended state variables and guards make up an incredibly powerful mechanism that can immensely simplify designs. But don’t let the fancy name (“guard”) fool you. When you actually code an extended state machine, the guards become the same `IF`s and `ELSE`s that you wanted to eliminate by using the state machine in the first place. Too many of them, and you’ll find yourself back in square one (“spaghetti”), where the guards effectively take over handling of all the relevant conditions in the system.

Indeed, abuse of extended state variables and guards is the primary mechanism of architectural decay in designs based on state machines. Usually, in the day-to-day battle, it seems very tempting, especially to programmers new to state machine formalism, to add yet another extended state variable and yet another guard condition (another if or an else) rather than to factor out the related behavior into a new qualitative aspect of the system—the control state. The likelihood of such an architectural decay is directly proportional to the overhead (actual or perceived) involved in adding or removing states. 

One of the main challenges in becoming an effective state machine designer is to develop a sense for which parts of the behavior should be captured as the qualitative” aspects (the “control state”) and which elements are better left as the “quantitative” aspects (extended state variables).


## State - transition topology is static
Capturing behavior as the “quantitative state” has its disadvantages and limitations, too. First, the state and transition topology in a state machine must be static and fixed at compile time, which can be too limiting and inflexible. Sure, you can easily devise state machines that would modify themselves at **runtime** (this is what often actually happens when you try to recode “stream spaghetti” as a state machine). However, this is like writing self-modifying code, which indeed was done in the early days of programming but was quickly dismissed as a generally bad idea. Consequently, “state” can capture only static aspects of the behavior that are known a priori and are unlikely to change in the future.

[//]: # (ref : Ref : A_Crash_Course_in_UML_State_Machines (Quantum Leaps)


## Does not do well with a high number of control states
While extended FSMs are an excellent tool for tackling smaller problems, it is also generally known that they tend to become unmanageable even for moderately involved systems. Due to the phenomenon known as "state explosion", the complexity of a traditional FSM tends to grow much faster than the complexity of the reactive system it describes.

The issue is not so much the number of states than the transition topology of the modelized reactive system. Some events can lead to the same action and target state, independently of the current state. This means repeating a transition as many times as there are control states, hence the exponential growth of the complexity.
This issue is considerably alleviated with the statecharts formalism (also called Hierarchical State Machines), but we chose to remain at a simpler and more accessible conceptual level of extended state machine. We will see other techniques which help in practice keep the number of control states low.

[//]: # (http://self.gutenberg.org/articles/Hierarchical_state_machine)

## Run-to-Completion execution model can over-simplify concurrency issues 
The key advantage of RTC processing is simplicity. Its biggest disadvantage is that the responsiveness of a state machine is determined by its longest RTC step. Achieving short RTC steps can sometimes significantly complicate real-time designs. 

In a future version of the library, event queuing and convenient interruption policies will be added.

[//]: # (A_Crash_Course_in_UML_State_Machines (Quantum Leaps))

# How to
## Perform operations at initialization
The common use case is that you might want to fetch some remote data and update the internal model of the state machine with some of that.  Two options :

1. Use `INIT` event in the state machine definition 
Every state machine has a `INIT` event which is sent when the state machine is activated. It is possible to configure a transition with an action which updates the model from its result
2. Use ONE event factory which immediately emits an event (i.e. synchronously)
When a state machine is activated, event factories are immediately executed, and the state machine starts listening on events coming from those event factories. If ONE of those event factories immediately emits an event, that event will immediately be processed (after the initial `INIT` event kicking off the state machine). It is better to have only one immediate event, as if there would be several, there is no guarantee on the order with which those events would be processed, hence there is no guarantee on their effects.
