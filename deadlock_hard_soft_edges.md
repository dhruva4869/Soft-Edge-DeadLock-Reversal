# Deadlock Detection: Hard vs Soft Edges

## 1. The basic problem

A deadlock happens when processes are waiting for each other in a cycle.

Example:

```text
A waits for B
B waits for C
C waits for A

A → B → C → A
```

`FindLockCycle(startProc)` starts from one process and follows the
wait-for relationships. If it eventually comes back to the original
starting process, a deadlock has been found.

The important part is that not every wait relationship is equally
"fixed". Some waits are caused by a process **actually holding a
conflicting lock**. Others are caused only because a process is
**ahead in a wait queue**.

That is the difference between a **hard edge** and a **soft edge**.

---

## 2. Hard edge

A hard edge means:

> "A is waiting for B because B already holds a lock that conflicts
> with A's requested lock."

Example:

```text
A wants:       EXCLUSIVE
B holds:       SHARE

A → B
```

A cannot simply move ahead of B in the wait queue.

B must release its lock first.

Therefore:

```text
A → B   = HARD
```

A hard edge represents a real lock dependency.

### Example deadlock

```text
A holds lock X
B holds lock Y

A wants Y  → waits for B
B wants X  → waits for A

A → B → A
```

These dependencies cannot be fixed by simply reordering a wait queue,
because the blockers are already holding conflicting locks.

---

## 3. Soft edge

A soft edge is different.

It means:

> "A is waiting for B because B is ahead of A in the wait queue."

Example:

```text
Wait queue:

B → C → A
          ^
          |
        A is waiting
```

Suppose B's requested lock conflicts with A's requested lock.

Then A has an edge:

```text
A → B
```

But B may not actually hold the lock. B is just **ahead in the queue**.

Therefore this dependency can potentially be removed by changing the
queue order:

```text
Before:

B → C → A

After:

A → B → C
```

So:

```text
A → B   = SOFT
```

Soft edges are important because they give the deadlock detector an
alternative to aborting a transaction.

---

## 4. Why distinguish hard and soft edges?

Consider:

```text
A → B → C → A
```

Suppose:

```text
A → B   = HARD
B → C   = SOFT
C → A   = SOFT
```

The deadlock detector can try to break the cycle by reversing a soft
edge.

For example, reverse:

```text
B → C
```

by moving B before C (or equivalently changing the relevant queue
ordering).

The goal is to find a new queue arrangement where the cycle disappears.

Hard edges cannot be freely reversed because they represent actual
lock ownership.

Soft edges can be reversed because they represent queue ordering.

---

# 5. What `FindLockCycle()` does

The simplified algorithm is:

```text
FindLockCycle(start):

    visited = []
    softEdges = []

    if DFS(start):
        return DEADLOCK, softEdges
    else:
        return NO_DEADLOCK
```

The DFS is approximately:

```text
DFS(proc):

    if proc was already visited:
        if proc == start:
            return TRUE       # came back to start
        else:
            return FALSE      # unrelated cycle

    visited.add(proc)

    # First check real lock holders
    for each conflicting lock holder:
        if DFS(holder):
            return TRUE       # HARD edge

    # Then check processes ahead in the queue
    for each conflicting waiter before proc:
        if DFS(waiter):
            softEdges.add(proc → waiter)
            return TRUE       # SOFT edge

    return FALSE
```

---

## 6. Why check hard edges first?

The algorithm first follows processes that **already hold conflicting
locks**.

Example:

```text
A waits for B

B actually holds the conflicting lock
```

So:

```text
A → B
```

is a hard dependency.

If following that dependency eventually reaches A:

```text
A → B → C → A
```

then there is a genuine deadlock.

The edge is not added to `softEdges` because we cannot solve it merely
by rearranging the queue.

---

## 7. How a soft edge is discovered

Suppose:

```text
Queue:

B → C → A
```

A is waiting for the lock.

The detector looks at the waiters **strictly ahead of A**:

```text
B
C
```

If B's requested lock conflicts with A's requested lock, the detector
can follow:

```text
A → B
```

If following B eventually gets back to A:

```text
A → B → C → A
```

then this edge was part of the deadlock cycle.

Because B was only ahead of A in the queue, the edge is classified as
soft:

```text
softEdges.add(A → B)
```

---

# 8. The actual solution: rearrange queues

Finding a soft edge gives the deadlock detector an escape route.

Suppose:

```text
A → B → C → A
```

and:

```text
A → B = SOFT
```

