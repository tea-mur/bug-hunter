# Scan Log

## 2026-02-18 — Tablet node lifecycle and concurrency

**Agent**: agent-8911
**Status**: done
**Files scanned**: `yt/yt/server/node/tablet_node/hunk_lock_manager.cpp`, `tablet_slot.cpp`, `master_connector.cpp`
**Findings**: 1 (high: 1, medium: 0, low: 0)
**Notes**: Found use-after-free bug in THunkLockManager where async transaction callbacks capture raw `this` pointer. Callbacks can fire after object destruction during tablet unmount. Other files (tablet_slot, master_connector) use correct MakeWeak pattern.
