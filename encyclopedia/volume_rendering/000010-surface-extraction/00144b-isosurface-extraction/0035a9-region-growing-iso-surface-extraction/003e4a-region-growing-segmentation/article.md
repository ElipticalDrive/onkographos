# Region Growing Segmentation

Also called: Region-based segmentation, Seed-based segmentation

## Overview
Region growing segmentation extracts a connected region of interest (ROI) from a 3D volume by expanding from a seed point based on pre-defined connectivity criteria, such as value range or gradient thresholds [S1, unofficial]. It differs from global optimization methods like [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md) by relying on local homogeneity checks rather than global energy minimization, and from [Watershed Segmentation](13875-watershed-segmentation.md) by growing from user or algorithmically defined seeds rather than flooding from topological minima [S1, unofficial][S6]. The primary output is a binary mask volume used to isolate specific anatomical structures or materials for subsequent volume rendering or analysis [S1, unofficial].

## How it works
**Inputs:**
*   $V$: Input 3D scalar volume (e.g., CT, MRI, or simulation data).
*   $seed$: A single voxel coordinate $(x_s, y_s, z_s)$ or a set of seed voxels.
*   $criteria$: A function or threshold defining connectivity. Common criteria include:
    *   Value range: $|V(p) - V(seed)| \leq T_{val}$ [S1, unofficial].
    *   Gradient threshold: $|\nabla V(p)| \leq T_{grad}$ [S1, unofficial].
    *   Statistical similarity: $V(p)$ falls within the standard deviation of the seed region [S1, unofficial].

**Outputs:**
*   $M$: Binary mask volume of the same dimensions as $V$, where $M(p) = 1$ if voxel $p$ belongs to the ROI, and $0$ otherwise.

**Data Structures:**
*   $M$: 3D array of booleans or uint8, initialized to 0.
*   $Q$: Queue (FIFO) or Stack (LIFO) for voxels to be processed.
*   $visited$: Boolean array or set to track processed voxels to prevent infinite loops and redundant checks.

**Algorithm Steps:**
1.  **Initialization:**
    *   Initialize mask $M$ to all zeros.
    *   Initialize $visited$ to all false.
    *   Push $seed$ into $Q$.
    *   Mark $seed$ as visited.
    *   Mark $M(seed) = 1$.

2.  **Expansion Loop:**
    *   While $Q$ is not empty:
        *   Pop current voxel $p$ from $Q$.
        *   For each neighbor $n$ of $p$ (typically 6-connected or 26-connected in 3D):
            *   If $n$ is within volume bounds:
                *   If $n$ is not visited:
                    *   Evaluate $criteria(p, n)$ (e.g., check if $V(n)$ is within the target range relative to $V(seed)$ or $V(p)$).
                    *   If $criteria$ is met:
                        *   Mark $n$ as visited.
                        *   Mark $M(n) = 1$.
                        *   Push $n$ into $Q$.

3.  **Termination:**
    *   Return $M$.

