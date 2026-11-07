# Parallel cache management with LRU

Also called: GPU-based cache management, Sarton et al. LRU cache management

## Overview
A method by Sarton et al. [SCRL20] that generalizes page-table hierarchies for non-rendering purposes and processes cache misses on the GPU itself using CUDA shared memory to minimize memory transfers between CPU and GPU [S1, unofficial]. It decouples cache residency management from the spatial subdivision, using a global usage buffer and an LRU buffer to keep frequently used bricks in the cache [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A brick cache with $N$ entries, a multi-channel page-table hierarchy, and a current timestamp $t$ (e.g., frame number).
*   **Output:** Updated LRU buffer order, and a list of missing brick IDs for the current timestamp.

**Data Structures:**
1.  **Global Brick-Usage Buffer:** An array of size $N$ (number of cache entries). Each element stores the last timestamp at which the corresponding cache entry was accessed [S1, unofficial].
2.  **LRU Buffer:** An array of size $N$ storing linear indices into the brick cache. The indices are sorted such that the most recently used entries are at the front (index 0) and the least recently used are at the back [S1, unofficial].
3.  **Global Request Buffer:** A 3D buffer (or equivalent mapping) with one entry per page/brick in the dataset. Stores the timestamp of the last request for that specific page [S1, unofficial].
4.  **Request List:** A dynamic list of brick IDs requested at a specific timestamp [S1, unofficial].

**Algorithm Steps:**

1.  **Record Usage:** When a brick-cache entry is accessed, write the current timestamp $t$ to the corresponding location in the Global Brick-Usage Buffer [S1, unofficial].
2.  **Update LRU Buffer:**
    a.  Generate a boolean mask $M$ of size $N$, where $M[i] = 1$ if `UsageBuffer[i] == t`, else $0$ [S1, unofficial].
    b.  Use mask $M$ to rearrange the previous LRU buffer. Entries where $M[i] == 1$ are moved to the front of the new LRU buffer, preserving their relative order from the previous buffer. Entries where $M[i] == 0$ are moved to the back, also preserving relative order [S1, unofficial].
3.  **Handle Cache Misses (Brick Request Processing):**
    a.  If a GPU thread accesses a page representing a brick that is not resident in the brick cache and is not known to be empty, write the current timestamp $t$ to the page's location in the Global Request Buffer [S1, unofficial].
    b.  To retrieve misses for timestamp $t$, read all pages from the Request Buffer with value $t$ [S1, unofficial].
    c.  Use a single stream compaction operation to compile a list of all brick IDs (from the Request List) that have been reported missing at timestamp $t$ [S1, unofficial].
    d.  Read back the Request List to the CPU [S1, unofficial].
    e.  The CPU streams the missing bricks from the server [S1, unofficial].

**Parameters:**
*   **Max Bricks per Timestamp:** A limit on the number of requested bricks per timestamp to control traffic between client and server. Can be a constant or variable (e.g., decreased during user interactions like viewport changes) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Traffic Control:** If the limit on requested bricks is not enforced, excessive traffic may occur during temporally incoherent working sets [S1, unofficial].
*   **Empty Bricks:** The method distinguishes between non-resident bricks and those known to be empty; only non-empty non-resident bricks trigger requests [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
usage_buffer = [0] * N  # Stores last access timestamp for each cache entry
lru_buffer = [0, 1, ..., N-1]  # Indices into cache, sorted by recency
request_buffer = {}  # Maps page_id -> last_request_timestamp
request_limit = 100  # Max bricks per timestamp

def update_lru(timestamp):
    """
    Updates the LRU buffer based on accesses at the current timestamp.
    """
    # Step 1: Generate mask for entries used at current timestamp
    mask = [usage_buffer[i] == timestamp for i in range(N)]
    
    # Step 2: Rearrange LRU buffer
    # Most recently used (mask is True) go to front, preserving relative order
    new_lru_front = [idx for idx in lru_buffer if mask[idx]]
    # Least recently used (mask is False) go to back, preserving relative order
    new_lru_back = [idx for idx in lru_buffer if not mask[idx]]
    
    lru_buffer = new_lru_front + new_lru_back

def handle_access(page_id, brick_id, timestamp):
    """
    Called when a brick is accessed.
    """
    # Record usage in global buffer
    cache_entry_index = get_cache_entry_index(page_id)
    usage_buffer[cache_entry_index] = timestamp
    
    # Check residency
    if not is_resident(cache_entry_index) and not is_empty(brick_id):
        handle_miss(page_id, brick_id, timestamp)

def handle_miss(page_id, brick_id, timestamp):
    """
    Processes a cache miss.
    """
    # Record request timestamp
    request_buffer[page_id] = timestamp
    
    # Note: The actual stream compaction and list generation happens
    # at the end of the frame/timestamp processing, not per-access.
    # This function just marks the miss.

def process_misses(timestamp):
    """
    Compiles and sends brick requests for the given timestamp.
    """
    # Find all pages requested at this timestamp
    requested_pages = [pid for pid, ts in request_buffer.items() if ts == timestamp]
    
    # Limit traffic
    if len(requested_pages) > request_limit:
        requested_pages = requested_pages[:request_limit]
        
    # Compile brick IDs
    brick_ids = [get_brick_id(pid) for pid in requested_pages]
    
    # Send to CPU
    send_to_cpu(brick_ids)
    
    # Clear request buffer for this timestamp (or manage lifecycle)
    for pid in requested_pages:
        del request_buffer[pid]
```

## Subtopics

### Brick Request Processing
An algorithm to process cache misses on the GPU and transmit brick requests to the CPU for streaming from the server [S1, unofficial]. It uses a 3D Request Buffer storing the last request timestamp for each page and a Request List compiled via stream compaction for a specific timestamp [S1, unofficial]. It allows limiting the number of requested bricks per timestamp to control traffic, which can be adjusted dynamically during user interactions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** LRU update requires scanning the usage buffer and rearranging the LRU buffer ($O(N)$). Miss compilation requires stream compaction on the request buffer ($O(P)$ where $P$ is number of pages).
*   **Memory Complexity:** Requires additional global buffers: Usage Buffer ($N$ integers), LRU Buffer ($N$ integers), and Request Buffer ($P$ integers) [S1, unofficial].
*   **Performance:** Minimizes CPU-GPU memory transfers by processing misses on the GPU using CUDA shared memory [S1, unofficial].
*   **Strengths:** Decouples cache residency from spatial subdivision; efficient GPU-side miss processing; scalable to exascale volumes [S1, unofficial].
*   **Limitations:** Requires synchronization between GPU and CPU for brick streaming; traffic control is necessary to prevent overload [S1, unofficial].
*   **When to use:** Large volumetric datasets where cache residency management is critical and CPU-GPU bandwidth is a bottleneck [S1, unofficial].

## Background
Originated with Sarton et al. [SCRL20], who generalized page-table hierarchies for volumetric datasets for non-rendering purposes [S1, unofficial]. The approach builds on earlier work by Fogal et al. [FSK13] who used a global hash table for reporting cache misses, but improves efficiency by processing misses on the GPU [S1, unofficial]. It is used in scalable web-based volume rendering of large biomedical datasets [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
