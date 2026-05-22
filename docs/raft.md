# RAFT Replication Log Management

This document provides a detailed explanation of RAFT log management in HomeStore, covering the architecture, append flow, compact/truncate flow, and rollback flow. It serves as a reference for code understanding and issue analysis.

## Architecture Overview

### Component Stack and Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│ NuRaft (nuraft library)                                     │
│ - Manages RAFT consensus protocol                           │
│ - Calls log_store interface to persist/retrieve logs        │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ nuraft::log_store interface
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ ReplLogStore                                                │
│ - Implements nuraft::log_store interface                    │
│ - Manages repl_req lifecycle and LSN mapping                │
│ - Coordinates data channel and raft channel                 │
│ - Inherits from HomeRaftLogStore                            │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Delegates to parent class
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ HomeRaftLogStore                                            │
│ - Converts between RAFT LSN and LogStore LSN               │
│ - Serializes/deserializes nuraft log entries               │
│ - Caches log entries for fast access                        │
│ - Contains m_log_store (HomeLogStore)                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Uses HomeLogStore for storage
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ HomeLogStore                                                │
│ - Manages log sequence numbers (LSNs)                       │
│ - Tracks logdev_key for each LSN                            │
│ - Maintains truncation boundaries                           │
│ - One logstore per RAFT log                                 │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Writes to LogDev
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ LogDev                                                      │
│ - Physical log device managing actual disk writes           │
│ - Batches multiple log records into LogGroups               │
│ - Assigns logdev_key {log_idx, dev_offset} to each record  │
│ - Currently: one LogDev per HomeLogStore (1:1 mapping)      │
│ - Performs physical truncation on journal vdev              │
└─────────────────────────────────────────────────────────────┘
```

### LSN and Key Concepts

#### LSN (Log Sequence Number) Mapping
- **RAFT LSN** (repl_lsn): Log index in RAFT, starts from 1
- **LogStore LSN** (store_lsn): Sequence number in HomeLogStore = `repl_lsn - 1`
- **Conversion formula**: `store_lsn = repl_lsn - 1`
- **Example**: RAFT LSN 100 → LogStore LSN 99

#### LogDev Key Structure
```cpp
struct logdev_key {
    logid_t idx;        // Log index in LogDev
    off_t dev_offset;   // Physical offset on journal device
}
```
- Each log written to LogDev gets a unique logdev_key
- The `idx` is assigned sequentially by LogDev, independent of store_lsn
- The `dev_offset` is the actual byte offset on the physical journal device

#### Replication Request (repl_req)
Represents a single write operation in the replication pipeline:
- **Contents**: user header, key, data buffer, LSN
- **State tracking**: DATA_WRITTEN, LOG_FLUSHED, etc.
- **Creation timing**:
  - **Leader**: Created in `async_alloc_write` before proposing to RAFT
  - **Follower**: Created when first receiving the entry, via two possible paths:
    - **RAFT channel**: In `ReplLogStore::append` when receiving RAFT log entry
    - **Data channel**: In `on_push_data_received` when receiving actual data
    - Whichever arrives first creates the repl_req; the other finds it already exists

---

## Append Flow

The append flow describes how a write operation is replicated and persisted. It involves two parallel channels:
- **Data Channel**: Transfers actual user data from leader to followers
- **RAFT Channel**: Transfers RAFT log entries for consensus

### High-Level Flow

```
Leader:
  1. async_alloc_write (allocate blocks, write data)
  2. raft_server()->append_entries_ext() (propose to RAFT)
  3. RAFT replicates log entries to all members

All Members (Leader & Followers):
  4. ReplLogStore::append (receive log entry)
  5. HomeRaftLogStore::append (add to logstore in-memory)
  6. LogDev::append_async (add to pending flush queue)
  7. ReplLogStore::end_of_append_batch (flush to disk)
  8. LogDev::flush (physical write)
