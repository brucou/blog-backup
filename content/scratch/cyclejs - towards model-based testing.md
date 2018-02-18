# Testing process

## Reviewing `runTestScenario`

- In fact we don't really want time diagrams
- We do want predictability of order of inputs
- so we will send each input one after one, separating data from event symbol

Example:

```javascript
const testSpace = {
  scenario1 : {
    caseA : {
      events : 'iccsctttscscS',
      eventData : [
        iData, cData, cData, sData, cData, tData, ...
      ]
    }
  }
}

const eventSymbols = {
  i : eventSimulatorI,
  c : eventSimulatorC,
  ...
}

const testData = testSpace.caseA;

const expected = {
  sinkA : [...],
  sinkB : [...]
};

runTestScenario(testData, eventSymbols, functionUnderTest)
  .then(analyzeResults)
  .catch(notifyErrors)
```

- this should allow to reuse `eventSymbols` between tests
- `testSpace` keeps scenario and cases together (cohesion)
- promise-based return from `runTestScenario` allow for running tests in parallel

TODO :

- see how to write `eventSimulatorI` to allow for [examples like in doc](https://brucou.github.io/posts/user-interfaces-as-reactive-systems/#visualization)
- or examples like before with `DOM!event@selector`, and `doc!property@selector`

## model-based testing

TODO :

- explain what it is
- API to design
  - essentially a search problem, as model is defined as a tree
