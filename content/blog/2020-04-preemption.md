+++
date = "2020-03-31"
title = "Reducing tail latencies with automatic cooperative task yielding"
description = "April 1, 2020"
menu = "blog"
weight = 982
+++

Tokio is a runtime for asynchronous Rust applications. It allows writing code
using `async` & `await` syntax. For example:

```rust
let mut listener = TcpListener::bind(&addr).await?;

loop {
    let (mut socket, _) = listener.accept().await?;

    tokio::spawn(async move {
        // handle socket
    });
}
```

The Rust compiler transforms this code into a state machine. The Tokio runtime
executes these state machines, multiplexing many tasks on a handful of threads.
Tokio's scheduler requires that the generated task's state machine yields control
back to the scheduler in order to multiplex tasks. Each `.await` call is an
opportunity to yield back to the scheduler. In the above example,
`listener.accept().await` will return a socket if one is pending. If there are
no pending sockets, control is yielded back to the scheduler.

This system works well in most cases. However, when a system comes under load,
it is possible for an asynchronous resource to never be "not ready". For
example, consider an echo server:

```rust
tokio::spawn(async move {
    let mut buf = [0; 1024];

    loop {
        let n = socket.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        // Write the data back
        socket.write(buf[..n]).await?;
    }
});
```

If data is received faster than it can be processed, it is possible that, by the
time the processing of a data chunk completes, more data has already been
received. In this case, `.await` will never yield control back to the scheduler,
other tasks will not be scheduled, resulting in starvation and large latency
variance.

Currently, the answer to this problem is that the user of Tokio is responsible
for adding yield points every so often. In practice, very few actually do this
and end up being vulnerable to this sort of problem.

A common solution to this problem is preemption. OS threads will interrupt
thread execution every so often in order to ensure fair scheduling of all
threads. Runtimes that have full control over execution (golang, erlang, ...)
will also use preemption to ensure fair scheduling of tasks. This is
accomplished by injecting yield points when compiling code that check if the
task has been executing for long enough and yielding back to the scheduler.
Unfortunately, Tokio is not able to use this technique as `async` Rust does not
inject any sort of yield point.

## Per-task operation budget

Even though Tokio is not able to **preempt**, there is still an opportunity to
nudge a task to yield back to the scheduler. As of [0.2.14], each Tokio task has
an operation budget. This budget is reset when the scheduler switches to the
task. Each Tokio resource (socket, timer, channel, ...) is now aware of this
budget. As long as the task as budget remaining, the resource operates as it did
previously. Each asynchronous operation (actions that users must `.await` on)
decrement the task's budget. Once the task is out of budget, all resources will
become "not ready" until the task yields back to the scheduler, at which point,
the budget is reset.

Going back to the echo server example from above. When the task is scheduled, it
is assigned a budget of 128 operations. When `socket.read(..)` and
`socket.write(..)` are called, the budget is decremented. If the budget is zero,
the task yields back to the scheduler. If either `read` or `write` cannot
proceed due to the underlying socket not being ready (no pending data or a full
send buffer), then the task also yield back to the scheduler.

The idea originated from a conversation I was having with [Ryan Dahl][ry]. He is
using Tokio as the underlying runtime for [Deno][deno]. When doing some HTTP
experimentation (a while back) with [Hyper], he was seeing some high tail
latencies in some benchmarks. The problem was due to a loop not yielding back to
the scheduler under load. Hyper ended up fixing the problem by hand in this one
case, but Ryan mentioned that, when he worked on [node.js][node], they handled
the problem by adding **per resource** limits. So, if a TCP socket was always
ready, it would force a yield every so often. I mentioned this conversation to
[Jon Gjenset][jonhoo], and he ended up coming up with the idea of placing the
limit on the task itself instead of on each resource.

The end result is that Tokio should be able to provide more consistent runtime
behavior under load. While the exact heuristics will most likely be tweaked over
time, initial measurements show that, in some cases, tail latencies are reduced
almost 3x.

