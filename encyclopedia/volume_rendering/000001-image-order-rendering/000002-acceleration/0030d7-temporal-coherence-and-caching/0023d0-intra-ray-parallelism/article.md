# Intra-ray Parallelism

Also called: GPU-acceleration through intra-ray parallelism

## Overview
Intra-ray parallelism is a rendering acceleration scheme that partitions each ray into depth bins to enable efficient parallel accumulation of primitive contributions during rendering [S1, unofficial]. It differs from sibling methods like [Inter-Ray Coherency](1030-inter-ray-coherency.md) by exploiting depth coherence within a single ray's path rather than spatial coherence between adjacent rays, allowing for batched processing of primitives against multiple depth intervals simultaneously.

## How it works
**Inputs:**
*   $w, h$: Width and height of the image to rasterize.
*   $b$: Number of depth bins.
*   $E$: Set of ellipsoid primitives, defined by world-space parameters (mean $\mu$, covariance $\Sigma$) [S1, unofficial].
*   $TF$: Transfer function mapping intensity to color and opacity [S1, unofficial].
*   $View$: View configuration of the current camera [S1, unofficial].

**Outputs:**
*   $I$: Rendered image.

**Data Structures:**
*   $T$: Tile grid including depth-wise subdivisions.
*   $E_{visible}$: Culled ellipsoids.
*   $L, K$: Lists of primitives and their associated 3D keys relative to tiles.
*   $R$: Ranges defining the start and end indices of primitives assigned to each tile.
*   $I_{bin}$: Per-bin canvas, initialized to 0.

**Steps:**
1.  Create tiles $T$ from image dimensions $w, h$ and bin count $b$. Tiles include depth-wise subdivisions [S1, unofficial].
2.  Cull ellipsoids $E$ based on the current $View$ to produce $E_{visible}$ [S1, unofficial].
3.  Duplicate visible ellipsoids with 3D keys relative to tiles $T$ to produce lists $L$ and $K$ [S1, unofficial].
4.  Sort lists $K$ and $L$ by tile keys. Note: No intra-tile depth sorting is performed [S1, unofficial].
5.  Identify ranges $R$ for each tile $t$, defining the range of primitives assigned to that 3D tile [S1, unofficial].
6.  Initialize per-bin canvas $I_{bin}$ to 0 [S1, unofficial].
7.  Loop over all tiles $t$ in $T$:
    a. Extract primitives $E_t$ assigned to tile $t$ from list $L$ using range $R[t]$ [S1, unofficial].
    b. For all pixels $i$ in $t$:
        i. Compute ray $ray[i, t_z]$ intensity sample based on $i$, $E$, and $E$ [S1, unofficial].
        ii. Blend the partial ray result into the per-bin canvas: $I_{bin}[i, t_z] \leftarrow \text{BlendPartialRay}(ray[i, t_z])$ [S1, unofficial].
8.  After processing all tiles, reset tile grid $z=1$ by creating new tiles $T$ with bin count 1: $T \leftarrow \text{CreateTiles}(w, h, 1)$ [S1, unofficial].
9.  Loop over all pixels $i$ in image $I$:
    a. Merge Z-bins by blending across $b$ bins: $I[i] \leftarrow \text{MergeZBins}(I_{bin}[i, 0:b])$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def RENDER(w, h, b, E, TF, View):
    # 1. Create tiles with depth-wise subdivisions
    T = CreateTiles(w, h, b)
    
    # 2. Cull ellipsoids based on view
    E_visible = CullEllipsoid(E, View)
    
    # 3. Duplicate visible ellipsoids with 3D keys relative to tiles
    L, K = DuplicateWithKeys3D(E_visible, T)
    
    # 4. Sort lists by tile keys (no intra-tile depth sorting)
    SortByTileKeys(K, L)
    
    # 5. Identify ranges of primitives per 3D tile
    R = IdentifyTileRanges(K)
    
    # 6. Initialize per-bin canvas
    I_bin = InitializeZero(w, h, b)
    
    # 7. Process tiles
    for t in T:
        # Extract primitives assigned to tile t
        start_idx = R[t].start
        end_idx = R[t].end
        E_t = L[start_idx:end_idx]
        
        # Process pixels in tile
        for i in PixelsInTile(t):
            # Compute ray intensity sample
            # Note: Source notation "IntensitySample(i, E, E)" implies sampling against primitives in E_t
            ray_val = IntensitySample(i, E_t, TF)
            
            # Blend partial ray result into per-bin canvas
            # t_z represents the depth bin index for tile t
            I_bin[i, t.z] = BlendPartialRay(ray_val, I_bin[i, t.z])
            
    # 8. Reset tile grid for final merge
    T_final = CreateTiles(w, h, 1)
    
    # 9. Merge Z-bins
    I = InitializeImage(w, h)
    for i in PixelsInImage(w, h):
        # Blend across b bins
        I[i] = MergeZBins(I_bin[i, 0:b])
        
    return I
```

**Parameters:**
*   $w, h$: Image dimensions [S1, unofficial].
*   $b$: Number of depth bins [S1, unofficial].
*   $TF$: Transfer function [S1, unofficial].
*   $View$: Camera view configuration [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The method explicitly states that no intra-tile depth sorting is performed [S1, unofficial]. This may lead to incorrect blending order if primitives within a tile overlap significantly in depth, requiring the blending function to handle depth-aware accumulation or the primitives to be pre-sorted by the caller [G].
*   Precision issues may arise when merging $b$ bins if the blending function is not associative or commutative in the presence of floating-point errors [G].

## Subtopics

### Depth Binning Strategy
A technique to partition ray space into discrete depth intervals, allowing batched primitive testing. It is chosen when depth coherence is high and primitives are sparse. Link to [Temporal Coherence and Caching](6372-temporal-coherence-and-caching.md).

### Tile-Based Primitive Assignment
Assigning primitives to 3D tiles based on spatial keys. It differs from pixel-based assignment by grouping rays, enabling coarser culling. Link to [Inter-Ray Coherency](1030-inter-ray-coherency.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Sorting by tile keys suggests $O(N \log N)$ for $N$ primitives, followed by linear traversal [S1, unofficial].
*   **Memory Complexity:** Requires storage for $I_{bin}$ of size $w \times h \times b$, plus lists $L$ and $K$ proportional to the number of visible primitives [S1, unofficial].
*   **Strengths:** Enables parallel accumulation of primitive contributions by grouping them into depth bins, potentially improving GPU occupancy [S1, unofficial].
*   **Limitations:** Lack of intra-tile depth sorting may compromise visual accuracy for overlapping primitives [S1, unofficial].
*   **When to Use:** Suitable for sparse volume rendering with ellipsoid primitives where depth coherence can be exploited [S1, unofficial].
*   **When Not to Use:** Not suitable for dense volumes or scenes with high depth complexity within tiles where sorting is critical [G].

## Background
The method originates from research on 3D ellipsoid-based sparse volume rendering, specifically leveraging structure-aware primitive learning and per-primitive ray sampling [S1, unofficial]. It was developed to accelerate rendering by partitioning rays into depth bins, allowing for efficient parallel processing on GPUs [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
