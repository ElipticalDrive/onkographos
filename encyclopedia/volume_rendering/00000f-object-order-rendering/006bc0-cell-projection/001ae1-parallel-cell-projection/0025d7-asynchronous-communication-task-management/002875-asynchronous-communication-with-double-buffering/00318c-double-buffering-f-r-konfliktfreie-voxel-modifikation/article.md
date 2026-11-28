# Double-Buffering für konfliktfreie Voxel-Modifikation

Also called: Ping-Pong Buffering, Puffer-Alternierung, Shared Memory Double-Buffered Volume Buffer, Volumeold/Volumenew Swap, Vortex Double Buffer

## Overview
Double-buffering for conflict-free voxel modification prevents visual artifacts such as flickering and data corruption caused by concurrent read and write accesses to the same voxel data during simulation or rendering updates [S1, unofficial]. By maintaining two separate volume buffers and swapping them between computation phases, it ensures that every thread operates on a consistent, untainted state from the previous time step, thereby eliminating race conditions inherent in single-buffered parallel voxel movement [S2, unofficial]. This approach is distinct from synchronization-heavy methods (e.g., atomic operations or multi-pass directional updates) by trading increased memory consumption for simplified, lock-free parallel execution [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid representing the current state of the volume, a set of voxel movement vectors (position $\vec{p}_i$ and direction $\vec{h}_i$), and a double-buffered memory structure.
*   **Output:** A voxel grid representing the updated state for the next time step, stored in the "new" buffer.

**Data Structures:**
The core data structure is a double-buffered volume container. Based on the implementation in [S3, unofficial], the layout in shared memory is defined as:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct voxel_double_buffer_t {
    pixel_t volume[2][VOXELS_COUNT]; // Two pages of voxel data
    uint8_t page;                    // Index of the active buffer (0 or 1)
    uint8_t bits_per_channel;        // Color depth configuration
    uint16_t debug_flags;            // Debugging state
    uint16_t revolutions_per_minute; // Metadata (context-specific)
    uint16_t microseconds_per_frame; // Timing metadata
};
```
*Note: `pixel_t` and `VOXELS_COUNT` are implementation-specific types/constants. The essential fields for the method are `volume[2]` and `page` [S3, unofficial].*

**Numbered Steps:**
1.  **Initialization:** Allocate two identical volume buffers, referred to as `Volumeold` and `Volumenew`. Initialize `Volumeold` with the initial voxel state. Set the active page pointer to 0 (pointing to `Volumeold`) [S1, unofficial][S2, unofficial].
2.  **Read and Delete:** For each voxel $i$ at position $\vec{p}_i$ with movement vector $\vec{h}_i$, read the voxel data from `Volumeold` (the buffer corresponding to the previous state). Mark the voxel at $\vec{p}_i$ in `Volumeold` as empty or delete it to prevent double-processing [S1, unofficial].
3.  **Write:** Calculate the new position $\vec{p}_{new} = \vec{p}_i + \vec{h}_i$. Write the voxel data to $\vec{p}_{new}$ in `Volumenew` (the buffer corresponding to the next state) [S1, unofficial].
    *   *Concurrency Note:* If multiple threads attempt to write to the same voxel in `Volumenew`, race conditions may still occur (e.g., voxels disappearing). The sources suggest using atomic compare-and-swap operators or multiple directional passes to resolve this, though the double-buffering itself primarily solves the read-write conflict between steps [S2, unofficial].
4.  **Swap:** After all voxels have been processed for the current time step, swap the references of `Volumeold` and `Volumenew`. The buffer that was just written to becomes the source for the next iteration [S1, unofficial][S2, unofficial].
    *   In shared memory implementations, this is achieved by toggling the `page` index: `page = !page` [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel_step(volume_buffer, active_page, num_voxels):
    """
    Processes voxel movements using double-buffering.
    
    Args:
        volume_buffer: A list/array of two buffers, [buffer0, buffer1].
        active_page: Integer (0 or 1) indicating the current source buffer.
        num_voxels: Number of voxels to process.
        
    Returns:
        new_active_page: The index of the buffer that now holds the latest state.
    """
    # Identify source and destination buffers
    src_idx = active_page
    dst_idx = 1 - active_page
    
    src_vol = volume_buffer[src_idx]
    dst_vol = volume_buffer[dst_idx]
    
    # Clear destination buffer for the new time step
    # [G] Implementation detail: Clearing ensures no stale data from previous iterations remains.
    clear_buffer(dst_vol)
    
    # Process each voxel in parallel
    for i in range(num_voxels):
        # 1. Read and Delete from Source
        # Read voxel data from current position in the source buffer
        voxel = src_vol[i]
        
        if voxel is not None:
            # Delete voxel from source to prevent re-processing
            src_vol[i] = None
            
            # 2. Calculate new position
            # Assuming voxel structure contains position (p) and velocity (h)
            new_pos = voxel.position + voxel.velocity
            
            # 3. Write to Destination
            # [G] Pitfall: Concurrent writes to new_pos in dst_vol may cause conflicts.
            # The double-buffering prevents read-write conflicts between src and dst,
            # but not write-write conflicts within dst. Atomic operations or 
            # multi-pass strategies are required for correctness here [S2, unofficial].
            dst_vol[new_pos] = voxel
            
    # 4. Swap Buffers
    # The destination buffer now holds the valid state for the next frame
    return dst_idx

def initialize_double_buffer(size):
    """
    Allocates and initializes the double-buffered volume.
    """
    # Allocate two buffers
    buffer0 = allocate_volume(size)
    buffer1 = allocate_volume(size)
    
    # Initialize with empty or initial state
    clear_buffer(buffer0)
    clear_buffer(buffer1)
    
    # Load initial data into buffer0
    load_initial_state(buffer0)
    
    return [buffer0, buffer1], 0
```

**Parameters and Typical Values:**
*   **Buffer Count:** Always 2 [S1, unofficial][S2, unofficial].
*   **Memory Allocation:** Requires $2 \times N \times C$ bytes, where $N$ is the number of voxels and $C$ is the size of a voxel (e.g., 4 bytes for a simple ID, or more for RGBA + metadata) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** The method doubles the required video/graphics memory compared to single-buffering [S1, unofficial].
*   **Write-Write Conflicts:** Double-buffering prevents read-write conflicts between time steps but does not inherently prevent two threads from writing to the same voxel in the *same* destination buffer simultaneously. This can lead to data loss (voxels disappearing) and requires additional synchronization (e.g., atomic CAS) or algorithmic changes (multi-pass) [S2, unofficial].
*   **Synchronization:** In shared memory contexts, the swap operation must be visible to all processes/threads. Using a simple pointer swap in local memory is insufficient if the buffer is shared via `shm_open` and `mmap`; the `page` index must be updated atomically or via memory barriers [S3, unofficial].

## Subtopics

### Ping-Pong Buffering
A general term for the technique of using two buffers and alternating between them as source and destination. It is the fundamental mechanism enabling the conflict-free updates described here. Choose this when you need to decouple reading from writing in iterative algorithms. [Link to general concept if available, otherwise self-contained].

### Shared Memory Double-Buffered Volume Buffer
An implementation variant using POSIX shared memory (`shm_open`, `mmap`) to allow multiple processes (e.g., a rendering process and a display controller) to access the double-buffered volume. It adds system-level synchronization and memory mapping overhead but enables cross-process consistency. Choose this when rendering and display are decoupled into separate processes. [Link to Asynchronous Communication with Double-Buffering](4280-asynchronous-communication-with-double-buffering.md).

## Operating characteristics
*   **Time Complexity:** $O(N)$ per time step, where $N$ is the number of voxels. The swap operation is $O(1)$ (pointer/index toggle) [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(2N)$, doubling the memory footprint compared to a single buffer [S1, unofficial].
*   **Performance:** Eliminates the need for complex locking mechanisms per voxel, allowing for high-throughput parallel processing on GPUs or multi-core CPUs. However, the memory bandwidth required to read from one buffer and write to another may become a bottleneck on memory-constrained hardware [S1, unofficial].
*   **Strengths:** Simple implementation, no deadlocks, consistent state for each thread.
*   **Limitations:** High memory cost, does not solve intra-buffer write conflicts without additional primitives [S1, unofficial][S2, unofficial].
*   **When to Use:** When concurrent voxel modification is required and memory is sufficient. When read-write conflicts are the primary concern.
*   **When Not to Use:** When memory is extremely constrained. When write-write conflicts are frequent and atomic operations are too expensive.

## Background
The technique originated in the context of real-time voxel visualization and simulation, where flickering artifacts were observed due to simultaneous read/write operations on the same voxel data structure [S1, unofficial]. It was formalized in academic work on raycasting-based voxel rendering and GPU-based voxel simulations to ensure algorithmic correctness without sacrificing parallelism [S1, unofficial][S2, unofficial]. The specific shared memory implementation was developed for embedded volumetric display systems (e.g., Multivox) to synchronize rendering and display controllers [S3, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S3] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
