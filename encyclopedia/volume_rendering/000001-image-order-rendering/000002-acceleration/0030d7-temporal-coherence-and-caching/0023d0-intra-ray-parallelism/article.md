# Intra-ray Parallelism

Also called: GPU-acceleration through intra-ray parallelism

## Overview
Intra-ray parallelism is a rendering acceleration scheme that partitions each ray into depth bins to enable efficient parallel accumulation of primitive contributions during rendering [S1, unofficial]. It differs from sibling methods like [Inter-Ray Coherency](1030-inter-ray-coherency.md) by exploiting parallelism within a single ray's traversal path rather than across multiple rays, and from [Temporal Coherence and Caching](6372-temporal-coherence-and-caching.md) by relying on spatial binning and sorting rather than temporal state reuse.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Image dimensions $w, h$; number of depth bins $b$; set of ellipsoid primitives $E$ (defined by mean $\mu$ and covariance $\Sigma$); transfer function $TF$; camera view configuration $View$.
*   **Outputs:** Rendered image $I$.

**Data Structures:**
*   **Tiles ($T$):** 3D grid cells defined by image coordinates $(x, y)$ and depth bin index $z \in [0, b-1]$.
*   **Lists ($L, K$):** Parallel arrays storing primitive data and 3D keys (tile identifiers) respectively.
*   **Ranges ($R$):** Start and end indices in $L$ for each tile.
*   **Per-bin Canvas ($I_{bin}$):** A 3D buffer of size $w \times h \times b$, initialized to zero, storing partial ray results.

**Steps:**
1.  **Tile Creation:** Create tiles $T$ from image dimensions $w, h$ and bin count $b$. Tiles include depth-wise subdivisions [S1, unofficial].
2.  **Culling:** Cull ellipsoids $E$ based on the current $View$ to produce visible ellipsoids $E_{visible}$ [S1, unofficial].
3.  **Key Assignment:** Duplicate visible ellipsoids with 3D keys relative to tiles $T$ to produce lists $L$ and $K$ [S1, unofficial].
4.  **Sorting:** Sort lists $K$ and $L$ by tile keys. Note: No intra-tile depth sorting is performed [S1, unofficial].
5.  **Range Identification:** Identify ranges $R$ for each tile $t$, defining the range of primitives assigned to that 3D tile [S1, unofficial].
6.  **Initialization:** Initialize per-bin canvas $I_{bin}$ to 0 [S1, unofficial].
7.  **Tile Processing:** Loop over all tiles $t$ in $T$:
    *   Extract primitives $E_t$ assigned to tile $t$ from list $L$ using range $R[t]$ [S1, unofficial].
    *   For all pixels $i$ in tile $t$:
        *   Compute ray $ray[i, t_z]$ through the tile's depth bin [S1, unofficial].
        *   Sample intensity from primitives $E_t$ along the ray [S1, unofficial].
        *   Blend the partial ray result into the per-bin canvas: $I_{bin}[i, t_z] \leftarrow \text{BlendPartialRay}(ray[i, t_z])$ [S1, unofficial].
8.  **Reset:** After processing all tiles, reset tile grid $z=1$ by creating new tiles $T$ with bin count $1$: $T \leftarrow \text{CreateTiles}(w, h, 1)$ [S1, unofficial].
9.  **Merging:** Loop over all pixels $i$ in image $I$: Merge Z-bins by blending across $b$ bins: $I[i] \leftarrow \text{MergeZBins}(I_{bin}[i, 0:b])$ [S1, unofficial].

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
    
    # 7. Process each tile
    for t in T:
        # Extract primitives assigned to tile t
        start_idx = R[t].start
        end_idx = R[t].end
        E_t = L[start_idx:end_idx]
        
        # Process pixels in the tile
        for i in PixelsInTile(t):
            # Compute ray for this pixel and depth bin
            ray = IntensitySample(i, E_t, E_t) # Note: Source notation implies sampling from E_t
            # Blend partial result into per-bin canvas
            I_bin[i, t.z] = BlendPartialRay(ray)
            
    # 8. Reset tile grid for merging
    T_merged = CreateTiles(w, h, 1)
    
    # 9. Merge Z-bins across all pixels
    I = EmptyImage(w, h)
    for i in PixelsInImage(w, h):
        I[i] = MergeZBins(I_bin[i, 0:b])
        
    return I
```

**Parameters:**
*   $w, h$: Width and height of the image to rasterize [S1, unofficial].
*   $b$: Number of depth bins [S1, unofficial].
*   $TF$: Transfer function to map intensity to color and opacity [S1, unofficial].
*   $View$: View configuration of current camera [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on accurate 3D key assignment for sorting. Incorrect key generation may lead to primitives being assigned to wrong tiles.
*   **Boundaries:** The culling step must correctly handle ellipsoids intersecting tile boundaries to ensure no contributions are missed or double-counted.
*   **Degenerate Input:** If $b=1$, the method reduces to standard tile-based rendering without intra-ray binning. If $E$ is empty, the output is black.

## Operating characteristics
*   **Time Complexity:** Dominated by sorting $K$ and $L$, which is $O(N \log N)$ where $N$ is the number of visible primitives. Tile processing is linear in the number of primitives per tile.
*   **Memory Complexity:** Requires storage for $L, K$ ($O(N)$), $I_{bin}$ ($O(w \times h \times b)$), and $R$ ($O(|T|)$).
*   **Strengths:** Efficiently utilizes GPU parallelism by processing multiple depth bins and primitives simultaneously within tiles.
*   **Limitations:** Requires significant memory for $I_{bin}$ if $b$ is large. No intra-tile depth sorting may affect blending accuracy in some cases.
*   **When to Use:** Suitable for real-time rendering of large-scale volumetric data with many primitives, where intra-ray parallelism can be exploited.
*   **When Not to Use:** Not suitable for scenes with very few primitives or where memory bandwidth is a bottleneck due to large $b$.

## Background
Intra-ray parallelism was introduced in the context of 3D Ellipsoid-based Sparse Volume Rendering (ESVR) [S1, unofficial]. The method aims to accelerate rendering by partitioning rays into depth bins, allowing for parallel accumulation of contributions. It is part of a broader effort to optimize volume rendering on GPUs by leveraging spatial coherence and parallel processing capabilities.

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
