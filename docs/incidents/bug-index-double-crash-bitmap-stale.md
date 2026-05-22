# RCA: Index Crash on Restart — Duplicate Insert After Double Crash

**Date:** 2026-05-22
**Jira:** SDSTOR-21939
**Component:** HomeStore / IndexWBCache / FixedBlkAllocator
**Severity:** Critical (crash on restart, blocks recovery)

---

## Symptom

After two consecutive unclean shutdowns (the second occurring before a vdev CP
completes), SM crashes during normal post-restart operation at
`wb_cache.cpp:117`:

```
HS_REL_ASSERT_EQ(done, true, "Unable to add alloc'd node to cache, low memory or duplicate inserts?");
```

`IndexWBCache::alloc_buf()` calls `m_cache.insert(node)` which returns `false`
because a node at the same `BlkId` (chunk=21, blk=45) is already present in
the cache. GDB confirms the persistent allocator bitmap (`m_disk_bm`) shows
blk45 as FREE, while a live node keyed by the same BlkId exists in the cache.

---

## Root Cause Chain

### Condition 1: New node data page is flushed to disk before CP12 completes

During CP12 flush, DAG traversal starts from leaves. The new node
`5910978805891162` (blkid = chunk21/blk45, `create_cp=12`) has its data page
written to disk successfully:

```
[04/18/26 11:05:08.271] cp 12 completed flushed for buf ... blkid blk#=45 count=1 chunk=21
```

SM is then killed. CP12's vdev flush — which would persist the allocator
bitmap — never completes.

**Evidence:** T1 log shows `cp 12 completed flushed` for the new node but no
corresponding "CP 12 flush done" entry.

### Condition 2: First recovery (T2) correctly calls `commit_blk()`, but the result is in-memory only

On restart, `IndexWBCache::recover()` replays the CP12 journal. It finds the
new node and its up buffer are both committed (`was_node_committed()` = true
for both), and calls `m_vdev->commit_blk({chunk=21, blk=45})`.

`commit_blk()` during recovery (`hs()->is_initializing() == true`):

```cpp
// virtual_dev.cpp:186
chunk->blk_allocator_mutable()->reserve_on_cache(blkid);  // adds blk45 to m_reserved_blks
// virtual_dev.cpp:188
chunk->blk_allocator_mutable()->reserve_on_disk(blkid);   // sets bit in m_disk_bm (IN MEMORY)
```

`reserve_on_disk` (`bitmap_blk_allocator.cpp:107`) sets
`m_disk_bm->set_bits(45)` and marks `m_is_disk_bm_dirty = true`. This is an
**in-memory-only** update. Persisting `m_disk_bm` to physical storage requires
a subsequent vdev CP flush (`BitmapBlkAllocator::cp_flush()`).

`recovery_completed()` then removes blk45 from `m_free_blk_q`. At this point
the in-memory allocator state is correct: blk45 = USED.

**Evidence:**
```
[04/18/26 11:07:45.252] recovering new buf ... node=[id=5910978805891162 ...] blkid blk#=45 chunk=21
[04/18/26 11:07:45.252] New buffer ... and the up buffer ... are committed
```

### Condition 3: SM crashes again before a vdev CP persists the allocator bitmap

T2 runs normal operations after recovery but is killed again before any vdev
CP completes (the vdev CP is what would flush `m_disk_bm` to physical
storage). All in-memory allocator state — including the blk45=USED mark from
Condition 2 — is lost.

**Evidence (T3 log):** CP13 had 0 dirty buffers, meaning no meaningful work
happened between T2 recovery completion and T3's kill. The disk bitmap was
never updated.

### Condition 4: Second recovery (T3) finds an empty CP journal — `commit_blk()` is never called for blk45

On T3 restart, the vdev loads `m_disk_bm` from physical storage. Since the
bitmap was never persisted with blk45=USED, it shows blk45 as **FREE**.
`FixedBlkAllocator::load()` scans the disk bitmap and enqueues all FREE blocks
into `m_free_blk_q`:

```cpp
// fixed_blk_allocator.cpp:43
if (!is_persistent() || get_disk_bitmap()->is_bits_reset(blk_num, 1)) {
    m_free_blk_q.write(blk_num);  // blk45 re-enters the free queue
}
```

`IndexWBCache::recover()` runs for CP13, which has 0 dirty buffers. It
performs no recovery work and calls no `commit_blk()`. blk45 remains in
`m_free_blk_q`.

