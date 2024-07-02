# Cylc 7 vs 8 Scheduling Comparison

Note: I've glossed over many details here in the interests of clarity.

## Cylc 7 Scheduling Algorithm (simplified)

Cylc 7 initializes its *task pool* with a *waiting* instance of every task
at the first cycle point. Those with no prerequisites can submit immediately.

When a task (e.g. `a.1`) submits, the scheduler spawns its next-cycle
instance (`a.2`) as *waiting*.

When a task completes an output, the scheduler matches completed outputs
with unsatisfied prerequisites across the task pool to see if other
tasks become ready.

*Succeeded* tasks can only be forgotten when there are no more *waiting*
tasks left that could potentially be satisfied by them (roughly, no waiting
tasks remaining in the same cycle point, modified by intercycle dependence).

*So Cylc 7 evolves the workflow forward by spawning the next-cycle instance
of each task at submit time, not by following the graph! The pre-spawned
tasks do then run according to the graph, however.*

Comments:
 - the historical justification for this algorithm is documented elsewhere
 - it works amazingly well, but has some notable problems resulting from:
   - spawning of tasks long before they are needed, and a large task pool
   - `O(n^2)` (in number of tasks) prerequisite-output matching
   - implicit dependence on previous-instance submit

## Cylc 8 Scheduling Algorithm (simplified)

The dependency graph is a network of `parent => child` relationships. Cylc 8
initializes its task pool with a waiting instance of every parentless task,
out to the runahead limit. These can all submit immediately (no prerequisites).

Whenever a task completes an output the scheduler spawns the associated
downstream child task "on demand".

*Cylc 8 evolves and runs the graph purely according to the dependencies.*

Comments:
- this is conceptually much simpler than Cylc 7
- it solves all of the problems that the Cylc 7 algorithm suffers from
- the much smaller task pool can make some interventions more difficult
  (this will be resolved by upcoming 8.x releases)

-----------

## Example

### Time Zero (start up)

![time 0](img/c78-comp-t0.png)

Cylc 7 starts with the first instance of every task.

Cylc 8 starts with just the parentless tasks at the top of each cycle

If the graph had 1000 tasks downstream of `x` the Cylc 7 task pool would
initially number 1001, but Cylc 8 would still have just 3 tasks.

### Time One (later)

![time 1](img/c78-comp-t1.png)

The Cylc 7 task pool expands across active cycle points. It may contain up
to N times the number of tasks per cycle, where N is the number of cycles. 

The Cylc 8 task pool is typically far smaller and is not affected by spread
over active cycle points.

### Time Two (later again)

![time 2](img/c78-comp-t2.png)

The Cylc 7 task pool continues to grow as the active cycle points fill out.
The scheduler can't clear out cycle 1 until no waiting tasks remain there.

Note the content of the Cylc 7 task pool can't be explained without
understanding the scheduling algorithm.

The Cylc 8 pool is easily explained: there are 3 active tasks (obvious) and
one waiting (spawned by `1/b:succeeded`, still waiting on `1/c:succeeded`). 

-------------

## Interventions: Matching and Operating on Tasks

### Cylc 7

In Cylc 7 you can only match and operate on tasks that remain in the task pool.

However, in clock-limited real-time workflows the bloated Cylc 7 pool makes it
likely that "all the tasks" remain in the pool for the current cycle, which makes
many interventions easier than one might expect for this model.

Beyond the task pool you must *insert* tasks before running anything:
- to rerun a sub-graph, insert all sub-graph tasks before triggering the first
  (because Cylc 7 doesn't automatically spawn tasks according to the graph)
- and the inserted tasks will spawn their next-cycle instances on submit
  (as per Cylc 7 normal) which will likely result in "stuck" waiting tasks
  that need to be removed in the next cycle

### Cylc 8

In Cylc 8 you can operate on individual tasks anywhere in the infinite graph.

Downstream activity will flow naturally with no setup (reset or insert)
required because the Cylc 8 algorithm automatically spawns future tasks
exactly as the graph dictates.

*Matching tasks by glob or family name currently (Cylc 8.3.0) only works
in the task pool.* This is just like Cylc 7, but the much leaner Cylc 8 task
pool can make mass interventions more difficult than for
all-tasks-still-in-the-pool Cylc 7 cases,
e.g. to target all members of a family by family name.

This will become much easier in future 8.x releases, however.

-------------

## What You See in the GUI

### Cylc 7

The Cylc 7 GUI shows only the current task pool, end of story. However, the
task pool is bloated and it is likely that "all the tasks" remain in the
current cycle of real-time clock-triggered workflows.

### Cylc 8

The Cylc 8 GUI shows (in all views) a configurable graph-based window around
the much leaner task pool. The default `n=1` window shows all tasks out to 1
graph edge from the (`n=0`) task pool.

How much of the workflow you see in the Cylc 8 GUI is just a choice.
