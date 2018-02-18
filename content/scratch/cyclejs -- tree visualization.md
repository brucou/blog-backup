# Motivation
bla bla about how important in a dataflow architecture to view the data flowing for finding, 
understanding and remediating errors.

- all component combinators call m
- would be nice to have different visual description for combinators too

# Algorithm
- m(spec, settings, components)
- every settings will have a reserved _path, and _name property
- DOC : all private settings should start with _
- _path should be the path in the component tree to a given component
- ?? PUT THE NAME IN THE SPECS?? probably but also, not instead
  - would be name of the combinator vs. name of the component

```javascript
App = m(spec, {_name:'App'}, [
  m(spec, {_name: 'Level 1'}, [
    m(spec, {_name: 'Level 1.1'}, [LeaveComponents])
  ]),
  m(spec, {_name: 'Level 2'}, [
    m(spec, {_name: 'Level 2.1'}, [LeaveComponents])
  ]),  
])
```

should give :

|---App----|
|          |-----App sinks----
|          |
------------
