# RaftReplDev and Log Storage Architecture

This document explains the full layered architecture of HomeStore's log storage subsystem, and how `RaftReplDev` uses it. It covers the relationship between **Log VDev**, **LogDev**, **LogStore**, and **RaftReplDev**, as well as how multiple Placement Groups (PGs) share chunks on the Log VDev.

---

## 1. Layered Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  RaftReplDev  (one per Raft replication group / PG)                │
│   m_data_journal    : ReplLogStore                                 │
│   m_free_blks_journal: HomeLogStore  (timeline consistency only)   │
├────────────────────────────────────────────────────────────────────┤
│  ReplLogStore : HomeRaftLogStore : nuraft::log_store               │
│   Adapter between the NuRaft log_store interface and HomeStore     │
├────────────────────────────────────────────────────────────────────┤
│  HomeLogStore   (one logical LSN namespace, one logstore_id)       │
│   Multiple HomeLogStores can share a single LogDev                 │
├────────────────────────────────────────────────────────────────────┤
│  LogDev   (one per PG, one logdev_id)                              │
│   Group-commit engine, flush scheduler, multi-store manager        │
├────────────────────────────────────────────────────────────────────┤
│  JournalVirtualDev  (singleton, shared by ALL LogDevs)             │
│  JournalVirtualDev::Descriptor  (one per LogDev, exclusive window) │
│  Physical disk chunks  (dynamically allocated from global pool)    │
└────────────────────────────────────────────────────────────────────┘
```

### Key files

| Layer | Files |
|-------|-------|
| Log VDev | `src/lib/device/journal_vdev.hpp`, `src/lib/device/journal_vdev.cpp` |
| LogDev | `src/lib/logstore/log_dev.hpp`, `src/lib/logstore/log_dev.cpp` |
| LogStoreService | `src/include/homestore/logstore_service.hpp`, `src/lib/logstore/log_store_service.cpp` |
| HomeLogStore | `src/include/homestore/logstore/log_store.hpp`, `src/lib/logstore/log_store.cpp` |
| HomeRaftLogStore | `src/lib/replication/log_store/home_raft_log_store.h`, `.cpp` |
| ReplLogStore | `src/lib/replication/log_store/repl_log_store.h`, `.cpp` |
| RaftReplDev | `src/lib/replication/repl_dev/raft_repl_dev.h`, `.cpp` |

---

## 2. Layer-by-Layer Description

### 2.1 JournalVirtualDev (Log VDev)

`JournalVirtualDev` (`src/lib/device/journal_vdev.hpp`) is the **raw storage backend** for all log data. There is exactly **one** `JournalVirtualDev` instance per HomeStore process, held by `LogStoreService::m_logdev_vdev` and shared by all LogDevs.

It manages a **global chunk pool** (`m_chunk_pool`). Chunks are dynamically allocated from this pool on demand and returned to it when freed by truncation.

Each LogDev gets its own private **Descriptor** — a sliding-window view into the VDev:

```cpp
// src/lib/device/journal_vdev.hpp
struct JournalChunkPrivate {
    logdev_id_t logdev_id{0};   // which LogDev owns this chunk
    bool is_head{false};        // is it the head of the chain?
    uint64_t created_at{0};     // creation timestamp (for crash recovery)
    uint64_t end_of_chunk{0};   // where valid data ends in this chunk
    chunk_num_t next_chunk{0};  // next chunk in this LogDev's chain
};

