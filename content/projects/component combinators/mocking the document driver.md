---
title: "Mocking the document driver"
date: 2018-01-27
lastmod: 2018-01-27
draft: false
tags: ["functional programming", "reactive programming", "user interface"]
categories: ["documentation", "programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# Motivation
In the context of our cyclejs-based architecture, to read DOM state (input field content, text 
field content, checkbox state, etc.) is realized by means of the `document` driver. That driver 
is simply an injection of the global browser's `document` object. For instance, to read an input 
field's value, assuming the css selector is `selector`, can be achieved by `sources.document.querySelector(selector).value`.
  
To be able to test applications written within that architecture, we propose a mechanism to mock 
 the document driver, hence avoiding the side-effects performed by such driver. 
 
 In connection with the `runTestScenario` [mock factory syntax](https://brucou.github.io/projects/component-combinators/runtestscenario/#mocking), we propose here a syntax 
 `document!selector@property` for mocked inputs. This will match a code written as `sources.document.querySelector(selector)[property]`.

# API
The `makeMockDocumentSource` function is exported to be used in conjunction with `runTestScenario` 
helper. For example : 

```javascript
  const inputs = [
    ...
    {
      [`document!value@${USER_APPLICATION_FORM_INPUT_SUPERPOWER_SELECTOR}`]: {
        diagram: 'u----', values: {u: ""}
      }
    },
    {
      [`document!value@${USER_APPLICATION_FORM_INPUT_LEGAL_NAME_SELECTOR}`]: {
        diagram: 'u----', values: {u: ""}
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
      [`document!value@${USER_APPLICATION_FORM_INPUT_SUPERPOWER_SELECTOR}`]: subjectFactory,
      [`document!value@${USER_APPLICATION_FORM_INPUT_LEGAL_NAME_SELECTOR}`]: subjectFactory,
      ...
    },
```

Note the mocked input syntax and the passing of timed diagrams. Also, as of now, 
`runTestScenario` requires the user to specify a factory for the input streams to which the 
mocked inputs will be sent. In the case of reading state from the DOM (i.e. behaviours), this would be a `() => new Rx.ReplaySubject(1)`. However, as of now, the subject factory is irrelevant. Only `querySelector` of the 
global window `document` object is mocked, and because the `document` driver just passes on the 
`document` object, there is no need for a subject factory. A subject factory must nonetheless 
appear in the `sourceFactory` setting for contract checking purposes (for every `x!y` mocked 
input, there must be a corresponding factory in `sourceFactory`).


# Contracts
- In `document!selector@property`, both `selector` and `property` must be non empty.

# Example
cf. [tests](https://github.com/brucou/component-combinators/blob/master/examples/volunteerApplication/test/app.specs.js#L159) from an example application.

# Tips
