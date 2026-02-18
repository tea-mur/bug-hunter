# Async Programming

**Keywords**: TFuture, TPromise, TCallback, BIND, Subscribe, Apply, MakeWeak, MakeStrong, async, callbacks, lambda

YT uses a future-based async programming model with `TFuture<T>` for representing asynchronous results. Understanding proper lifetime management in callbacks is critical to avoid use-after-free bugs.

## TFuture and TCallback

- **Location**: `yt/yt/core/actions/future.h`, `yt/yt/core/actions/callback.h`
- `TFuture<T>` represents an asynchronous result that will be available in the future
- `TPromise<T>` is used to set the result of a future
- `TCallback<T>` is a type-erased callable that can store lambdas or bound functions
- Common operations: `Apply()`, `Subscribe()`, `Via()` for chaining async operations

## BIND and lifetime management

The `BIND` macro is used to create callbacks. When binding member functions or lambdas that capture `this`, **lifetime management is critical**:

### CORRECT patterns (ref-counted objects):

```cpp
// MakeWeak(this) - callback is silently dropped if object is destroyed
someSignal->Subscribe(
    BIND(&MyClass::OnEvent, MakeWeak(this))
        .Via(GetInvoker()));

// MakeStrong(this) - keeps object alive while callback is pending
someFuture.Apply(
    BIND(&MyClass::ProcessResult, MakeStrong(this))
        .AsyncVia(GetInvoker())
        .Run());

// Lambda with MakeStrong
someFuture.Subscribe(
    BIND([this, this_ = MakeStrong(this)] (const TResult& result) {
        ProcessResult(result);
    }));
```

### INCORRECT pattern - DANGLING POINTER:

```cpp
// BAD: raw 'this' pointer in async callback
someFuture
    .Apply(BIND([=, this] (const TArg& arg) {
        return this->DoWork(arg);  // this may be dangling!
    }))
    .Subscribe(BIND([=, this] (const TResult& result) {
        this->HandleResult(result);  // use-after-free if object destroyed!
    }));
```

**Why this is wrong**: The lambda captures `this` as a raw pointer. If the object is destroyed before the future completes, the callback will access freed memory.

## When to use MakeWeak vs MakeStrong

- **MakeWeak**: When you want the callback to be dropped if the object is destroyed. Common for event subscriptions where missing an event is acceptable.
- **MakeStrong**: When the callback *must* execute and the object must stay alive until completion. Common for critical cleanup or state updates.
- **Rule of thumb**: For async operations (futures, RPCs, transactions), prefer `MakeStrong` unless you explicitly want to cancel the operation on object destruction.

## Gotcha: StopEpoch/Finalize don't wait for futures

In tablet node code, lifecycle methods like `StopEpoch()` and `Finalize()` typically stop periodic executors and reset state, but **they do not cancel or wait for in-flight futures**. This means:

1. If you start an async operation in `StartEpoch()`
2. Store no reference to the future
3. Object gets destroyed in `StopEpoch()` / `Finalize()`
4. Future completes later → callback accesses destroyed object → **use-after-free**

**Example bug found**: `yt/yt/server/node/tablet_node/hunk_lock_manager.cpp:522` - `ToggleLock` starts a transaction and subscribes a callback with raw `this`, but `StopEpoch()` doesn't wait for the transaction to complete.

## BIND_NO_PROPAGATE

`BIND_NO_PROPAGATE` is a variant that doesn't propagate cancellation. Often used with `MakeWeak` for event handlers that should simply be dropped (not actively canceled) if the object is gone.

## Common APIs that return TFuture

- RPC calls: `proxy.SomeMethod()->Invoke()`
- Transactions: `transaction->Commit(options)`
- I/O operations: file reads, network fetches
- Invoker operations: `BIND(...).AsyncVia(invoker).Run()`

All of these can complete after the caller is destroyed - must use proper lifetime management!
