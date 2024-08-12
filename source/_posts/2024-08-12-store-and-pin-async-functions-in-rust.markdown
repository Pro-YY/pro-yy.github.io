---
layout: post
title: "Store and Pin Async Functions in Rust"
date: 2024-08-12 10:19:06 +0800
comments: true
categories: 
---

In a typical business flow implementation, we usually store the function pointers in some collection data structures (list, hashmap..), and dynamically invoke them according to the inputs/events/conditions, with multiple thread.

But how to with those async function? In dynamic language like JavaScript/Python, this is fairly simple and straightforward. However, this is not the case in the Rust world.

As we've tried, storing async functions in a Vec can be challenging because Rust's async functions (and closures) typically return different types that all implement the Future trait. However, these types are not the same, even if they have the same signature. In Rust, we have to pin and box the Future returned as well as the whole function.

## Define the Boxed Future
What we need to do first iss defining a type alias for a boxed future:
```rust
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;
```

`Pin<Box<...>>`: The Pin type ensures that the Future inside the Box cannot be moved in memory, which is important for certain async operations.
`dyn Future<Output = T> + Send + 'a`:

- `dyn Future<Output = T>`: A dynamically dispatched future returning a value of type T.
- `Send`: Ensures the future can be sent across thread boundaries.
- `'a`: Ensures that all references within the future are valid for at least as long as the lifetime 'a.

## Pin and Box the async function with wrapper
Then we need to wrap our async functions, making them return the boxed future.
Say our async functions are all of the same signature:
```
async fn async_fn1(v: i32) -> Result<i32, Box<dyn Error + Send>>
```

The wrapper should be:

```
type BoxAsyncFn = Box<dyn Fn(i32) -> BoxFuture<'static, Result<i32, Box<dyn Error + Send>>> + Send + Sync>;

fn box_async_fn<F, Fut>(f: F) -> BoxAsyncFn
where
    F: Fn(i32) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = Result<i32, Box<dyn Error + Send>>> + Send + 'static,
{
    Box::new(move |v| Box::pin(f(v)))
}

```

To make our async function can be called/refered, we add `Send` and `Sync` trait bounds to our boxed function.Notice that the result Future it returns need not to be `Sync`, for we don't need to refer it in different threads usually, we just consume it on the fly. As a recap, `Send` means we can safely transfer ownership among threads, and `Sync` means that we can safely transfer reference among threads.


## An example for all
With them all, our final code should be as:
```
use std::future::Future;
use std::pin::Pin;
use std::collections::VecDeque;
use std::error::Error;
use std::boxed::Box;

// Define a type alias for a boxed future with a specific lifetime
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

// Define a type alias for a boxed async function that takes an i32 parameter
type BoxAsyncFn = Box<dyn Fn(i32) -> BoxFuture<'static, Result<i32, Box<dyn Error + Send>>> + Send + Sync>;

// Example async function that takes an i32 parameter and returns a Result
async fn async_fn1(v: i32) -> Result<i32, Box<dyn Error + Send>> {
    Ok(v + 1)
}

// Another example async function that takes an i32 parameter and returns a Result
async fn async_fn2(v: i32) -> Result<i32, Box<dyn Error + Send>> {
    Ok(v * 2)
}

// Function to box the async functions that take an i32 parameter
fn box_async_fn<F, Fut>(f: F) -> BoxAsyncFn
where
    F: Fn(i32) -> Fut + Send + Sync + 'static,
    Fut: Future<Output = Result<i32, Box<dyn Error + Send>>> + Send + 'static,
{
    Box::new(move |v| Box::pin(f(v)))
}

fn main() {
    // Create a vector to store boxed async functions
    let mut async_fns: VecDeque<BoxAsyncFn> = VecDeque::new();

    // Add async functions to the vector without parameters
    async_fns.push_back(box_async_fn(async_fn1));
    async_fns.push_back(box_async_fn(async_fn2));

    // Parameters to pass to the async functions
    let params = vec![10, 20];

    // Execute the async functions with parameters
    for (fut, &param) in async_fns.iter().zip(params.iter()) {
        match futures::executor::block_on(fut(param)) {
            Ok(result) => println!("Success: {}", result),
            Err(e) => eprintln!("Error: {}", e),
        }
    }
}


```

Some explanations here:
- BoxFuture Type Alias simplifies the type signature for boxed futures.
- Async Functions, like `async_fn1`, are our example async functions, containing the business logic.
- The wrapper function `box_async_fn`, it **box** the async functions into the `BoxFuture` type.
- With the Storing Vec, or you can use any collection type, like Vector or Hashmap, we store our boxed function (namely function pointer), in it.

And of course, you can then use an async runtime like Tokio (or async-std) to run our functions for better performance, but this basic example juse demonstrates the concept.

Fear no fraid and happy Pinning(Boxing) your async functions with Rust!
