---
title: "Mocking the DOM driver"
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
In the context of our cyclejs-based architecture, updates to the DOM are realized by emitting 
objects containing a virtual description of the target DOM content, to a dedicated DOM driver. To
 be able to test applications written within that architecture, we propose a mechanism to mock 
 the DOM driver, hence avoiding the side-effects performed by such driver. 
 
 As of now, We will not seek to mock the DOM **update**, but we will mock the DOM **event 
 handling**, in order to be able to simulate inputs arriving through the DOM. A typical test 
 session will simulate inputs without incurring in effects (binding event handlers for example), 
 and match the update commands received by the DOM driver vs. the expected commands as derived 
 from the specifications of the application under test.
 
 Typically, in a code source, the DOM source passed by the DOM driver is used as follows : `sources.DOM.select('selector').events(myEvent)`. In connection with the `runTestScenario` [mock factory syntax](https://brucou.github.io/projects/component-combinators/runtestscenario/#mocking), we propose here a syntax 
 `DOM!selector@myEvent` for mocked inputs.

# API
The `makeMockDOMSource` function is exported to be used in conjunction with `runTestScenario` 
helper. For example : 

```javascript
    const inputs = [
      {
        'DOM!input@click': {
          diagram: 'xy|', values: {
            x: makeDummyClickEvent('a-0'), y: makeDummyClickEvent('a-1')
          }
        }
      },
      {
        'DOM!a@hover': {
          diagram: '-xyz|', values: {
            x: makeDummyHoverEvent('a-0'),
            y: makeDummyHoverEvent('a-1'),
            z: makeDummyHoverEvent('a-2'),
          }
        }
      },
      ...
    ]
    
    ...
    
    runTestScenario(inputs, expected, testFn, {
      ...
      mocks: {
        DOM: makeMockDOMSource
      },
      sourceFactory: {
        'DOM!input@click': () => new Rx.Subject(),
        'DOM!a@hover' : () => new Rx.Subject(),
      },
      ...
    });
  });
```

Note the mocked input syntax and the passing of timed diagrams. Also, as of now, 
`runTestScenario` requires the user to specify a factory for the input streams to which the 
mocked inputs will be sent. In the standard case (events), and in the context of `Rxjs` as a 
streaming library, this will be a `() => new Rx.Subject()`. 

# Example
cf. [`runTestScenario` tests](https://github
.com/brucou/component-combinators/blob/master/test/runTestScenario.specs.js). Also available, 
some [tests](https://github.com/brucou/component-combinators/blob/master/examples/volunteerApplication/test/app.specs.js#L86) from an example application.

# Tips
