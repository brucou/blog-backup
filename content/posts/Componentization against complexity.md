---
title: "Componentization against complexity"
date: 2017-07-01
lastmod: 2018-01-12
draft: false
tags: ["functional programming", "reactive programming", "components"]
categories: ["programming"]
author: "brucou"
toc: true
comment: false
mathjax: true
---

# The power of abstraction
We have [previously](/posts/user-interfaces-as-reactive-systems/) seen that a reactive system can be described by equations involving a reactive function `f` such that `actions = f(state, events)`. In theory, the function `f` is as complex as the specification of the reactive system to implement. In practice, it is often much more so, as dictated by the particular implementation choices made.

Complexity resists to a uniformally useful definition. However, it is generally accepted that there is a component of complexity which cannot be reduced by any particular technique. <em>How many systems do we have? In what convoluted manner are they connected? What the requirement for operating these systems? How many parameters or subtasks are necessary to specify a task?</em>. That complexity increases every year, as the trend is into interconnecting more and more systems to satisfy ever astringent user requirements to do more things in less time. To reuse a commonly used term, we will call this the essential complexity of the system under specification.

Oddly enough, at the same time, on a subjective level, things that were complex last year are less complex now. That subjective but very real component of complexity relates the number of things that cause problems to our ability to deal with them. Our ability to deal with them goes up as we develop better languages, tools, interfaces, architecture, infrastracture, etc. and are able to do more with less. We would define this component of complexity as accidental complexity, and as being the complexity that an implementation approach adds to the essential complexity corresponding to the specification of a system.

As a side note, a user interface itself is precisely an attempt at shielding the user from the complexity of underlying systems so he can focus on the domain at hand and the tasks it encompasses. In other words, the user interface **abstracts** out the underlying system layer. As a matter of fact, most advances against accidental complexity (in **any** field) are made by better and/or more powerful abstractions.

As a matter of fact, we can list a short range of abstractions taking aim at specific sources of 
complexity, at the system implementation level :

- domain : domain modeling
- programming language : domain-specific language
- persistence : databases
- asynchrony : promises, tasks, streams (!!)
- memory : garbage collector, constructors/destructors
- testing : model-based testing
- design : componentization
- programming paradigm : declarative programming (including functional programming, dataflow programming, logic programming)  

In the current state of my knowledge, I believe that the most effective levers in mastering complexity are domain modelling, domain-specific languages, and components, glued together by means of declarative programming.

We will detail in what follows what makes componentization an abstraction, the benefits to be expected, but also the costs and barriers associated, concluding that componentization is a strategic decision to be pondered over.

# Componentization as an abstraction
The major goal in software construction is to build software that is robust (i.e. performs correctly and does not crash even in unforeseen cases), flexible (i.e. can be used in many seemingly different applications), extensible (i.e. can be extended with minimal if not no modification of existing code), easy to maintain (i.e. to modify to improve performance or fix "bugs"). The main tool of the trade of software developers can be summarized in one word: abstraction.

The abstraction process involves several steps:

- Eliminate all unnecessary details and get to the essential elements of the problem and express them clearly and concisely as invariants
- Encapsulate those which vary and express them as variants.  The variants of abstractly equivalent behaviors are grouped into an appropriate taxonomy of abstract classes and subclasses
- Define the dynamics of the system in terms of the  interplay between the invariants and the variants.  Sometimes the invariants/variants interplays can be straightforward, but more than often they can be subtle like the case when an invariant can be expressed in terms of many variants

The effort spent in identifying and delineating the invariants from the variants usually leads to the construction of what is called a component framework system where the invariants form the framework and the variants constitute the components.

