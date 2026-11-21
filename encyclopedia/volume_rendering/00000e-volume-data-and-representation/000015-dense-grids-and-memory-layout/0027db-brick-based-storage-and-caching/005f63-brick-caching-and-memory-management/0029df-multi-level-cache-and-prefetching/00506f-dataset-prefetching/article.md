# Dataset Prefetching

Also called: Asynchronous Data Loading, Dataset.run

## Overview
Dataset Prefetching is a technique that decouples data retrieval from the rendering loop by running a background thread to prefetch batches into a queue [S1, unofficial]. It solves the problem of CPU-bound data loading stalling the GPU by ensuring that volume data is available in memory before it is needed for rendering [S1, unofficial]. Unlike sibling methods that focus on spatial caching hierarchies (e.g., [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md) or [Rolling 3-Row Slab Cache](4747-rolling-3-row-slab-cache.md)), this method focuses on temporal overlap of I/O and computation [S1, unofficial].

## How it works
**Inputs:** A dataset object providing iteration methods (e.g., `_next_train`, `_next_test`) and a configuration for batch size.
**Outputs:** A queue containing pre-fetched data batches ready for immediate consumption by the renderer.

**Data Structures:**
1.  **Prefetch Queue:** A thread-safe FIFO queue.
    *   **Size:** 3 [S1, unofficial].
    *   **Content:** Batches of volume data (e.g., textures, ray samples, or metadata) [S1, unofficial].

**Steps:**
1.  Initialize the prefetch queue with a capacity of 3 [S1, unofficial].
2.  Start a background thread in daemon mode [S1, unofficial].
3.  The thread enters an infinite loop:
    a.  Call the appropriate dataset next function (`_next_train` or `_next_test`) to retrieve the next batch [S1, unofficial].
    b.  Put the retrieved batch into the queue [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import queue
import threading

class DatasetPrefetcher:
    def __init__(self, dataset, batch_size=1):
        self.dataset = dataset
        self.batch_size = batch_size
        # Initialize queue with size 3 as per source [S1, unofficial]
        self.queue = queue.Queue(maxsize=3)
        self.thread = None
        self.running = False

    def _prefetch_loop(self):
        """Background thread loop."""
        while self.running:
            try:
                # Call appropriate next function based on training/testing mode
                # Note: Implementation details of _next_train/_next_test are dataset-specific
                batch = self.dataset._next_train() 
                # Put result into queue. Blocks if queue is full.
                self.queue.put(batch)
            except Exception as e:
                # Handle end of dataset or errors gracefully
                break

    def start(self):
        """Start the prefetching thread."""
        self.running = True
        self.thread = threading.Thread(target=self._prefetch_loop, daemon=True)
        self.thread.start()

    def get_batch(self):
        """Retrieve a batch from the queue, blocking if empty."""
        return self.queue.get()

    def stop(self):
        """Stop the prefetching thread."""
        self.running = False
        if self.thread:
            self.thread.join()
```

**Parameters:**
*   **Queue Size:** 3 [S1, unofficial]. This value balances memory usage against the likelihood of the renderer waiting for data.
*   **Thread Mode:** Daemon [S1, unofficial]. Ensures the thread does not prevent program termination.

**Edge Cases and Pitfalls:**
*   **Queue Full:** If the renderer consumes data slower than the prefetcher produces it, `queue.put()` will block. The fixed size of 3 limits memory overhead but may cause the prefetcher to stall if the renderer is significantly slower [S1, unofficial].
*   **Queue Empty:** If the renderer consumes data faster than the prefetcher produces it, `queue.get()` will block. This negates the benefit of prefetching and introduces latency [S1, unofficial].
*   **Dataset Exhaustion:** The loop must handle the end of the dataset to avoid exceptions. In training loops, this often involves resetting the dataset or stopping the thread [S1, unofficial].

## Subtopics

### DHCS (Distributed Hierarchical Cache System) Asynchronous Overlap
This variant extends asynchronous prefetching to distributed volume rendering. It launches all asynchronous data loads for the current frame, determines the local memory state at the end of the frame (when loads are achieved), and launches binary all-to-all communication overlapped with rendering [S2]. Crucially, it renders the first bricks even if all requests are not yet satisfied [S2]. This approach avoids the performance decrease of sequential implementations where data loading, cache state update, volume slicing, and rendering are executed serially [S2].

## Operating characteristics
*   **Time Complexity:** Reduces effective I/O latency by overlapping data loading with rendering/computation. Theoretical speedup depends on the ratio of I/O time to computation time.
*   **Memory Complexity:** $O(K \times B)$, where $K$ is the queue size (3) and $B$ is the batch size [S1, unofficial].
*   **Strengths:** Hides I/O latency; simple to implement; decouples data pipeline from rendering loop [S1, unofficial].
*   **Limitations:** Fixed queue size may lead to under-utilization or blocking; does not address spatial locality or cache misses within the volume data itself [S1, unofficial].
*   **When to Use:** When data loading is a bottleneck and the dataset can be iterated sequentially or randomly with low overhead [S1, unofficial].
*   **When Not to Use:** When spatial locality is critical and data must be fetched from remote storage in a non-sequential manner without additional caching logic (see siblings) [S1, unofficial][S2].

## Background
The concept of asynchronous data loading via background threads is commonly used in deep learning pipelines for volume rendering, such as in NeRF implementations [S1, unofficial]. The specific pattern of using a queue size of 3 and daemon threads is documented in the CaSE-NeRF project [S1, unofficial]. The principle of overlapping data loading with rendering to avoid serial execution penalties is also described in the context of distributed shared memory systems for large volumes, such as DHCS [S2].

## Sources
[S1] CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
[S2] Distributed Shared Memory for Roaming Large Volumes, https://doi.org/10.1109/tvcg.2006.135, fetched 2026-10-26
