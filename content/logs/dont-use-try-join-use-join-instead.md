---
title: Don't use `tokio::try_join!`, use `tokio::join!` instead.
description: "tokio::try_join! might look convenient with built in error handling and parellalism in one line, but this presents some major design flaws in this api."
template: pages/log.html
date: 2025-09-09
---

> **NOTE**
>
> Before reading this article, read this. This isn't targeted to the tokio team at all. The `futures` crate also has this issue but `tokio` is much more popular so this is why this article mentions `tokio`.
> The tokio team is extremely talented and maintains one of the most widely used libraries in the rust ecosystem. Tokio is a wonderful piece of tech.
> Please look at this article as raising awareness of this api issue, and an explanation of how the issue actually affects production code, as that is its intended purpose.

Whilst the `tokio::try_join!` macro is very convenient to use in many scenarios, with it's builtin error handling and parellalism in just one line, it can present some major bugs.
To understand these bugs, we must first understand the [`Future`](https://doc.rust-lang.org/stable/std/future/trait.Future.html) trait.

## What is a future?

```rust
// std::future

pub trait Future {
    type Output;

    // Required method
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

A Future represents a value which will be available sometime in the future.
The rust team designed a `Future` to be a trait with a `poll` method defined. This method is used to drive progress in the future.
This method has two arguments:
 - A mutable (`Pin` means non-movable, so self is a non-movable) reference to itself
 - A Context paramter. The context struct only consists of a waker, more on this later.

 The poll method expects to return `Poll<Self::Output>`:

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

- If `Future::poll` returns `Poll::Pending`, it means that the Future isn't ready to return the value, and **IS REQUIRED** to store the waker somewhere, so that it [can call it](https://doc.rust-lang.org/stable/std/task/struct.Waker.html#method.wake).
- `Future::poll` is not called in a tight loop, it is only called when the waker is called, this puts responsiblilty of the Future implementor to only call the waker in a optimised way.
- When the waker is called (can be anywhere in the program), it tells the runtime that the linked `Future` should be `Future::poll`ed again.
- The runtime assumes that the waker will be called in some point in the future. If the waker never gets called, the future **never completes**.
- When `Future::poll` returns `Poll::Ready(Self::Output)`, it is undefined behaviour if `Future::poll` gets called again. Additionally, this also means it's undefined behaviour if a waker is called somewhere in your system after the linked Future has returned `Poll::Ready(Self::Output)`.
- In rust a Future is "cold", which means that if you don't poll it, nothing will happen. This in contrast to javascript's Promises which execute in the background,  even if you dont `.await`.

Here's an example of a super simple Future impl:
```rust
use std::{task::{Poll, Context}, future::Future};

#[derive(Default)]
pub struct FutureThatReturnsPendingAtFirstAndThenReturnsPending {
    has_returned_pending: bool
}

impl Future for FutureThatReturnsPendingAtFirstAndThenReturnsPending {
  type Output = i32;

  fn poll(self: Pin<&mut Self>, context: &mut Context) -> Poll<Self::Output> {
    if !self.has_returned_pending {
      self.has_returned_pending = true;

      // NEVER CALL THE WAKER IN THE SAME FUTURE IN REAL CODE. VERY UNEFFICIENT
      context.waker().wake_by_ref();

      println!("pending");

      Poll::Pending
    } else {
      println!("ready");
      Poll::Ready(0)
    }
  }
}

fn main() {
  let mut fut = FutureThatReturnsPendingAtFirstAndThenReturnsPending::default();

  // A fake waker as this isn't used for any functionality.
  let waker = futures_task::noop_waker();

  let poll_result = Pin::new(&mut fut)
    .poll(&mut Context::from_waker(waker.clone())); // print: "pending"

  assert!(poll_result == Poll::Pending);

  let poll_result = Pin::new(&mut fut)
    .poll(&mut Context::from_waker(waker.clone())); // print: "ready"

  assert!(poll_result == Poll::Ready(0));
}
```

## `async { ... }`

This `Future` thing is just a trait, so it needs to be implemented for people to use Futures right? Yes, and rust choose the `async`/`await` syntax for this.
An async block is what is called an anonumys Future. That's because it cannot be encoded as a type without generics:

```rust
let fut: Box<dyn Future<Output = ()>> = Box::new(async { () })
```

The rust compiler implements Future on async blocks for you. It does this by making every `.await` statement a pollable point. It turns your async block into a state-machine where at every `.await` point, is a "step" in the state machine.
The poll method tries to poll the inner future thats `.await`ed. If the inner future returns:
- `Poll::Pending`, `async {}` Future also returns `Poll::Pending`.
- `Poll::Ready(...)`, `async {}` continues execution until the next `.await` point and repeats or captures the return value and itself returns `Poll::Ready(return_value)`.

> **NOTE**
>
> The state machine/`async { ... }` Future doesn't need to register the waker at all. This is because it has passed the waker to the child-future.
> The child-future has registered the waker, As a general rule: If only calling child-futures the parent future doesn't need to worry about wakers.


Also, `async fn` statements:
```rust
async fn do_something(arg: i32) -> i32 {
    arg
}
```

get compiled to this:

```rust
fn do_something(arg: i32) -> impl Future<Output = i32> /* also: `+ Send` if future is Send */ {
    // 'move' is for borrowing rules to comply sometimes.
    async move {
        arg
    }
}
```

Now that we know this, we can understand what big problem `tokio::try_join!` is.

### `tokio::try_join!`

According to [tokio docs.rs](https://docs.rs/tokio/latest/tokio/macro.try_join.html):

```text
Waits on multiple concurrent branches, returning when all branches

complete with Ok(_) or on the first Err(_).

...

Similar to join!, the try_join! macro takes a list of async expressions
and evaluates them concurrently on the same task. Each async expression
evaluates to a future and the futures from each expression are multiplexed
on the current task. The try_join! macro returns when all branches return
with Ok or when the first branch returns with Err.
```

This api seems very convenient but it is flawed.

Take an example:

```rust

type Fut = FutureThatReturnsPendingAtFirstAndThenReturnsPending;
#[tokio::main]
async fn main() {
    let outer_fut1 = async {
      let mut fut1 = Fut::default();
      // First await point, async's poll method will call poll twice
      // on this line, one where
      // `FutureThatReturnsPendingAtFirstAndThenReturnsPending::poll` returns
      // Poll::Pending, and the other where it returns Poll::Ready(0)
      fut1.await;
    };

    let outer_fut2 = async {
      let mut fut1 = Fut::default();
      // same here
      fut1.await;

      let mut fut2 = Fut::default();
      // same here
      fut2.await;
    };

    let outer_fut3 = async {
      let mut fut1 = Fut::default();
      // same here
      fut1.await;

      let mut fut2 = Fut::default();
      // same here
      fut2.await;

      let mut fut3 = Fut::default();
      // same here
      fut3.await;
    };

    let result = tokio::try_join!(outer_fut1, outer_fut2, outer_fut3);

    assert!(Ok(((), (), ())) == result);
}
```

This works great. We can see roughtly the order of completion. 'cycle' refers to the async runtime's event loop cycle count.

| Futures       | cycle 1 | cycle 2 | cycle 3 |
|---------------|---------|---------|---------|
| outer_fut1    | fut1    |         |         |
| outer_fut2    | fut1    | fut2    |         |
| outer_fut3    | fut1    | fut2    | fut3    |

But what happens if any future returns an error? Well this happens:


```rust

// Taken from article, scroll up for definition.
type Fut = FutureThatReturnsPendingAtFirstAndThenReturnsPending;
#[tokio::main]
async fn main() {
    let outer_fut1 = async {
      let mut fut1 = Fut::default();
      // First await point, async's poll method will call poll twice
      // on this line, one where
      // `FutureThatReturnsPendingAtFirstAndThenReturnsPending::poll` returns
      // Poll::Pending, and the other where it returns Poll::Ready(0)
      fut1.await;
    };

    let outer_fut2 = async {
      let mut fut1 = Fut::default();
      // same here
      fut1.await;

      let mut fut2 = FutureThatReturnsErrInsidePollReady::default();
      // same here (here fut2.await = Err(...))
      fut2.await?;
    };

    let outer_fut3 = async {
      let mut fut1 = Fut::default();
      // same here
      fut1.await;

      let mut fut2 = FutureThatReturnsErrInsidePollReady::default();
      // same here (here fut2.await = Err(...))
      fut2.await?;

      let mut fut3 = Fut::default();
      // same here
      fut3.await;
    };

    let result = tokio::try_join!(outer_fut1, outer_fut2, outer_fut3);

    assert!(Err(...) == result);
}
```

Here both outer_fut3:fut2 and outer_fut2:fut2 are guarranteed to throw the error. Lets look at the table again:
- X means the future never is executed
- Y means the future is guarranteed to be done.
- M means the future maybe is executed to completion.

| Futures       | cycle 1 | cycle 2    | cycle 3 |
|---------------|---------|------------|---------|
| outer_fut1    | fut1 Y  |            |         |
| outer_fut2    | fut1 Y  | fut2 M     |         |
| outer_fut3    | fut1 Y  | fut2 M     | fut3 X  |

_<u>Bad parts:</u>_
- outer_fut3:fut3 is **never** executed.
- We don't know from which future the error came from.
- This becomes exponentially less predictable as Futures can have different amount of poll calls, `async` blocks can have different amount of `.await` points.
- **It is not guarranteed which of outer_fut2:fut2 or outer_fut3:fut2 will error and which will not be run at all**. This depends entirely of the runtime implementation (both the `try_join!` macro and the runtime internals).

The last point becomes exponentially worse when the runtime uses threads to poll futures in parallel.

## "Rust is full of these minefields!!!" i hear you say

In my opinion, this isn't rusts fault. Rust values zero-cost abstraction and if control over execution is required (as it often is when choosing rust), this is the best design of async i've ever seen.
This issue is not the `Future` trait, or how `async { ... }` is compiled, it's just a library API mistake which leaks internal functionality on to the library user.