**Parameters:**
*   $T_{val}$: Tolerance for intensity difference. Typical values depend on the data type (e.g., Hounsfield Units in CT). Not given in the sources.
*   Connectivity: 6, 18, or 26 neighbors. 6-connected is standard for strict connectivity; 26-connected allows diagonal growth [S1, unofficial].
*   Seed selection: Can be manual (user click) or automatic (e.g., finding local maxima) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Leaking:** If the criteria are too loose, the region may "leak" through weak boundaries (e.g., noise or low-contrast interfaces) [S1, unofficial]. This is mitigated by using stricter gradient-based criteria or [Restricted Region Growing](9750-restricted-region-growing.md) which restricts seeds to areas with low gradients [S1, unofficial].
*   **Precision:** Floating-point comparisons should use epsilon tolerances.
*   **Boundaries:** Neighbors outside the volume must be checked explicitly to avoid out-of-bounds errors.
*   **Degenerate Input:** If no neighbors meet the criteria, the region remains a single voxel.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_grow_segmentation(volume: np.ndarray, seed: tuple, tolerance: float, connectivity: int = 6) -> np.ndarray:
    """
    Performs region growing segmentation on a 3D volume.
    
    Args:
        volume: 3D numpy array of scalar values.
        seed: Tuple (x, y, z) of the starting voxel.
        tolerance: Maximum allowed difference in intensity between seed and neighbor.
        connectivity: Number of neighbors to consider (6, 18, 26).
        
    Returns:
        Binary mask array.
    """
    shape = volume.shape
    mask = np.zeros(shape, dtype=np.uint8)
    visited = np.zeros(shape, dtype=bool)
    
    # Queue for BFS (FIFO)
    queue = [seed]
    visited[seed] = True
    mask[seed] = 1
    
    # Define neighbors based on connectivity
    # 6-connected: +/- x, y, z
    # 18-connected: 6-connected + face diagonals
    # 26-connected: 18-connected + corner diagonals
    if connectivity == 6:
        offsets = [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1)]
    elif connectivity == 18:
        offsets = [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1),
                   (-1,-1,0), (-1,1,0), (1,-1,0), (1,1,0),
                   (-1,0,-1), (-1,0,1), (1,0,-1), (1,0,1),
                   (0,-1,-1), (0,-1,1), (0,1,-1), (0,1,1)]
    elif connectivity == 26:
        offsets = [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1),
                   (-1,-1,0), (-1,1,0), (1,-1,0), (1,1,0),
                   (-1,0,-1), (-1,0,1), (1,0,-1), (1,0,1),
                   (0,-1,-1), (0,-1,1), (0,1,-1), (0,1,1),
                   (-1,-1,-1), (-1,-1,1), (-1,1,-1), (-1,1,1),
                   (1,-1,-1), (1,-1,1), (1,1,-1), (1,1,1)]
    else:
        raise ValueError("Connectivity must be 6, 18, or 26")

    seed_val = volume[seed]
    
    while queue:
        current = queue.pop(0) # BFS
        # current_val = volume[current] # Optional: use current voxel value for local criteria
        
        for dx, dy, dz in offsets:
            nx, ny, nz = current[0] + dx, current[1] + dy, current[2] + dz
            
            # Check bounds
            if not (0 <= nx < shape[0] and 0 <= ny < shape[1] and 0 <= nz < shape[2]):
                continue
                
            if visited[nx, ny, nz]:
                continue
                
            # Check criteria: intensity difference from seed
            if abs(volume[nx, ny, nz] - seed_val) <= tolerance:
                visited[nx, ny, nz] = True
                mask[nx, ny, nz] = 1
                queue.append((nx, ny, nz))
                
    return mask
