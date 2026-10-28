# Formal Model

## 1. Ordinary Mealy model

A conventional Mealy machine can be summarized as:

```text
S × E -> S × A
```

where:

- `S` is the state space,
- `E` is the input/event alphabet,
- `A` is the action/output space.

For many RxJS operators this admits combinations that are semantically impossible. For example, an inner-stream notification cannot occur when no inner subscription exists.

## 2. State-indexed event family

Refine the event alphabet into a family indexed by state:

```text
E : S -> Type
```

Then a transition receives only events legal for its current state:

```text
(state : S) ->
(event : E state) ->
...
```

This turns state-dependent trigger legality into part of the type.

## 3. State/event-indexed action family

Actions may depend on both the source state and incoming event:

```text
A : (state : S) -> E state -> Type
```

A still stronger formulation may let the action type depend on the resulting state:

```text
A :
  (state : S) ->
  (event : E state) ->
  (nextState : S) ->
  Type
```

## 4. Dependent result

The operator step can then return a dependent pair:

```text
step :
  (state : S) ->
  (event : E state) ->
  Σ (nextState : S).
    A state event nextState
```

This is the central behavioral type used in this repository.

## 5. Interpretation in RxJS terms

```text
state      = what the operator currently remembers
event      = what just happened
nextState  = what the operator remembers next
actions    = what the operator does because of that event
```

Examples of events:

```text
SourceNext(value)
SourceComplete
SourceError(error)

TimerElapsed(timer)

NotifierNext(value)
NotifierComplete
NotifierError(error)

InnerNext(inner, value)
InnerComplete(inner)
InnerError(inner, error)

Subscribe(subscriber)
Unsubscribe(subscriber)
```

Examples of actions:

```text
Emit(value)
Complete
Error(error)

SubscribeInner(inner)
UnsubscribeInner(inner)

StartTimer(timer)
CancelTimer(timer)

Store(value)
Discard(value)
NoAction
```

## 6. Relation to ROB-SN

ROBSN-style notation:

```text
f(S, E) -> (S', A)
```

can be refined to:

```text
f :
  (s : S) ->
  (e : E s) ->
  Σ (s' : S). A s e s'
```

The dependent version says not only what transition occurs, but which state/event/action combinations are admissible.

## 7. Relation to BDD

A BDD scenario:

```text
Given an inner stream is active
When a new outer value arrives
Then unsubscribe from the current inner
     project the new value
     subscribe to the new inner
```

corresponds structurally to:

```text
state      = Active(oldInner)
event      = OuterNext(value)
nextState  = Active(newInner)
actions    = [
  UnsubscribeInner(oldInner),
  SubscribeInner(newInner)
]
```

and therefore to a dependent step inhabitant:

```text
step
  (Active oldInner)
  (OuterNext value)
:
Σ (nextState : SwitchState).
  SwitchActions
    (Active oldInner)
    (OuterNext value)
    nextState
```

## 8. Core principle

An operator specification becomes stronger when it can state:

```text
Illegal behavior has no inhabitant.
```

Instead of documenting that a stale timer must not emit, or that `switchMap` must not retain two active inners, the type-level model can exclude those outcomes from the legal result space.
