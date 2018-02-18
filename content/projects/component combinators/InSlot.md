---
title: "InSlot"
date: 2018-01-02
lastmod: 2018-01-07
draft: false
tags: ["functional programming", "reactive programming", "user interface"]
categories: ["documentation", "programming"]
author: "brucou"
toc: true
mathjax: true
comment: false
---
# Motivation
The `InSlot` combinator allows to assign a named slot to DOM content. This is to be used in 
conjunction with container components, which marks the position of those slots within the DOM 
content of the container component.

In simple cases with only DOM content, slots can be directly set with the slot module (`div({slot : 'xxx'}, [dom content])`). In the case of components which can return any number of sinks, assigning a slot to their DOM content can only be done with the `InSlot` combinator.

Together with the slot module, the `InSlot` is a fundamental piece in implementing a slot 
mechanism à la web components.
 
# API

## InSlot :: InSlotSettings -> [Component] -> SlottedComponent

### Description
The `InSlot` combinator only take **one** component passed within an array as parameter. This is 
to keep the same signature as other combinators. The `InSlotSettings` is not an object this time 
but a string which is the name of the slot to be allocated.

The behaviour is as follows :

- the component is executed and produces its sinks
- the DOM sink has its content updated **in place** to add the slot name via the slot module. As 
such, if there was already a slot name assigned, it is replaced.

A more thorough presentation of the slot mechanism is given in [another article](/projects/component-combinators/m-component---merge-default-functions/#slotted-dom-merge). 

### Types
- `InSlotSettings :: SlotName`
- `SlotName :: String`

### Contracts
None. We have been lazy here. We should check that the slot name is indeed a string.

# Example
cf. [tests for `m` combinator](https://github.com/brucou/component-combinators/blob/master/test/m.specs.js#L873)

# Tips