struct Descriptor {
    logdev_id_t m_logdev_id;
    off_t m_data_start_offset;              // left boundary (advances on truncate)
    off_t m_end_offset;                     // right boundary (advances on new chunk)
    std::vector<shared<Chunk>> m_journal_chunks; // ordered chunk chain
    std::atomic<uint64_t> m_write_sz_in_total;
    uint64_t m_reserved_sz;                 // space reserved but not yet written
    uint64_t m_total_size;
};
```

**All offsets are monotonically increasing — they never wrap around.**

`tail_offset() = m_data_start_offset + m_write_sz_in_total + m_reserved_sz`

A **critical invariant**: a single LogGroup write must be smaller than one chunk size. Writes are therefore guaranteed not to span chunk boundaries:
```cpp
// journal_vdev.cpp:259
RELEASE_ASSERT_LT(sz, chunk_size, "Size requested greater than chunk size");
```

### 2.2 LogDev

`LogDev` (`src/lib/logstore/log_dev.hpp`) is a logical "log band" with its own `logdev_id`. Each PG owns one exclusive LogDev.

Key responsibilities:
- **Group commit**: batches multiple `log_record`s into a `LogGroup` and writes them atomically. At most 2 `LogGroup` objects exist simultaneously (`max_log_group = 2`).
- **Manages multiple HomeLogStores**: a LogDev can contain several stores, each with a distinct `logstore_id`.
- **Flush scheduling**: supports three modes — `INLINE`, `TIMER`, and `EXPLICIT` (Raft always uses `EXPLICIT`).
- **Owns a monotonic global log index** (`m_log_idx`) used to determine truncation safety.

```cpp
// log_dev.hpp (key members)
std::atomic<logid_t> m_log_idx{0};            // ever-increasing log index
std::shared_ptr<JournalVirtualDev> m_vdev;
shared<JournalVirtualDev::Descriptor> m_vdev_jd; // exclusive descriptor
std::unordered_map<logstore_id_t, logstore_info> m_id_logstore_map;
LogGroup m_log_group_pool[max_log_group];      // pool of 2 log groups
flush_mode_t m_flush_mode;
```

### 2.3 HomeLogStore

`HomeLogStore` (`src/include/homestore/logstore/log_store.hpp`) is an independent logical LSN namespace. Multiple stores can coexist within one LogDev.

Each store holds:
- `m_store_id`: its unique ID within the LogDev
- `m_logdev`: back-pointer to its LogDev
- `m_records`: a `StreamTracker` mapping `logstore_seq_num → logdev_key{idx, dev_offset}`
- `m_start_lsn`, `m_next_lsn`, `m_tail_lsn`

Write path:
```cpp
// log_store.cpp:64
auto ret = m_logdev->append_async(m_store_id, req->seq_num, req->data, ...);
```

Read path:
```cpp
// log_store.cpp:132-143
const auto record = m_records.at(seq_num);
const logdev_key ld_key = record.m_dev_key;
const auto b = m_logdev->read(ld_key);
```

### 2.4 LogStoreService

`LogStoreService` (`src/include/homestore/logstore_service.hpp`) is the global manager (singleton via `logstore_service()`). It holds the one `JournalVirtualDev` and a map of `logdev_id → LogDev`.

---

## 3. RaftReplDev and the Log Storage Stack

### 3.1 Ownership Structure

```
LogStoreService (singleton)
  ├── m_logdev_vdev : JournalVirtualDev         ← one global VDev
  └── m_id_logdev_map : { logdev_id → LogDev }
        └── LogDev  (one per PG)
              ├── m_vdev_jd : Descriptor          ← exclusive VDev window
              ├── m_id_logstore_map:
              │     ├── HomeLogStore  (data_journal,  append_mode=true)
              │     └── HomeLogStore  (free_blks_journal, append_mode=false, optional)
              └── m_logdev_meta : superblock (persists logstore_id list)

RaftReplDev  (one per Raft group)
  ├── m_data_journal : ReplLogStore
  │     └── HomeRaftLogStore
  │           └── m_log_store : HomeLogStore      ← same object as above
  └── m_free_blks_journal : HomeLogStore (optional, for timeline consistency)

m_rd_sb persists: { logdev_id, logstore_id }     ← used for recovery
```

### 3.2 Creation (New PG)

```cpp
// raft_repl_dev.cpp:71-86  (new RaftReplDev, load_existing=false)
m_data_journal = std::make_shared<ReplLogStore>(*this, *m_state_machine);
m_rd_sb->logdev_id   = m_data_journal->logdev_id();
m_rd_sb->logstore_id = m_data_journal->logstore_id();

// If timeline_consistent:
m_free_blks_journal = logstore_service().create_new_log_store(
    m_rd_sb->logdev_id, false /* append_mode */);
