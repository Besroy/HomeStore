# RCA: Root Split Crash Leaves SB Pointing to Old Root

**Date:** 2026-05-22
**Component:** HomeStore / IndexTable / BTree / IndexWBCache
**Severity:** Critical (crash on restart, blocks index recovery)

---

## Symptom

After a SIGKILL, the process crashes during restart with a B-tree sanity failure.

In release builds:

```
Child node level mismatch ... child level: 1, expected: 0
```

In debug builds, recovery may abort earlier inside `IndexTable::repair_root_node`:

```
root already has a valid edge ..., so we should have found the new root node
```

---

## Root Cause Chain

### Condition 1: `on_root_changed` is called before `split_node` completes

`Btree::check_split_root` (`src/include/homestore/btree/detail/btree_mutate_impl.ipp`):

```
1. Allocate new_root (level=2)
2. Call on_root_changed(new_root)      ← SB and journal updated here
3. split_node(new_root, old_root, ...) ← old root modified here
```

`IndexTable::on_root_changed` updates the in-memory SB (`index_table_sb.root_node`,
`btree_depth`) and calls `wb_cache().transact_bufs(meta, root)`, which links
`meta → new_root` and appends a meta/root transaction record to the journal.

At this moment the new root may still be CLEAN (not yet dirtied via `write_node_impl`).
**Evidence:** `CLEAN_BUF_DEBUG: Adding CLEAN down_buf 1407379178523696 to up_buf 0
... may lead to up_buf being stuck in cp flush`

### Condition 2: The old root's `edge_info` is set to a same-level interior node

`split_node(new_root, old_root, ...)` restructures the old root into an interior node
and sets its `edge_info` to point to the sibling of the original split. At this
intermediate state:

- `old_root.level = 1`
- `old_root.edge_info.m_bnodeid = 1125904201813044` (an interior node also at `level=1`)

For a `level=1` node all children (including edge child) must be `level=0` leaves.
This state is only safe transiently — it is resolved when the new root takes ownership.
**Evidence (gdb on coredump):** `node_id=1125904201810030, level=1,
edge_info.m_bnodeid=1125904201813044` (confirmed `level=1` interior node).

### Condition 3: SB persistence is deferred to end-of-CP flush

Even though `on_root_changed` updated the in-memory SB, the persisted superblock
write happens only after the last buffer completes during CP flush
(`IndexWBCache::process_write_completion` → `index_service().write_sb(ordinal)`
in `src/lib/index/wb_cache.cpp`).

A SIGKILL between Condition 1 and this deferred write leaves the on-disk SB still
pointing to the old root.
**Evidence (gdb):** `index_table_sb.root_node=1125904201810030, btree_depth=1`
— matches the old root, not the new level-2 root.

### Condition 4: A new level-2 root exists on disk but is not referenced by the SB

Recovery DAG logs show: `id=1407379178523696 ... INTERIOR level=2 ... NEW`

The new root was written and is present on disk, but the persisted SB still points
to `old_root` (Condition 3). Recovery therefore starts from the old root.

### Condition 5: `repair_root_node` cannot handle old root with a pre-existing `edge_info`

`IndexWBCache::recover_buf` → `IndexService::update_root` → `IndexTable::repair_root_node`
(`src/include/homestore/index/index_table.hpp`).

`repair_root_node` is designed to repair a root-change marker by converting
`next_bnode` into `edge_info`. It contains a hard assumption: if the SB root already
has a valid `edge_info`, it asserts/FC because "we should have found the new root node."

With the on-disk state from Conditions 2 and 3:
- SB root = old root (level=1)
- old root already has `edge_info` set (to a same-level interior node)

Recovery hits the assertion and aborts.

---

## Trigger Conditions

The following sequence must all occur:

1. A B-tree root split is triggered (tree height increases from level=1 to level=2)
2. `on_root_changed` runs and links `meta → new_root` while `new_root` is still CLEAN
3. `split_node` sets old root's `edge_info` to the sibling interior node
4. SIGKILL arrives after step 3 but before CP flush completes and writes the updated SB

The critical window is the duration of the CP flush after a root split. Because the
new root enters the dependency graph as CLEAN, it may stall or delay the flush,
widening the window.

---

## Skipped Conditions

None. All five conditions confirmed:

- Conditions 1 and 2: confirmed via static code analysis of `check_split_root` and `on_root_changed`
- Condition 3: confirmed via code analysis of `process_write_completion`
- Condition 4: confirmed via recovery DAG log showing the level-2 root as NEW
- Condition 5: confirmed via gdb coredump showing `edge_info` set on old root, and via code analysis of `repair_root_node`

---

## Fix Direction

**Option A — Fix root split ordering (root cause):**
Follow the same "write-before-root-change" discipline as the root collapse fix.
Dirty the new root (call `write_node_impl`) before calling `on_root_changed`, ensuring
the new root enters the CP dependency graph as DIRTY and is covered by the flush
before the SB is updated.

**Option B — Harden recovery (defensive):**
Enhance `repair_root_node` (or the recovery path above it) to handle the case where
the SB root already has a valid `edge_info`. In that case, treat `edge_info` as a
hint to search for the new root rather than asserting. If a level-2 root is found,
update the SB to point to it and continue recovery normally.

Both options are independent and can be applied together. Option A eliminates the
window; Option B prevents recovery from aborting if the window is ever hit.

**Relevant files:**

- `src/include/homestore/btree/detail/btree_mutate_impl.ipp` — `check_split_root`
- `src/include/homestore/index/index_table.hpp` — `on_root_changed`, `repair_root_node`
- `src/lib/index/wb_cache.cpp` — `async_cp_flush`, `process_write_completion`, `recover`, `recover_buf`
- `src/include/homestore/btree/detail/btree_node.hpp` — node header definition
- `src/lib/index/index_cp.hpp`, `src/lib/index/index_cp.cpp` — journal ordering
