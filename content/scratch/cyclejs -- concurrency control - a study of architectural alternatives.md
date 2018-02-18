---
title: "cyclejs and concurrency control - a study of architectural alternatives"
date: 2018-01-12
lastmod: 2018-01-12
draft: false
tags: ["functional programming", "reactive programming", "components"]
categories: ["programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# concurrency control
cf. Redux saga
- all actions in one sink
- several concurrent processes listening on actions
- coordinated by ? shared-memory? message-passing? streams (declarative concurrncy)? investigate
- https://redux-saga.js.org/, https://redux-saga.js.org/docs/recipes/, http://konkle.us/master-complex-redux-workflows-with-sagas/

cf. Concepts, Techniques, and Models of Computer Programming ou PROGRAMMATION Concepts, 
techniques et modèles for concurrency models
