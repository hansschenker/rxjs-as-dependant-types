# RxJS as Dependant Types

A behavioral model of RxJS operators using dependent-type ideas.

> **Core idea:** Cω notation describes the shape of an RxJS dataflow; dependent types describe which state/event/action combinations are legal; Mealy machines describe the transition semantics; BDD provides the human-readable story.

## Why this repository exists

An ordinary RxJS operator type tells us what kinds of values flow through the operator:

```text
T* -> R*
```

That is useful, but it does not describe the operator's internal behavioral contract:

- which events are legal in a given state,
- which next states may follow,
- which actions are permitted,
- which invariants must always hold.

Dependent types give us a vocabulary for expressing those relationships.

## The central formulation

A conventional Mealy-style operator model can be written as:

```text
S × E -> S × A
```

For RxJS this is often too permissive because not every event is meaningful in every state.

A more precise dependent formulation is:

```text
step :
  (state : S) ->
  (event : E state) ->
  Σ (nextState : S).
    Actions state event nextState
```

Read it as:

> Given the current state, accept only an event legal for that state, then produce a next state together with actions that are legal for that state/event/next-state combination.

The dependent pair `Σ` carries both:

```text
(nextState, actions)
```

while allowing the type of `actions` to depend on `nextState`.

## Four complementary views

```text
Cω        = What can flow?
Dependent = What is legal?
Mealy     = What happens?
BDD       = How do we explain it?
```

These views are complementary rather than competing.

### Cω-style stream notation

```text
T!   exactly one
T?   zero or one
T+   one or more
T*   zero or more

A × B   both
A + B   either
A -> B  transformation
(T*)*   stream of streams
```

Example:

```text
scan:
(A × T -> A) -> T* -> A*
```

This describes dataflow shape and cardinality.

### Dependent behavioral type

```text
step :
  (s : S) ->
  (e : E s) ->
  Σ (s' : S). A s e s'
```

This describes legal behavior.

### Mealy machine

```text
T : S × E -> S
G : S × E -> A
```

or, when refined with dependencies:

```text
step :
  (s : S) ->
  (e : E s) ->
  Σ (s' : S). A s e s'
```

### BDD

```text
Given <state/precondition>
When  <event>
Then  <actions>
```

BDD exposes the same behavior in a user-facing form.

## Example: scan

Ordinary type:

```text
(A × T -> A) -> T* -> A*
```

Behavioral step:

```text
step :
  (state : A) ->
  (event : Next T) ->
  Result(
    nextState = reducer(state, event.value),
    action    = Emit(nextState)
  )
```

The accumulator is the operator state. A source value triggers the reducer, the reducer result becomes the next state, and that state is emitted.

## Example: switchMap

Possible states:

```text
Idle
Active(inner)
OuterCompleted(inner)
Done
```

Events can depend on the state:

```text
Event Idle =
    OuterNext T
  + OuterComplete
  + OuterError Error

Event Active =
    OuterNext T
  + OuterComplete
  + OuterError Error
  + InnerNext R
  + InnerComplete
  + InnerError Error
```

`InnerNext` is therefore not even a legal event while the machine is `Idle`.

The transition type is:

```text
transition :
  (s : SwitchState) ->
  Event s ->
  Σ (s' : SwitchState). SwitchActions s event s'
```

A new outer value while an inner is active produces the policy:

```text
Active(oldInner) + OuterNext(x)
  ->
Active(newInner)
  × [
      Unsubscribe(oldInner),
      Subscribe(newInner)
    ]
```

This is the type-level form of:

```text
switchMap = only latest
```

## Example: debounceTime

State:

```text
DebounceState T =
    Idle
  + Waiting T Timer
```

Legal events differ by state:

```text
Event Idle =
    SourceNext T
  + SourceComplete
  + SourceError Error

Event (Waiting value timer) =
    SourceNext T
  + TimerElapsed timer
  + SourceComplete
  + SourceError Error
```

The key behavioral refinement is that:

```text
TimerElapsed timer
```

belongs to the active timer. A stale timer is not a valid trigger for the current state.

This lets the specification encode an important RxJS invariant rather than merely describe it in prose.

## Operator invariants as types

Dependent types can make behavioral invariants explicit.

For `switchMap`:

```text
activeInners <= 1
```

Conceptually:

```text
step :
  (s : SwitchState) ->
  (e : SwitchEvent s) ->
  { s' : SwitchState | activeInners(s') <= 1 }
```

The result cannot represent a state with two active inners.

The same idea exposes the four flattening policies:

```text
mergeMap   = allow overlap
concatMap  = queue
switchMap  = only latest
exhaustMap = ignore while busy
```

with invariants such as:

```text
mergeMap(concurrency = n): activeInners <= n
concatMap:                activeInners <= 1, pendingInners >= 0
switchMap:                activeInners <= 1
exhaustMap:               Active + OuterNext -> same state + NoAction
```

## Refined operator definition

A useful evolution of:

```text
Operator = State Transition + Action Policy
```

is:

```text
Operator Behavior
=
Dependent State Transition
+
Dependent Action Policy
```

Or as one combined type:

```text
Step :
  (s : S) ->
  (e : E s) ->
  Σ (s' : S). A s e s'
```

This keeps the state transition and the action policy synchronized by construction.

## Why this matters for RxJS

RxJS operators are not merely value transformations. Many operators have state-dependent trigger sets and policies involving:

- source values,
- notifier values,
- timer events,
- inner notifications,
- completion,
- errors,
- cancellation,
- subscription lifecycle.

Dependent types provide a precise way to say:

> an event is legal only in the states where the operator can actually respond to it.

That makes illegal state/event combinations unrepresentable in the formal model.

## TypeScript

TypeScript can approximate this style with:

- discriminated unions,
- generics,
- conditional types,
- branded or phantom types,
- typestate patterns.

But TypeScript is not a fully dependently typed language. Languages and proof assistants such as Idris, Agda, Lean, and Coq can express these dependencies more directly.

The goal of this repository is therefore not necessarily to reimplement RxJS in a dependently typed language. The immediate value is to use dependent-type notation as a **semantic specification language for RxJS operator behavior**.

## Relationship to the wider RxJS formalization work

```text
                 RxJS Operator
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
      Cω Type     Dependent Type    Mealy
        |              |              |
   data shape     legal behavior    execution
        |              |              |
        +--------------+--------------+
                       |
                       v
                      BDD
                       |
                       v
               human explanation
```

The combination gives us a compact progression:

```text
Cω notation
    ↓
Dataflow shape

Dependent types
    ↓
Behavioral constraints and invariants

Mealy / ROB-SN
    ↓
Operational semantics

BDD
    ↓
Readable operator story
```

## Central takeaway

The most compact formulation developed here is:

```text
RxJS Operator Behavior

step :
  (state : S) ->
  (event : E state) ->
  Σ (nextState : S).
    Actions state event nextState
```

It captures the fundamental RxJS question:

> Given what the operator currently remembers and what just happened, what states and actions are legally possible next?

---

## Contributors

- Hans Schenker — concept direction, RxJS operator modeling, Cω/Mealy/BDD synthesis
- **OpenAI GPT-5.6 Sol** — main contributor to the dependent-type analysis and formalization

