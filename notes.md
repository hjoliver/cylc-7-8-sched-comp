## Simplifying assumptions in this example

- skipping over the *submitted* task state
- no queues etc. that put more waiting tasks in the pool
- all dependencies are of the form `a:succeeded => b`
- only one cycling sequence in the graph
- no external triggers (including clock triggers)
- the runahead limit is 3 (or more) cycle points
- runahead-limited task instances are omitted
- tasks complete all their outputs successfully
