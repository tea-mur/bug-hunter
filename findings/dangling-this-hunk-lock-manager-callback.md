# Dangling `this` pointer in THunkLockManager async callbacks

**Confidence**: high
**Severity**: major
**Category**: lifetime
**Files**: yt/yt/server/node/tablet_node/hunk_lock_manager.cpp:494, yt/yt/server/node/tablet_node/hunk_lock_manager.cpp:522

## Description

In `THunkLockManager::ToggleLock`, two lambdas capture `this` as a raw pointer and are passed to asynchronous future operations (`Apply` and `Subscribe`). These callbacks can be invoked after the `THunkLockManager` object is destroyed, leading to use-after-free bugs.

The lambdas access:
- Member methods like `OnBoggleLockFailed` (line 524)
- Member variables via `Context_` and `Tablet_` (lines 495, 500, 508)

When a tablet is unmounted or destroyed, `THunkLockManager::StopEpoch()` is called, which stops the periodic unlock executor but **does not wait for in-flight transaction futures to complete**. If a transaction initiated by `ToggleLock` is still pending when the tablet/manager is destroyed, the `Subscribe` callback will fire after the object is gone.

## Evidence

**yt/yt/server/node/tablet_node/hunk_lock_manager.cpp:493-528:**
```cpp
transactionFuture
    .Apply(BIND([=, this] (const NNative::ITransactionPtr& transaction) {
        auto tabletCellId = Context_->GetCellId();  // Accesses Context_ member

        NTabletClient::NProto::TReqToggleHunkTabletStoreLock hunkRequest;
        ToProto(hunkRequest.mutable_tablet_id(), hunkTabletId);
        ToProto(hunkRequest.mutable_store_id(), hunkStoreId);
        ToProto(hunkRequest.mutable_locker_tablet_id(), Tablet_->GetId());  // Accesses Tablet_ member
        // ... more setup ...
        return transaction->Commit(commitOptions);
    }))
    .Subscribe(BIND([=, this] (const TErrorOr<NApi::TTransactionCommitResult>& resultOrError) {
        if (!resultOrError.IsOK()) {
            OnBoggleLockFailed(hunkStoreId, lock, resultOrError);  // Calls method on this
        }
    })
        .Via(GetCurrentInvoker()));
```

Both lambdas use `[=, this]` which captures `this` as a raw pointer. The `Subscribe` callback is registered on a future that represents a distributed transaction commit, which can take significant time (network round-trips, 2PC coordination).

**yt/yt/server/node/tablet_node/hunk_lock_manager.cpp:123-133 (StopEpoch):**
```cpp
void StopEpoch() override
{
    YT_ASSERT_THREAD_AFFINITY(AutomatonThread);

    if (UnlockExecutor_) {
        YT_UNUSED_FUTURE(UnlockExecutor_->Stop());
    }
    UnlockExecutor_.Reset();

    ClearTransientState();
}
```

`StopEpoch` does not cancel or wait for in-flight transaction futures. When the tablet is unmounted, this is called, and then the `THunkLockManager` can be destroyed while callbacks are still pending.

**yt/yt/server/node/tablet_node/hunk_lock_manager.h:**
```cpp
struct IHunkLockManager
    : public TRefCounted
{
    // ...
};
```

The class is ref-counted, so the fix should use `MakeWeak(this)` or `MakeStrong(this)` in the BIND macro to properly manage lifetime.

**Correct pattern seen elsewhere in the same file (line 104):**
```cpp
configManager->SubscribeConfigChanged(
    BIND(&THunkLockManager::OnDynamicConfigChanged, MakeWeak(this))
        .Via(Context_->GetAutomatonInvoker()));
```

This correctly uses `MakeWeak(this)` for a callback subscription.

## Similar occurrences

Checked other methods in the same file - `Initialize()` and `StartEpoch()` use `MakeWeak(this)` correctly. The bug appears isolated to the `ToggleLock` method.

I searched for similar patterns in tablet_node with:
```bash
grep -r "BIND.*\[.*this" server/node/tablet_node/ | grep -v "MakeWeak\|MakeStrong\|this_"
```

Found a few other instances:
- `server/node/tablet_node/lookup.cpp`: Similar pattern with `BIND([this, ...`
- `server/node/tablet_node/unittests/tablet_cell_write_manager_ut_helpers.h`: Test code

The lookup.cpp case should be investigated separately.
