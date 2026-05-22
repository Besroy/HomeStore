# RCA: Root Split Crash Leaves SB Pointing to Old Root

## Summary

After a force-kill and restart, the index recovery path aborts with a B-tree sanity failure.
The failure is caused by a **crash during a root split / CP flush window** that leaves the index table superblock (SB) still pointing to the **old root**, while the old root’s persistent header has already been modified into an **intermediate state** where its `edge_info` points to a same-level interior node.

This violates the fundamental B-tree invariant:

- `child.level == parent.level - 1`

and triggers validation failure and/or `repair_root_node()` assertions during recovery.

---

## User-visible symptoms

- On restart after force kill, recovery hits a sanity check failure:
  - “Child node level mismatch ... child level: 1, expected: 0”

- In debug builds, recovery may abort earlier inside `IndexTable::repair_root_node` with:
  - “root already has a valid edge ..., so we should have found the new root node”

---

## Evidence

### 1) Superblock still points to old root

From gdb on coredump:

- `index_table_sb.root_node = 1125904201810030`
- `index_table_sb.btree_depth = 1`

So the persisted SB still represents a tree of height 1 and points to the old root.

### 2) Old root persistent header is in an invalid intermediate state

From gdb reading `persistent_hdr_t` directly:

- `node_id = 1125904201810030`
- `level = 1`
- `edge_info.m_bnodeid = 1125904201813044`

But `1125904201813044` is an **interior** node with **level=1** (same level as the old root).

For a `level=1` interior node, children must be `level=0` leaves; therefore `edge_info` is not a valid child pointer.

### 3) Recovery DAG shows a newly created level-2 root exists

From recovery DAG logs:

- `id=1407379178523696 ... INTERIOR level=2 ... NEW`

This indicates that a root split occurred (tree height increased), and a new root was created.

### 4) CP flush logs indicate meta/root dependency was created with the new root treated as CLEAN

Log snippet:

- `CLEAN_BUF_DEBUG: Adding CLEAN down_buf 1407379178523696 to up_buf 0 ... may lead to up_buf being stuck in cp flush`

This demonstrates that the meta buffer was linked to the new root before the new root became DIRTY in the active CP.

---

## Code-level analysis

### A) Root split ordering creates a vulnerable window

Root split happens in:

- `src/include/homestore/btree/detail/btree_mutate_impl.ipp`
  - `Btree::check_split_root(...)`

Current ordering (simplified):

1. Allocate `new_root`
2. **Call `on_root_changed(new_root)` before splitting the old root**
3. Split old root into children under `new_root` via `split_node(new_root, old_root, ...)`

`IndexTable::on_root_changed`:

- Updates in-memory SB state (`index_table_sb.root_node`, `btree_depth`, etc.)
- Calls `wb_cache().transact_bufs(meta, root)` which:
  - links `meta -> root` (`link_buf`)
  - appends a meta/root transaction record to the journal

However, at this moment the new root may still be CLEAN (not yet dirtied via `write_node_impl`).
This is exactly what the `CLEAN_BUF_DEBUG` warning indicates.

### B) SB persistence is deferred to end-of-CP flush

Even if `on_root_changed` updated the in-memory SB, the persisted superblock update happens only at the end of CP flush:

- `src/lib/index/wb_cache.cpp`
  - `IndexWBCache::process_write_completion(...)`:
    - after the last buffer completes, it calls `index_service().write_sb(ordinal)`

Therefore, if the process crashes mid-flush, the persisted SB may still point to the old root.

### C) Recovery root repair assumes old root cannot already have a valid edge

Recovery calls:

- `IndexWBCache::recover_buf` -> `IndexService::update_root` -> `IndexTable::repair_root_node`

`repair_root_node` is intended to repair a root-change marker by converting `next_bnode` into `edge_info`.
But it contains a strong assumption:

- If the SB root already has a valid edge, it asserts/FC because it “should have found the new root node”.

In the intermediate state produced by the crash:

- The SB still points to the old root
- The old root already has `edge_info` populated (but pointing to the wrong object: a same-level interior node)

So recovery aborts.

---

## Root cause

A crash occurred during a root split / CP flush window such that:

- The new root (level=2) was created and partially integrated into the dependency/journal system.
- The old root was modified into an intermediate state where its `edge_info` points to a same-level interior node.
- The persisted index table SB was not updated (SB writes are deferred to end-of-flush).
- Recovery logic does not handle this intermediate state and aborts.

---

## Why this manifests as “child level mismatch”

The B-tree validation (`Btree::validate_node`) expects:

- for a parent node at `level=1`, all children (including edge child) must be at `level=0`.

But the old root’s `edge_info` points to an interior node at `level=1`.

---

## Relevant files

- Node header definition:
  - `src/include/homestore/btree/detail/btree_node.hpp`

- Root split:
  - `src/include/homestore/btree/detail/btree_mutate_impl.ipp`

- Root change + meta/root transaction:
  - `src/include/homestore/index/index_table.hpp` (`on_root_changed`, `repair_root_node`)

- CP flush ordering and SB persistence timing:
  - `src/lib/index/wb_cache.cpp` (`async_cp_flush`, `process_write_completion`)

- Recovery flow:
  - `src/lib/index/wb_cache.cpp` (`recover`, `recover_buf`)

- Journal ordering:
  - `src/lib/index/index_cp.hpp`, `src/lib/index/index_cp.cpp`

---

## Notes

This document describes the observed behavior and explains why the failure is deterministic given the persisted state.
A follow-up fix would likely involve either:

- Making root split follow the same “write-before-root-change” discipline as the root collapse fix, and/or
- Enhancing recovery to discover the new root and update the SB when the old root already has a valid edge.
