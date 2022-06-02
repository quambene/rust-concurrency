<!-- markdownlint-disable MD033 -->

# Rust Concurrency Cheat Sheet

- [Safety](#safety)
- [Overview](#overview)
- [Futures](#futures)
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
Concurrency | Single-threaded concurrency | Future | `Future` | Futures run concurrently on the same thread | [`futures::future::join`](https://docs.rs/futures/latest/futures/future/fn.join.html), [`futures::join`](https://docs.rs/futures/latest/futures/macro.join.html)
Concurrency<br>+Parallelism | Multithreaded concurrency | Task | `T: Future + Send` | Tasks run concurrently to other tasks; the task may run on the current thread, or it may be sent to a different thread | [`async_std::task::spawn`](https://docs.rs/async-std/latest/async_std/task/fn.spawn.html), [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/fn.spawn.html)

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

## Share state

&nbsp; | Threads | Tasks
------- | ------- | -------
channel | `std::sync::mpsc` (`Send`), `crossbeam::channel` (`Send`, `Sync`) | `tokio::sync::mpsc`, `tokio::sync::oneshot`, `tokio::sync::broadcast`, `tokio::sync::watch`, `async_channel::unbounded`, `async_channel::bounded`
mutex | `std::sync::Mutex` | `tokio::sync::Mutex`

## Marker traits

- `Send`: safe to send it to another thread
- `Sync`: safe to share between threads

Type | `Send` | `Sync`
------- | ------- | -------
`Rc<T>` | No | No
`Arc<T>` | Yes (if `T` is `Send`) | Yes (if `T` is `Sync`)
`Mutex<T>` | Yes (if `T` is `Send`) | Yes (if `T` is `Send`)
`RwLock<T>` | Yes (if `T` is `Send`) | Yes (if `T` is `Send` and `Sync`)

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

**Marker trait**: Used to give the compiler certain guarantees (see [`std::marker`](https://doc.rust-lang.org/std/marker/index.html)).

**[Data race](https://doc.rust-lang.org/nomicon/races.html)**: Two or more threads concurrently accessing a location of memory; one or more of them is a write; one or more of them is unsynchronized.

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

**Executor**: Runs asynchronous tasks.

**Generator**: Used internally by the compiler. Can stop (or *yield*) its execution and resume (`poll`) afterwards from its last yield point by inspecting the previously stored state in `self`.

**`poll`ing**: Attempts to resolve the future into a final value.

**[io_uring](https://en.wikipedia.org/wiki/Io_uring)**: A Linux kernel system call interface for storage device asynchronous I/O operations.

## References

- Steve Klabnik and Carol Nichols, [The Rust Programming Language](https://doc.rust-lang.org/book/)
- Jon Gjengset, Rust for Rustaceans
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/intro.html)
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
- [Tokio's work-stealing scheduler](https://tokio.rs/blog/2019-10-scheduler#schedulers-how-do-they-work)
- [Actix user guide](https://actix.rs/book/actix/)
