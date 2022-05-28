# Rust Concurrency Cheat Sheet

- [Safety](#safety)
- [Overview](#overview)
- [Futures](#futures)
- [Threads and tasks](#threads-and-tasks)
- [Thread-safety and marker traits](#thread-safety-and-marker-traits)
- [Concurreny models](#concurreny-models)
- [Terminology](#terminology)
- [References](#references)

## Safety

Rust ensures data race safety through the type system (`Send` and `Sync` marker traits) as well as the ownership and borrowing rules: it is not allowed to alias a mutable reference, so it is not possible to perform a data race.

## Overview

&nbsp; | Problem
------- | -------
Parallelism | Multi-core utilization
Concurrency | Single-core idleness

&nbsp; | Solution | Primitive | Type | Description |
------- | ------- | ------- | ------- | -------
Parallelism | Multithreading | Thread | `T: Send` | Do work simultaneously on different threads
Concurrency | Single-threaded concurrency | Future | `Future` | Futures run concurrently on the same thread
Concurrency+Parallelism | Multithreaded concurrency | Task | `T: Future + Send` | Tasks run concurrently to other tasks; the task may run on the current thread, or it may be sent to a different thread

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

- `Future` has to be `poll`ed (by the *runtime*) to resume where it last yielded and make progress (async is *lazy*)
- `&mut Self` contains state (state machine)
- `Pin` the memory location because the future contains self-referential data
- `Context` contains the `Waker` to notify the *executor* that progress can be made
- async/await on futures is implemented by *generators*

Futures form a tree of futures. The leaf futures commmunicate with the executor. The root future of a tree is called a *task*.

## Threads and tasks

&nbsp; | sync (blocking) | async (non-blocking)
------- | ------- | -------
spawning | `std::thread::spawn` | `async_std::task::spawn`, `tokio::task::spawn`
channel | `std::sync::mpsc` (`Send`), `crossbeam::channel` (`Send`, `Sync`) | `tokio::sync::mpsc`, `tokio::sync::oneshot`, `tokio::sync::broadcast`, `tokio::sync::watch`, `async_channel::unbounded()`, `async_channel::bounded()`
mutex | `std::sync::Mutex` | `tokio::sync::Mutex`

Share state between threads:

- shared-memory data type like `Mutex`, or
- communicate via channels

## Thread-safety and marker traits

- `Send`: safe to send it to another thread
- `Sync`: safe to share between threads

Type | `Send` | `Sync`
------- | ------- | ------- |
`Rc<T>` | No | No
`Arc<T>` | Yes | Yes
`Mutex<T>` | Yes | Yes

## Concurreny models

- shared memory
- worker pools
- actors

Tokio (multithreaded) | Actix
------------ | -------------
thread pool with work-stealing strategy | actor framework

## Terminology

**[Data race](https://doc.rust-lang.org/nomicon/races.html)**: Two or more threads concurrently accessing a location of memory; one or more of them is a write; one or more of them is unsynchronized.

**Thread**: A native OS thread.

**[Green threads (or virtual threads)](https://en.wikipedia.org/wiki/Green_threads)**: Threads that are scheduled by a runtime library or virtual machine (VM) instead of natively by the underlying operating system (OS).

**Future** (cf. promise): A single value produced asynchronously.

**Stream**: A series of values produced asynchronously.

**Sink**: Write data asynchronously.

**Task**: An asynchronous green thread.

**Channel**: Enables communication between threads.

**Mutex** (mutual exclusion): Shares data between threads.

**Executor**: Runs asynchronous tasks.

**[io_uring](https://en.wikipedia.org/wiki/Io_uring)**: A Linux kernel system call interface for storage device asynchronous I/O operations

## References

- Steve Klabnik and Carol Nichols, [The Rust Programming Language](https://doc.rust-lang.org/book/)
- Jon Gjengset, Rust for Rustaceans
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/intro.html)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
