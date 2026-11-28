# Speculated Raytracing Kernel Inputs Construction
Also called: Build Speculated Raytracing Kernel Inputs

## Overview
This method constructs the input buffers required for a raytracing kernel when speculative intersection detection is enabled. It solves the problem of non-unique ray identifiers in the active ray ID buffer, where a single ray ID may appear multiple times corresponding to potential intersections in different blocks. It differs from the non-speculative input construction by introducing an offset buffer to uniquely index each ray-block intersection pair, enabling the kernel to output multiple results per ray for later compositing [S1].

## How it works
**Inputs:**
- $M_{RAct}$: Buffer marking active ray-block intersections, produced during the compaction of active ray IDs [S1].
- $I_{RAct}$: Active ray ID buffer, containing ray IDs repeated $N_{Spec}$ times [S1].

**Outputs:**
- $O_{Spec}$: Offset buffer assigning a unique index to each ray-block intersection [S1].
- $I_{BVis}$: List of visible block IDs [S1].
- $N_{BRays}$: Number of rays to process for each visible block [S1].
- $O_{BRays}$: Offsets into the ray ID buffer for each visible block [S1].

**Data Structures:**
- $O_{Spec}$: An integer buffer with the same layout as $I_{RAct}$, storing unique indices for each entry in $I_{RAct}$ [S1].
- $I_{RAct}$: Array of ray IDs, compacted to active entries and sorted by block ID [S1].

**Steps:**
1. **Scan $M_{RAct}$ to produce $O_{Spec}$:** Iterate through the buffer $M_{RAct}$ which marks active ray-block intersections. Assign a unique sequential index to each active intersection. This index serves as the unique identifier for that specific ray-block pair, resolving the ambiguity caused by repeated ray IDs in $I_{RAct}$ [S1].
2. **Compact and Sort $O_{Spec}$:** Compact $O_{Spec}$ to retain only entries corresponding to active ray-block intersections. Sort these entries by block ID to ensure alignment with the sorted order of $I_{RAct}$ [S1].
3. **Produce Block Metadata:** Generate the list of visible block IDs ($I_{BVis}$), the count of rays per block ($N_{BRays}$), and the starting offsets for each block's rays ($O_{BRays}$) using the same logic as the non-speculative case, but operating on the now-aligned and unique data structures [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_speculated_kernel_inputs(M_RAct, I_RAct):
    # Step 1: Scan M_RAct to produce unique indices
    # M_RAct marks which entries in I_RAct are active intersections
    O_Spec = []
    unique_index = 0
    
    for i in range(len(M_RAct)):
        if M_RAct[i] is active:
            O_Spec.append(unique_index)
            unique_index += 1
        else:
            # Placeholder for inactive entries, to be removed in compaction
            O_Spec.append(None) 
            
    # Step 2: Compact O_Spec to active entries only
    O_Spec_compact = [idx for idx in O_Spec if idx is not None]
    
    # Step 3: Sort O_Spec_compact by block ID
    # This requires pairing O_Spec_compact with the corresponding block IDs from I_RAct
    # Assuming I_RAct is already sorted by block ID or we have a parallel block ID array
    # For clarity, we assume I_RAct and O_Spec_compact are aligned by index
    # We need to sort them together based on block ID
    
    # Extract block IDs corresponding to active rays
    # This step assumes access to block ID mapping for each ray in I_RAct
    # Let's assume we have a function get_block_id(ray_id) or a parallel array
    # Since the source says "sorted by block ID to match the order of I_RAct",
    # we perform a stable sort or re-indexing.
    
    # Simplified representation of sorting:
    # Create pairs of (block_id, unique_index)
    pairs = []
    for i, idx in enumerate(O_Spec_compact):
        # Placeholder: In practice, block ID is derived from the ray or block context
        # The source implies I_RAct is sorted by block ID, so we sort O_Spec accordingly
        block_id = get_block_id_for_active_entry(i) # [G] Helper to retrieve block ID
        pairs.append((block_id, idx))
        
    # Sort by block ID
    pairs.sort(key=lambda x: x[0])
    
    # Extract sorted unique indices
    O_Spec_sorted = [idx for _, idx in pairs]
    
    # Step 3: Produce block metadata (I_BVis, N_BRays, O_BRays)
    # This is done "as before" using the sorted I_RAct and O_Spec_sorted
    I_BVis, N_BRays, O_BRays = produce_block_metadata(I_RAct, O_Spec_sorted)
    
    return O_Spec_sorted, I_BVis, N_BRays, O_BRays
```

**Edge Cases and Pitfalls:**
- **Precision:** Not given in the sources.
- **Boundaries:** The number of speculated intersections $N_{Spec}$ must be consistent across all rays to ensure buffer sizes are predictable [S1].
- **Degenerate Input:** If $M_{RAct}$ is empty, the output buffers will be empty, and no raytracing kernel launch is required [S1].

## Subtopics

### Speculated Raytracing Kernel Outputs
This variant modifies the raytracing kernel to output intersection results to a new RGBZ buffer instead of directly to the framebuffer. This allows multiple intersections per ray to be stored for later depth compositing. The kernel writes color and depth values for intersections at offsets given in $O_{Spec}$ for the ray-block intersection [S1].

## Operating characteristics
**Time and Memory Complexity:**
- The construction involves a scan and a sort, which are linear and $O(N \log N)$ respectively with respect to the number of active ray-block intersections [S1].
- Memory overhead includes the additional $O_{Spec}$ buffer, which has the same size as the active portion of $I_{RAct}$ [S1].

**Measured Performance:**
- Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Enables handling of multiple intersections per ray, which is necessary for accurate isosurface visualization in massive volumes where a ray may intersect the same isosurface in multiple blocks [S1].
- **Limitations:** Requires additional memory for $O_{Spec}$ and a subsequent depth compositing step to resolve the multiple intersections [S1].

**When to Use:**
- Use when visualizing isosurfaces in massive volumes where rays may intersect the same surface in multiple blocks, and memory constraints prevent storing all intersections in the framebuffer directly [S1].

## Background
This method is part of a speculative progressive raycasting pipeline for memory-constrained isosurface visualization of massive volumes. It was introduced in the context of browser-based visualization of 1TB volumes [S1]. The approach allows for efficient handling of large datasets by deferring the final pixel color determination to a compositing step, thereby reducing memory bandwidth pressure during the raytracing phase [S1].

## Sources
[S1] Speculative Progressive Raycasting for Memory Constrained Isosurface Visualization of Massive Volumes, https://doi.org/10.48550/arxiv.2309.10212, fetched 2026-10-26
