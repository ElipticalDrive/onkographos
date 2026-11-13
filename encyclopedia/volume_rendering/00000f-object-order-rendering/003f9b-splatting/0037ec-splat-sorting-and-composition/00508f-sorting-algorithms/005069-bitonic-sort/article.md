# Bitonic Sort

Also called: ABisort, Bitonic Sort for GPGPU, Govindaraju et al. Bitonic Sort, Particle Sorting (Bitonic Sort)

## Overview
Bitonic Sort is a data-independent network sorting algorithm used on the GPU to achieve exact depth ordering of splatted voxels for every frame [S1, unofficial]. It is chosen over partial-ordering methods like Odd-Even Merge Sort because volume rendering requires strict front-to-back or back-to-front ordering per frame to correctly composite transparency [S1, unofficial]. It is preferred over Adaptive Bitonic Sort (ABisort) in some implementations due to library accessibility and ease of adaptation, despite ABisort having potentially better theoretical complexity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An array of view-transformed voxels already resident in GPU memory, typically stored in a texture (View Transformed Voxels Texture). Each voxel is represented as a `<key, value>` pair, where the key is the depth value $b$ and the value is the voxel index [S1, unofficial].
*   **Output:** A sorted array of voxels stored in a texture (Depth-Sorted Voxels Texture), ordered by depth [S1, unofficial].

**Data Structures:**
The algorithm uses the Multiple Render Target (MRT) extension to store keys and values in separate render targets [S1, unofficial]. A one-to-one correspondence exists between the key texture and the value texture [S1, unofficial]. To optimize memory, four `<key, value>` instances can be packed into a single RGBA texel [S1, unofficial].

**Parameters:**
*   $uPass$: Current pass number in the sort algorithm [S2, unofficial].
*   $uStage$: Current stage number within the pass [S2, unofficial].
*   $uHalfVector$: The half-angle vector used for sorting particles [S2, unofficial].
*   $N$: Total number of particles/voxels to sort.

**Algorithm Steps:**
1.  **Initialization:** If the sorting direction needs to be reversed (e.g., due to camera movement), reset pass and stage counters [S2, unofficial].
2.  **Ping-Pong Buffering:** Use two framebuffers (current and previous) to swap read/write roles between passes [S1, unofficial].
3.  **Comparison Logic:** For each element at index $i$, determine the comparison direction and partner index based on the current pass and stage [S2, unofficial].
4.  **Fetch and Compare:** Fetch the partner data and perform a comparison using the depth key and the half-vector [S2, unofficial].
5.  **Write Result:** Write the sorted result to the appropriate render target [S1, unofficial].
6.  **Finalization:** After all passes, a fragment shader fills the final Depth-Sorted Voxels Texture from the sorted render targets [S1, unofficial].

**Pseudocode:**
The following pseudocode describes the fragment shader logic for a single comparison step in the Bitonic Sort network [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
#   i: Global particle index derived from texture coordinates
#   uPass: Current pass number
#   uStage: Current stage number
#   uHalfVector: Half-angle vector for sorting direction
#   prevPositions: Texture containing previous sort state (key=value, value=index)

def bitonic_sort_step(i, uPass, uStage, uHalfVector, prevPositions):
    # 1. Determine comparison direction and partner offset
    # The direction depends on the stage and pass in the bitonic network
    compare = determine_compare_direction(uPass, uStage, i)
    
    # 2. Calculate partner address
    # uPass represents the distance to the partner in the bitonic sequence
    adr = i + compare * uPass
    
    # 3. Fetch data from previous render target
    # Fetch key (depth) and value (index) for self and partner
    self_data = prevPositions.sample(i)
    partner_data = prevPositions.sample(adr)
    
    self_key = self_data.key      # Depth value b
    self_val = self_data.value    # Voxel index
    partner_key = partner_data.key
    partner_val = partner_data.value
    
    # 4. Perform comparison using dot product with half vector
    # This logic implements the bitonic merge step
    # If selfProjectedLength * compare < partnerProjectedLength * compare, keep self
    # Note: 'compare' acts as a sign multiplier for the direction
    lhs = self_key * compare
    rhs = partner_key * compare
    
    if lhs < rhs:
        # Self is "smaller" (or in correct order relative to direction)
        output_key = self_key
        output_val = self_val
    else:
        # Partner is "smaller"
        output_key = partner_key
        output_val = partner_val
        
    return output_key, output_val

def determine_compare_direction(uPass, uStage, i):
    # Standard Bitonic Sort logic for determining swap direction
    # This is a placeholder for the specific bitonic network index logic
    # which depends on the binary representation of i relative to stage/pass
    # In standard bitonic sort, direction is determined by the bit at position uStage
    # within the block defined by uPass.
    # [G] Standard bitonic sort uses the bit at index uStage of i to determine direction.
    # If the bit is 0, compare with lower; if 1, compare with higher.
    # The 'compare' variable here likely represents +1 or -1.
    if (i >> uStage) & 1:
        return -1  # Compare with higher index
    else:
        return 1   # Compare with lower index
```

**Edge Cases and Pitfalls:**
*   **Data Independence:** Unlike Quicksort, Bitonic Sort does not branch on data values, making it suitable for GPU SIMD/SIMT architectures but potentially slower for small datasets due to fixed work [S1, unofficial].
*   **Exact Ordering:** Requires all passes to complete for exact ordering; partial results are insufficient for correct compositing [S1, unofficial].
*   **Memory Packing:** Packing four values per texel requires careful handling of texture coordinates and swizzling to ensure correct key/value alignment [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log^2 N)$ comparisons and swaps [S1, unofficial]. Adaptive Bitonic Sort (ABisort) claims $O((N \log N)/P)$ complexity, where $P$ is the number of stream processors [S1, unofficial].
*   **Performance:** On an NVIDIA GeForce 7800, Govindaraju's Bitonic Sort was approximately twice as fast as the best CPU QuickSort implementation at the time of the source [S1, unofficial].
*   **Strengths:** Provides exact ordering every frame; highly parallelizable; cache-efficient memory access patterns in optimized implementations [S1, unofficial].
*   **Limitations:** Fixed number of operations regardless of data distribution; not suitable for dynamic datasets where partial ordering is acceptable (Odd-Even Merge Sort is preferred there) [S1, unofficial].
*   **When to Use:** When exact depth sorting is required for every frame in volume splatting [S1, unofficial].
*   **When Not to Use:** When partial ordering is sufficient (e.g., particle systems spread over multiple frames) or when adaptive sorting performance is critical and library access permits [S1, unofficial].

## Background
Bitonic Sort was originally developed as a network sorting algorithm. In the context of GPU volume rendering, Govindaraju et al. [GRHM05] improved early implementations [KW05] by introducing cache-efficient memory accesses, leading to high performance on GPUs like the NVIDIA GeForce 7800 [S1, unofficial]. Gress and Zachmann [GZ06] later designed Adaptive Bitonic Sort (ABisort) with optimal complexity [S1, unofficial]. In the specific implementation described in the sources, Govindaraju's method was chosen over ABisort due to the accessibility and adaptability of the library [S1, unofficial].

## Sources
[S1] Image-space sheet-buffered splatting on the GPU, https://openalex.org/W2419486389, fetched 2026-10-27 (unofficial)
[S2] GitHub - zk8085454/VolumeParticleShadows: Half angle slicing technique ..., https://github.com/zk8085454/VolumeParticleShadows/, fetched 2026-10-28 (unofficial)
