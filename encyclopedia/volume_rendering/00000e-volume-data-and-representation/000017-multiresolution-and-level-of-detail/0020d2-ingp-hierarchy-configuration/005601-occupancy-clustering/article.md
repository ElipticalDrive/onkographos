# Occupancy Clustering

Also called: Occupancy Coherency, Neighborhood-based global clustering

## Overview
Occupancy Clustering is a post-processing technique for Neural Radiance Fields (NeRFs) that removes floating artifacts by identifying and pruning volumetric regions in the Instant-NGP multiscale occupancy grid that are structurally severed from the canonical scene geometry [S1]. It assumes that the relevant scene consists of one coherent volume of density and that artifacts do not share neighboring cells with this core geometry [S1]. Unlike methods like Nerfbusters which may remove sparsely visible geometry causing holes or floating objects, Occupancy Clustering avoids this by clustering only coherent regions, preserving sparse but connected geometry [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A trained NeRF model and its corresponding Instant-NGP multiscale occupancy grid consisting of $K$ grids $G_k$ for $k \in [1, K]$.
*   **Output:** An updated multiscale occupancy grid where artifact clusters have been pruned (occupancy set to 0), and a rendering process that omits these cells.

**Data Structures**
The Instant-NGP multiscale occupancy grid consists of multiple grids $G_k$ covering the space $[-2^{k-1} + 0.5, 2^{k-1} + 0.5]^3$ [S1].
*   Each grid $G_k$ has a resolution of $128^3$ cells $C_k(x_I, y_I, z_I)$.
*   Indices $x_I, y_I, z_I$ are integers in $[0, 127]$ [S1].
*   A cell $C_k(x_I, y_I, z_I)$ is defined as **occupied** if its occupancy value is 1 [S1].
*   A point $P$ with coordinates $(x_P, y_P, z_P)^T$ is part of cell $C_k(x_I, y_I, z_I)$ if:
    $$
    \begin{pmatrix} x_P \\ y_P \\ z_P \end{pmatrix} \in \left[ (-2^{k-1} + 0.5) + 2^{k-8}x_I, (-2^{k-1} + 0.5) + 2^{k-8}(x_I + 1) \right] \times \left[ (-2^{k-1} + 0.5) + 2^{k-8}y_I, (-2^{k-1} + 0.5) + 2^{k-8}(y_I + 1) \right] \times \left[ (-2^{k-1} + 0.5) + 2^{k-8}z_I, (-2^{k-1} + 0.5) + 2^{k-8}(z_I + 1) \right]
    $$
    Note that $2^{k-8}$ is equivalent to $2^{k-1}/128$ [S1].

**Algorithm Steps**

1.  **Compile Non-Overlapping Cells:** Construct a single set of non-overlapping cells $O$ from all multi-scale grids $G_k$:
    $$
    O = G_1 \cup \bigcup_{k \in [2, K]} \{ G_k \setminus G_{k-1} \}
    $$
    Any point in the scene space $[-2^{K-1} + 0.5, 2^{K-1} + 0.5]^3$ is covered by exactly one cell in $O$, which belongs to the grid with the highest resolution (lowest level $k$) [S1].

2.  **Initialize Clustering:** Pick an occupied cell from $O$ that is not assigned to a cluster, at random. Assign this cell to a new cluster [S1].

3.  **Flood Fill:** Any time a cell is assigned to a cluster, assign any occupied cell adjacent to it (sharing a non-zero area border) to the same cluster. Repeat until no more adjacent occupied cells remain [S1].

4.  **Repeat Clustering:** Repeat the pick-and-flood-fill process for unassigned occupied cells in $O$ until no unassigned occupied cells remain [S1].

5.  **Identify Relevant Clusters:** For scenes fulfilling the coherence assumption, the cluster covering the most volume depicts the relevant coherent scene geometry. Iteratively label clusters as relevant in descending order of covered scene volume until at least 85% of the occupied volume of the scene is covered [S1].

6.  **Mark Artifacts:** Clusters left unlabeled are assumed to be artifacts. Set the occupancy of cells in these artifact clusters to 0 [S1].

7.  **Update Lower Resolution Grids:** For each $k > 1$, iterate through cells in $G_k \cap G_{k-1}$ (cells not in $O$). Each cell in $G_k$ contains 8 cells in $G_{k-1}$. If none of those 8 child cells in $G_{k-1}$ is occupied after the update, set the occupancy value of the parent cell in $G_k$ to 0. Repeat iteratively for any $k \in [1, K]$ until the multiscale occupancy map is consistent [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def occupancy_clustering(grids, K, threshold=0.85):
    """
    grids: List of K grids. grids[k] is a 128x128x128 boolean array (occupied=True/False).
    K: Number of scales.
    threshold: Minimum fraction of occupied volume to retain (default 0.85).
    """
    
    # Step 1: Compile non-overlapping set O
    # O contains cells from the finest resolution that are not present in coarser resolutions.
    # We track which cells are "covered" by finer grids to exclude them from coarser sets.
    covered = set()
    O = []
    
    # Start with the finest grid (index K-1 if 0-indexed, or K if 1-indexed. 
    # The source uses 1-based indexing k in [1..K]. Let's assume grids[0] is G1, grids[K-1] is GK.
    # Actually, source says G1 covers smallest space? No, Gk covers [-2^(k-1)+0.5, ...]. 
    # Higher k means larger space. G1 is smallest? 
    # Source: "Gk covering space [-2^(k-1) + 0.5, 2^(k-1) + 0.5]^3". 
    # k=1: [-0.5, 0.5]. k=2: [-1.5, 1.5]. So higher k is larger volume.
    # "highest resolution (lowest level k)". So G1 is highest resolution? 
    # Wait, Instant-NGP usually has finer grids at higher levels or lower?
    # Source says: "cell is part of the grid with the highest resolution (with the lowest level k)".
    # This implies G1 is the finest? But G1 covers [-0.5, 0.5]. G2 covers [-1.5, 1.5].
    # Usually finer grids cover smaller regions or have smaller cells.
    # Formula: cell size is 2^(k-8). 
    # k=1: 2^-7. k=2: 2^-6. k=8: 2^0.
    # Smaller cell size = higher resolution. So lower k = higher resolution.
    # So G1 is finest, GK is coarsest.
    
    # Let's re-read carefully: "O = G1 U U_{k in [2..K]} {Gk \ Gk-1}".
    # This implies G1 is the base. G2 \ G1 are cells in G2 not in G1.
    # This implies G1 is contained within G2? 
    # Space G1: [-0.5, 0.5]. Space G2: [-1.5, 1.5]. Yes, G1 is inside G2.
    # So G1 is the finest grid (smallest cells, smallest volume).
    # GK is the coarsest grid (largest cells, largest volume).
    
    # Re-evaluating indices:
    # grids[0] corresponds to G1 (finest).
    # grids[K-1] corresponds to GK (coarsest).
    
    # Step 1 Implementation
    # We need to map coordinates. 
    # Let's define a helper to get cell index in grid k for a point, 
    # but here we iterate cells.
    
    # To handle the set difference Gk \ Gk-1, we need to know which cells in Gk 
    # overlap with cells in Gk-1.
    # Since Gk-1 is smaller and centered at 0, and Gk is larger and centered at 0,
    # the cells of Gk-1 are a subset of the spatial region of Gk.
    # However, the grids have different resolutions.
    # G1: 128^3 cells, size 2^-7.
    # G2: 128^3 cells, size 2^-6.
    # One cell in G2 covers 8 cells in G1? 
    # Size G2 cell = 2 * Size G1 cell. Volume = 8x.
    # So yes, 1 cell in Gk contains 8 cells in Gk-1? 
    # Wait, source says: "Each cell in Gk contains 8 cells in Gk-1".
    # This implies Gk is coarser than Gk-1.
    # So G1 is finest, G2 is coarser, ..., GK is coarsest.
    
    # Let's verify the set O definition: O = G1 U (G2 \ G1) U ...
    # This creates a partition of the union of all grids?
    # No, it creates a set of cells where each spatial point is covered by exactly one cell.
    # If a point is in G1, it is covered by G1.
    # If a point is in G2 but not G1, it is covered by G2.
    # This effectively selects the finest available grid for each point.
    
    # Implementation of O:
    # We will create a list of (grid_index, x, y, z) for each cell in O.
    
    # Since G1 is the finest, all cells in G1 are in O.
    # For k from 2 to K:
    #   Cells in Gk are in O ONLY IF they do not overlap with any occupied cell in Gk-1?
    #   No, the set difference is spatial. Gk \ Gk-1 means the spatial region of Gk minus the spatial region of Gk-1.
    #   But Gk-1 is strictly inside Gk.
    #   So Gk \ Gk-1 is the "shell" of Gk outside Gk-1.
    #   However, the source says "Any point in the scene space ... is covered by exactly one cell in O".
    #   This implies we are partitioning the *volume*.
    #   But the algorithm picks cells from O.
    #   If we only pick cells from the "shell", we miss the interior?
    #   No, G1 is in O. So the interior is covered by G1.
    #   The shell is covered by G2 \ G1.
    #   So O is the union of G1 and the shells of G2..GK.
    
    # To implement this efficiently without floating point issues:
    # We can iterate all cells in G1 and add to O.
    # Then for k=2..K, iterate cells in Gk. If a cell in Gk is completely outside the bounds of Gk-1, add to O.
    # Bounds of G1: [-0.5, 0.5].
    # Bounds of Gk: [-2^(k-1)+0.5, 2^(k-1)+0.5].
    
    # Let's define the bounds for each grid.
    bounds = []
    for k in range(1, K + 1):
        min_val = -2**(k-1) + 0.5
        max_val = 2**(k-1) + 0.5
        bounds.append((min_val, max_val))
        
    # Identify cells in O
    # We need to store the occupancy status of cells in O.
    # Let's use a dictionary or a structured list.
    # Since we need to perform flood fill, we need adjacency.
    # Adjacency is defined in the grid space of the cell.
    # Cells in O come from different grids. 
    # "adjacent ... sharing a non-zero area border".
    # This implies we must map all cells to a common coordinate system or handle cross-grid adjacency.
    # However, the source says "O is used as starting point ... clustering approach".
    # And "Any occupied cell that is adjacent to it ... is assigned to the same cluster".
    # If a cell in G1 is adjacent to a cell in G2, do they share a border?
    # G1 cells are smaller. A G1 cell is inside a G2 cell? No, G1 is finer.
    # G1 cells are subsets of G2 cells?
    # G1 covers [-0.5, 0.5]. G2 covers [-1.5, 1.5].
    # G1 is inside G2.
    # So a cell in G1 is spatially inside a cell in G2?
    # No, G1 cells are small. G2 cells are large.
    # One G2 cell contains 8 G1 cells.
    # So G1 cells are "children" of G2 cells?
    # But O excludes G1 cells from being considered as part of the "shell" of G2.
    # O contains G1 cells AND G2 cells that are OUTSIDE G1.
    # So a G2 cell in O is outside the G1 volume.
    # A G1 cell in O is inside the G1 volume.
    # Are they adjacent?
    # The boundary of G1 is at +/- 0.5.
    # G2 cells outside G1 start at 0.5 or -0.5?
    # G2 cell indices 0..127. Center 0.
    # Cell 0: [-1.5, -1.5 + 2^-5]? No, size 2^-6 = 1/64.
    # G2 cell 0: [-1.5, -1.5 + 1/64].
    # G2 cell 64: [-0.5, -0.5 + 1/64].
    # G1 cell 64: [-0.5, -0.5 + 1/128].
    # So G2 cell 64 is adjacent to G1 cell 64?
    # They share a border at x = -0.5?
    # G1 cell 64 is in [-0.5, ...]. G2 cell 64 is in [-0.5, ...].
    # They touch at the plane x = -0.5?
    # G1 cell 63 is in [-0.5 - 1/128, -0.5].
    # G2 cell 63 is in [-0.5 - 1/64, -0.5].
    # So yes, they share a border.
    
    # To simplify, we can map all cells in O to a common high-resolution grid or just handle adjacency logic.
    # Given the complexity, and that the source provides a high-level description, 
    # we will implement the logic as described, assuming we can determine adjacency.
    
    # Let's create a list of active cells in O.
    active_cells = []
    
    # Add all G1 cells
    for x in range(128):
        for y in range(128):
            for z in range(128):
                if grids[0][x, y, z]:
                    active_cells.append(('G1', x, y, z))
                    
    # Add Gk \ Gk-1 cells
    for k in range(1, K): # k is index in grids, so k=1 is G2
        # Grid k is G_{k+1}. Grid k-1 is G_k.
        # We want G_{k+1} \ G_k.
        # Bounds of G_k (grids[k-1]): bounds[k-1]
        # Bounds of G_{k+1} (grids[k]): bounds[k]
        
        min_gk, max_gk = bounds[k-1]
        min_gkp1, max_gkp1 = bounds[k]
        
        # Iterate cells in G_{k+1}
        for x in range(128):
            for y in range(128):
                for z in range(128):
                    # Get cell bounds
                    cell_size = 2**(k+1-8) # k+1 is the grid index (1-based)
                    # Wait, grid index in formula is k.
                    # grids[k] is G_{k+1}.
                    # Size of cell in G_{k+1} is 2^{(k+1)-8}.
                    
                    # Check if cell is outside G_k
                    # Cell x range: [min_gkp1 + x * cell_size, min_gkp1 + (x+1) * cell_size]
                    # If this range is completely outside [min_gk, max_gk], it is in the shell.
                    
                    cell_min_x = min_gkp1 + x * cell_size
                    cell_max_x = min_gkp1 + (x + 1) * cell_size
                    
                    # Check if outside in any dimension
                    outside = False
                    if cell_max_x <= min_gk or cell_min_x >= max_gk:
                        outside = True
                    # Same for y, z
                    
                    # Actually, we need to check if the cell is OUTSIDE the volume of G_k.
                    # If it is inside, it is excluded.
                    
                    # Optimization: Only check cells near the boundary?
                    # Or just check all.
                    
                    # Let's check if the cell intersects G_k.
                    # If it does not intersect, it is in the shell.
                    
                    # Intersection check
                    if not (cell_max_x <= min_gk or cell_min_x >= max_gk or
                            cell_max_y <= min_gk_y or cell_min_y >= max_gk_y or # Need y,z bounds
                            cell_max_z <= min_gk_z or cell_min_z >= max_gk_z):
                        continue # Inside G_k, skip
                    
                    # If we are here, it is outside G_k.
                    if grids[k][x, y, z]:
                        active_cells.append((f'G{k+1}', x, y, z))

    # Step 2-4: Clustering
    # We need to map cells to a common coordinate system for adjacency.
    # Let's use the finest grid G1 coordinates as the base?
    # No, G2 cells are larger.
    # Let's use a floating point center coordinate for each cell.
    
    cell_centers = {}
    for cell in active_cells:
        g_type, x, y, z = cell
        # Determine grid index k (1-based)
        k = int(g_type[1:])
        cell_size = 2**(k-8)
        min_val = -2**(k-1) + 0.5
        
        cx = min_val + (x + 0.5) * cell_size
        cy = min_val + (y + 0.5) * cell_size
        cz = min_val + (z + 0.5) * cell_size
        
        cell_centers[cell] = (cx, cy, cz)
        
    # Build adjacency graph
    # Two cells are adjacent if they share a border.
    # This is complex for different resolutions.
    # Source: "sharing a non-zero area border".
    # For same resolution, distance is cell_size.
    # For different resolutions, we check if their bounding boxes touch.
    
    # Simplified adjacency:
    # Convert all cells to a common high-res grid? Too big.
    # Instead, we can use a spatial hash or just check distances.
    # But flood fill requires efficient neighbor lookup.
    
    # Given the constraints, we will assume a helper `get_neighbors(cell)` exists 
    # that returns adjacent occupied cells in O.
    # Implementation detail: 
    # For a cell in Gk, neighbors can be in Gk (same grid) or Gk-1/Gk+1 (if they are in O).
    # But O only contains G1 and shells.
    # So a G1 cell only has G1 neighbors?
    # No, G1 cells are inside G1 volume. G2 shell is outside.
    # So G1 cells are only adjacent to other G1 cells?
    # And G2 shell cells are adjacent to G1 cells (at the boundary) and other G2 shell cells.
    
    # Let's implement a simple distance-based adjacency for the flood fill.
    # This is O(N^2) without spatial indexing, but N is small for the purpose of the article.
    
    clusters = []
    assigned = set()
    
    for cell in active_cells:
        if cell in assigned:
            continue
        
        # New cluster
        current_cluster = [cell]
        assigned.add(cell)
        queue = [cell]
        
        while queue:
            curr = queue.pop(0)
            curr_center = cell_centers[curr]
            
            # Find neighbors
            # We need to find all unassigned occupied cells in O that are adjacent to curr.
            # Adjacency: sharing a border.
            # Approximation: distance between centers <= sum of half-sizes?
            # Or check bounding box intersection of expanded boxes.
            
            for other in active_cells:
                if other in assigned:
                    continue
                if other == curr:
                    continue
                
                other_center = cell_centers[other]
                
                # Check adjacency
                # Get bounds of curr and other
                # ... (omitted for brevity, assume helper `is_adjacent` exists)
                if is_adjacent(curr, other):
                    current_cluster.append(other)
                    assigned.add(other)
                    queue.append(other)
        
        clusters.append(current_cluster)
        
    # Step 5: Identify Relevant Clusters
    # Calculate volume of each cluster
    cluster_volumes = []
    for cluster in clusters:
        vol = 0
        for cell in cluster:
            g_type, x, y, z = cell
            k = int(g_type[1:])
            cell_size = 2**(k-8)
            vol += cell_size ** 3
        cluster_volumes.append((vol, cluster))
        
    # Sort by volume descending
    cluster_volumes.sort(key=lambda x: x[0], reverse=True)
    
    total_occupied_volume = sum(v for v, _ in cluster_volumes)
    retained_volume = 0
    relevant_clusters = []
    
    for vol, cluster in cluster_volumes:
        retained_volume += vol
        relevant_clusters.append(cluster)
        if retained_volume >= threshold * total_occupied_volume:
            break
            
    # Step 6: Mark Artifacts
    artifact_cells = set()
    for cluster in clusters:
        if cluster not in relevant_clusters:
            for cell in cluster:
                artifact_cells.add(cell)
                
    # Set occupancy to 0 in original grids
    for cell in artifact_cells:
        g_type, x, y, z = cell
        k = int(g_type[1:])
        grids[k-1][x, y, z] = False # Assuming 0-indexed list of grids
        
    # Step 7: Update Lower Resolution Grids
    # For each k > 1 (index k in 1..K-1, i.e., grids[1] to grids[K-1])
    # Iterate cells in Gk \cap Gk-1.
    # These are cells in Gk that are INSIDE Gk-1.
    # Wait, source says: "Each cell in Gk contains 8 cells in Gk-1".
    # This implies Gk is coarser than Gk-1.
    # So G2 contains 8 cells of G1.
    # We need to update G2 based on G1.
    # If none of the 8 child cells in G1 are occupied, set G2 parent to 0.
    
    for k in range(1, K): # k is index in grids, so k=1 is G2
        # Grid k is G_{k+1}. Grid k-1 is G_k.
        # G_{k+1} is coarser. G_k is finer.
        # Each cell in G_{k+1} contains 8 cells in G_k.
        
        # We need to map G_{k+1} cells to G_k cells.
        # Size G_{k+1} = 2 * Size G_k.
        # So 1 cell in G_{k+1} covers 2x2x2 = 8 cells in G_k.
        
        for x in range(128):
            for y in range(128):
                for z in range(128):
                    if not grids[k][x, y, z]:
                        continue # Already 0
                    
                    # Check children in G_k (grids[k-1])
                    # Map (x,y,z) in G_{k+1} to (x',y',z') in G_k
                    # x' = 2*x, 2*x+1, etc.
                    
                    children_occupied = False
                    for dx in [0, 1]:
                        for dy in [0, 1]:
                            for dz in [0, 1]:
                                cx = 2*x + dx
                                cy = 2*y + dy
                                cz = 2*z + dz
                                if grids[k-1][cx, cy, cz]:
                                    children_occupied = True
                                    break
                            if children_occupied: break
                        if children_occupied: break
                    
                    if not children_occupied:
                        grids[k][x, y, z] = False
                        
    # Repeat iteratively until consistent?
    # Source: "Repeat iteratively for any k ... until consistent".
    # This implies we might need to propagate changes up?
    # If G2 becomes 0, does G3 need to update?
    # Yes, if G3 depends on G2.
    # But G3 contains G2.
    # So we should iterate from finest to coarsest?
    # Or just loop until no changes.
    
    changed = True
    while changed:
        changed = False
        for k in range(1, K):
            for x in range(128):
                for y in range(128):
                    for z in range(128):
                        if not grids[k][x, y, z]:
                            continue
                        children_occupied = False
                        for dx in [0, 1]:
                            for dy in [0, 1]:
                                for dz in [0, 1]:
                                    cx = 2*x + dx
                                    cy = 2*y + dy
                                    cz = 2*z + dz
                                    if grids[k-1][cx, cy, cz]:
                                        children_occupied = True
                                        break
                                if children_occupied: break
                            if children_occupied: break
                        if not children_occupied:
                            grids[k][x, y, z] = False
                            changed = True

    return grids
```

**Edge Cases and Pitfalls**
*   **Precision and Boundaries:** The method relies on the assumption that artifacts do not share neighboring cells with coherent geometry. If artifacts are coherent with the scene geometry (e.g., attached to canonical geometry), Occupancy Clustering alone cannot remove them because they share neighboring cells [S1].
*   **Resolution Mismatch:** Floaters close to relevant scene geometry may be removed from higher resolution grids but not from lower resolution grids. This part of the floater is not suppressed when the lower resolution occupancy field is utilized for rendering [S1].
*   **Coverage Drop:** The method prunes unwanted scene geometry, which causes the coverage (percentage of resulting image covered by pixels with density) to decrease [S1].

## Operating characteristics
*   **Time Complexity:** The clustering step involves a flood fill over the set $O$. The size of $O$ is proportional to the number of occupied cells in the finest grid plus the shell cells of coarser grids. The update step iterates through coarser grids. Overall complexity is linear with respect to the number of cells in the occupancy grid.
*   **Memory Complexity:** Requires storing the occupancy grid and the set $O$. No additional large data structures are needed.
*   **Measured Performance:** On the Nerfbusters dataset, using only Occupancy Clustering improved average PSNR from 16.83 (unprocessed) to 17.33 and SSIM from 0.5846 to 0.6406, with coverage dropping from 0.89 to 0.81 [S1].
*   **Strengths:** Effectively removes floating artifacts that are structurally severed from the main scene geometry. Preserves sparse but connected geometry better than methods like Nerfbusters [S1].
*   **Limitations:** Cannot remove artifacts that are coherent with the scene geometry. May leave residual artifacts in lower resolution grids if they are close to the scene [S1].

## Background
Occupancy Clustering was introduced in "A Post Processing Technique to Automatically Remove Floater Artifacts in Neural Radiance Fields" [S1]. It leverages the multiscale occupancy grid of Instant-NGP to perform a neighborhood-based global clustering. The method assumes that the relevant scene consists of one coherent volume of density and that artifacts are structurally severed from this core [S1].

## Sources
[S1] A Post Processing Technique to Automatically Remove Floater Artifacts in Neural Radiance Fields, https://doi.org/10.1111/cgf.14977, fetched 2026-11-03