[![benchmark](https://user-images.githubusercontent.com/176295/73222456-4a103300-4131-11ea-9131-4e437ecb9a04.png)](https://user-images.githubusercontent.com/176295/73222456-4a103300-4131-11ea-9131-4e437ecb9a04.png)

"master" is before the automatic yielding and "preempt" is after. Click for a
bigger version, also see the original [PR comment][pr] for more details.

## A note on blocking

While automatic cooperative task yielding improves many cases, it cannot preempt
tasks. Users of Tokio must still take care to avoid CPU intensive work and using
blocking APIs. The [`spawn_blocking`][spawn_blocking] function can be used to
"asyncify" these sorts of tasks by running them on a thread pool.

Tokio does not, and will not attempt to detect blocking tasks and automatically
compensate by adding threads to the scheduler. This question has come up a
number of times in the past so allow me to elaborate.

For context, the idea is for the scheduler to include a monitoring thread. This
thread would poll scheduler threads every so often and check that workers are
making progress. If a worker is not making progress, it is assumed that the
worker is executing a blocking task and a new thread should be spawned to
compensate.

This idea is not new. The first occurence of this strategy that I am aware of is
in the .NET thread pool and was introduced more than ten years ago.
Unfortunately, the strategy has a number of problems and because of this, has
not been featured in other thread pools / schedulers (golang, java, erlang,
...).

The first problem is that it is very hard to define "progress". A naive
definition of progress is whether or not a task has been scheduled for over some
unit of time. For example, if a worker has been stuck scheduling the same task
for more than 100ms, then that worker is flagged as blocked and a new thread is
spawned. In this definition, how does one detect scenarios where spawning a new
thread **reduces** throughput? This can happen when the scheduler is generally
under load and adding threads would make the situation much worse. To combat
this, the .NET thread pool uses [hill climbing][hill].

The second problem is that any automatic detection strategy will be vulnerable
to bursty or otherwise uneven workloads. This specific problem has been the bane
of the .NET thread pool and is known as the "stuttering" problem. The hill
climbing strategy requires some period of time (hundreds of milliseconds) to
adapt to load changes. This time period is needed, in part, to be able to
determine that adding threads is improving the situation and not making it
worse.

The stuttering problem can be managed with the .NET thread pool, in part,
because the pool is designed to schedule **coarse** tasks, i.e. tasks that
execute in the order of hundreds of milliseconds to multiple seconds. However,
asynchronous task schedulers are designed to schedule tasks that should run in
the order of micro seconds to tens of milliseconds at most. In this case, any
stutttering problem from a heuristic based scheduler will result in far greater
latency variations.

The most common follow up question I've received after this is "doesn't the Go
scheduler automatically detect blocked tasks?". The short answer is: no. Doing
so would result in the same stuttering problems as mentioned above. Also, Go has
no need to have generalized blocked task detection because Go is able to
preempt. What the Go scheduler **does** do is annotate potentially blocking
system calls. This is roughly equivalent to the Tokio APIs
[`spawn_blocking`][spawn_blocking] and [`block_in_place`][block_in_place].

In short, as of now, the automatic cooperative task yielding strategy that has
just been introduced is the best we have found for reducing tail latencies.

<div style="text-align:right">&mdash;Carl Lerche</div>


[0.2.14]: #
[ry]: https://github.com/ry
[deno]: https://github.com/denoland/deno
[Hyper]: github.com/hyperium/hyper/
[node]: https://nodejs.org
[jonhoo]: https://github.com/jonhoo/
[pr]: https://github.com/tokio-rs/tokio/pull/2160#issuecomment-579004856
[spawn_blocking]: https://docs.rs/tokio/0.2/tokio/task/fn.spawn_blocking.html
[block_in_place]: https://docs.rs/tokio/0.2/tokio/task/fn.block_in_place.html
[hill]: https://en.wikipedia.org/wiki/Hill_climbing