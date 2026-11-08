# CPU Cache with Write-Back

Also called: Dirty tracking, Warm cache

## Overview
CPU DRAM serves as a warm cache between SSD storage and GPU VRAM, maintaining an LRU cache over volume blocks with per-block dirty bits [S1, unofficial]. It decouples GPU residency from SSD write latency by staging evicted blocks in CPU memory before persisting them, turning frequent small updates into batched sequential appends [S1, unofficial]. This method builds on [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by adding a write-back policy that distinguishes between clean and dirty blocks, preventing unnecessary SSD writes for blocks that are still being actively reused [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of volume blocks currently resident in GPU VRAM ($R_{VRAM}$), a set of blocks to be accessed or evicted, and a CPU cache capacity ($C_{CPU}$).
*   **Output:** Updated CPU cache state, potentially flushed dirty blocks written to SSD patch segments.

**Data Structures:**
*   **CPU Cache Entry:** Contains the block data (voxel values/parameters), a metadata header, and a **dirty bit** (1 bit).
    *   `data`: The block's parameter values.
    *   `is_dirty`: Boolean flag. True if the block has been updated by GPU-side training since it was last persisted to SSD.
*   **LRU State:** A doubly-linked list or hash map maintaining the access order of all entries in the CPU cache.

**Algorithm Steps:**
1.  **Access/Insertion:** When a block is accessed or evicted from VRAM:
    *   If the block is already in the CPU cache, update its LRU position to the most recently used [S1, unofficial].
    *   If the block is new to the CPU cache:
        *   Determine its dirty status. A block is marked **dirty** if its parameters were updated by GPU-side training (e.g., gradient updates) [S1, unofficial].
        *   If the CPU cache is full, evict the least recently used (LRU) block [S1, unofficial].
        *   Insert the new block into the CPU cache with its dirty bit set appropriately.

2.  **Eviction from CPU Cache:** When a block is evicted from the CPU cache due to memory pressure:
    *   Check the `is_dirty` bit.
    *   If **clean**: Discard the data (it is already persisted or not needed).
    *   If **dirty**: Asynchronously flush the block to SSD patch segments [S1, unofficial]. This involves appending a new version of the block to the SSD, avoiding random in-place overwrites [S1, unofficial].

3.  **Consistency Barriers:** At explicit consistency points (e.g., checkpointing, shutdown), all dirty blocks remaining in the CPU cache are flushed to SSD [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class CPU_Cache_Entry:
    def __init__(self, block_data, is_dirty):
        self.data = block_data
        self.is_dirty = is_dirty
        self.last_access_time = current_time()

class CPU_Cache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.entries = {}  # Map: block_id -> CPU_Cache_Entry
        self.lru_order = []  # List of block_ids, oldest first

    def access(self, block_id, block_data, is_dirty):
        # Update LRU position
        if block_id in self.entries:
            self.entries[block_id].data = block_data
            self.entries[block_id].is_dirty = is_dirty
            self._move_to_front(block_id)
        else:
            # Evict if full
            if len(self.entries) >= self.capacity:
                self._evict_lru()
            # Insert new entry
            self.entries[block_id] = CPU_Cache_Entry(block_data, is_dirty)
            self.lru_order.append(block_id)

    def _evict_lru(self):
        if not self.lru_order:
            return
        # Get least recently used block
        evict_id = self.lru_order.pop(0)
        entry = self.entries.pop(evict_id)
        
        # Write-back if dirty
        if entry.is_dirty:
            self._flush_to_ssd(evict_id, entry.data)

    def _move_to_front(self, block_id):
        # Remove from current position and append to end (most recent)
        self.lru_order.remove(block_id)
        self.lru_order.append(block_id)

    def _flush_to_ssd(self, block_id, data):
        # Append data to SSD patch segment
        # This is an asynchronous operation in practice
        pass

    def flush_all(self):
        # Consistency barrier: flush all remaining dirty blocks
        for block_id, entry in self.entries.items():
            if entry.is_dirty:
                self._flush_to_ssd(block_id, entry.data)
        self.entries.clear()
        self.lru_order.clear()
```

**Parameters:**
*   **CPU Cache Capacity ($C_{CPU}$):** Not given in the sources.
*   **Dirty Bit:** 1 bit per block.

**Edge Cases:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** The LRU policy is updated on every access, independent of the dirty bit. Frequently reused dirty blocks remain in CPU memory and are not immediately persisted [S1, unofficial].
*   **Degenerate Input:** If the SSD is full or fails, the flush operation may block or fail. Not given in the sources.

## Subtopics

### Lazy Write-Back
When a dirty block is evicted from VRAM, it is staged to the CPU cache as dirty. It is appended to SSD patch segments only when it is later evicted from the CPU cache [S1, unofficial]. Explicit consistency barriers may also flush dirty CPU-cache entries as needed [S1, unofficial]. This design amortizes write-back, turning frequent block updates into batched sequential appends on the SSD, avoiding random in-place overwrites [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** $O(C_{CPU})$ where $C_{CPU}$ is the CPU cache capacity in blocks [S1, unofficial].
*   **Time:** Access is $O(1)$ for cache lookup and LRU update. Write-back is asynchronous and amortized over multiple updates [S1, unofficial].

**Measured Performance:**
*   Achieves high sustained throughput for writes by converting random in-place overwrites into sequential appends on SSD [S1, unofficial].
*   Decouples GPU residency from SSD write latency, improving training efficiency [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Reduces SSD wear and I/O overhead by batching writes. Prevents unnecessary persistence of blocks that are still being actively used in VRAM.
*   **Limitations:** Requires additional CPU memory for the cache. Consistency barriers (checkpointing) may cause latency spikes if many dirty blocks need to be flushed [S1, unofficial].

**When to Use:**
*   Use when GPU training involves frequent parameter updates (gradients) and SSD write bandwidth is a bottleneck.
*   Use when checkpointing or shutdown consistency is required.

**When Not to Use:**
*   Use when memory is extremely constrained and CPU cache cannot hold a significant portion of the working set.
*   Use when real-time consistency between GPU and SSD is required without explicit barriers.

## Background
Originated in TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization [S1, unofficial]. The method addresses the challenge of training large-scale 3D Gaussian Splatting models where the number of primitives exceeds GPU memory, requiring out-of-core optimization with frequent parameter updates [S1, unofficial].

## Sources
[S1] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
