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

*Succeeded* tasks can be dropped from the pool once there are no *waiting*
tasks left that they could potentially satisfy (roughly, no waiting tasks
remaining in the same cycle point).

*So Cylc 7 extends the workflow by spawning the next instance of each task
at submit time, not by following the graph! The pre-spawned tasks do then
run according to the dependencies, however.*

## Cylc 8 Scheduling Algorithm (simplified)

The dependency graph is a network of `parent => child` relationships. Cylc 8
initializes its task pool with a waiting instance of every parentless task,
out to the runahead limit. These can all submit immediately (no prerequisites).

Whenever a task completes an output the scheduler spawns the associated
downstream child "on demand" (so unused branches do not get spawned at all).

*Cylc 8 evolves and runs the graph purely according to the dependencies.*
This is conceptually much simpler than Cylc 7 algorithm and solves all of its
problems - see below.

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
Beyond that you must first *insert* tasks into the pool.

To rerun a sub-graph in Cylc 7 you have to "reset" existing (pool) sub-graph
members to waiting and insert any ex-pool members, to set up the re-run before
triggering the initial tasks.

However, in clock-limited workflows the bloated Cylc 7 pool makes it likely
that "all the tasks" remain in the pool for the current cycle, which makes
many interventions easier than one might expect for this model.

### Cylc 8

In Cylc 8 you can operate on individual tasks anywhere in the infinite graph,
and downstream consequences will flow naturally with no pre-setup (reset or
insert) required.

However, matching tasks by glob or family name currently (Cylc 8.3.0) only works
in the task pool. This is just like Cylc 7, but the much leaner pool (e.g. no
succeeded tasks in the Cylc 8 pool) can make some mass interventions more
involved than for Cylc 7, e.g. to target individual family members. This will
become much easier soon though, in future releases.

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