m_rd_sb->free_blks_journal_id = m_free_blks_journal->get_store_id();
```

Inside `HomeRaftLogStore` constructor:
```cpp
// home_raft_log_store.cpp:94-98
// logstore_id == UINT32_MAX means brand-new
m_logdev_id  = logstore_service().create_new_logdev(flush_mode_t::EXPLICIT);
m_log_store  = logstore_service().create_new_log_store(m_logdev_id, true /* append_mode */);
m_logstore_id = m_log_store->get_store_id();
```

Each Raft group gets its **own exclusive LogDev** with `flush_mode = EXPLICIT`.

### 3.3 Recovery (Existing PG)

```cpp
// raft_repl_dev.cpp:42-49
m_data_journal = std::make_shared<ReplLogStore>(
    *this, *m_state_machine,
    m_rd_sb->logdev_id, m_rd_sb->logstore_id,
    [this](logstore_seq_num_t lsn, log_buffer buf, void* key) { on_log_found(...); },
    [this](auto hs, auto lsn) { m_log_store_replay_done = true; ... });
```

Inside `HomeRaftLogStore` constructor:
```cpp
// home_raft_log_store.cpp:100-113
m_logdev_id   = logdev_id;
m_logstore_id = logstore_id;
logstore_service().open_logdev(m_logdev_id, flush_mode_t::EXPLICIT);
m_log_store_future = logstore_service()
    .open_log_store(m_logdev_id, logstore_id, true, log_found_cb, log_replay_done_cb)
    .thenValue([this](auto log_store) { m_log_store = std::move(log_store); });
```

### 3.4 LSN Offset Convention

Raft LSNs start from 1; HomeLogStore `seq_num` starts from 0. They differ by exactly 1:

```cpp
// home_raft_log_store.cpp:40-44
static constexpr logstore_seq_num_t to_store_lsn(uint64_t raft_lsn) {
    return static_cast<logstore_seq_num_t>(raft_lsn - 1);
}
// home_raft_log_store.h:258
static constexpr repl_lsn_t to_repl_lsn(store_lsn_t store_lsn) { return store_lsn + 1; }
```

### 3.5 Write Path (End to End)

```
NuRaft::append_entries()
  → ReplLogStore::append()               [repl_log_store.cpp:9]
      → m_sm.localize_journal_entry_finish()   // translate remote data pointers to local
      → HomeRaftLogStore::append()        [home_raft_log_store.cpp:164]
          → m_log_store->append_async()   // HomeLogStore (auto-assigns next LSN)
              → LogDev::append_async()    // assigns logid, stores to StreamTracker

  → ReplLogStore::end_of_append_batch()  [repl_log_store.cpp:41]
      → m_rd.notify_after_data_written() // wait for block data to be persisted
      → HomeRaftLogStore::end_of_append_batch()
          → m_log_store->flush()
              → LogDev::flush_under_guard()
                  → LogDev::flush()
                      → prepare_flush()   // pack pending log_records into LogGroup
                      → m_vdev_jd->alloc_next_append_blk()  // may trigger append_chunk()
                      → m_vdev_jd->sync_pwritev()            // write to physical disk
                      → on_flush_completion()                 // notify HomeLogStore callbacks
```

---

## 4. How 10 PGs Share 100 Chunks

Assume: 1 `JournalVirtualDev` with 100 chunks (e.g., 256 MB each), 10 PGs.

### 4.1 Chunk Allocation Principle

**Chunks are NOT pre-partitioned across PGs.** All 100 chunks start in the global `ChunkPool`. Each LogDev starts with zero chunks and acquires them on demand.

When a flush needs space and `tail_offset + write_size >= m_end_offset`:

```cpp
// journal_vdev.cpp:261-277  alloc_next_append_blk()
if ((tail_offset() + sz) >= m_end_offset) {
    append_chunk();   // dequeue one chunk from the global pool
}
```

```cpp
// journal_vdev.cpp:208-253  append_chunk()
auto new_chunk = m_vdev.m_chunk_pool->dequeue();  // take from global pool
m_total_size += new_chunk->size();
m_end_offset += new_chunk->size();

