# GPU-based Hash Table for Ray Guidance

Also called: concurrent brick request table, lock-free hash table

## Overview
This method provides an efficient mechanism for communicating ray guidance information (specifically, missing brick identifiers) from the GPU, where ray casting occurs, to the CPU I/O layer, which manages volume data residency [S1]. It functions as a concurrent set where keys and values are identical (brick IDs), indicating the presence of a non-resident brick required for rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D index representing a requested brick, composed of spatial coordinates $(x, y, z)$ and a Level of Detail (LoD) index [S1].
*   **Output:** An update to a GPU-resident hash table marking the brick ID as present/needed. The table is read back to the CPU at the end of the frame to trigger data loading [S1].

**Data Structures:**
The hash table is a 1D array of integers [S1]. Each entry stores a brick ID, and the value `0` is reserved to indicate an empty slot [S1].

**Algorithm Steps:**
1.  **Index Mapping:** Convert the 4D brick index $(x, y, z, \text{LoD})$ into a unique 1D hash index by treating the 4D index as if it were stored in a contiguous 1D array [S1].
2.  **Offset Adjustment:** Increment the calculated 1D index by 1 so that the value `0` can be used to indicate that there is no entry at a location [S1].
3.  **Concurrent Insert:** Attempt to insert the brick ID into the table at the calculated index using lock-free atomic primitives [S1].
4.  **Probing:** If the concurrent insert fails due to a race condition (another thread wrote to the slot), probe the table a fixed number of times before giving up [S1].
5.  **Readback:** At the end of the rendering frame, read the hash table from GPU memory to the CPU [S1]. The CPU processes the non-zero entries as a list of bricks to fetch into the volume pool [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_PROBES = 10
EMPTY_VALUE = 0
# Dimensions not given in sources; replace with actual grid strides.
DIM_X = 1
DIM_Y = 1
DIM_Z = 1
DIM_LOD = 1

# Simulated GPU atomic operations (in GLSL: atomicLoad, atomicCompSwap)
def atomic_load(table, idx):
    return table[idx]

def atomic_compare_and_swap(table, idx, expected, desired):
    if table[idx] == expected:
        table[idx] = desired
        return True
    return False

# Function to map 4D index to 1D hash index
def map_4d_to_1d(x, y, z, lod):
    # Converts 4D index to 1D form as if stored in a 1D array.
    index = x + y * DIM_X + z * DIM_X * DIM_Y + lod * DIM_X * DIM_Y * DIM_Z
    return index

# Function to insert a missing brick ID into the hash table
def insert_missing_brick(table, brick_id, x, y, z, lod):
    # Step 1 & 2: Map and offset
    index = map_4d_to_1d(x, y, z, lod)
    index = index + 1  # Offset by 1 to allow 0 as empty

    # Step 3 & 4: Lock-free insert with probing
    for probe in range(MAX_PROBES):
        # Attempt to swap EMPTY_VALUE with brick_id
        if atomic_compare_and_swap(table, index, EMPTY_VALUE, brick_id):
            return True  # Success

        # If failed, probe to next index
        index = index + 1

    return False  # Gave up after MAX_PROBES

# Note: The source states it is not an error if a missing brick is not recorded.
# The invariant is: volume rendering is done, OR at least one ray can make progress.
```

**Parameters:**
*   **Probe Count:** 10 attempts before giving up on a concurrent insert [S1].

**Edge Cases and Pitfalls:**
*   **GLSL Lock Limitations:** GLSL lacks native lock primitives, necessitating lock-free structures [S1].
*   **Concurrent Write Hazards:** Multiple rays may attempt to write the same or adjacent slots [S1]. The implementation handles this via probing rather than locking [S1].
*   **Incomplete Recording:** It is not an error if a missing brick is not recorded in the hash table due to probe exhaustion [S1]. The system invariant ensures progress as long as *some* missing bricks are recorded, ensuring at least one ray can continue [S1].

## Operating characteristics
*   **Memory:** The table is described as "small" and requires significantly less memory than multi-render-target approaches [S1]. Not given in the sources.
*   **Performance:** Lock-free design reduces contention compared to locked structures [S1]. Probing on failure handles collisions without stalling threads [S1].
*   **Strengths:** Efficient communication from GPU to CPU; low memory overhead; handles concurrency without locks [S1].
*   **Limitations:** Potential for missed requests if probes are exhausted (though acceptable per invariant); requires readback from GPU to CPU each frame [S1].

## Background
This method is part of a scalable GPU-based ray-guided volume rendering system [S1]. It addresses the bottleneck of transferring ray guidance data (missing brick IDs) from the GPU renderer to the CPU I/O layer [S1]. It contrasts with Crassin et al.'s approach which uses multiple render targets to avoid write hazards, offering a more memory-efficient solution at the cost of implementing lock-free logic [S1].

## Sources
[S1] An Analysis of Scalable GPU-Based Ray-Guided Volume Rendering, https://pmc.ncbi.nlm.nih.gov/articles/PMC4260263/, fetched 2026-10-26
