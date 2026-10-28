# Flattening Operators as Dependent Behavioral Types

The four canonical higher-order mapping operators share the same broad dataflow shape but differ in their concurrency and cancellation policy.

```text
mergeMap   = allow overlap
concatMap  = queue
switchMap  = only latest
exhaustMap = ignore while busy
```

A useful common higher-order shape is:

```text
(T -> R*) -> T* -> R*
```

The Cω-style type does not reveal the policy difference. The dependent behavioral model does.

## Common state vocabulary

Conceptually:

```text
Idle

Active(
  activeInners,
  pendingOuterValues
)

OuterCompleted(
  activeInners,
  pendingOuterValues
)

Done
```

Each operator refines the valid shape of these collections.

---

## mergeMap

Policy:

```text
allow overlap
```

For bounded concurrency `n`:

```text
Invariant:
activeInners <= n
```

When a new outer value arrives:

```text
Given activeInners < n
When OuterNext(value)
Then project value
     subscribe to new inner
     add inner to activeInners
```

If the concurrency bound is reached, the implementation may queue pending work according to the operator's semantics.

Dependent result sketch:

```text
step :
  (s : MergeState n) ->
  (e : MergeEvent s) ->
  Σ (s' : MergeState n).
    MergeActions s e s'
```

where every inhabitant of `MergeState n` proves:

```text
activeInners <= n
```

---

## concatMap

Policy:

```text
queue
```

Invariant:

```text
activeInners <= 1
pendingOuterValues >= 0
```

When busy:

```text
Given one inner is active
When OuterNext(value)
Then enqueue value
     keep current inner active
```

When the inner completes:

```text
Given one inner is active
And queue is non-empty
When InnerComplete
Then dequeue oldest outer value
     project it
     subscribe to its inner
```

The type-level state can distinguish:

```text
Idle
Active(inner, queue)
```

with no representable state containing two active inners.

---

## switchMap

Policy:

```text
only latest
```

Invariant:

```text
activeInners <= 1
```

When a new outer value arrives while busy:

```text
Given an inner is active
When OuterNext(value)
Then unsubscribe current inner
     project value
     subscribe to new inner
```

Dependent state-indexed events ensure that `InnerNext` exists only when an inner is active.

A stronger identity-aware model can additionally require that an inner event carries the identity of the currently active inner, excluding stale inner emissions.

---

## exhaustMap

Policy:

```text
ignore while busy
```

Invariant:

```text
activeInners <= 1
```

Critical rule:

```text
Given an inner is active
When OuterNext(value)
Then keep same state
     NoAction
```

This can be stated directly:

```text
step
  (Active inner)
  (OuterNext value)
=
(
  Active inner,
  NoAction
)
```

No queue and no replacement action are legal for that transition.

---

## Same shape, different legal behaviors

All four operators can share the same broad stream type:

```text
(T -> R*) -> T* -> R*
```

but differ in the legal inhabitants of:

```text
(state : S) ->
(event : E state) ->
Σ (nextState : S).
  Actions state event nextState
```

That gives a precise separation:

```text
Cω type        = higher-order dataflow shape
Dependent type = concurrency/cancellation legality
Mealy          = transition semantics
BDD            = readable policy
```

This is especially useful for teaching because the value transformation can remain identical while the execution policy is exposed independently.
