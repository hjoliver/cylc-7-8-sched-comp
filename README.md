# Cylc 7 vs 8 Scheduling Comparison

This is a conceptual look at the Cylc 7 and 8 scheduling models,
how they impact manual interventions and what you see in the GUIs.

(Some details are omitted for clarity).

Terminology:
- **task pool:** the subset of all the tasks in the potentially infinite graph
that are currently held in memory to support the scheduling algorithm. In Cylc
8 this is also called the "active task pool" and the "n=0 window" (which refers
to the GUI window constructed by moving n graph edges out from the task pool. 

## Cylc 7 Scheduling Algorithm (simplified)

A Cylc 7 **task** knows its own prerequisites and its own outputs. It does
not know what other tasks depend on those outputs.

Cylc 7 initializes its task pool with a *waiting* instance of every task
at the first cycle point. Tasks with no prerequisites can submit immediately.

Then, when a task completes an output (e.g. `a.1:succeeded`), the scheduler
matches all completed outputs with all unsatisfied prerequisites across the
task pool, to see if other tasks become ready.

Additionally, when a task (e.g. `a.1`) submits, the scheduler spawns its
next-cycle instance (e.g. `a.2`) as *waiting*.

*Succeeded* tasks can only be forgotten when there are no more *waiting*
tasks left that could potentially be satisfied by them (roughly, no waiting
tasks remaining in the same cycle point, modified by intercycle dependence).
This makes the task pool quite bloated (roughly, one waiting instance of
every task, and all succeeded instances across the range of active cycles
must be held in memory).

*Cylc 7 evolves the workflow forward by spawning the next-cycle instance
of each task at submit time, not by following the graph! The pre-spawned
tasks do then run according to the graph, however.*

Comments:
 - the historical justification for this algorithm, briefly, is that Cylc was
    originally a self-organising scheduler in which the dependency graph
    emerged at run time rather then being specified up front
 - this works amazingly well, but there are some problems resulting from:
   - spawning of tasks long before they are needed, and a large task pool
   - `O(n^2)` (in number of tasks) prerequisite-to-output matching
   - implicit dependence on previous-instance submit, for every task
   - the spawn-on-submit model does not perpetuate the flow naturally if you
     need to manually trigger a sub-graph beyond the main task pool

### Cylc 7 Manual Interventions

In Cylc 7 you can only match and operate on tasks that remain in the task pool.
However, in clock-limited real-time workflows the bloated task pool makes it
likely that "all the tasks" remain in the pool for the current cycle, which makes
mass interventions easier than one might expect for this model.

Beyond the task pool you must *insert* tasks back into the pool:
- to rerun a sub-graph, insert all sub-graph tasks before triggering the first
  (because Cylc 7 doesn't automatically spawn tasks according to the graph)
- and the inserted tasks will spawn their next-cycle instances on submit
  (as per Cylc 7 normal) which will likely result in "stuck" waiting tasks
  that need to be removed in the next cycle

### What You See in the Cylc 7 GUI

The Cylc 7 GUI shows (in all views) only the current task pool, end of story.

However, the task pool is bloated and it is likely that "all the tasks" remain
in the current cycle of real-time clock-triggered workflows.

--------------

## Cylc 8 Scheduling Algorithm (simplified)

A Cylc 8 **task** knows its own prerequisites, its own outputs, **and** the
downstream tasks (children) that depend on those outputs.

The dependency graph is a network of `parent => child` relationships. Cylc 8
initializes its task pool with a waiting instance of every parentless task,
out to the runahead limit. These can all submit immediately (because, by
definition, they have no prerequisites).

Then whenever a task completes an output the scheduler spawns the downstream
children of that output "on demand".

Tasks can be forgotten immediately once they reach a final status (succeeded,
failed, expired) so long as their required outputs are complete. 

*Cylc 8 both evolves and runs the workflow according to the graph.*

Comments:
- this is conceptually much simpler than Cylc 7
- it solves all of the problems that afflict the Cylc 7 scheduling algorithm
- (it also supports a powerful new capability: multiple concurrent runs through
  the graph - "flows" - starting from any task(s) in the graph)  

### Cylc 8 Manual Interventions

Unlike Cylc 7, in Cylc 8 you can operate on individual tasks anywhere in
the infinite graph.

Downstream activity flows naturally with no setup (task insert and reset)
required, because downstream tasks spawn "on demand" as the graph dictates.

Just like Cylc 7, matching tasks by glob or family name (currently) only
works in the task pool. However, selecting tasks was often easier in Cylc 7
because the bloated nature of the pool was such that tasks you wanted to
select were generally (if not always) present.

With Cylc 8 we don't have a bloated pool to run globs over, so selecting 
(e.g.) families of tasks is tricky as we would have to work out what tasks
"could" enter the pool rather than what tasks "are currently" in the pool.

*We will address out-of-pool task matching by family and glob in upcoming
 8.x releases.*

### What You See in the Cylc 8 GUI

The Cylc 8 GUI shows (in all views) a configurable graph-based window around
the task pool. The default `n=1` window shows all tasks out to 1 graph edge
from the (`n=0`) task pool.

How much of the workflow you see around the active tasks is just a choice.

-----------

## Cylc 7 vs Cylc 8 Scheduling - Illustrated Example

The following images show, side-by-side, the Cylc 7 and 8 task pools
and indicates how they evolve by spawning future tasks into the pool.

### Time Zero (start up)

![time 0](img/c78-comp-t0.png)

Cylc 7 starts with the first instance of every task.

Cylc 8 starts with just the parentless tasks at the top of each cycle.

If the graph had 1000 tasks downstream of `x` the Cylc 7 task pool would
initially number 1001, but Cylc 8 would still have just 3 tasks.

-------

### Time One (later)

![time 1](img/c78-comp-t1.png)

The Cylc 7 task pool expands across active cycle points. It may contain up
to N times the number of tasks per cycle, where N is the number of cycles. 

The Cylc 8 task pool is typically far smaller and is not affected by spread
over active cycle points.

-------

### Time Two (later again)

![time 2](img/c78-comp-t2.png)

The Cylc 7 task pool continues to grow as the active cycle points fill out.
The scheduler can't clear out cycle 1 until no waiting tasks remain there.

Note the content of the Cylc 7 task pool can't be explained without
understanding the scheduling algorithm.

The Cylc 8 pool is easily explained: there are 3 active tasks (obvious) and
one waiting (spawned by `1/b:succeeded`, still waiting on `1/c:succeeded`). 

-------------

## Cylc 7 vs Cylc 8 Manual Intervention - Illustrated Example

### Cylc 7

Cylc 7 tasks:
 - know only their own prerequisites and outputs
 - prerequisites
   - can only be satisfied naturally by upstream task outputs
 - outputs can be completed:
   - naturally as task jobs run
   - or manually but indirectly by resetting task state (e.g. resetting a task
     to the "succeeded" state also sets its succeeded output)

In effect, Cylc 7 tasks wait on their parent outputs, rather than on their own
prerequisites (because their prerequisites can only be satisfied by upstream
outputs). So to rerun a past task in Cylc 7 (without *triggering* it) you need to:
 1. *insert* the task back into the pool in the *waiting* (unsatisfied) state
 2. *insert* its upstream parents back into the pool in the *waiting* state
 3. *reset* the upstream parents to *succeeded*, so the scheduling algorithm can
    use their outputs to satisfy the child's prerequisites 
 4. (the flow will only continue from there so far as you have reset all downstream
     tasks to *waiting*, and it will likely spawn stuck off-flow tasks that need
    removal to avoid a stall - via the next-instance spawn-on-submit mechanism)

### Cylc 8

Cylc 8 tasks:
 - know their own prerequisites, outputs, and the downstream tasks (children)
   that depend on those outputs
 - prerequisites can be satisified individually:
   - on demand, when upstream tasks complete outputs
   - manually (`cylc set --pre`)
   - this contributes to a tasks readiness to run
 - outputs can be satisfied individually:
   - naturally, when a task completes an output
   - manually (`cylc set --out`)
   - this contributes to completion of the task's outputs
   - and spawns downstream children (with corresponding prerequisites satisfied)

In effect, Cylc 8 tasks wait only on their own prerequisites, rather than on
the corresponding upstream outputs (because you can manually set individual
prerequisites without touching upstream tasks).
So to rerun a past task in Cylc 8 (without just *triggering* it) you can:
 - set its prerequisites (`cylc set --pre`)
 OR
- set the relevant outputs of its parents 

Then the flow will continue downstream naturally (so long as you set
`--flow=new` to allow retraversal of graph sections that already ran)
