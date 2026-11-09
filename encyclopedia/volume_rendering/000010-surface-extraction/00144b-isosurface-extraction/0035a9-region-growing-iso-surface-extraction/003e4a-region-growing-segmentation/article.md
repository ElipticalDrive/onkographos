# Region Growing Segmentation

Also called: Region-based segmentation, Connected component analysis (in specific contexts)

## Overview
Region growing segmentation is a technique to extract a connected region from a 3D volume based on pre-defined connecting criteria, used to isolate a region of interest via a mask volume [S1, unofficial]. It solves the problem of identifying specific anatomical or material structures within a scalar field by expanding from seed points until boundary conditions are met. It sets itself apart from pixel-based methods like the Magic Wand tool [S5, unofficial] by operating directly on volumetric connectivity, and from global optimization methods like Min-Cut [9748-graph-based-isosurface-segmentation-min-cut.md] by relying on local homogeneity criteria rather than global energy minimization.

## How it works
**Inputs:**
- A 3D scalar volume $V$ of size $D_x \times D_y \times D_z$.
- A seed point (or set of seed points) $S_{seed}$.
- A target value range $[v_{min}, v_{max}]$ defining homogeneity [S1, unofficial].
- A connectivity criterion (typically 6-, 18-, or 26-neighbor connectivity) [S1, unofficial].

**Outputs:**
- A binary mask volume $M$ of the same dimensions as $V$, where $M[i,j,k] = 1$ if the voxel belongs to the region of interest, and $0$ otherwise [S1, unofficial].

**Data Structures:**
- **Queue/Stack:** To manage voxels pending expansion. A queue implements Breadth-First Search (BFS), while a stack implements Depth-First Search (DFS).
- **Visited Set:** A boolean volume or hash set to track processed voxels to prevent infinite loops and redundant processing.
- **Mask Volume:** The output binary volume.

**Steps:**
1. **Initialization:** Create an empty mask volume $M$ initialized to zeros. Initialize a `visited` set. Initialize a queue $Q$ with the seed point(s) $S_{seed}$. Mark seed points as visited.
2. **Expansion Loop:** While $Q$ is not empty:
    a. Dequeue a voxel $v = (x, y, z)$.
    b. Mark $M[x,y,z] = 1$.
    c. Identify all 6-connected (or specified connectivity) neighbors $n$ of $v$.
    d. For each neighbor $n$:
        i. Check if $n$ is within volume bounds.
        ii. Check if $n$ is not in `visited`.
        iii. Check if the scalar value $V[n]$ satisfies the homogeneity criterion: $v_{min} \le V[n] \le v_{max}$ [S1, unofficial].
        iv. If all conditions are met, add $n$ to `visited` and enqueue $n$.
3. **Termination:** The algorithm terminates when $Q$ is empty. The resulting $M$ contains the segmented region [S1, unofficial].