**Evidence:**
```
[04/20/26 23:23:05.338] Detected unclean shutdown, prior cp=13 had to flush 0 nodes, recovering...
[04/20/26 23:23:05.338] Recovery processing begins
[04/20/26 23:23:05.338] Recovery processing Ends
[04/20/26 23:23:05.338] Index Recovery detected 0 nodes out of 0 as new/freed nodes
```

GDB on T3:
```
bit45 = (words[0] >> 45) & 1 = 0  → blk45 is FREE in allocator persistent bitmap (m_disk_bm)
```

### Condition 5: The btree tree on disk still references the node at blk45

The index tree structure (parent node `6755403736023126`) was also written to
disk as part of CP12 or an earlier CP. On T3, when any btree operation
traverses through this part of the tree, `read_buf()` loads the node at blk45
into the WBCache:

```cpp
// wb_cache.cpp:153-166
m_vdev->sync_read(blk45) → node = node_initializer(idx_buf)
m_cache.insert(node)  ← node with blkid=chunk21/blk45 is now in cache
```

### Condition 6: A new allocation picks blk45 — duplicate insert assertion fires

A subsequent insert triggers a split, calling `alloc_buf()`. The allocator
pops blk45 from `m_free_blk_q` (it is FREE per disk bitmap). `alloc_buf()`
creates a new node with blkid=chunk21/blk45 and calls `m_cache.insert()`:

```cpp
// wb_cache.cpp:116-117
bool done = m_cache.insert(node);
HS_REL_ASSERT_EQ(done, true, "Unable to add alloc'd node to cache...");
// ↑ done == false because the old node at blk45 is already in cache → CRASH
```

---

## Trigger Conditions

All of the following must occur simultaneously:

1. A CP flush begins and writes at least one **new node's** data page to disk
2. The process is killed **after** the new node's data page is flushed but
   **before** the vdev CP flush (allocator bitmap persistence) completes
3. The process restarts, correctly recovers the new node (calls `commit_blk()`),
   updating `m_disk_bm` in memory
4. The process is killed **again** before the vdev CP flush completes —
   specifically before `BitmapBlkAllocator::cp_flush()` writes `m_disk_bm` to
   physical storage
5. On the third restart, the index CP journal is empty (0 dirty buffers in the
   new CP), so no `commit_blk()` is called for the block recovered in step 3

This is a **double-crash** scenario. The crash window in step 4 is any time
between T2 recovery completion and the next successful vdev CP flush.

---

## Persistence Domain Mismatch (Core Structural Issue)

The bug exposes a durability gap between two independent persistence domains:

| Domain | What it tracks | Persisted by |
|--------|---------------|--------------|
| Index CP journal (meta blk) | Which blkids are new/freed per CP | `IndexWBCache::async_cp_flush()` |
| Allocator bitmap (`m_disk_bm`) | Which blocks are allocated | `BitmapBlkAllocator::cp_flush()` via vdev CP |

After index recovery calls `commit_blk()`, the update lives only in `m_disk_bm`
(in-memory). It is **not** recorded in the index CP journal (that journal is
already consumed) and **not** yet written to the allocator's physical bitmap.
A crash in this window makes the update permanently invisible to any future
recovery.

---

## Skipped Conditions

**SKIP — Condition 3 exact crash timing:** We cannot confirm the exact moment
T2 was killed relative to any vdev CP activity. The T2 log shows CP13 had 0
dirty buffers, which is consistent with a very early kill (immediately after
index recovery completed), but we cannot rule out that CP13 was partially in
flight when the kill occurred.

**Impact on fix:** The fix must close the durability gap unconditionally — not
just for the narrow "killed immediately after recovery" case. The SKIP does not
change the fix direction.

---

## Fix Direction

**Root cause fix:** After `IndexWBCache::recover()` finishes committing all
recovered blkids via `commit_blk()`, force a synchronous flush of the allocator
bitmap to physical storage before returning. This ensures that blkids committed
during index recovery are durably recorded in the allocator's persistent bitmap
before normal operations (and any further crash) can occur.

Concretely, add a synchronous allocator bitmap flush at the end of
`IndexWBCache::recover()`, just before `m_vdev->recovery_completed()` is
called (`wb_cache.cpp:761`). This makes blk45=USED durable on T2, so T3's
bitmap load would correctly show blk45=USED and never enqueue it into
`m_free_blk_q`.

**Defensive fix (independent):** In `IndexWBCache::alloc_buf()`, when
`m_cache.insert()` returns false, treat it as a fatal allocator inconsistency
with a more actionable error message (e.g., print the conflicting blkid and the
existing cache entry's CP metadata) before asserting, to aid future diagnosis.
