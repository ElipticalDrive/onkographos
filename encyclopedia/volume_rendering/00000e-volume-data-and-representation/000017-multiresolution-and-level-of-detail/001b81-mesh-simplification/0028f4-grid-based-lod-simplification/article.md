# Grid-based LOD Simplification

Also called: Coordinate Rounding LOD, Geometry Quantization LOD, LOD Simplification, POP Buffer LOD, Progressively Ordered Primitive Buffer, Snapping LOD

## Overview
Grid-based LOD Simplification is a method for rendering large-scale voxel or blocky meshes at multiple levels of detail by quantizing vertex coordinates to a coarse grid and implicitly managing primitive visibility. It solves the problem of rendering distant geometry efficiently without storing multiple explicit mesh copies on the GPU, by sorting primitives into a "Progressively Ordered Primitive" (POP) buffer [S1, unofficial]. Unlike [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md) which modifies topology, or [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md) which requires complex graph operations, this method relies on simple coordinate rounding and sorting [S1, unofficial]. It is distinct from [Linear Spline Simplification](3483-linear-spline-simplification.md) as it operates on discrete integer grids rather than continuous spline approximations [S2, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A mesh composed of axis-aligned quads (or triangles) with integer vertex coordinates.
- **Output**: A sorted array of primitives (POP buffer) and an offset table mapping each LOD level to the start index of primitives at that level.

### Data Structures
1.  **Primitive Array**: An array of quad definitions. Each quad is defined by its four vertex coordinates.
2.  **LOD Bucket Array**: An array of size $L_{max}$ (total number of LOD levels), initialized to zero. This serves as both the counter and the prefix-sum offset table.

### LOD Computation
The core of the method is determining the coarsest LOD level at which a primitive remains non-degenerate.

1.  **Vertex LOD**: For a vertex $v$ with integer coordinate $x$, the LOD is defined as the number of trailing zeros in its binary representation:
    $$ lod(v) = \text{countTrailingZeros}(x) $$
    [S1, unofficial]

2.  **Primitive LOD**:
    -   For a triangle with vertices $v_1, v_2, v_3$:
        $$ lod(t) = \max(lod(v_1), lod(v_2), lod(v_3)) $$
        [S1, unofficial]
    -   For a general axis-aligned quad, the LOD is the minimum LOD along each axis. For a single axis interval $[lo, hi]$, the LOD is the smallest integer $i$ such that $(lo \gg i) == (hi \gg i)$. This can be computed efficiently using the XOR of the endpoints:
        $$ LOD = \text{countLeadingZeros}(lo \oplus hi) $$
        [S1, unofficial]

3.  **Degeneracy Check**: If rounding causes two vertices of a cell to map to the same point (edge collapse), the primitive is considered degenerate at that LOD and is removed from that level's bucket.

### Algorithm Steps

1.  **Initialize Buckets**: Create an array `buckets` of size $L_{max}$, initialized to 0.
2.  **Count LODs**: Iterate through all primitives. For each primitive, compute its LOD using the formulas above. Increment `buckets[lod]`.
3.  **Compute Prefix Sums**: Transform the counts into starting offsets.
    $$ \text{offset}[i] = \sum_{j=0}^{i-1} \text{count}[j] $$
    [S1, unofficial]
4.  **Sort Primitives**: Place each primitive into its correct position in the sorted array using the computed offsets. This is typically done in-place using a stable partitioning or counting sort variant.
5.  **Handle Squashed Faces (Optional)**: After sorting, multiple primitives may map to the same screen space (over-drawing). A final pass can move "squashed" faces to the next higher LOD level to improve fragment processing performance. [S1, unofficial]
6.  **Render**: To render at a specific LOD $L$, draw primitives from the sorted array starting at `offset[L]` up to `offset[L+1]`. In the vertex shader, round the vertex coordinates to the grid size $2^L$ to visualize the simplification. [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_pop_buffer(quads, max_lod):
    """
    Builds a Progressively Ordered Primitive Buffer.
    quads: List of quad objects, each having a method or property to compute its LOD.
    max_lod: Integer, the maximum number of LOD levels.
    Returns:
        sorted_quads: List of quads sorted from coarsest to finest LOD.
        offsets: List of length max_lod + 1, where offsets[i] is the start index
                 for LOD level i.
    """
    # Step 1: Count primitives per LOD
    counts = [0] * max_lod
    for q in quads:
        lod = compute_quad_lod(q)
        if lod < max_lod:
            counts[lod] += 1
        else:
            # Handle primitives that require higher detail than max_lod
            # Option A: Clamp to max_lod
            counts[max_lod - 1] += 1
            # Option B: Discard or handle error. 
            # Based on [S1, unofficial], we assume quads fit within max_lod or are handled.
    
    # Step 2: Compute prefix sums (offsets)
    offsets = [0] * (max_lod + 1)
    current_offset = 0
    for i in range(max_lod):
        offsets[i] = current_offset
        current_offset += counts[i]
    offsets[max_lod] = current_offset
    
    # Step 3: Partition quads into sorted array
    # We need a temporary array for the sorted result to avoid overwriting
    # while reading, or use an in-place stable sort algorithm.
    # The source [S1, unofficial] describes an in-place partitioning using the buckets as pointers.
    
    sorted_quads = [None] * len(quads)
    
    # Reset buckets to act as write pointers
    write_ptrs = list(offsets[:max_lod])
    
    # Iterate backwards to maintain stability if required, 
    # though for LOD sorting stability among same-LOD items is often not critical.
    # The source [S1, unofficial] iterates backwards.
    for i in range(len(quads) - 1, -1, -1):
        q = quads[i]
        lod = compute_quad_lod(q)
        if lod >= max_lod:
            lod = max_lod - 1
            
        ptr = write_ptrs[lod]
        sorted_quads[ptr] = q
        write_ptrs[lod] += 1
        
    return sorted_quads, offsets

def compute_quad_lod(quad):
    """
    Computes the LOD for an axis-aligned quad.
    Assumes quad has vertices with integer coordinates.
    """
    # Get bounding box of the quad
    min_x = min(v.x for v in quad.vertices)
    max_x = max(v.x for v in quad.vertices)
    min_y = min(v.y for v in quad.vertices)
    max_y = max(v.y for v in quad.vertices)
    
    # Compute LOD for X axis
    lod_x = interval_lod(min_x, max_x)
    # Compute LOD for Y axis
    lod_y = interval_lod(min_y, max_y)
    
    # The LOD of the quad is the minimum of the axis LODs
    # (Coarser LOD = Higher LOD index? No, LOD 0 is finest, LOD N is coarsest.
    # Wait, [S1, unofficial] says "rounding down to previous power of two".
    # lod(v) = countTrailingZeros. 
    # If v=1 (binary 1), tz=0. LOD 0.
    # If v=2 (binary 10), tz=1. LOD 1.
    # So higher LOD index = Coarser.
    # The quad is valid at LOD L if all vertices round to distinct points?
    # Actually, [S1, unofficial] says: "If two vertices of a cell round to the same point... delete that cell from that LOD."
    # So we find the HIGHEST LOD (coarsest) where it is still valid?
    # No, we sort coarse-to-fine.
    # The LOD value assigned to the quad is the coarsest level it survives.
    # If it collapses at LOD 2, it exists at LOD 0 and 1.
    # So we want the smallest LOD index where it is valid?
    # Let's re-read [S1, unofficial]: "sort the primitives of the mesh from coarse-to-fine".
    # "buckets now contains the prefixes for each LOD".
    # If LOD 0 is finest, LOD N is coarsest.
    # A quad that collapses at LOD 2 should be in buckets 0 and 1.
    # So the "LOD of the quad" for sorting purposes is the coarsest LOD it appears in.
    # Which is the max LOD where it is non-degenerate.
    
    # However, the formula lod(v) = countTrailingZeros gives the resolution.
    # If we round to 2^L, we keep bits >= L.
    # Two vertices collide if they share the same upper bits.
    # The quad is valid at LOD L if no two vertices collide when rounded to 2^L.
    # We want the maximum L such that it is valid.
    
    # Let's check the interval LOD function.
    # intervalLOD(lo, hi) returns i where (lo >> i) == (hi >> i).
    # This is the first bit position where they differ? No.
    # If lo=0 (000), hi=1 (001). 
    # i=0: 0>>0=0, 1>>0=1. Diff.
    # i=1: 0>>1=0, 1>>1=0. Same.
    # Returns 1.
    # So LOD 1 is the level where they collide?
    # If they collide at LOD 1, they are invalid at LOD 1.
    # So they are valid at LOD 0.
    # So the quad's LOD for the bucket should be 0.
    
    # Therefore, the LOD index for the bucket is the MINIMUM LOD where it is valid?
    # Or is it the MAXIMUM LOD where it is valid?
    # [S1, unofficial] says: "sort ... from coarse-to-fine".
    # Bucket 0: Coarsest? Or Finest?
    # "round the vertices down to the previous power of two".
    # Usually LOD 0 is Full Detail. LOD N is Coarse.
    # If we sort coarse-to-fine, we want LOD N first, then N-1... then 0.
    # But the buckets array is indexed 0..MAX_LOD.
    # If bucket[0] is coarsest, then we draw bucket[0] first.
    
    # Let's look at the code:
    # buckets[quadLOD(quads[i])] += 1
    # If quadLOD returns the coarsest valid LOD, then it goes into that bucket.
    # If it collapses at LOD 1, it is valid at LOD 0.
    # Does it go into bucket 0 or bucket 1?
    # If it goes into bucket 1, it is drawn at LOD 1 (coarse).
    # If it goes into bucket 0, it is drawn at LOD 0 (fine).
    # We want it drawn at the coarsest possible level.
    # So if it is valid at 0 and 1, we want it in bucket 1 (if 1 is coarsest).
    
    # Let's assume LOD 0 = Finest, LOD N = Coarsest.
    # intervalLOD returns the LOD where collision happens.
    # So valid LODs are < intervalLOD.
    # Max valid LOD = intervalLOD - 1.
    
    return min(lod_x, lod_y) - 1

def interval_lod(lo, hi):
    """
    Returns the smallest i such that (lo >> i) == (hi >> i).
    This is the LOD level where the interval collapses.
    """
    if lo == hi:
        return 0 # Collapses at LOD 0? Or is it a point?
                 # If lo==hi, it's a point. It collapses at LOD 0.
                 # So it is invalid at LOD 0.
                 # It has no valid LOD?
                 # A point quad is degenerate.
    
    # Using countLeadingZeros on XOR
    xor_val = lo ^ hi
    if xor_val == 0:
        return 0
        
    # In Python, bit_length() gives the number of bits required to represent an integer.
    # The highest set bit is at position bit_length - 1.
    # countLeadingZeros is not standard in Python int, but we can simulate.
    # For 32-bit integers:
    clz = 32 - xor_val.bit_length()
    return clz
```

### Parameters
-   **Distance Constants**: In some implementations (e.g., [S2, unofficial]), distance constants for LOD selection are 4 and 8. The specific logic for selecting LOD based on camera distance is handled by the renderer.
-   **Bias**: A bias parameter in the geomorphing LOD function, based on display resolution, FOV, and hardware requirements. [S1, unofficial]

### Edge Cases and Pitfalls
-   **Cracks at LOD Transitions**: Abrupt switching between LOD levels in adjacent chunks causes geometric discontinuities (cracks). [S1, unofficial]
    -   *Solution 1*: Use skirts or transition seams, which increases implementation complexity and draw overhead.
    -   *Solution 2 (Geomorphing)*: Interpolate between LOD levels using vertex morphing. The LOD parameter $l$ is computed as a function of vertex coordinate $v$ and distance to camera $d$:
        $$ l = \text{bias} + \text{viewDist}(v) $$
        [S1, unofficial]
    -   *Stable Rounding*: To avoid discontinuities from the logarithmic LOD function, compute a fixed point for the LOD value. Let $L_{max}$ be the maximum LOD for the chunk and neighbors. Iterate 2-3 times in the vertex shader to stabilize the LOD value. [S1, unofficial]
-   **Squashed Faces**: Rounding multiple faces to the same level can cause over-drawing. A final pass can move these faces to the next LOD level. [S1, unofficial]
-   **Greedy Meshing Trade-off**: LOD reduction is less effective on greedy-meshed meshes than on naive meshes because snapped quads may not have matching counterparts to merge with (absorption). [S2, unofficial]

## Subtopics

### [Mesh simplification via valence pattern removal](1931-mesh-simplification-via-valence-pattern-removal.md)
A topology-based approach that removes vertices based on valence patterns. Choose when preserving exact geometric boundaries is less important than reducing vertex count via topological simplification.

### [Linear Spline Simplification](3483-linear-spline-simplification.md)
Uses linear splines to approximate geometry. Choose when smooth interpolation is required over discrete grid snapping.

### [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md)
Performs edge collapse operations on the GPU. Choose when dynamic, real-time decimation of arbitrary meshes is needed, rather than pre-computed grid-based LOD.

## Operating characteristics
-   **Time Complexity**: Building the POP buffer takes $O(n)$ time using counting sort, where $n$ is the number of primitives. [S1, unofficial]
-   **Memory Complexity**: $O(n)$ for the sorted primitive array and $O(L_{max})$ for the offset table. No multiple mesh copies are stored. [S1, unofficial]
-   **Performance**: The method is described as faster and simpler to implement than alternatives like Sea of Memes' approach. [S1, unofficial]
-   **Scaling**: Scales well with large numbers of primitives due to the linear sorting step.
-   **Strengths**: Implicit LOD (no multiple meshes), fast construction, smooth transitions with geomorphing.
-   **Limitations**: Discontinuities (cracks) if not handled with geomorphing or seams; less effective on greedy-meshed data; requires integer coordinates or quantization.
-   **When to Use**: Blocky voxel games or scenes with axis-aligned primitives where pre-computation is acceptable.
-   **When Not to Use**: Scenes with arbitrary curved geometry or dynamic meshes that cannot be pre-sorted.

## Background
The method originates from the "POP Buffer" (Progressively Ordered Primitive Buffer) concept introduced by M. Limper, Y. Jung, J. Behr, and M. Alexa in their paper "The POP Buffer: Rapid Progressive Clustering by Geometry Quantization" (Pacific Graphics 2013) [S1, unofficial]. It was adapted for voxel engines to handle level of detail transitions smoothly. The specific implementation details regarding interval LOD and stable rounding were popularized in blog posts discussing voxel engine optimization, such as those by 0fps.net [S1, unofficial]. It is used in games like box3.dao3.fun [S1, unofficial].

## Sources
[S1] A level of detail method for blocky voxels, https://0fps.net/2018/03/03/a-level-of-detail-method-for-blocky-voxels/, fetched 2026-10-26 (unofficial)
[S2] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
