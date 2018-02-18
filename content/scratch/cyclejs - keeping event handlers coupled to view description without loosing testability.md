---
title: "cyclejs - coupling event handlers to view description, while keeping testability"
date: 2018-01-12
lastmod: 2018-01-12
draft: false
tags: ["functional programming", "reactive programming", "components"]
categories: ["programming"]
author: "brucou"
toc: true

# you can close something for this content if you open it in config.toml.
comment: false
mathjax: true
---

# Having event handlers declared together with the view

- In sources, inject a subject factory
- In component, with that subject factory, create subject
  - subject will have automatically generatd unique id from the trace settings property 
- In view, with helper function, link the event listener to the subject
- in component, returns the subject 
- when testing, send values through the subject

think some more I am close

- conclusion : whatever I do is the same as what I have, 
- event@selector is necessary to test anyways. Even if I remove it from implementation, I need it
 for test, so it is all the same to keep it separated or joined.
  - so keep mock : `DOM!event@selector`
  - BUT change syntax to remove subject factory declaration in runTestScenario.
  - example : `handler!event@selector` -> factory will be for events (E) -> new Subject() 
  - example : `document!property@selector` -> factory will be for state (S) -> new 
     - the latter means I need to change document driver to have common syntax to access DOM state
     - possibly reimplementing the DOM interface, so us 80-20 approach
     - for the 20, well, test by hand or with selenium
- BUT maybe it is worth to have the factor as dependency in sources
  - sources.eventHandler(selector, event) curried to accept only selector
  - in the snabbdom, change module listener to accept {handlers : {event : curriedHandler}
    - instead of `on` so no telescoping, and fit the naming of the source
  - in the implementation of module apply curriedHandler to `[event]`, and attach handler which 
  passes the event to the subject as any handler
  - pass the subject as output
  - BENEFITS :
    - no more DOM driver, just snabbdom!
    - BUT : we still have to declare handler on body! do it through a specific driver -> so DOM 
    driver again? no, there is always a body so should be simple. How to manage the lifecycle? 
    body never goes away. Same : explicit removal, just like explicit creation
      - NO, can work with same curried API? TO THINK ABOUT
        - YES : pass the subject as argument in the body driver, i.e. just like snabbdom does, 
        but isntead of snabbdom driver, body driver
    - KEEP document driver to query DOM state, call it domQuery driver? like domainQuery driver?
      - YES, same as domainQuery, a factory which returns a shared replay stream, mock as usual
        - instead of getCurrent(domain entity) I will have dom.getCurrent(selector, property)
  - handler driver : that is a subject factory, `makeHandlerDriver(subjectFactory)`
  - FOR TESTS:
    - example : `handler!event@selector` -> factory will be for events (E) -> new Subject() 
    - example : `document!property@selector` -> factory will be for state (S) -> new ReplaySubject(1)
       - be careful in implementation vs. test
         - if test returns stream (replayed), then `sources.document` must also return a stream. To
          keep semantics, that stream should have only one value and be queried with `withLatestFrom` 
          always, or sample (beware of termination semantics, there is only one value). Ok, that 
          is how it works, query driver returns a stream no matter what
SOLVED!!! I can test and I get rid of the dom driver. I also solve the body selector problem

- React
  - does not allow to define handler on body either