if (m_journal_chunks.empty()) {
    // First chunk: mark as head, stamp logdev_id
    new_chunk_private->is_head    = true;
    new_chunk_private->logdev_id  = m_logdev_id;
    new_chunk_private->end_of_chunk = chunk_size;
} else {
    // Subsequent chunk: link previous chunk's next_chunk pointer to this one
    // and record the actual end_of_chunk in the previous chunk
    last_chunk_private->next_chunk = new_chunk->chunk_id();
    last_chunk_private->end_of_chunk = offset_in_chunk;   // watermark
    m_vdev.update_chunk_private(last_chunk, ...);          // persist metadata
}
m_journal_chunks.push_back(new_chunk);
m_vdev.update_chunk_private(new_chunk, ...);
```

The chunk's private metadata is **persisted to disk immediately** each time ownership changes, enabling crash-safe recovery.

### 4.2 Chunk Exclusivity

A chunk belongs to exactly one entity at any moment:

| State | Owner |
|-------|-------|
| In `ChunkPool` | No LogDev — free for any |
| In a `Descriptor::m_journal_chunks` | That LogDev exclusively |

There is no sharing of chunks between LogDevs. The `logdev_id` field in `JournalChunkPrivate` records current ownership.

### 4.3 Physical Layout on Disk

Chunks from different LogDevs are **physically interleaved** on disk in no particular order. The logical ordering within a LogDev is maintained solely through the `next_chunk` linked-list pointers stored in each chunk's private metadata.

Example snapshot (chunks from 10 PGs scattered on disk):

```
Physical disk:
  chunk[0]  → ChunkPool (free)
  chunk[1]  → PG_2 (head, logdev=2, next=55)
  chunk[2]  → ChunkPool (free)
  chunk[3]  → PG_0 (head, logdev=0, next=17)
  chunk[4]  → ChunkPool (free)
  ...
  chunk[17] → PG_0 (logdev=0, next=42)
  chunk[23] → PG_1 (head, logdev=1, next=0, currently writing)
  chunk[42] → PG_0 (logdev=0, next=0, currently writing)
  chunk[55] → PG_2 (logdev=2, next=78)
  chunk[78] → PG_2 (logdev=2, next=0, currently writing)
  ...
  remaining → ChunkPool (free)
```

Each LogDev's virtual offset space is independent:

```
PG_0 Descriptor (virtual tape):
  ├── [0         .. chunk_size)    chunk[3]  (head)
  ├── [chunk_size .. 2*chunk_size) chunk[17]
  └── [2*chunk_sz .. 3*chunk_size) chunk[42] ← m_end_offset

  m_data_start_offset = 0          (no truncation yet)
  tail_offset()       ≈ 2*chunk_sz + bytes_written_in_chunk[42]