![Component framework](https://www.clear.rice.edu/comp310/JavaResources/frameworks/frameworks.png)

Hence, in essence, component frameworks break the system down into variant "components" that represent abstract processing that the system supports. These components are generally thought of as "pluggable" in the sense that the system can be configured with a set of components to accomplish a particular task or set of tasks.   In many systems, the components can dynamically swapped in and out.   The framework, on the other hand, is the invariant "superstructure" that manages the components, providing services to each component that, when combined with the services/behaviors supplied by that component, create the net behavior defined by that component in that framework.    The framework has the effect of decoupling or isolating the components **from each other** while coupling them to the framework.

# Benefits
As previously mentioned, the benefits are :

- productivity, coming from a component framework design targeted at reuse, and composability
- readability
  - when a given component addresses a specific concern, and do so exclusively (i.e. achieving separation of concerns), the reactive system is easier to reason about
- maintainability
  - components can be replaced easily, by a better or equivalent version, as long as its 
  interface its kept ([program to an interface, not to an implementation](https://tuhrig.de/programming-to-an-interface/))  
- reliability
  - in the measure that components and the component framework are properly tested and reliable, the surface area on which bugs can attach themselves is considerably lower

# Barriers to reuse
Despite all the potential of componentization, significant barriers to component reuse have been observed :

- The business model is unclear
- It costs a client too much to understand and use a component
- Components have conflicting world views

Componentization is hence a strategic decision which will depend on a company's strategic context.

## Unclear business model
Design is expensive, and reusable designs are very expensive. It costs between ½ and 2 times as much to build a module with a clean interface that is well-designed for your system as to just write some code, depending on how lucky you are. But a reusable component costs 3 to 5 times as much as a good module.

The extra money pays for:

- generality
  - A reusable module must meet the needs of a fairly wide range of ‘foreign’ clients, not just of people working on the same project. Figuring out what those needs are is hard, and designing an implementation that can meet them efficiently enough is often hard as well.
- simplicity
  - Foreign clients must be able to understand the interface to a module fairly easily, or it’s no use to them.
- customization
  - To make the module general enough, it probably must be customizable, either with some well-chosen parameters or with some kind of programmability
- testing
  - Foreign clients have higher expectations for the quality of a module, and they use it in more different ways. The generality and customization must be tested as well
- documentation
  - Foreign clients need more documentation, since they can’t come over to your office
- stability
  - Foreign clients are not tied to the release cycle of a system. Ideally for them, a module’s behaviour should remain unchanged (or upward compatible) for years, probably for the lifetime of their system

Designing for all those aspects is an investment that must be recouped in the future. Apart from very large companies, such as google or facebook, and some popular open-source projects, or large software development firms, the business case for such investment is not clear.

## Cost to understand
To use a component, the client must understand its behaviour. This is not just the functional specification, but also the resource consumption, the exceptions it raises, its customization facilities, its bugs, and what workarounds to use when it doesn’t behave as expected or desired.

Furthermore, because the written spec is often quite inadequate, there is uncertainty about the cost to discover the things that are not in the spec, and about the cost to deal with the surprises that turn up. If the module has been around for a while and has many satisfied users, these risks are of course smaller, but it is difficult to reach this happy 
state.

## Conflicting world views
Components are tied to a component framework, which defines its interface and lifecycle, by thus limiting the scope of their reuse out of that framework. Adapters and bridges can be built to palliate that situation. However that is not always possible or practical.

As a matter of fact, Angular2 components cannot be reused out of Angular2, React components can be adapted to React-like frameworks but it takes some efforts, etc. Web components, which are supposed to be a common standard, allowing reuse in all browsers, have been arriving for many years, and are still a work in progress, illustrating the difficulty of designing simultaneously for many targets.

## Componentization is a strategic decision
The option of componentization to implement a reactive system, must, like any architectural decision, be weighted in the context of the specific implementing company/project/team. A start-up who will anyways trash its minimum viable product the next year after receiving funding  might prefer to avoid going through the trouble, while companies which are constantly developing software might want to invest in carefully designed components.

# Conclusion
The eternal quest for productivity in software development is a quest for managing complexity, in
 particular the accidental complexity that we create ourselves by the particular design, set of 
 tools, implementation, etc. thrown at the problem at hand. Four abstractions, in my (unscientific) 
 opinion, give the best result for complexity reduction : declarative programming, domain modeling, 
 domain specific languages, and componentization. 
 
Componentization, however, is a double-edge sword : increasing productivity through reuse in the long-term, but costly in the short-term. As such, the decision of embarking on a componentization effort, and calibrating the scope of such, is a strategic decision which must be taken on a case-by-case basis.

# Bibliography

- F. P. Brooks Jr. No silver bullet - essence and accidents of software engineering. IEEE 
Computer, 20(4):10–19, 1987
- Fundamental Approaches to Software Engineering: 9th International Conference, FASE 2006, Held 
as Part of the Joint European Conferences on Theory and Practice of Software, ETAPS 2006, Vienna, Austria, March 27-28, 2006, Proceedings
- [Why Software Reuse has Failed and How to Make It Work for You](https://www.dre.vanderbilt.edu/~schmidt/reuse-lessons.html)
- [Reducing Complexity in Software & Systems](https://www.sei.cmu.edu/podcasts/podcast_episode.cfm?episodeid=443886)
- [Software Component Models - Component Life Cycle](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.450.9230&rep=rep1&type=pdf)
- [Component Based Systems - 2011](http://www.win.tue.nl/~johanl/educ/2II45/ADS.09.CBSE.pdf)
- [Component-Framework Systems](https://www.clear.rice.edu/comp310/JavaResources/frameworks)
- [A Generic Component Framework for System Modeling](https://link.springer.com/content/pdf/10.1007/3-540-45923-5_3.pdf)
- [Software Components: Only The Giants Survive](https://www.microsoft.com/en-us/research/publication/software-components-only-the-giants-survive/)