```

## Subtopics

### [Valley Cell Construction](9747-valley-cell-construction.md)
Valley Cell Construction decomposes a feature space (typically a 2D density plot or scalar field) into non-overlapping regions called valley cells, where each cell contains exactly one isolated local maximum. It solves the problem of structuring the feature space for clustering or segmentation by identifying the boundaries between basins of attraction for local maxima. It differs from [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md) by using topological region growing from maxima rather than global cut optimization, and from [Restricted Region Growing](9750-restricted-region-growing.md) by explicitly handling topological degeneracies (multi-maximum saddles) via node duplication to ensure a valid cell decomposition.

### [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md)
This method segments a 3D volume by treating voxels containing a specific isosurface as nodes in a graph, connected via neighborhood connectivity, and finding an optimal cut to separate user-defined foreground and background seed regions. Unlike traditional volume segmentation which operates on the entire voxel grid and suffers from label volume precision limits and computational expense, this approach restricts the graph to the sparse subset of voxels intersecting the isosurface, integrating directly with isosurface rendering pipelines. It is distinguished from its siblings by its explicit formulation of the segmentation problem as a min-cut optimization on the isosurface domain, whereas [Valley Cell Construction](9747-valley-cell-construction.md) focuses on geometric valley detection and [Restricted Region Growing](9750-restricted-region-growing.md) relies on iterative classification of doubt voxels.

### [Restricted Region Growing](9750-restricted-region-growing.md)
Restricted Region Growing is a segmentation technique that defines rough locations for each material part in a volumetric model by growing regions from seed voxels, restricted by noise distribution to prevent leaking through weak boundaries. It serves as a fast pre-processing step to classify the main bodies of materials, thereby reducing the graph size for subsequent Graph-cut operations. Unlike general region growing, which is prone to leaking through weak spots in object boundaries, this method ensures that selected initial seeds have neighborhoods with similar CT values and low gradients, preventing selection in boundary areas.

### [Watershed Segmentation](13875-watershed-segmentation.md)
Watershed Segmentation is a region-growing technique used to extract connected regions from 3D volumes, specifically applied here for brain mask extraction and vessel mask extraction in MRI data. It isolates regions of interest by treating the volume as a topographic landscape, where intensity values define elevation. Unlike [Restricted Region Growing](9750-restricted-region-growing.md) which relies on pre-defined connectivity criteria, this method leverages the gradient structure of the image to define boundaries. It serves as a preprocessing step for volume rendering, enabling the blending of anatomical details with pathological tissue indicators.

### Out-of-core Region Growing
An out-of-core region growing algorithm used to find a mask volume for segmentation of massive volumes (one or more orders of magnitude larger than free memory) [S1, unofficial]. It uses slab-projection slices and out-of-core bricking with a directed acyclic graph (DAG) of relative brick distances to manage data that exceeds available RAM [S1, unofficial].

### Watershed-based Morse-Smale Approximation
Approximates the discrete Morse-Smale complex by intersecting watershed segmentations of a scalar field and its complement, avoiding explicit critical point extraction [S2, unofficial]. Steps: 1. Perform watershed segmentation on the normalized discrete scalar field $f$. 2. Perform watershed segmentation on the inverted field $1-f$. 3. Compute the intersection of the two resulting watershed segmentations to approximate the discrete Morse-Smale complex. The approximation is valid only if $f$ is a Morse function, which is not generally true for discrete scalar fields [S2, unofficial].

### Global/Local Segmentation
A segmentation approach that uses global criteria to avoid errors from local decision making, often by confining automatic segmentation to a user-specified mask [S3]. Steps: 1. User interactively indicates a subset of $V$ (mask) for automatic segmentation. 2. Automatic segmentation is confined to this mask. 3. Optionally, automatically find globally optimal boundaries using 2D dynamic programming within the mask. A shell around the boundary is often all that is needed to minimize user interaction time [S3]. Considerable success is reported in automatically finding globally optimal boundaries using 2D dynamic programming when the search is confined to a mask [S3].

### Connected Component Segmentation
Applies Otsu's method to calculate an automatic threshold value for the volume data. Creates a binary segmentation mask where voxels greater than the threshold are 1, others 0. Labels connected components in the binary mask using connectivity=1 and returns the number of labels. Identifies the largest connected component by area among all labeled regions. Creates the final spine segmentation mask by checking if labeled voxels equal the largest region label, casting to uint8 [S4, unofficial].

### Magic Wand Tool for Sample Selection
A 2D segmentation tool based on Perona and Malik's anisotropic diffusion, used to alleviate the difficulty of perfectly drawing over feature boundaries with a lasso tool [S5, unofficial]. Solves the diffusion equation $\frac{\partial S(t,x,y)}{\partial t} = \text{div}(g(|\nabla V(t,x,y)|) \nabla S(t,x,y))$ numerically using the finite difference method with a given iteration number $T$. The conductivity term is $g(s) = v \cdot \exp(-s^2 / \kappa^2)$, where $s$ is the gradient magnitude. Parameter $K$ ($\kappa$) governs how fast $g(s)$ goes to zero for high gradients. Regular term $v$ is 1. Normalization term $h$ is $1/(n+1)$ for numerical stability, where $n=8$ [S5, unofficial].

### Connected Component Labeling for Volume Extraction
Classifies each voxel in the source volume into either a cloud voxel or an empty voxel. If the density $\rho$ at voxel $p$ is greater than a user-specified threshold $\epsilon$, it is classified as a cloud voxel. Applies a labeling process to the classified volume to assign unique labels to each region consisting of connected cloud voxels. Creates a bounding box for each connected region and extracts the volume enclosed by the bounding box as the cloud volume [S6].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the volume, as each voxel is visited at most once. However, the actual runtime depends heavily on the size of the resulting region and the connectivity check overhead.
*   **Memory Complexity:** $O(N)$ for the mask and visited arrays. For out-of-core variants, memory usage is reduced to fit within available RAM by processing slabs or bricks [S1, unofficial].
*   **Performance:** Fast for small to medium regions. Performance degrades if the region is large and connectivity is high (26-connected). Out-of-core methods add I/O overhead but enable processing of massive datasets [S1, unofficial].
*   **Strengths:** Simple to implement; intuitive for users; effective for homogeneous regions; can be restricted to prevent leaking [S1, unofficial].
*   **Limitations:** Prone to leaking through weak boundaries if criteria are not strict; sensitive to seed placement; may require manual intervention for complex structures [S1, unofficial].
*   **When to use:** When a user can provide a reliable seed point; when regions are homogeneous; as a pre-processing step for more complex segmentation [S1, unofficial].
*   **When not to use:** When boundaries are weak or noisy; when global context is required to resolve ambiguities; when the region is not connected [S1, unofficial].

## Background
Region growing is a classical image segmentation technique that has been widely used in medical imaging and volume visualization for decades. It was introduced as a method to isolate regions of interest based on local homogeneity criteria [S1, unofficial]. The technique is foundational in volume rendering pipelines, where it is used to create mask volumes that restrict rendering to specific anatomical structures or materials [S1, unofficial]. Variants such as out-of-core region growing have been developed to handle massive datasets that exceed available memory [S1, unofficial]. The method is often combined with other techniques, such as [Restricted Region Growing](9750-restricted-region-growing.md) for pre-processing or [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md) for refinement [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
[S3] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
[S4] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S5] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S6] A sketch-based system for cloud volume retrieval from simulated dataset for realistic image synthesis, https://doi.org/10.1145/2817675.2817690, fetched 2026-11-05
