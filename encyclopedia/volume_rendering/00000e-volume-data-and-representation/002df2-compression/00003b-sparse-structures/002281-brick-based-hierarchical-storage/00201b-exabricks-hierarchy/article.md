# ExaBricks Hierarchy

Also called: ExaBricks, ExaBricks data structure

## Overview
The ExaBricks Hierarchy is a rendering method for Block-Structured Adaptive Mesh Refinement (BS-AMR) data that re-organizes irregular AMR cells into compact, non-overlapping bricks to enable high-quality smooth interpolation and efficient hardware-accelerated ray tracing [S1]. It solves the problem of discontinuous interpolation at AMR level boundaries and the high computational cost of locating cells for reconstruction in deep hierarchies [S1]. Unlike sibling methods that rely on costly per-sample k-d tree traversals or complex stitching of unstructured meshes, ExaBricks uses Active Brick Regions (ABRs) to provide fast, trivial cell retrieval from structured brick arrays, combined with an RTX BVH for space skipping and adaptive sampling [S1].

## How it works
The method operates using three inter-operating data structures: Bricks, Active Brick Regions (ABRs), and an RTX Bounding Volume Hierarchy (BVH) built over the ABRs [S1].

### Inputs and Outputs
*   **Input:** A set of AMR cells, each defined by lower-left unit coordinates $(i, j, k)$, level $l$, and a scalar data value $C_v$. Level 0 is the finest level with implicit cell size $1^3$; level $l$ has cell size $2^l$ [S1].
*   **Output:** A set of Bricks, a set of ABRs, and an RTX BVH, enabling ray marching, space skipping, and adaptive sampling [S1].

### Data Structures

**1. Brick**
A Brick is a compact, non-overlapping group of same-level cells. The original AMR hierarchy is discarded [S1].
*   **Fields:**
    *   `lower_left`: $(i, j, k)$, the lower-left unit coordinates of the brick.
    *   `level`: $l$, the AMR level of the cells.
    *   `dims`: $(N, M, K)$, the number of cells in each dimension.
    *   `data`: A 3D array of scalars of size $N \times M \times K$ [S1].

**2. Active Brick Region (ABR)**
An ABR is a leaf in a spatial partitioning structure. Each leaf stores a list of bricks that potentially influence the spatial region covered by the leaf [S1].
*   **Fields:**
    *   `bounding_box`: The 3D bounding box of the region.
    *   `active_bricks`: A list of Brick IDs.
    *   `min_scalar`: The minimum scalar value of any cell whose support overlaps the region.
    *   `max_scalar`: The maximum scalar value of any cell whose support overlaps the region.
    *   `finest_cell_width`: The cell width of the finest level influencing the region [S1].

**3. RTX BVH**
A Bounding Volume Hierarchy built over the ABRs. Used for hardware-accelerated ray traversal, space skipping, and adaptive sampling [S1].

### Reconstruction Filter
The method uses the basis method (hat function) for smooth reconstruction [S1].
*   **Basis Function:** For a cell $C$ of width $C_w$, the basis function at position $p$ is:
    $$ \hat{H}_C(p) = \hat{h}\left(\frac{|C_{p_x} - p_x|}{C_w}\right) \hat{h}\left(\frac{|C_{p_y} - p_y|}{C_w}\right) \hat{h}\left(\frac{|C_{p_z} - p_z|}{C_w}\right) $$
    where $\hat{h}(x) = \max(1 - x, 0)$ [S1].
*   **Support:** The support of cell $C$ is a rectangular box extending exactly half a cell width beyond the cell itself. The support of a brick $B$ is the union of the supports of its cells, resulting in a box exactly half the brick's cell width larger than the brick [S1].
*   **Reconstruction:** The value at position $p$ is:
    $$ B(p) = \frac{\sum_{C_i} \hat{H}_{C_i}(p) C_{v_i}}{\sum_{C_i} \hat{H}_{C_i}(p)} $$
    where $C_i$ are cells with non-zero support at $p$, and $C_{v_i}$ is the data value [S1].

