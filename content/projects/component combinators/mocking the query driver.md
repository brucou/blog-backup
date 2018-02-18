---
title: "Mocking the query driver"
date: 2018-01-27
lastmod: 2018-01-29
draft: false
tags: ["functional programming", "reactive programming", "user interface"]
categories: ["documentation", "programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# Motivation
In the context of our cyclejs-based architecture, to read external system's state (database content, external device, etc.) is realized by means of a `domainQuery` driver. That driver relies on a domain model abstraction : 
a query has a context (for example a bounded context, or a domain entity), and parameters. As of 
now, the `domainQuery` only allows one type of domain operation (typically a `get` operation).

Here is an example of configuration for a domain with two entities (projects and opportunities) :

```javascript
export const domainObjectsQueryMap = {
  [PROJECTS]: {
    get: function getProjectByProjectKey(repository, context, params) {
      const { projectKey } = params;
      const localforageKey = [PROJECTS_REF, projectKey].join(KEY_SEP);

      return repository.getItem(localforageKey);
    }
  },
  [OPPORTUNITY]: {
    get: function getOpportunityByOppKey(repository, context, params) {
      const { opportunityKey } = params;
      const localforageKey = [OPPORTUNITY_REF, opportunityKey].join(KEY_SEP);

      return repository.getItem(localforageKey);
    }
  },
  ...
``` 
  
Here is an example of using the `domainQuery` driver :

```javascript
export function fetchUserApplication(domainQuery, opportunityKey, userKey) {
  return domainQuery.getCurrent(USER_APPLICATION, { userKey, opportunityKey })
}
```

To be able to test applications written within our architecture, we propose a mechanism to mock 
 the `domainQuery` driver, hence avoiding the side-effects performed by such driver. 
 
 In connection with the `runTestScenario` [mock factory syntax](https://brucou.github.io/projects/component-combinators/runtestscenario/#mocking), we propose here a syntax 
 `domainQuery!stringifiedParams@context` for mocked inputs. This will match a code written as `sources.domainQuery.getCurrent(context, parsedParams)`, where `parsedParams = JSON.parse(stringifiedParams)`. 

# API
The `makeDomainQuerySource` function is exported to be used in conjunction with `runTestScenario` 
helper. For example : 

```javascript
  const inputs = [
    ...
    {
      [`domainQuery!${mockUserAppParams}@${USER_APPLICATION}`]: {
        diagram: 'u----', values: { u: null }
      }
    },
    {
      [`domainQuery!${mockTeamsParams}@${TEAMS}`]: {
        diagram: 'v----', values: { v: teams[`${TEAMS_REF}!${fakeProjectKey}`] }
      }
    },
    ...
    ];

  runTestScenario(inputs, testResults, ProcessApplication, {
    tickDuration: 5,
    waitForFinishDelay: 20,
    analyzeTestResults: analyzeTestResults(assert, done),
    mocks: {
      DOM: makeMockDOMSource,
      document: makeMockDocumentSource,
      domainQuery: makeDomainQuerySource
    },
    sourceFactory: {
      ...
      [`domainQuery!${mockUserAppParams}@${USER_APPLICATION}`]: subjectFactory,
      [`domainQuery!${mockTeamsParams}@${TEAMS}`]: subjectFactory,
      ...
    },
```

Note the mocked input syntax and the passing of timed diagrams. Also, as of now, 
`runTestScenario` requires the user to specify a factory for the input streams to which the 
mocked inputs will be sent. In a live query case, this factory would typically be a `() => new Rx.ReplaySubject(1)`. In a standard case, it will usually be a `() => new Rx.Subject()`, but it will depend on the particular 
component function under test. 

To get a deeper understanding of the consequences of each choice, 
one can refer to the [corresponding tests](https://github.com/brucou/component-combinators/tree/master/test/mockDomainQuery.specs.js), 
and play with changing the type of subjects used, and see how the computed output changes. For 
instance, in the [once query test], changing a standard subject to a memoized subject will lead 
to the last output of `ANOTHER_SOURCE` to change to `${YET_ANOTHER_PREFIX}-${A_STRING}` from
`${YET_ANOTHER_PREFIX}-${A_NUMBER}`, as the source inputs are `null, A_STRING, A_NUMBER`. With a 
memoized subject, the memoized input at the moment of execution of `getCurrent` is used, hence the
 `A_STRING`.  With a standard subject, at the moment of execution of `getCurrent`, the first value
  emitted (in the future) is taken, hence `A_NUMBER`.

# Contracts

- In `domainQuery!stringifiedParams@context`, both `stringifiedParams` and `context` must be non empty.
- both `stringifiedParams` and `context` cannot have a `@` character
- `stringifiedParams` must correspond to a JSON string (meaning for instance that `undefined` is  
not a valid value)
- `context` **may** start with `LIVE_QUERY_TOKEN`. This however must be consistent in the whole 
test parameterization, i.e. it is not allowed in the same test to have the same context with a 
different syntax.

# Example
cf. [tests](https://github.com/brucou/component-combinators/blob/master/examples/volunteerApplication/test/app.specs.js#L199) from an example application.

Also [core tests](https://github.com/brucou/component-combinators/blob/master/test/mockDomainQuery.specs.js) available. 

# Tips
- choose carefully the type of subject factory in function of th implementation under test. 
Nothing more frustating than writing tests that are wrong, with an implementation that is right 
(you can't test the tests, so write the tests right).