**Parameters:**
- **Connectivity:** 6-connected (face-sharing) is standard for strict connectivity; 26-connected (corner-sharing) allows diagonal growth [S1, unofficial].
- **Value Range:** $[v_{min}, v_{max}]$ must be defined based on the data's intensity distribution [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Leakage:** If the value range is too wide or boundaries are noisy, the region may "leak" through weak boundaries into adjacent structures [S1, unofficial]. This is mitigated by Restricted Region Growing [9750-restricted-region-growing.md].
- **Seed Selection:** Incorrect seed placement results in segmentation of the wrong component. Automated seed selection (e.g., using Otsu's thresholding) can help [S4, unofficial].
- **Memory:** For massive volumes, standard in-core queues may exceed memory limits. Out-of-core variants are required [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_grow_segmentation(volume: Volume, seeds: List[Tuple[int, int, int]], 
                             v_min: float, v_max: float, connectivity: int = 6) -> MaskVolume:
    """
    Segments a 3D volume using region growing.
    
    Args:
        volume: 3D array of scalar values.
        seeds: List of (x, y, z) coordinates for seed points.
        v_min: Lower bound of target intensity range.
        v_max: Upper bound of target intensity range.
        connectivity: Number of neighbors to consider (6, 18, or 26).
        
    Returns:
        Binary mask volume where 1 indicates segmented voxels.
    """
    dims = volume.shape
    mask = np.zeros(dims, dtype=np.uint8)
    visited = np.zeros(dims, dtype=bool)
    
    # Define neighbor offsets based on connectivity
    # 6-connected: +/- x, +/- y, +/- z
    # 18-connected: 6-connected + face diagonals
    # 26-connected: 18-connected + corner diagonals
    neighbors = get_neighbor_offsets(connectivity)
    
    queue = deque(seeds)
    for seed in seeds:
        visited[seed] = True
        
    while queue:
        x, y, z = queue.popleft()
        mask[x, y, z] = 1
        
        for dx, dy, dz in neighbors:
            nx, ny, nz = x + dx, y + dy, z + dz
            
            # Check bounds
            if not (0 <= nx < dims[0] and 0 <= ny < dims[1] and 0 <= nz < dims[2]):
                continue
                
            # Check visited
            if visited[nx, ny, nz]:
                continue
                
            # Check homogeneity
            val = volume[nx, ny, nz]
            if v_min <= val <= v_max:
                visited[nx, ny, nz] = True
                queue.append((nx, ny, nz))
                
    return mask

def get_neighbor_offsets(connectivity: int) -> List[Tuple[int, int, int]]:
    """Returns list of (dx, dy, dz) offsets for given connectivity."""
    if connectivity == 6:
        return [
            (1, 0, 0), (-1, 0, 0),
            (0, 1, 0), (0, -1, 0),
            (0, 0, 1), (0, 0, -1)
        ]
    elif connectivity == 18:
        base = get_neighbor_offsets(6)
        diagonals = [
            (1, 1, 0), (1, -1, 0), (-1, 1, 0), (-1, -1, 0),
            (1, 0, 1), (1, 0, -1), (-1, 0, 1), (-1, 0, -1),
            (0, 1, 1), (0, 1, -1), (0, -1, 1), (0, -1, -1)
        ]
        return base + diagonals
    elif connectivity == 26:
        base = get_neighbor_offsets(18)
        corners = [
            (1, 1, 1), (1, 1, -1), (1, -1, 1), (1, -1, -1),
            (-1, 1, 1), (-1, 1, -1), (-1, -1, 1), (-1, -1, -1)
        ]
        return base + corners
    else:
        raise ValueError("Connectivity must be 6, 18, or 26")
```

## Subtopics

### [Valley Cell Construction](9747-valley-cell-construction.md)
Decomposes the feature space into valley cells, each containing an isolated maximum, using region growing from maxima [9747-valley-cell-construction.md]. It sets itself apart by focusing on the topological decomposition of the scalar field's maxima rather than arbitrary seed points. Choose this when analyzing complex topological structures in scalar fields. See [Valley Cell Construction](9747-valley-cell-construction.md).

### [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md)
Segments voxels containing a specific isosurface by treating them as an interconnected network (graph) and finding an optimal cut to separate foreground and background seed points [9748-graph-based-isosurface-segmentation-min-cut.md]. It sets itself apart by using global energy minimization (min-cut) rather than local homogeneity, making it robust against noise and weak boundaries. Choose this when precise boundary adherence is critical and noise is present. See [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md).

### [Restricted Region Growing](9750-restricted-region-growing.md)
Defines rough locations for each material part by growing regions from seed voxels, restricted by noise distribution to prevent leaking through weak boundaries [9750-restricted-region-growing.md]. It sets itself apart by incorporating statistical noise models into the connectivity criterion. Choose this when dealing with noisy data where standard region growing would leak. See [Restricted Region Growing](9750-restricted-region-growing.md).

### Out-of-core Region Growing
An out-of-core region growing algorithm used to find a mask volume for segmentation of massive volumes (one or more orders of magnitude larger than free memory). It sets itself apart by using slab-projection slices and a directed acyclic graph (DAG) of brick distances to manage data that cannot fit in RAM. Choose this for petabyte-scale or similarly massive datasets. See [Out-of-core Region Growing](9750-restricted-region-growing.md) (Note: No dedicated article exists, but this is a variant described in).

### Watershed-based Morse-Smale Approximation
Approximates the discrete Morse-Smale complex by intersecting watershed segmentations of a scalar field $f$ and its complement $1-f$, avoiding explicit critical point extraction. It sets itself apart by providing a topological approximation without computing critical points. The approximation is valid only if $f$ is a Morse function, which is not generally true for discrete scalar fields. Choose this for topological analysis of scalar fields. See [Watershed-based Morse-Smale Approximation](9750-restricted-region-growing.md) (Note: No dedicated article exists, but this is a variant described in).

### Global/Local Segmentation
A segmentation approach that uses global criteria to avoid errors from local decision making, often by confining automatic segmentation to a user-specified mask. It sets itself apart by combining user interaction (masking) with global optimization (e.g., 2D dynamic programming) within that mask. Choose this when user guidance is acceptable and global optimality is desired. See [Global/Local Segmentation](9750-restricted-region-growing.md) (Note: No dedicated article exists, but this is a variant described in).

### Connected Component Segmentation
Applies Otsu's method to calculate an automatic threshold, creates a binary mask, labels connected components, and identifies the largest component as the final segmentation. It sets itself apart by automating seed selection via thresholding and component size analysis. Choose this for simple binary segmentation tasks where the largest object is the target. See [Connected Component Segmentation](9750-restricted-region-growing.md) (Note: No dedicated article exists, but this is a variant described in).

### Magic Wand Tool for Sample Selection
A 2D segmentation tool based on Perona and Malik's anisotropic diffusion, used to alleviate the difficulty of perfectly drawing over feature boundaries with a lasso tool. It sets itself apart by using partial differential equations (PDEs) for diffusion-based boundary adherence. Choose this for interactive 2D slice selection. See [Magic Wand Tool for Sample Selection](9750-restricted-region-growing.md) (Note: No dedicated article exists, but this is a variant described in).

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the volume, as each voxel is processed at most once [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the mask and visited arrays [S1, unofficial]. Out-of-core variants reduce memory usage to $O(B)$ where $B$ is the brick size [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with volume size in-core. Out-of-core methods scale with disk I/O bandwidth [S1, unofficial].
- **Strengths:** Simple to implement; fast for homogeneous regions; intuitive [S1, unofficial].
- **Limitations:** Prone to leakage in noisy data; sensitive to seed placement; requires manual parameter tuning for value ranges [S1, unofficial].
- **When to use:** When regions are homogeneous and boundaries are strong; when interactive seed placement is feasible.
- **When not to use:** When noise is high and boundaries are weak; when global optimality is required.

## Background
Region growing has been researched for many years as a fundamental technique for masking regions of interest in volumetric data [S1, unofficial]. It is commonly used in medical volume rendering to isolate anatomical structures [S1, unofficial]. The technique is foundational for many advanced segmentation methods, including those used in out-of-core visualization architectures [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
[S3] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
[S4] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S5] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