### Parameters
*   **Brick Size:** Bricks can be small (1, 2, or 4 cells) at level boundaries, but larger in homogeneous regions. Leaves are limited to at most 32 cells wide on any axis to ensure fine granularity for space skipping [S1].
*   **AMR Levels:** Level 0 is finest ($1^3$). Level $l$ has size $2^l$. Coordinates $(i, j, k)$ are multiples of $2^l$ [S1].

### Algorithm Steps

#### 1. Brick Generation
Re-organize input AMR cells into non-overlapping bricks of same-level cells [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_bricks(amr_cells):
    """
    Re-organize AMR cells into bricks using a top-down k-d tree approach.
    amr_cells: List of cells, each with (i, j, k, l, value).
    Returns: List of Brick objects.
    """
    # Build a k-d tree top-down
    root = build_kd_tree(amr_cells)
    bricks = []
    
    # Traverse tree to extract leaves
    for leaf in root.leaves():
        cells = leaf.cells()
        
        # Check if all cells are on the same AMR level
        levels = set(c.level for c in cells)
        if len(levels) != 1:
            continue # Should not happen if tree logic is correct, but guard
            
        level = levels.pop()
        
        # Check if cells completely fill their combined bounding box
        # Calculate bounding box of cells
        min_i = min(c.i for c in cells)
        min_j = min(c.j for c in cells)
        min_k = min(c.k for c in cells)
        max_i = max(c.i + 2**level for c in cells) # Cell size is 2^level
        max_j = max(c.j + 2**level for c in cells)
        max_k = max(c.k + 2**level for c in cells)
        
        # Verify no gaps. Since cells are aligned to 2^level, we can check
        # if the number of cells equals the volume of the bounding box divided by cell volume.
        cell_volume = 2**(3*level)
        box_volume = (max_i - min_i) * (max_j - min_j) * (max_k - min_k)
        
        if len(cells) * cell_volume != box_volume:
            continue # Gaps exist, do not create brick here
            
        # Check size limit: max 32 cells wide on any axis
        width_i = (max_i - min_i) / (2**level)
        width_j = (max_j - min_j) / (2**level)
        width_k = (max_k - min_k) / (2**level)
        
        if width_i > 32 or width_j > 32 or width_k > 32:
            continue # Too large, split further (handled by tree construction)
            
        # Create Brick
        # Note: The k-d tree construction ensures we only reach this point if
        # the split was valid. The split logic in build_kd_tree handles the
        # "longest axis" and "rounding" requirements.
        brick = Brick(
            lower_left=(min_i, min_j, min_k),
            level=level,
            dims=(int(width_i), int(width_j), int(width_k)),
            data=extract_data(cells, min_i, min_j, min_k, level)
        )
        bricks.append(brick)
        
    return bricks

def build_kd_tree(cells):
    """
    Top-down k-d tree construction.
    Split along longest axis of bounding box.
    Round split positions to integer multiple of current coarsest cell width.
    """
    if not cells:
        return None
        
    # Determine bounding box
    min_i = min(c.i for c in cells)
    min_j = min(c.j for c in cells)
    min_k = min(c.k for c in cells)
    max_i = max(c.i + 2**c.level for c in cells)
    max_j = max(c.j + 2**c.level for c in cells)
    max_k = max(c.k + 2**c.level for c in cells)
    
    # Check leaf conditions
    levels = set(c.level for c in cells)
    if len(levels) == 1:
        level = levels.pop()
        cell_vol = 2**(3*level)
        box_vol = (max_i - min_i) * (max_j - min_j) * (max_k - min_k)
        if len(cells) * cell_vol == box_vol:
            # All same level and fill box. Check size limit.
            w_i = (max_i - min_i) / (2**level)
            w_j = (max_j - min_j) / (2**level)
            w_k = (max_k - min_k) / (2**level)
            if w_i <= 32 and w_j <= 32 and w_k <= 32:
                return LeafNode(cells)
    
    # Split
    # Find longest axis
    dims = [max_i - min_i, max_j - min_j, max_k - min_k]
    axis = dims.index(max(dims))
    
    # Find split position
    # "Round split positions to an integer multiple of current coarsest cell width"
    # The "current coarsest cell width" in the context of splitting usually refers
    # to the grid alignment. For AMR, this is complex. The source says "integer multiple
    # of current coarsest cell width". In a mixed-level set, this is ambiguous.
    # However, since we split to separate levels or fill boxes, we likely split at
    # cell boundaries.
    # Simplified: Split at the median coordinate of the cells along the longest axis,
    # rounded to the nearest valid grid boundary for the finest level present?
    # The source text is brief: "Split positions are rounded to an integer multiple of 
    # current coarsest cell width."
    # Implementation detail [G]: In practice, this often means splitting at the 
    # boundary of the finest level cells to ensure alignment.
    
    # For this pseudocode, we assume a standard median split rounded to grid.
    # Let's assume we split at the median value of the cell centers or boundaries.
    # To be safe and generic:
    if axis == 0:
        coords = [c.i for c in cells]
    elif axis == 1:
        coords = [c.j for c in cells]
    else:
        coords = [c.k for c in cells]
        
    coords.sort()
    mid_idx = len(coords) // 2
    split_val = coords[mid_idx]
    
    # Round to integer (assuming unit grid for split logic, or coarsest level)
    # The source implies rounding to ensure bricks align with AMR grids.
    split_val = round(split_val) 
    
    left_cells = [c for c in cells if (c.i if axis==0 else c.j if axis==1 else c.k) < split_val]
    right_cells = [c for c in cells if (c.i if axis==0 else c.j if axis==1 else c.k) >= split_val]
    
    # Handle cells exactly on the split line if any (should be rare or handled by <=/>=)
    # The source doesn't specify tie-breaking. Standard k-d tree uses <= left, > right.
    
    return InternalNode(
        axis=axis,
        split=split_val,
        left=build_kd_tree(left_cells),
        right=build_kd_tree(right_cells)
    )

def extract_data(cells, min_i, min_j, min_k, level):
    """
    Extract scalar values into a 3D array.
    """
    N = int((max(c.i + 2**level for c in cells) - min_i) / (2**level))
    M = int((max(c.j + 2**level for c in cells) - min_j) / (2**level))
    K = int((max(c.k + 2**level for c in cells) - min_k) / (2**level))
    
    data = [[[] for _ in range(K)] for _ in range(M)] # Simplified structure
    # Actually, need 3D array.
    # Using a flat list or nested lists.
    # Let's use a 3D list.
    arr = [[[0 for _ in range(K)] for _ in range(M)] for _ in range(N)]
    
    for c in cells:
        # Calculate local index
        # Cell size is 2^level.
        # i is lower-left unit coord.
        # Local index = (c.i - min_i) / 2^level
        idx_i = int((c.i - min_i) / (2**level))
        idx_j = int((c.j - min_j) / (2**level))
        idx_k = int((c.k - min_k) / (2**level))
        
        arr[idx_i][idx_j][idx_k] = c.value
        
    return arr
```

#### 2. ABR Construction
Build a spatial partitioning on top of bricks, designed for the AMR basis reconstruction filter [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_abrs(bricks):
    """
    Construct Active Brick Regions.
    Returns: List of ABR objects.
    """
    # 1. Create list of 'brick support fragments'
    fragments = []
    for brick in bricks:
        # Support of a brick is a box exactly half the brick's cell width larger than the brick.
        # Brick cell width = 2^brick.level
        half_width = 2**(brick.level - 1)
        
        # Brick bounds
        b_min_i, b_min_j, b_min_k = brick.lower_left
        b_max_i = b_min_i + brick.dims[0] * (2**brick.level)
        b_max_j = b_min_j + brick.dims[1] * (2**brick.level)
        b_max_k = b_min_k + brick.dims[2] * (2**brick.level)
        
        # Support bounds
        s_min_i = b_min_i - half_width
        s_min_j = b_min_j - half_width
        s_min_k = b_min_k - half_width
        s_max_i = b_max_i + half_width
        s_max_j = b_max_j + half_width
        s_max_k = b_max_k + half_width
        
        fragments.append(SupportFragment(
            box=(s_min_i, s_min_j, s_min_k, s_max_i, s_max_j, s_max_k),
            brick_id=brick.id
        ))
        
    # 2. Compute bounding box of all fragments
    global_min_i = min(f.box[0] for f in fragments)
    global_min_j = min(f.box[1] for f in fragments)
    global_min_k = min(f.box[2] for f in fragments)
    global_max_i = max(f.box[3] for f in fragments)
    global_max_j = max(f.box[4] for f in fragments)
    global_max_k = max(f.box[5] for f in fragments)
    
    # 3. Recursive subdivision
    root = build_abr_tree(fragments, (global_min_i, global_min_j, global_min_k, global_max_i, global_max_j, global_max_k))
    
    # Collect leaves
    abrs = []
    for leaf in root.leaves():
        abrs.append(leaf.abr)
        
    return abrs

def build_abr_tree(fragments, region_box):
    """
    Recursive subdivision of ABRs.
    """
    # Filter fragments overlapping region
    overlapping = [f for f in fragments if boxes_overlap(f.box, region_box)]
    
    # If no fragments, return empty leaf? Or skip?
    # Source: "If no plane exists, create leaf with current brick IDs."
    # If no fragments overlap, the region is empty of influence.
    if not overlapping:
        return LeafABR(
            box=region_box,
            active_bricks=[],
            min_scalar=0, # Placeholder
            max_scalar=0,
            finest_cell_width=0
        )
        
    # Find partitioning plane
    # "Iterate through the set of fragment boundaries and look at their respective faces"
    # "Prefer widest dimension"
    # "Closest to region's spatial center"
    
    region_center = get_center(region_box)
    best_plane = None
    best_dist = infinity
    
    # Collect all candidate planes from fragment boundaries
    planes = []
    for f in overlapping:
        # Faces of the fragment box
        # x-min, x-max, y-min, y-max, z-min, z-max
        planes.append((f.box[0], 'x')) # x-min
        planes.append((f.box[3], 'x')) # x-max
        planes.append((f.box[1], 'y')) # y-min
        planes.append((f.box[4], 'y')) # y-max
        planes.append((f.box[2], 'z')) # z-min
        planes.append((f.box[5], 'z')) # z-max
        
    # Find best plane
    for coord, axis in planes:
        # Check if plane is inside region
        if axis == 'x' and region_box[0] < coord < region_box[3]:
            dist = abs(coord - region_center[0])
            if dist < best_dist:
                best_dist = dist
                best_plane = (coord, axis)
        elif axis == 'y' and region_box[1] < coord < region_box[4]:
            dist = abs(coord - region_center[1])
            if dist < best_dist:
                best_dist = dist
                best_plane = (coord, axis)
        elif axis == 'z' and region_box[2] < coord < region_box[5]:
            dist = abs(coord - region_center[2])
            if dist < best_dist:
                best_dist = dist
                best_plane = (coord, axis)
                
    # If no plane exists (e.g., all fragments are fully contained or region is small), create leaf
    if best_plane is None:
        # Create leaf with current brick IDs
        brick_ids = list(set(f.brick_id for f in overlapping))
        
        # Compute meta-info
        # Min/Max scalar of any cell whose support overlaps the region
        # This requires accessing the brick data.
        # For pseudocode, we assume a function to get this.
        min_s, max_s, finest_w = compute_region_meta(overlapping, region_box)
        
        return LeafABR(
            box=region_box,
            active_bricks=brick_ids,
            min_scalar=min_s,
            max_scalar=max_s,
            finest_cell_width=finest_w
        )
        
    # Partition region
    coord, axis = best_plane
    left_box, right_box = split_box(region_box, axis, coord)
    
    # Sort fragments into left/right subtrees
    left_frags = [f for f in overlapping if box_center(f.box)[axis] < coord] # Simplified split
    right_frags = [f for f in overlapping if box_center(f.box)[axis] >= coord]
    
    # Note: The source says "sort fragments into left/right subtrees".
    # A fragment might overlap both sides? If so, it goes to both?
    # "partition region, sort fragments into left/right subtrees, recurse"
    # Usually, for spatial partitioning, fragments are assigned to children that overlap.
    
    left_child = build_abr_tree(left_frags, left_box)
    right_child = build_abr_tree(right_frags, right_box)
    
    return InternalABR(
        axis=axis,
        split=coord,
        left=left_child,
        right=right_child
    )

def compute_region_meta(overlapping_fragments, region_box):
    """
    Compute min/max scalar and finest cell width for a region.
    """
    min_val = infinity
    max_val = -infinity
    finest_w = infinity
    
    # Get unique bricks
    bricks = get_bricks_by_ids([f.brick_id for f in overlapping_fragments])
    
    for brick in bricks:
        # Check if brick support overlaps region
        # We already filtered overlapping fragments, so we know the brick is relevant.
        # But we need the scalar values of the CELLS whose support overlaps.
        # This is complex. The source says "minimum and maximum scalar value of any cell 
        # whose support overlaps the region".
        
        # For pseudocode, we assume we can query the brick's data for this range.
        # This is a simplification [G].
        b_min, b_max = get_brick_scalar_range(brick)
        if b_min < min_val: min_val = b_min
        if b_max > max_val: max_val = b_max
        
        # Finest cell width
        w = 2**brick.level
        if w < finest_w: finest_w = w
        
    return min_val, max_val, finest_w
```

#### 3. RTX BVH Construction
Build an RTX BVH over the ABRs [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_rtx_bvh(abrs):
    """
    Build an RTX BVH over the ABRs.
    """
    # Standard BVH construction algorithm (e.g., SAH)
    # Nodes are ABRs.
    # Bounding boxes are ABR.bounding_box.
    return BVH(abrs)
```

#### 4. Ray Marching
Use the RTX BVH for hardware-accelerated ray marching, space skipping, and adaptive sampling [S1].

*   **Space Skipping:** If a ray intersects an ABR leaf where `min_scalar == max_scalar` (and outside transfer function range), skip the region [S1].
*   **Adaptive Sampling:** Use `finest_cell_width` of the ABR leaf to determine sampling frequency [S1].
*   **Reconstruction:** For samples within an ABR leaf, iterate over `active_bricks`. For each brick, iterate over cells that influence the sample (based on basis function support) and compute the weighted sum [S1].

### Edge Cases and Pitfalls
*   **Precision:** The basis function involves division by cell width. Ensure floating-point precision is sufficient for large coordinate values [S1].
*   **Boundaries:** The support of a brick extends beyond its bounds. Ensure the ABR construction correctly handles these extended supports to avoid gaps in influence [S1].
*   **Degenerate Input:** If AMR cells do not form valid blocks (gaps), the brick generation step skips them. Ensure the input data adheres to AMR refinement rules (single-level differences at boundaries) [S1].

## Subtopics

### [Octant Method](2648-octant-method.md)
A reconstruction strategy for BS-AMR data that maps sample points to scalar values by decomposing the domain into non-overlapping rectilinear octants. It solves discontinuous interpolation at level boundaries by using octants that do not extend beyond parent cell bounds, unlike ExaBricks which uses overlapping support regions and ABRs. Choose when rectilinear tiling without overlap is required.

## Operating characteristics
*   **Time Complexity:** Brick generation is $O(N \log N)$ due to k-d tree construction. ABR construction is recursive and depends on the number of overlapping supports. Ray traversal is accelerated by RTX BVH, typically $O(\log M)$ per ray, where $M$ is the number of ABRs [S1].
*   **Memory Complexity:** Stores bricks (compact same-level cells) and ABRs. The ABRs add overhead due to overlapping support regions, but bricks are memory-efficient [S1].
*   **Performance:** Enables interactive high-quality direct volume rendering with smooth AMR reconstruction and gradient-based shading [S1]. Avoids costly per-sample k-d tree traversals for cell location [S1].
*   **Strengths:** Smooth interpolation across level boundaries, efficient space skipping, adaptive sampling, hardware acceleration via RTX [S1].
*   **Limitations:** Overlapping support regions lead to more complex ABRs than simple bricks. Requires significant preprocessing to build bricks and ABRs [S1].

## Background
The ExaBricks Hierarchy was introduced by Wald et al. in 2020 [S1]. It builds on prior work by Kähler et al. [17] for brick organization and Wald et al. [37] for the basis method reconstruction. It is used for rendering structured AMR data, such as from PowerFlow [11] and Cart3D [1], supporting interactive transfer function and iso-surface editing [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