Instead of immediately aborting A, try reversing the soft dependency.

Conceptually:

```text
Before:

B
C
A

After:

A
B
C
```

The exact ordering is determined using a topological sort.

The topological sort answers:

> "Given all the queue-order constraints we have accumulated,
> is there a valid ordering of this queue?"

---

## 9. Topological sort vs `FindLockCycle()`

These two checks answer different questions.

### Topological sort

Checks:

> "Can I construct a queue ordering that satisfies all the proposed
> rearrangement constraints?"

Example:

```text
Constraints:

C before A
```

Possible:

```text
C → A → B
```

So topo sort succeeds.

### `FindLockCycle()`

Checks:

> "After considering this proposed rearrangement, does the actual
> wait-for graph still contain a deadlock cycle?"

So:

```text
Topo sort
    ↓
Is the proposed queue arrangement possible?
    ↓ YES
FindLockCycle()
    ↓
Does the resulting configuration still have a deadlock?
```

Both checks are necessary.

---

# 10. Why check `FindLockCycle(A)` and `FindLockCycle(B)` again?

Suppose a proposed rearrangement moves A relative to B.

The topological sort only tells us that the queue ordering is valid.

It does not prove that the entire wait-for graph is now cycle-free.

The rearrangement can expose or create another cycle involving the
processes that were moved.

Therefore the algorithm checks:

```text
FindLockCycle(original start)
FindLockCycle(A)
FindLockCycle(B)
```

If no cycle is found, the proposed rearrangement is valid.

If another cycle is found, the algorithm takes a soft edge from that
cycle and tries another reversal.

---

# 11. Recursive search

The overall algorithm is roughly:

```text
Find cycle
    ↓
Pick a soft edge
    ↓
Try reversing it
    ↓
Topological sort
    ↓
Can the queue be arranged?
    |
    +-- NO → reject this reversal
    |
    +-- YES
          ↓
      FindLockCycle()
          |
          +-- NO cycle → SUCCESS
          |
          +-- Cycle remains
                 ↓
             Pick another soft edge
                 ↓
             Try another reversal
```

This is recursive because a first rearrangement may not be enough.

Example:

```text
Initial cycle:

A → B → C → A
```

Reverse one soft edge.

Now there is another cycle:

```text
D → E → F → D
```

The algorithm can take a soft edge from this new cycle and add another
rearrangement constraint.

It keeps doing this until:

1. All cycles disappear → use the rearrangement, or
2. Every possible rearrangement fails → abort the transaction.

---

# 12. Why try many rearrangements?

There can be multiple possible ways to break a deadlock.

The algorithm does not care which valid arrangement it finds first.

For example:

```text
Current queue:

A → B → C
```

Possible constraints might eventually require:

```text
C before A
B before C
```

A valid final ordering is:

```text
C → A → B
```

The algorithm may first try a different ordering, discover that it
still contains a cycle, and then recursively try the additional
constraint.

The important guarantee is:

> It should eventually try every relevant queue rearrangement that
> could lead to a deadlock-free configuration.

If none works, the fallback is:

```text
abort original transaction
```

---

# 13. Easy mental model

Think of a restaurant queue.

```text
B → C → A
```

A is waiting because B and C are ahead.

If B is already **eating** and holding the table:

```text
A → B = HARD
```

You cannot simply move A ahead of B.

But if B is merely **standing in the queue**:

```text
A → B = SOFT
```

you can potentially say:

```text
A, you go ahead of B.
```

That queue rearrangement can break a deadlock without killing a
transaction.

---

# 14. Final mental model

```text
HARD EDGE
---------
A → B

B actually holds the conflicting lock.

Cannot fix by queue rearrangement.


SOFT EDGE
---------
A → B

B is only ahead of A in the wait queue.

Can potentially fix by moving A ahead of B.


DEADLOCK SOLUTION
-----------------
Find cycle
    ↓
Find soft edges in cycle
    ↓
Reverse/rearrange soft edges
    ↓
Topo sort = "is this queue arrangement possible?"
    ↓
FindLockCycle = "is the resulting wait graph deadlock-free?"
    ↓
Success → reorder queues + wake waiters
Failure → try another rearrangement
    ↓
No rearrangement works
    ↓
Abort transaction
```

The core idea is:

> **Hard edges represent dependencies we cannot rearrange. Soft edges
> represent queue-order dependencies we can potentially rearrange.
> The deadlock detector tries to use those soft edges to resolve the
> deadlock without aborting a transaction.**