```

---

### Step 1: Leader Proposes to RAFT

**Entry Point**: `RaftReplDev::async_alloc_write`

**What happens**:
1. Leader receives write request with header, key, and data
2. Allocates blocks from data service
3. Writes actual data to allocated blocks
4. Creates `repl_req` to track this write operation
5. Serializes write info into RAFT log entry
6. Calls `raft_server()->append_entries_ext()` to propose entry to RAFT

**Key Point**: Leader writes data BEFORE proposing to RAFT. When log is flushed later, data is already on disk.

**Output**: RAFT begins replicating this log entry to followers

---

### Step 2: RAFT Replicates to All Members

**What happens**:
- NuRaft library handles replication via network
- All members (including leader) receive log entries via `append()` callback
- RAFT batches multiple log entries together for efficiency

**Key Point**: From this point on, leader and followers follow the same append path through logstore.

---

### Step 3: Receive Log Entry in ReplLogStore

**Entry Point**: `ReplLogStore::append`

**What happens**:
1. Check if entry is app_log (skip config entries)
2. **Create or retrieve repl_req**:
   - **For Leader**: repl_req already exists from Step 1
   - **For Follower**: Create repl_req now (first time seeing this entry)
     - Can be triggered by **RAFT channel** (this function) OR **data channel** (`on_push_data_received`)
     - Whichever arrives first creates the repl_req
3. Link the LSN to repl_req for future lookup
4. Delegate to parent class `HomeRaftLogStore::append`

**Output**: Returns RAFT LSN (repl_lsn)

---

### Step 4: Add to LogStore (In-Memory)

**Entry Point**: `HomeRaftLogStore::append`

**What happens**:
1. Serialize nuraft log entry to buffer format:
   ```
   [8 bytes: term][1 byte: log_val_type][N bytes: user data]
   ```
2. Convert RAFT LSN to LogStore LSN: `store_lsn = repl_lsn - 1`
3. Call `HomeLogStore::append_async` with serialized buffer
4. Cache the log entry at position `lsn % cache_size` for fast access
5. Return RAFT LSN

**Key Point**: Log is still in MEMORY, not on disk yet.

**Output**: Log entry added to logstore's in-memory pending queue

---

### Step 5: Enqueue to LogDev (In-Memory)

**Entry Point**: `LogDev::append_async`

**What happens**:
1. Create `log_record` structure:
   ```cpp
   log_record {
       store_id: which logstore this belongs to
       seq_num: logstore sequence number (from HomeLogStore)
       data: serialized log entry
       context: pointer to logstore_req
   }
   ```
2. Assign a unique log_idx (LogDev's internal sequence number)
3. Add record to `m_log_records` stream tracker (in-memory queue)
4. Increment `m_pending_flush_size` by data size
5. Return the seq_num to caller

**Two Index Values**:
- **log_idx returned by write_at inside this function**: LogDev's internal index, **NOT USED** 
- **seq_num in log_record**: This is the logstore's LSN (from `m_next_lsn`), which is the actual LSN used

**Key Point**: Multiple logstores can share one LogDev. Each record is tagged with store_id.

**Output**: log_idx assigned, waiting for batch flush

---

### Step 6: End of Append Batch - Coordination

**Entry Point**: `ReplLogStore::end_of_append_batch`

This is called by NuRaft after appending a batch of entries (RAFT channel callback).

**What happens**:

1. **Separate requests into two groups**:
   - **Proposer requests** (leader's own writes): Data already written in Step 1
   - **Non-proposer requests** (follower's received writes): Data may not be written yet

2. **For non-proposer requests (followers)**:
   - Call `notify_after_data_written()` to check if data is written
   - Future completes when data is confirmed written to data service
   - **Critical**: MUST ensure data write completion before log flush

3. **Flush logs to disk**:
   - Call `HomeRaftLogStore::end_of_append_batch`
   - This triggers actual disk flush

4. **Mark completion**:
   - Mark all repl_reqs as `LOG_FLUSHED`
   - Set `is_volatile = false` on all repl_reqs

**Why wait for data?**
If logs are flushed before data is written, upon recovery we'd replay logs but data would be missing, causing data inconsistency.

**Output**: Logs are now durable on disk, data is written

---

### Step 7: Flush LogStore

**Entry Point**: `HomeRaftLogStore::end_of_append_batch`

**What happens**:
1. Calculate end LSN: `end_lsn = store_lsn(start + count - 1)`
2. Call `m_log_store->flush(end_lsn)`
3. Update `m_last_durable_lsn = end_lsn`

**Entry Point**: `HomeLogStore::flush`

**What happens**:
- Delegates to LogDev to perform actual flush
- No-op if already flushed

---

### Step 8: Physical Flush to Device

**Entry Point**: `LogDev::flush`

This is where logs physically hit the disk.

**What happens**:

1. **Gather pending records**:
   - Get all log records from `m_log_records` between `m_last_flush_idx` and current `m_log_idx`
   - These are the in-memory log records waiting to be flushed

2. **Create LogGroup**:
   - LogGroup is the unit of physical write
   - Contains header, multiple log records, inline data, OOB data, and footer
   - Format (based on code comments in log_dev.hpp):
     ```
     [LogGroup Header: #records, oob_area_offset, inline_area_offset, ...]
     [Record 1: size, offset, is_inlined, store_seq_num, store_id]
     [Record 2: size, offset, is_inlined, store_seq_num, store_id]
     ...
     [Inline Data Area: data for small records]
     [OOB (Out-of-Band) Data Area: data for large records]
     [Footer: magic, start_log_idx]
     ```

   **Inline vs OOB**:
   - Small records: Data stored in inline area
   - Large records: Data stored in OOB area
   - Record's `is_inlined` flag indicates which area to read from

3. **Allocate space on journal vdev**:
   - Call `m_vdev_jd->alloc_next_append_blk(total_size)`
   - Returns physical offset on journal device
   - Store this in `lg->m_log_dev_offset`

4. **Assign logdev_key to each record**:
   **All records in same LogGroup share the same dev_offset**
     ```cpp
     logdev_key {
         idx: unique log_idx for this record (from LogDev sequence)
         dev_offset: lg->m_log_dev_offset (SAME for entire LogGroup) 
     }
     ```

5. **Write to physical device**:
   - Prepare iovecs (scatter-gather list) from LogGroup
   - Call `m_vdev_jd->sync_pwritev(iovecs, offset)`
   - This is a synchronous write ensuring data is on disk

6. **Update tracking**:
   - Update `m_last_flush_idx` to last flushed log_idx
   - Update `m_last_flush_ld_key` with the logdev_key of last flushed log
   - Decrement `m_pending_flush_size`

7. **Notify completion**:
   - For each log record in LogGroup:
     - Find corresponding HomeLogStore via store_id
     - Call logstore's `on_write_completion(req, logdev_key, flush_ld_key)`
     - LogStore updates its in-memory mapping: `seq_num → {logdev_key, trunc_key}`
     - Call user's completion callback

**Key Points**:
- Multiple log records are batched into one LogGroup for efficiency
- Each log gets a unique logdev_key for later retrieval
- Writes are synchronous (blocking) to ensure durability

**Output**: Logs are now physically durable on journal device

---

### Follower Data Channel Flow

Followers receive data via data channel in parallel with raft channel.

**Entry Point**: `RaftReplDev::on_push_data_received`

**What happens**:
1. Leader calls `push_data_to_all_followers` after Step 1
2. Follower receives push data RPC
3. Create or retrieve repl_req (may already exist from RAFT channel)
4. Save pushed data buffer in repl_req
5. Write data to data service: `data_service().async_write()`
6. Mark repl_req as `DATA_WRITTEN` and fulfill promise
7. If RAFT channel is waiting in Step 6, it now proceeds

**Race Handling**:
- **Data arrives first**: Data written, RAFT channel finds data ready
- **RAFT arrives first**: RAFT channel waits for data via future
- Either order works correctly

---

## Compact/Truncate Flow

Compact/truncate reclaims space from old logs that have been snapshotted and are no longer needed for recovery or replication.

### Trigger and Context

**When it happens**:
- RAFT creates snapshot periodically
- After snapshot completes, RAFT calls `compact(upto_lsn)` to remove old logs
- Manually triggered via `trigger_snapshot_creation`

**Key Constraint - truncation_upper_limit**:
- Calculated by leader based on all followers' replication progress
- Definition: `max LSN that can be safely truncated without affecting any follower`
- Computed in `RaftReplDev::propose_truncate_boundary`:
  ```
  minimum_repl_idx = min(follower1_idx, follower2_idx, ...)
  truncation_upper_limit = max(commit_lsn - reserve_threshold, minimum_repl_idx)
  ```
- Ensures no follower needs logs we're about to delete

---

### Step 1: Compute Effective Compact LSN

**Entry Point**: `ReplLogStore::compact`

**What happens**:
1. RAFT wants to compact up to `compact_upto_lsn` (from snapshot LSN)
2. Get current `truncation_upper_limit` from ReplDev
3. Compute safe compact point:
   ```cpp
   effective_compact_lsn = min(compact_upto_lsn, truncation_upper_limit)
   ```
4. Call `m_rd.on_compact(effective_compact_lsn)`:
   - Updates ReplDev's `m_compact_lsn` to effective_compact_lsn
   - This value will be persisted later during checkpoint flush
5. Proceed to compact LogStore: `HomeRaftLogStore::compact(effective_compact_lsn)`

**Why limit by truncation_upper_limit?**
- RAFT snapshot may be at LSN 10000
- But a slow follower may only be at LSN 8000
- We can't truncate logs beyond 8000 or follower can't catch up
- So effective_compact_lsn = min(10000, 8000) = 8000

**Output**: Safe LSN determined for truncation

---

### Step 2: Compact in HomeRaftLogStore

**Entry Point**: `HomeRaftLogStore::compact`

**What happens**:
1. Validate compact_lsn:
   - Check if `compact_lsn > current_max_lsn`
   - If true, log a warning (may indicate holes in log)
   - Still proceed with truncation
2. Convert RAFT LSN to LogStore LSN:
   ```cpp
   store_lsn = compact_lsn - 1
   ```
3. Call `m_log_store->truncate(store_lsn, false)`:
   - `false` means perform physical truncation (not in-memory only)

**Output**: Request to truncate logstore up to store_lsn

---

### Step 3: Truncate LogStore (In-Memory)

**Entry Point**: `HomeLogStore::truncate`

**What happens**:

1. **Flush any pending logs first**:
   - Call `flush()` to ensure all in-memory logs are on disk
   - This prevents truncating logs that aren't yet durable

2. **Determine truncation key**:
   - If `upto_lsn <= m_tail_lsn` (normal case):
     - Get record at `upto_lsn`: `rec = m_records.at(upto_lsn)`
     - Extract `m_trunc_ld_key = rec.m_trunc_key`
     - This logdev_key marks the truncation boundary
   - If `upto_lsn > m_tail_lsn` (baseline resync case):
     - Update `m_tail_lsn` and `m_next_lsn` to upto_lsn
     - Insert empty record at upto_lsn
     - This handles cases where snapshot LSN exceeds current logs

3. **Truncate in-memory records**:
   - Call `m_records.truncate(upto_lsn)`
   - Removes all records with `seq_num <= upto_lsn` from memory
   - These LSNs are now invalid for future reads

4. **Update start LSN**:
   - Set `m_start_lsn = upto_lsn + 1`
   - Future log reads/writes must be >= upto_lsn + 1

5. **Trigger physical truncation**:
   - If `!in_memory_truncate_only`: call `m_logdev->truncate()`
   - This propagates truncation to LogDev layer

**Key State**:
- `m_start_lsn`: First valid LSN in logstore (upto_lsn + 1)
- `m_tail_lsn`: Last written LSN
- `m_trunc_ld_key`: LogDev key for truncation boundary

**Output**: In-memory state updated, physical truncation triggered

---

### Step 4: Truncate LogDev (Physical)

**Entry Point**: `LogDev::truncate`

**What happens**:

1. **Lock order** (important for avoiding deadlock):
   - Acquire flush_guard (prevents concurrent flush)
   - Acquire store_map read lock (protects logstore map)
   - Acquire meta_mutex (protects metadata updates)

2. **Find minimum safe truncation point across all logstores**:
   ```cpp
   logdev_key min_safe_ld_key = out_of_bound;

   for each logstore on this logdev:
       (trunc_lsn, trunc_ld_key, tail_lsn) = logstore->truncate_info()

       // Update logstore superblock with new start LSN
       logstore_sb.m_first_seq_num = trunc_lsn + 1
       persist(logstore_sb)

       // Track minimum across all stores
       if trunc_ld_key.idx < min_safe_ld_key.idx:
           min_safe_ld_key = trunc_ld_key
   ```

   **Why find minimum?**
   - One LogDev can serve multiple logstores (though currently 1:1)
   - Can only truncate up to the slowest logstore's boundary
   - Ensures no logstore loses its needed data

3. **Handle edge cases**:
   - If all logstores are empty: `min_safe_ld_key = m_last_flush_ld_key`
   - If no truncation needed (min_safe_ld_key <= m_last_truncate_idx):
     - Still persist logstore superblocks
     - Return 0 (no space reclaimed)

4. **Physical truncation on journal vdev**:
   - Call `m_vdev_jd->truncate(min_safe_ld_key.dev_offset)`
   - Reclaims space on journal device up to this offset
   - All data before this offset is now freed

5. **Update LogDev metadata**:
   - Set new `start_dev_offset = min_safe_ld_key.dev_offset`
   - Set new `start_log_idx = min_safe_ld_key.idx`
   - Persist metadata via `m_logdev_meta.persist()`
   - Update `m_last_truncate_idx = min_safe_ld_key.idx`

6. **Return reclaimed records count**:
   ```cpp
   num_records = min_safe_ld_key.idx - m_last_truncate_idx
   ```

**LogDev Metadata Persistence**:
The truncation info is stored in logdev superblock:
```cpp
struct logdev_superblk {
    start_dev_offset: physical offset to start reading from
    key_idx: log_idx to start reading from
    ...
}
```

During recovery (`LogDev::do_load`), logs are read from this offset forward.

**Output**: Physical space reclaimed on journal device

---

### Step 5: Persist Compact LSN in Checkpoint

**Entry Point**: `RaftReplDev::cp_flush`

This happens during periodic checkpoint flush.

**What happens**:
1. Read current `m_compact_lsn` from ReplDev (updated in Step 1)
2. Write to repl dev superblock:
   ```cpp
   m_rd_sb->compact_lsn = m_compact_lsn.load()
   m_rd_sb.write()
   ```
3. Superblock is persisted to meta service

**Why persist?**
- On recovery, need to know what LSN was compacted to
- Determines safe starting point for log replay
- Prevents re-processing already-snapshotted logs

**Checkpoint Flush Trigger**:
- Periodic: triggered by CP manager
- Manual: via `trigger_snapshot_creation`
- Before snapshot: to ensure clean state

**Output**: Compact LSN durable on disk

---

### Complete Flow Summary

```
Step 1: ReplLogStore::compact
   - Limit by truncation_upper_limit
   - effective_compact_lsn = min(snapshot_lsn, truncation_upper_limit)
   - Update m_compact_lsn in ReplDev

Step 2: HomeRaftLogStore::compact
   - Convert repl_lsn to store_lsn

Step 3: HomeLogStore::truncate
   - Flush pending logs
   - Get m_trunc_ld_key from record
   - Truncate in-memory records
   - Update m_start_lsn

Step 4: LogDev::truncate
   - Find min truncation point across all logstores
   - Persist logstore superblocks
   - Truncate journal vdev physically
   - Persist logdev metadata

Step 5: RaftReplDev::cp_flush (during checkpoint)
   - Persist m_compact_lsn to repl dev superblock
```

---

## Rollback Flow

Rollback handles the case when a follower (or old leader) has uncommitted logs that conflict with the new leader's logs. This happens during leader elections.

### When Rollback Occurs

**Scenario**:
1. Old leader appended logs at LSN 100-105 but crashed before committing
2. New leader elected, has committed logs only up to LSN 99
3. New leader sends its own logs starting at LSN 100 (different from old leader's)
4. Follower (old leader) must rollback LSN 100-105 and accept new leader's logs

**NuRaft's Two-Phase Process**:
1. **Phase 1 - Rollback**: Rollback invalid logs in **reverse order** (105 → 104 → ... → 100)
2. **Phase 2 - Overwrite**: Write new logs in **forward order** (100 → 101 → ...)

---

### Phase 1: Rollback Invalid Logs (Reverse Order)

**Entry Point**: NuRaft calls `state_machine->rollback_ext()` for each log in reverse

**NuRaft's Logic**:
```cpp
// Rollback from last log backwards to conflict point
for (idx = my_last_log_idx; idx >= conflict_idx; idx--) {
    log_entry = log_store_->entry_at(idx)
    if (log_entry.type == app_log) {
        state_machine_->rollback_ext(idx, log_entry.buffer)
    } else if (log_entry.type == conf) {
        state_machine_->rollback_config(idx, config)
    }
}
```

**What happens**:
1. **For each log from 105 down to 100**:
   - Read the log entry from logstore
   - If app_log: Call `RaftStateMachine::rollback_ext(idx, buffer)`
   - If config_log: Call `RaftStateMachine::rollback_config(idx, config)`

2. **In RaftStateMachine::rollback_ext**:
   - Notify application layer via listener: `m_listener->on_pre_commit_rollback(lsn, ...)`
   - Application can undo any in-memory state changes
   - Does NOT remove logs from logstore yet

**Key Point**: This is a notification phase, allowing application to clean up state before logs are physically removed.

---

### Phase 2: Overwrite with New Logs (Forward Order)

After rollback phase completes, NuRaft proceeds to overwrite.

**Entry Point**: NuRaft calls `log_store_->write_at(index, entry)` for each new log

**NuRaft's Logic**:
```cpp
// Overwrite logs in forward order
while (log_idx < next_slot() && has_more_entries) {
    entry = new_entries[count]
    log_store_->write_at(log_idx, entry)

    if (entry.type == app_log) {
        state_machine_->pre_commit_ext(log_idx, entry.buffer)
    }

    log_idx++
    count++
}
```

**What happens**:
1. **For each new log from 100 onwards**:
   - Call `ReplLogStore::write_at(index, entry)`
   - Call `RaftStateMachine::pre_commit_ext()` to notify application

---

### Step 1: Write At Specific Index

**Entry Point**: `ReplLogStore::write_at`

**What happens**:
1. Check if entry is app_log (skip config entries)
2. Create or retrieve repl_req from journal entry
3. Link LSN to repl_req
4. Delegate to `HomeRaftLogStore::write_at(index, entry)`

**Key Point**: This is called for EACH log in the overwrite range (100, 101, 102, ...)

---

### Step 2: Rollback and Append Single Log

**Entry Point**: `HomeRaftLogStore::write_at` (called for EACH index)

**What happens**:

1. **Serialize new log entry**:
   - Format: `[term][type][data]`

2. **Rollback logstore to before this index**:
   - Call `m_log_store->rollback(to_store_lsn(index) - 1)`
   - Example: For index=100, rollback to store_lsn=99
   - This removes log at index 100 and all after it

3. **Reset durable LSN**:
   - Set `m_last_durable_lsn = -1`
   - Will be recalculated on next flush

4. **Append new log at this index**:
   - Call `m_log_store->append_async(new_data)`
   - Since we just rolled back to 99, next append is at 100

5. **Validate LSN matches**:
   - Assert `appended_lsn == index`

6. **Clear cache entries after this index**:
   - Remove cached entries with LSN > index
   - Prevents serving stale data

7. **Flush immediately**:
   - Call `end_of_append_batch(index, 1)`
   - Ensures new log is durable

**Why rollback before each write_at?**
- First write_at(100): Removes 100-105, writes new 100
- Second write_at(101): Removes 101 onwards (already gone), writes new 101
- Ensures each new log is written cleanly

---

### Step 3: Rollback In-Memory State

**Entry Point**: `HomeLogStore::rollback`

**What happens**:

1. **Rollback next LSN**:
   - Set `m_next_lsn = upto_lsn + 1`
   - Example: rollback(99) sets m_next_lsn = 100

2. **Truncate in-memory records**:
   - Call `m_records.truncate(upto_lsn)`
   - Removes all records with `seq_num > upto_lsn`

3. **Update tail LSN** (if needed):
   - Set `m_tail_lsn = upto_lsn`

**Key Point - No Physical Truncation**:
- Does NOT call `m_logdev->truncate()`
- Old logs remain on disk but are logically invalid
- Will be overwritten when new logs are appended
- Physical space reclaimed during next compact

**Why no physical truncation?**
1. **Efficiency**: Rollback is rare, no need for expensive truncation
2. **Simplicity**: Just overwrite in-place
3. **Safety**: Old data remains until overwritten

---

### Complete Rollback Flow Example

```
Initial State (Old Leader):
   RAFT LSN:     97  98  99  100 101 102 103 104 105
   Committed:    ✓   ✓   ✓   ✗   ✗   ✗   ✗   ✗   ✗

Leader Change → New Leader has different logs at 100-102

Phase 1 - NuRaft Rollback (Reverse Order):
   rollback_ext(105) → Notify app to undo state for 105
   rollback_ext(104) → Notify app to undo state for 104
   rollback_ext(103) → Notify app to undo state for 103
   rollback_ext(102) → Notify app to undo state for 102
   rollback_ext(101) → Notify app to undo state for 101
   rollback_ext(100) → Notify app to undo state for 100

Phase 2 - NuRaft Overwrite (Forward Order):
   write_at(100, new_entry_100):
      → rollback(99) removes 100-105 from logstore
      → append new log 100
      → flush

   write_at(101, new_entry_101):
      → rollback(100) removes 101 onwards (already gone)
      → append new log 101
      → flush

   write_at(102, new_entry_102):
      → rollback(101) removes 102 onwards (already gone)
      → append new log 102
      → flush

Final State:
   RAFT LSN:     97  98  99  100 101 102
   Committed:    ✓   ✓   ✓   ✗   ✗   ✗   (will commit later)

   Logs 100-102 now contain NEW data from new leader
   Old logs 100-105 are orphaned on disk
```

---

### Rollback vs Compact

| Aspect | Rollback | Compact |
|--------|----------|---------|
| **When** | Leader change, uncommitted logs | After snapshot, committed logs |
| **Scope** | From rollback point to end | From start to compact point |
| **Physical** | No disk truncation | Physical disk truncation |
| **Data** | Overwritten in-place | Reclaimed via truncate |
| **Frequency** | Rare (only on failures) | Regular (after snapshots) |

---

## Recovery Flow

Recovery rebuilds the in-memory state of RAFT logs from persistent storage after restart.

### Overview

```
Restart → LogDev::do_load() → Read logs from disk →
on_log_found callbacks → Rebuild logstore mappings →
RAFT replay → Rebuild state machine
```

---

### Step 1: Load LogDev Metadata

**Entry Point**: `LogDev::start` (when format=false)

**What happens**:
1. Read logdev superblock from meta service:
   ```cpp
   struct logdev_superblk {
       start_dev_offset: where to start reading logs
       key_idx: log_idx to start from
       num_stores: how many logstores on this logdev
       ...
   }
   ```

2. Read logstore superblocks for each store:
   ```cpp
   struct logstore_superblk {
       m_first_seq_num: first valid LSN for this logstore
   }
   ```

3. Initialize LogDev state:
   - Set `m_log_idx = key_idx` (resume log index sequence)
   - Prepare to read from `start_dev_offset`

**Output**: LogDev knows where to start reading

---

### Step 2: Scan and Parse Logs

**Entry Point**: `LogDev::do_load`

**What happens**:

1. **Create log stream reader**:
   - Start reading from `start_dev_offset`
   - Uses `log_stream_reader` to parse LogGroups

2. **Read LogGroup by LogGroup**:
   ```cpp
   while not end of device:
       LogGroup = read_next_log_group()
       validate_header(LogGroup.header)
       for each record in LogGroup:
           dispatch to correct logstore
   ```

3. **Validate LogGroup**:
   - Check magic number: `LOG_GROUP_HDR_MAGIC`
   - Verify CRC matches
   - Check prev_grp_crc links to last group

4. **Extract log records**:
   - Parse each `serialized_log_record` from LogGroup
   - Structure:
     ```cpp
     serialized_log_record {
         size: size of log data
         offset: offset within LogGroup
         is_inlined: whether data is inline or OOB
         store_seq_num: logstore LSN
         store_id: which logstore this belongs to
     }
     ```

5. **Call logstore callback**:
   - For each record: `on_logfound(store_id, seq_num, logdev_key, flush_ld_key, data, ...)`
   - Passes to correct HomeLogStore based on store_id

**Key Points**:
- Reads sequentially from start_dev_offset to end
- Stops at first invalid LogGroup (end of valid logs)
- Handles inline vs OOB data within LogGroup

**Output**: All log records dispatched to logstores

---

### Step 3: Rebuild HomeLogStore State

**Entry Point**: `HomeLogStore::on_log_found`

**What happens**:

1. **Filter by start LSN**:
   - If `seq_num < m_start_lsn`: skip this record
   - `m_start_lsn` comes from logstore superblock (after truncation)

2. **Update tail LSN**:
   - If `seq_num > m_tail_lsn`: update `m_tail_lsn = seq_num`
   - Track highest LSN seen

3. **Determine truncation key**:
   ```cpp
   if seq_num > m_tail_lsn:
       trunc_key = flush_ld_key  // This is new tail
   else:
       trunc_key = m_records.at(m_tail_lsn).m_trunc_key  // Inherit from current tail
   ```

4. **Create logstore record**:
   - Build in-memory record:
     ```cpp
     logstore_record {
         m_dev_key: logdev_key (where to read this log)
         m_trunc_key: logdev_key (truncation boundary)
     }
     ```
   - Insert into `m_records` at seq_num

5. **Update next LSN**:
   - Set `m_next_lsn = max(m_next_lsn, seq_num + 1)`
   - Ensures next append doesn't conflict

6. **Call user callback** (if registered):
   - User-provided `log_found_cb(seq_num, data, context)`
   - Allows upper layer to examine logs during recovery

**Output**: HomeLogStore has complete `seq_num → logdev_key` mapping

---


### Recovery Example

**Persistent State**:
```
LogDev Superblock:
   start_dev_offset: 1024
   key_idx: 50

LogStore Superblock:
   m_first_seq_num: 100 (after last truncation)

Journal Device:
   [Offset 1024]: LogGroup containing records 100-105
   [Offset 2048]: LogGroup containing records 106-110
   ...
```

**Recovery Steps**:
1. Read from offset 1024
2. Parse LogGroups
3. Extract records 100-110 (skip <100)
4. Rebuild mapping: 100→{idx:50, off:1024}, 101→{idx:51, off:1056}, ...
5. RAFT replays logs 100-110
6. Resume at LSN 111

---

## Summary

### Append Flow
**Purpose**: Persist new write operations to disk
**Key Steps**:
1. Leader writes data to data service
2. Leader proposes to RAFT via `raft_server()->append_entries_ext()`
3. RAFT replicates to all members via `append()`
4. All members persist to LogStore (in-memory)
5. LogDev batches records into LogGroup
6. `end_of_append_batch` triggers flush to disk
7. LogDev physically writes to journal device

**Critical Path**: Data must be written before log flush (for followers)

---

### Compact/Truncate Flow
**Purpose**: Reclaim space from old snapshotted logs
**Key Steps**:
1. RAFT creates snapshot at LSN X
2. Compute `effective_compact_lsn = min(X, truncation_upper_limit)`
3. Truncate LogStore in-memory (update start_lsn)
4. LogDev finds minimum truncation point across all stores
5. Physically truncate journal device
6. Persist compact_lsn in checkpoint

**Key Constraint**: `truncation_upper_limit` ensures no follower is left behind

---

### Rollback Flow
**Purpose**: Resolve log conflicts during leader changes
**Key Steps**:
1. New leader sends `write_at(index, new_entry)`
2. Rollback LogStore to index-1 (in-memory only)
3. Append new log at same index
4. Immediately flush to disk
5. Old logs orphaned, reclaimed during next compact

**Key Point**: No physical truncation, just in-memory rollback + overwrite

---

### Recovery Flow
**Purpose**: Rebuild state after restart
**Key Steps**:
1. Read LogDev metadata (start_dev_offset, start_lsn)
2. Scan LogGroups from start_dev_offset
3. Dispatch records to logstores via callbacks
4. Rebuild `seq_num → logdev_key` mapping
5. RAFT replays logs from snapshot_lsn + 1
6. Resume normal operation

**Key Invariant**: `logstore_lsn = raft_lsn - 1`

---
