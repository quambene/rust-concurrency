<!-- markdownlint-disable MD033 -->

# Rust Concurrency Cheat Sheet

- [Safety](#safety)
- [Overview](#overview)
- [Futures](#futures)
- [Tasks and threads](#tasks-and-threads)
- [Streams](#streams)
- [Share state](#share-state)
- [Marker traits](#marker-traits)
- [Concurrency models](#concurrency-models)
- [Terminology](#terminology)
- [References](#references)

## Safety

Rust ensures data race safety through the type system (`Send` and `Sync` marker traits) as well as the ownership and borrowing rules: it is not allowed to alias a mutable reference, so it is not possible to perform a data race.

## Overview

&nbsp; | Problem
------- | -------
Parallelism | Multi-core utilization
Concurrency | Single-core idleness

&nbsp; | Solution | Primitive | Type | Description | Examples
------- | ------- | ------- | ------- | ------- | -------
Parallelism | Multithreading | Thread | `T: Send` | Do work simultaneously on different threads | [`std::thread::spawn`](https://doc.rust-lang.org/std/thread/fn.spawn.html)
Concurrency | Single-threaded concurrency | Future | `Future` | Futures run concurrently on the same thread | [`futures::future::join`](https://docs.rs/futures/latest/futures/future/fn.join.html), [`futures::join`](https://docs.rs/futures/latest/futures/macro.join.html), [`tokio::join`](https://docs.rs/tokio/latest/tokio/macro.join.html)
Concurrency<br>+Parallelism | Multithreaded concurrency | Task | `T: Future + Send` | Tasks run concurrently to other tasks; the task may run on the current thread, or it may be sent to a different thread | [`async_std::task::spawn`](https://docs.rs/async-std/latest/async_std/task/fn.spawn.html), [`tokio::task::spawn`](https://docs.rs/tokio/latest/tokio/fn.spawn.html)

## Futures

``` rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

- `Future` has to be `poll`ed (by the *executor*) to resume where it last yielded and make progress (async is *lazy*)
- `&mut Self` contains state (state machine)
- `Pin` the memory location because the future contains self-referential data
- `Context` contains the `Waker` to notify the executor that progress can be made
- async/await on futures is implemented by *generators*
- `async fn` and `async` blocks return `impl Future<Output = T>`
- calling `.await` attempts to resolve the `Future`: if the `Future` is blocked, it yields control; if progress can be made, the `Future` resumes

Futures form a tree of futures. The leaf futures commmunicate with the executor. The root future of a tree is called a *task*.

## Tasks and threads

Computation | Examples
------- | -------
Lightweight (e.g. <100 ms) | [`async_std::task::spawn`](https://docs.rs/async-std/latest/async_std/task/fn.spawn.html), [`tokio::task::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html)
Expensive (e.g. >100 ms or I/O bound) | [`async_std::task::spawn_blocking`](https://docs.rs/async-std/latest/async_std/task/fn.spawn_blocking.html), [`tokio::task::spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html)
Massive (e.g. running forever or CPU-bound) | [`std::thread::spawn`](https://doc.rust-lang.org/std/thread/fn.spawn.html)

## Streams

``` rust
pub trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) { ... }
}
```

- `Stream<Item = T>` is an asynchronous version of `Iterator<Item = T>`, i.e., it does not block between each item yield

Operation | Relationship | Operations
------- | ------- | -------
Create | | [`futures::stream::iter`](https://docs.rs/futures/latest/futures/stream/fn.iter.html), [`futures::stream::once`](https://docs.rs/futures/latest/futures/stream/fn.once.html), [`futures::stream::repeat`](https://docs.rs/futures/latest/futures/stream/fn.repeat.html), [`futures::stream::repeat_with`](https://docs.rs/futures/latest/futures/stream/fn.repeat_with.html), [`async_stream::stream`](https://docs.rs/async-stream/latest/async_stream/macro.stream.html)
Create (via channels) | | [`futures::channel::mpsc::Receiver`](https://docs.rs/futures/latest/futures/channel/mpsc/struct.Receiver.html), [`tokio_stream::wrappers::ReceiverStream`](https://docs.rs/tokio-stream/latest/tokio_stream/wrappers/struct.ReceiverStream.html)
Looping | | [`futures::stream::StreamExt::next`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.next), [`futures::stream::StreamExt::for_each`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.for_each), [`futures::stream::StreamExt::for_each_concurrent`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.for_each_concurrent)
Transform | 1-1 | [`futures::stream::StreamExt::map`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.map), [`futures::stream::StreamExt::then`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.then), [`futures::stream::StreamExt::flatten`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.flatten)
Filter | 1-1 | [`futures::prelude::stream::StreamExt::filter`](https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.filter), [`futures::prelude::stream::StreamExt::take`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.take), [`futures::prelude::stream::StreamExt::skip`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.skip)
Buffering | 1-1 | [`futures::prelude::stream::StreamExt::buffered`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffered), [`futures::prelude::stream::StreamExt::buffer_unordered`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffer_unordered)
Combine | n-1 | [`futures::stream::StreamExt::chain`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.chain), [`futures::stream::StreamExt::zip`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.zip), [`tokio_stream::StreamExt::merge`](https://docs.rs/tokio-stream/0.1.9/tokio_stream/trait.StreamExt.html#method.merge), [`tokio_stream::StreamMap`](https://docs.rs/tokio-stream/latest/tokio_stream/struct.StreamMap.html), [`tokio::select`](https://docs.rs/tokio/latest/tokio/macro.select.html)
Split | 1-n | [`futures::channel::oneshot::Sender::send`](https://docs.rs/futures/latest/futures/channel/oneshot/struct.Sender.html#method.send), [`async_std::channel::Sender::send`](https://docs.rs/async-std/latest/async_std/channel/struct.Sender.html#method.send)

## Share state

&nbsp; | Threads | Tasks
------- | ------- | -------
channel | [`std::sync::mpsc`](https://doc.rust-lang.org/std/sync/mpsc/) (`Send`), [`crossbeam::channel`](https://docs.rs/crossbeam-channel/latest/crossbeam_channel/) (`Send`, `Sync`) | [`tokio::sync::mpsc`](https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html), [`tokio::sync::oneshot`](https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html), [`tokio::sync::broadcast`](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html), [`tokio::sync::watch`](https://docs.rs/tokio/latest/tokio/sync/watch/index.html), [`async_channel::unbounded`](https://docs.rs/async-channel/latest/async_channel/fn.unbounded.html), [`async_channel::bounded`](https://docs.rs/async-channel/latest/async_channel/fn.bounded.html)
mutex | [`std::sync::Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) | [`tokio::sync::Mutex`](https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html)

## Marker traits

- [`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html): safe to send it to another thread
- [`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html): safe to share between threads (`T` is `Sync` if and only if `&T` is `Send`)

Type | `Send` | `Sync` | Owners | Interior mutability
------- | ------- | ------- | ------- | -------
[`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html) | No | No | multiple | No
[`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) | Yes (if `T` is `Send` and `Sync`) | Yes (if `T` is `Send` and `Sync`) | multiple | No
[`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html) | Yes (if `T` is `Send`) | Yes (if `T` is `Sync`) | single | No
[`Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) | Yes (if `T` is `Send`) | Yes (if `T` is `Send`) | single | Yes
[`RwLock<T>`](https://doc.rust-lang.org/std/sync/struct.RwLock.html) | Yes (if `T` is `Send`) | Yes (if `T` is `Send` and `Sync`) | single | Yes
[`MutexGuard<'a, T: 'a>`](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html) | No | Yes (if `T` is `Sync`) | single | Yes
[`Cell<T>`](https://doc.rust-lang.org/std/cell/struct.Cell.html) | Yes (if `T` is `Send` | No | single | Yes
[`RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html) | Yes (if `T` is `Send`) | No | single | Yes

## Concurrency models

Model | Description
------- | -------
shared memory | threads operate on regions of shared memory
worker pools | many identical threads receive jobs from a shared job queue
actors | many different job queues, one for each actor; actors communicate exclusively by exchanging messages

Runtime | Description
------- | -------
[tokio](https://crates.io/crates/tokio) (multithreaded) | thread pool with work-stealing scheduler: each processor maintains its own run queue; idle processor checks sibling processor run queues, and attempts to steal tasks from them
[actix_rt](https://docs.rs/actix-rt/latest/actix_rt/) | single-threaded async runtime; futures are `!Send`
[actix](https://crates.io/crates/actix) | actor framework
[actix-web](https://crates.io/crates/actix-web) | constructs an application instance for each thread; application data must be constructed multiple times or shared between threads

## Terminology

**Shared reference**: An immutable reference (`&T`); can be copied/cloned.

**Exclusive reference**: A mutable reference (`&mut T`); cannot be copied/cloned.

**Aliasing**: Having several immutable references.

**Mutability**: Having one mutable reference.

**[Data race](https://doc.rust-lang.org/nomicon/races.html)**: Two or more threads concurrently accessing a location of memory; one or more of them is a write; one or more of them is unsynchronized.

**[Race condition](https://en.wikipedia.org/wiki/Race_condition)**: The condition of a software system where the system's substantive behavior is dependent on the sequence or timing of other uncontrollable events.

**[Deadlock](https://en.wikipedia.org/wiki/Deadlock)**: Any situation in which no member of some group of entities can proceed because each waits for another member, including itself, to take action.

**[Heisenbug](https://en.wikipedia.org/wiki/Heisenbug)**: A heisenbug is a software bug that seems to disappear or alter its behavior when one attempts to study it. For example, time-sensitive bugs such as race conditions may not occur when the program is slowed down by single-stepping source lines in the debugger.

**Marker trait**: Used to give the compiler certain guarantees (see [`std::marker`](https://doc.rust-lang.org/std/marker/index.html)).

**Thread**: A native OS thread.

**[Green threads (or virtual threads)](https://en.wikipedia.org/wiki/Green_threads)**: Threads that are scheduled by a runtime library or virtual machine (VM) instead of natively by the underlying operating system (OS).

[**Context switch**](https://en.wikipedia.org/wiki/Context_switch): The process of storing the state of a process or thread, so that it can be restored and resume execution at a later point.

**Synchronous I/O**: blocking I/O.

**Asynchronous I/O**: non-blocking I/O.

**Future** (cf. promise): A single value produced asynchronously.

**Stream**: A series of values produced asynchronously.

**Sink**: Write data asynchronously.

**Task**: An asynchronous green thread.

**Channel**: Enables communication between threads or tasks.

**Mutex** (mutual exclusion): Shares data between threads or tasks.

**Interior mutability**: A design pattern that allows mutating data even when there are immutable references to that data.

**Executor**: Runs asynchronous tasks.

**Generator**: Used internally by the compiler. Can stop (or *yield*) its execution and resume (`poll`) afterwards from its last yield point by inspecting the previously stored state in `self`.

**Reactor**: Leaf futures register event sources with the *reactor*.

**Runtime**: Bundles a reactor and an executor.

**`poll`ing**: Attempts to resolve the future into a final value.

**[io_uring](https://en.wikipedia.org/wiki/Io_uring)**: A Linux kernel system call interface for storage device asynchronous I/O operations.

**[CPU-bound](https://en.wikipedia.org/wiki/CPU-bound)**: Refers to a condition in which the time it takes to complete a computation is determined principally by the speed of the CPU.

**[I/O bound](https://en.wikipedia.org/wiki/I/O_bound)**: Refers to a condition in which the time it takes to complete a computation is determined principally by the period spent waiting for input/output operations to be completed. This is the opposite of a task being CPU bound.

## References

- Steve Klabnik and Carol Nichols, [The Rust Programming Language](https://doc.rust-lang.org/book/)
- Jon Gjengset, Rust for Rustaceans
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/intro.html)
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
- [Tokio's work-stealing scheduler](https://tokio.rs/blog/2019-10-scheduler#schedulers-how-do-they-work)
- [Actix user guide](https://actix.rs/book/actix/)