```

### 4.4 Flush Principle

Flush modes (set per LogDev):

| Mode | Trigger | Used by |
|------|---------|---------|
| `EXPLICIT` | Caller invokes `flush_under_guard()` | Raft (`end_of_append_batch`) |
| `INLINE` | After `append_async`, if `pending_size >= threshold` | Non-Raft stores |
| `TIMER` | Periodic timer checks if there is pending unflushed data | Non-Raft stores |

The `LogGroup` header carries a CRC and `prev_grp_crc` chain for integrity validation during recovery.

**Write alignment constraint**: each `LogGroup` flush must fit within a single chunk:
```cpp
RELEASE_ASSERT_LT(sz, chunk_size, "Size requested greater than chunk size");
```
If the remaining space in the current chunk cannot fit the next `LogGroup`, a new chunk is obtained first, then the write proceeds.

### 4.5 Truncate Principle

Truncation flows through all layers:

#### Step 1 — HomeLogStore layer (in-memory)
```cpp
// log_store.cpp:208-258
bool HomeLogStore::truncate(logstore_seq_num_t upto_lsn, bool in_memory_only) {
    m_records.truncate(upto_lsn);      // release in-memory StreamTracker entries
    m_start_lsn.store(upto_lsn + 1);
    if (!in_memory_only) m_logdev->truncate();
}
```

#### Step 2 — LogDev layer (find safe truncation point)
```cpp
// log_dev.cpp:631-665  LogDev::truncate()
logdev_key min_safe_ld_key = logdev_key::out_of_bound_ld_key();
for (auto& [store_id, store] : m_id_logstore_map) {
    auto [trunc_lsn, trunc_ld_key, tail_lsn] = store.log_store->truncate_info();
    if (trunc_ld_key.idx < min_safe_ld_key.idx) {
        min_safe_ld_key = trunc_ld_key;   // take the most conservative point
    }
}
// All stores (data_journal AND free_blks_journal) must agree before truncating.
m_vdev_jd->truncate(min_safe_ld_key.dev_offset);
m_logdev_meta.set_start_dev_offset(min_safe_ld_key.dev_offset, min_safe_ld_key.idx, ...);
```

#### Step 3 — Descriptor layer (release chunks)
```cpp
// journal_vdev.cpp:569-687  Descriptor::truncate(truncate_offset)
// 1. Find which chunk contains truncate_offset → this becomes the new head.
// 2. Persist is_head=true on the new head chunk (crash-safe: two heads may
//    exist transiently; recovery picks the one with higher created_at).
// 3. Walk the chunk list from the old head:
for (auto it = m_journal_chunks.begin(); ...) {
    if (cover_offset <= truncate_offset) {
        m_total_size -= chunk->size();
        m_journal_chunks.erase(it);
        m_vdev.release_chunk_to_pool(chunk);   // return to global pool
    }
}
// 4. Update m_data_start_offset = truncate_offset.
// 5. Reduce m_write_sz_in_total by size_to_truncate.
```

```cpp
// journal_vdev.cpp:156-166  release_chunk_to_pool()
*data = JournalChunkPrivate{};          // clear logdev_id and all metadata
update_chunk_private(chunk, data);      // persist the clear (crash-safe)
m_chunk_pool->enqueue(chunk);           // chunk is now available to any LogDev
```

**A chunk is released only when the truncation offset completely covers it.** The chunk containing the new `m_data_start_offset` is retained.

### 4.6 Recovery: Reconstructing Chunk Chains

At startup, `JournalVirtualDev::init()` reconstructs each LogDev's chunk chain from persisted metadata:

```cpp
// journal_vdev.cpp:72-138
void JournalVirtualDev::init() {
    // 1. Scan all chunks; build: logdev_id → head_chunk, chunk_id → chunk
    for (auto& [_, chunk] : m_all_chunks) {
        if (data->is_head) {
            // Among multiple is_head chunks for the same logdev_id,
            // pick the one with the highest created_at timestamp.
            logdev_head_map[logdev_id] = { chunk_id, created_at };
        }
    }

    // 2. For each logdev, follow next_chunk pointers to rebuild the ordered list.
    for (auto& [logdev_id, head] : logdev_head_map) {
        auto journal_desc = std::make_shared<Descriptor>(*this, logdev_id);
        auto chunk_num = head.chunk_num;
        while (chunk_num != 0) {
            journal_desc->m_journal_chunks.push_back(chunk_map[chunk_num]);
            chunk_num = data->next_chunk;
        }
    }

    // 3. Chunks not in any chain are orphans — remove them.
    for (auto& [_, chunk] : m_all_chunks) {
        if (!visited_chunks.count(chunk->chunk_id())) {
            remove_journal_chunks(orphan_chunks);
        }
    }
}
```

The **crash-safe two-head mechanism**: when truncation persists `is_head=true` on the new head before releasing old chunks, a crash leaves two chunks with `is_head=true`. Recovery resolves this by selecting the one with the **higher `created_at`** timestamp, treating the other as orphaned.

---

## 5. Log VDev Capacity and Chunk Allocation Internals

This section explains why `LogStoreService::create_vdev()` passes `vdev_size=0` and `num_chunks=0` for the Log VDev, how the system still knows the allocatable capacity, and where the actual allocation logic lives.

### 5.1 Why `vdev_size=0` and `num_chunks=0` still works

The Log VDev (`JournalVirtualDev`) is created with a *dynamic size* configuration (see `src/lib/logstore/log_store_service.cpp`, `LogStoreService::create_vdev()`): it does **not** pre-create a fixed number of chunks at format time.

Instead, it relies on an internal **ChunkPool** which can *create* new chunks on demand. The real upper bound is determined by the **physical device data region** and by available chunk-id / chunk-info slots.

### 5.2 Where chunks come from: `ChunkPool` producer thread

`JournalVirtualDev` constructs a `ChunkPool` (`src/lib/device/journal_vdev.cpp:47-57`). The pool maintains a background producer thread which creates chunks via `DeviceManager::create_chunk()` when the pool drops below a threshold.

Key code path:

- `ChunkPool::producer()` (`src/lib/device/device_manager.cpp:873-899`)
  - calls `DeviceManager::create_chunk(dev_type, vdev_id, chunk_size, private_data)`

### 5.3 Global allocation constraints

A new chunk can be created only if all of the following remain available:

1. **Global chunk-id space** (DeviceManager-level)
   - `DeviceManager::create_chunk()` reserves a new `chunk_id` from `m_chunk_id_bm`.
   - If there are no remaining IDs, it throws: `"System has no room for additional chunk"`.

2. **Per-PDev chunk-info slots** (PhysicalDev-level)
   - `PhysicalDev::create_chunk()` allocates a free *slot* in `m_chunk_info_slots`.
   - If there are no remaining slots, it throws: `"System has no room for additional chunk"`.

3. **Per-PDev free space in the data region** (PhysicalDev-level)
   - This is enforced by `PhysicalDev::find_next_chunk_area()`.

### 5.4 The physical device data region: `data_start_offset()` and `data_end_offset()`

Chunk data cannot be placed at arbitrary offsets on a physical device. It must be placed inside the device's **data region**, which excludes the HomeStore superblocks/metadata areas.

- The data start offset is computed during formatting in `DeviceManager::populate_pdev_info()` (`src/lib/device/device_manager.cpp:708-713`):
  - `pinfo.data_offset = round_up(first_block_offset + total_superblock_size, phys_page_size)`

- PhysicalDev exposes:
  - `data_start_offset() = m_pdev_info.data_offset` (`src/lib/device/physical_dev.hpp:226`)
  - `data_end_offset()` is the end of usable device space, adjusted if the superblock is mirrored in the footer (`src/lib/device/physical_dev.hpp:227-229`).

Because chunk data must not overlap the superblock areas, chunk allocation searches only inside `[data_start_offset, data_end_offset)`.

### 5.5 How free space is tracked: `m_chunk_data_area`

Each `PhysicalDev` maintains an in-memory interval set of allocated chunk data ranges:

- `PhysicalDev::m_chunk_data_area` (`src/lib/device/physical_dev.hpp:138`)

It is populated in two ways:

1. **Recovery path**: as existing chunks are loaded from the superblock, their occupied ranges are inserted
   - `PhysicalDev::load_chunks()` inserts `[chunk_start_offset, chunk_start_offset + chunk_size)` into `m_chunk_data_area` (`src/lib/device/physical_dev.cpp:400-403`).

2. **Creation path**: when a new chunk is created, the newly selected range is inserted
   - `PhysicalDev::populate_chunk_info()` inserts the chosen interval (`src/lib/device/physical_dev.cpp:448-453`).

Chunk deletion removes the corresponding range:
- `PhysicalDev::free_chunk_info()` erases the interval (`src/lib/device/physical_dev.cpp:464-467`).

### 5.6 The actual placement algorithm: `find_next_chunk_area()`

The physical placement decision is made by `PhysicalDev::find_next_chunk_area(size)` (`src/lib/device/physical_dev.cpp:474-485`). It performs a simple first-fit scan:

- Start with the candidate interval `[data_start_offset, data_start_offset + size)`.
- If it overlaps an existing allocated interval, move the candidate to start at `exist_ival.upper()`.
- If the candidate's upper bound exceeds `data_end_offset`, allocation fails with `"Physical dev has no room for additional chunk"`.

This means **chunks from different VDevs can be physically interleaved on the same physical device**, because all VDevs allocate from the same per-PDev data region and are only separated by chunk ownership metadata (`chunk_info.vdev_id`).

---

## 6. Summary of Key Properties

| Property | Behavior |
|----------|----------|
| Chunk allocation | Demand-driven from global pool; no pre-partitioning |
| Chunk exclusivity | A chunk belongs to exactly one LogDev at any time |
| Physical layout | Chunks from different LogDevs are interleaved on disk |
| Offset space | Per-LogDev, monotonically increasing, never wraps |
| Write boundary | A single LogGroup flush must fit within one chunk |
| Flush mode (Raft) | `EXPLICIT` — triggered by `end_of_append_batch` only |
| Truncation safety | Governed by the minimum `trunc_ld_key` across all LogStores in a LogDev |
| Chunk release | Only when truncation fully covers the chunk |
| Recovery | Per-chunk metadata (`logdev_id`, `next_chunk`, `is_head`, `created_at`) |
| Crash safety | New head persisted before old chunks released; two-head resolved by `created_at` |
