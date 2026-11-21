# GPU-based Hash Table for Ray Guidance

Also called: concurrent brick request table, lock-free hash table

## Overview
This method provides efficient communication of ray guidance information (specifically, brick IDs) from the GPU, where rays are cast, to the CPU I/O layer, which manages volume data residency [S1]. It solves the problem of identifying non-resident bricks required for ray continuation without the overhead of locks or complex synchronization primitives [S1]. Unlike a standard hash map, it functions as a set where keys are values, storing only the presence of requested bricks to trigger data loading [S1]. It is distinct from sibling structures like [Two-Level Sparse Voxel Grid Layout](5065-two-level-sparse-voxel-grid-layout.md) by focusing specifically on the GPU-to-CPU feedback loop for missing data rather than general traversal indexing [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of ray requests for bricks. Each request consists of a 4D index comprising a 3D spatial coordinate and a Level of Detail (LoD) integer [S1].
*   **Output:** A hash table on the GPU containing unique brick IDs, which is read back to the CPU to populate the volume pool [S1].

**Data Structures:**
*   **Hash Table:** A 1D array of integers. The size is determined by the maximum expected number of concurrent missing brick requests per frame [S1].
*   **Entry Layout:** Each slot stores a 1D brick index. The value `0` indicates an empty slot [S1].

**Mapping 4D Index to 1D Index:**
The 4D index $(x, y, z, \text{LoD})$ is mapped to a unique 1D index $I_{raw}$ as if the 4D space were flattened into a 1D array [S1]. The specific stride dimensions for this flattening are not given in the sources [G]. The final hash table index $I$ is calculated as:
$$ I = I_{raw} + 1 $$
This offset ensures that `0` can represent an empty slot [S1].

**Algorithm:**
1.  **Request:** When a ray encounters a missing brick, it computes the brick's 4D index [S1].
2.  **Hash:** Compute the 1D index $I$ using the mapping described above [S1].
3.  **Insert:** Attempt to insert $I$ into the hash table using lock-free atomic primitives (e.g., `atomicAdd` or `atomicCAS` depending on the specific implementation of the set logic) [S1].
    *   If the slot is empty (`0`), write $I$ [S1].
    *   If the slot is occupied, the insert is considered successful (as the brick is already recorded) [S1].
    *   If the slot is occupied by a different value or the atomic operation fails due to contention, probe the next available slot [S1].
4.  **Probe:** If the initial insert fails or the slot is occupied by a different brick, probe up to 10 times before giving up [S1].
5.  **Readback:** At the end of the frame, read the hash table from GPU memory to the CPU [S1].
6.  **Process:** The CPU iterates through the table, collecting all non-zero entries as a list of missing bricks to load into the volume pool [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_PROBES = 10
EMPTY_SLOT = 0

# Global GPU State
hash_table = [0] * TABLE_SIZE  # Initialized to 0

def request_brick(spatial_x, spatial_y, spatial_z, lod):
    """
    Requests a brick and records its ID in the GPU hash table.
    spatial_x, spatial_y, spatial_z: int
    lod: int
    """
    # 1. Map 4D index to 1D index
    # Note: The specific stride calculation for 4D to 1D flattening is not provided in sources [G]
    # Assuming a standard row-major flattening for illustration, but exact strides are implementation-dependent
    # I_raw = ((spatial_z * DIM_Z + spatial_y) * DIM_Y + spatial_x) * DIM_LOD + lod
    # The sources state: "converting the 4D index into its equivalent 1D form"
    i_raw = compute_4d_to_1d(spatial_x, spatial_y, spatial_z, lod)
    
    # 2. Offset by 1 so 0 indicates empty
    target_index = i_raw + 1
    
    # 3. Lock-free insert with probing
    for probe in range(MAX_PROBES):
        # Attempt to write target_index into hash_table[target_index % TABLE_SIZE]
        # Using a simple atomic compare-and-swap or add logic to ensure uniqueness
        # Since it's a set, if the value is already there, we are done.
        # If it's empty (0), we write it.
        # If it's occupied by a different value, we probe next.
        
        slot_idx = target_index % TABLE_SIZE
        current_val = hash_table[slot_idx]
        
        if current_val == EMPTY_SLOT:
            # Try to claim the slot
            if atomic_compare_and_swap(hash_table, slot_idx, EMPTY_SLOT, target_index):
                return # Successfully recorded
            else:
                # Another thread claimed it, or it was filled between check and swap
                # If it was filled with the same value, we are also done, but probing handles contention
                pass
        elif current_val == target_index:
            return # Already recorded
        else:
            # Occupied by different brick, probe next
            target_index += 1
            
    # If we exhaust probes, we give up. This is not an error [S1].
    pass

def compute_4d_to_1d(x, y, z, lod):
    # Placeholder for the specific 4D flattening logic not detailed in sources
    # The sources only specify "converting the 4D index into its equivalent 1D form"
    return (z * 1000 + y) * 1000 + x  # Example logic, not from source [G]
```

**Parameters:**
*   **Probe Count:** 10 attempts before giving up on a concurrent insert [S1].

**Edge Cases and Pitfalls:**
*   **GLSL Locks:** GLSL lacks traditional locks, necessitating lock-free structures [S1].
*   **Concurrent Write Hazards:** Multiple rays may request the same brick simultaneously. The probing mechanism handles this by allowing multiple threads to compete for slots [S1].
*   **Missing Records:** It is not an error if a missing brick is not recorded in the hash table due to probe exhaustion [S1]. The system invariant is that volume rendering is either complete or at least one ray can make progress [S1].

## Subtopics

### Sparse Hash Map CPU Index with GPU Shallow Tree
This variant describes a two-tier indexing system where the CPU uses a sparse hash map keyed by brick coordinate (int32 in Planet-frame coordinates) for management, while the GPU uses a shallow tree structure for ray traversal [S2, unofficial]. It differs from the primary method by separating the management index (CPU) from the traversal index (GPU), whereas the primary method focuses on the feedback hash table for missing data [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion is $O(1)$ amortized with a constant probe limit (10) [S1]. Readback is $O(N)$ where $N$ is the table size [S1].
*   **Memory Complexity:** Proportional to the maximum number of concurrent missing brick requests per frame [S1]. Significantly less memory than multiple render target approaches [S1].
*   **Performance:** Efficient for concurrent inserts with low contention overhead compared to locked structures [S1].
*   **Strengths:** Lock-free, low memory footprint, simple set semantics [S1].
*   **Limitations:** Requires readback from GPU to CPU every frame [S1]. Probe exhaustion can lead to missing brick requests, though the system invariant ensures progress [S1].
*   **When to Use:** When ray guidance data needs to be communicated from GPU to CPU for dynamic data loading in volume rendering [S1].
*   **When Not to Use:** When strict completeness of missing brick recording is required for every single ray in a single frame, or when GPU-CPU synchronization overhead is prohibitive [S1].

## Background
Originating from research on scalable GPU-based ray-guided volume rendering, this method addresses the bottleneck of communicating missing brick information from the GPU renderer to the CPU I/O layer [S1]. It was developed as an alternative to multiple render target approaches which required more memory [S1]. The approach leverages the fact that a set of missing bricks is sufficient for the CPU to make progress in loading data, even if not all missing bricks are recorded in a single frame [S1].

## Sources
[S1] An Analysis of Scalable GPU-Based Ray-Guided Volume Rendering, https://pmc.ncbi.nlm.nih.gov/articles/PMC4260263/, fetched 2026-10-26
[S2] Choose voxel data structure & LOD scheme #9, https://github.com/theHimanshuShekhar/bhayanakvoxel/issues/9, fetched 2026-10-28 (unofficial)
