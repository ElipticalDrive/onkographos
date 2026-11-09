# Graph-based Isosurface Segmentation (Min-Cut)

Also called: Graph Segmentation on Isosurface, Isosurface Segmentation, Isosurface Segmentation via Min-Cut, Surface structure extraction

## Overview
This method segments a 3D volume by treating voxels containing a specific isosurface as nodes in a graph, connected via neighborhood connectivity, and finding an optimal cut to separate user-defined foreground and background seed regions [S1, unofficial]. Unlike traditional volume segmentation which operates on the entire voxel grid and suffers from label volume precision limits and computational expense, this approach restricts the graph to the sparse subset of voxels intersecting the isosurface, integrating directly with isosurface rendering pipelines [S1, unofficial]. It is distinguished from its siblings by its explicit formulation of the segmentation problem as a min-cut optimization on the isosurface domain, whereas [Valley Cell Construction](9747-valley-cell-construction.md) focuses on geometric valley detection and [Restricted Region Growing](9750-restricted-region-growing.md) relies on iterative classification of doubt voxels.

## How it works
**Inputs:**
*   A 3D volume dataset with scalar values (e.g., CT or MRI intensities).
*   An isovalue defining the specific isosurface to segment.
*   User-selected seed voxels for the foreground and background, selected via an interactive interface (e.g., mouse clicks/drag on the rendering view) [S1, unofficial].

**Outputs:**
*   A segmentation of the isosurface voxels into connected components (surface structures).
*   A label assignment for each segmented isosurface voxel, stored in an 8-bit label volume where values 1–255 represent distinct surface structures and 0 represents empty space [S1, unofficial].

**Data Structures:**
1.  **Graph Nodes:** A list of voxel IDs $(x, y, z)$ that contain the isosurface. The isosurface presence is determined by ray-isosurface intersection tests or trilinear interpolation checks [S1, unofficial].
2.  **Graph Edges:** Undirected edges connecting nodes in a 6-neighborhood (for 3D) or 4-neighborhood (for 2D illustrations) [S1, unofficial].
3.  **Edge Weights:** A scalar value $w_{pq}$ for each edge connecting voxel $p$ and voxel $q$, defined as the length of the intersection between the isosurface and the voxel boundary shared by $p$ and $q$ [S1, unofficial].
4.  **Seed Storage:** Two sorted 'set' data structures (optimized for searching) storing the voxel IDs of the foreground seeds ($S_{fg}$) and background seeds ($S_{bg}$) [S1, unofficial].

**Algorithm Steps:**

1.  **Voxel Selection and Seed Initialization:**
    Identify voxels containing the isosurface. Collect user-selected seed voxel IDs into two sets, $S_{fg}$ and $S_{bg}$, stored as sorted sets for efficient lookup [S1, unofficial].

2.  **Graph Construction:**
    Construct a graph $G = (V, E)$ where $V$ is the set of isosurface voxels. Add an edge between any two voxels $p, q \in V$ if they are 6-neighbors (adjacent along x, y, or z axes) [S1, unofficial].

3.  **Edge Weight Calculation:**
    For each edge $(p, q) \in E$, calculate the weight $w_{pq}$ as the length of the intersection segment between the isosurface and the boundary between voxel $p$ and voxel $q$ [S1, unofficial].
    *   If the isosurface intersects the boundary in a single continuous segment, calculate the length using numerical integration [S1, unofficial].
    *   If trilinear interpolation results in two distinct intersection segments (a rare case), divide the total length of both segments by 2 to approximate the weight [S1, unofficial].

4.  **Min-Cut Optimization:**
    Solve the min-cut problem to partition $V$ into two sets $V_{fg}$ and $V_{bg}$ such that the sum of weights of edges crossing the cut is minimized, subject to $S_{fg} \subseteq V_{fg}$ and $S_{bg} \subseteq V_{bg}$ [S1, unofficial]. Use the Boykov-Kolmogorov optimization algorithm [S1, unofficial].

5.  **Label Assignment:**
    Assign the label corresponding to the foreground component to all voxels in $V_{fg}$ and the background component to $V_{bg}$ in the output label volume [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def graph_based_isosurface_segmentation(volume, isovalue, seed_fg_ids, seed_bg_ids):
    """
    Segments isosurface voxels using min-cut.
    
    Args:
        volume: 3D array of scalar values.
        isovalue: The scalar value defining the isosurface.
        seed_fg_ids: List of (x,y,z) tuples for foreground seeds.
        seed_bg_ids: List of (x,y,z) tuples for background seeds.
        
    Returns:
        label_volume: 3D array with 0 for empty, 1 for foreground structure, 
                      2 for background structure (or distinct labels for multiple structures).
    """
    
    # Step 1: Identify isosurface voxels
    # A voxel contains the isosurface if the isovalue lies between the min and max 
    # of its 8 corners (trilinear interpolation bounds).
    isosurface_voxels = []
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                if voxel_contains_isosurface(volume, x, y, z, isovalue):
                    isosurface_voxels.append((x, y, z))
    
    # Map voxel ID to node index for graph construction
    voxel_to_node = {voxel: i for i, voxel in enumerate(isosurface_voxels)}
    num_nodes = len(isosurface_voxels)
    
    # Step 2: Construct Graph and Calculate Weights
    # Using Boykov-Kolmogorov compatible structure: 
    # Nodes 0..N-1 are voxels, Node N is Source (fg), Node N+1 is Sink (bg)
    num_terminal_nodes = 2
    total_nodes = num_nodes + num_terminal_nodes
    
    # Initialize graph weights
    # t-links (terminals)
    t_links_fg = [0] * total_nodes
    t_links_bg = [0] * total_nodes
    
    # n-links (neighbors)
    # Store as list of (u, v, weight)
    n_links = []
    
    for i, p in enumerate(isosurface_voxels):
        # Assign seeds to terminals
        if p in seed_fg_ids:
            t_links_fg[i] = INF
        elif p in seed_bg_ids:
            t_links_bg[i] = INF
            
        # Check 6-neighbors to build edges
        # Only check neighbors with higher indices to avoid duplicates
        neighbors = get_6_neighbors(p)
        for q in neighbors:
            if q in voxel_to_node:
                j = voxel_to_node[q]
                if i < j:
                    # Calculate weight: intersection length of isosurface with voxel boundary
                    weight = calculate_intersection_length(volume, p, q, isovalue)
                    n_links.append((i, j, weight))
    
    # Step 3: Solve Min-Cut
    # The min-cut algorithm minimizes: sum(w_ij for i in S, j in T) + sum(t_links)
    # We use a standard max-flow/min-cut solver (e.g., Boykov-Kolmogorov)
    # Note: The specific implementation of the solver is external, but the input format is standard.
    labels = solve_min_cut(total_nodes, t_links_fg, t_links_bg, n_links)
    
    # Step 4: Map results to label volume
    label_volume = np.zeros(volume.shape, dtype=np.uint8)
    
    for i, p in enumerate(isosurface_voxels):
        # labels[i] == 0 means Source (Foreground), 1 means Sink (Background)
        # Assuming 1 for FG, 2 for BG, 0 for empty
        if labels[i] == 0:
            label_volume[p] = 1
        elif labels[i] == 1:
            label_volume[p] = 2
            
    return label_volume

def voxel_contains_isosurface(volume, x, y, z, isovalue):
    """Check if isovalue is within the range of trilinear interpolation bounds of the voxel."""
    # Get min and max of the 8 corners
    min_val = INF
    max_val = -INF
    for dx in [0, 1]:
        for dy in [0, 1]:
            for dz in [0, 1]:
                val = volume[x+dx, y+dy, z+dz]
                if val < min_val: min_val = val
                if val > max_val: max_val = val
    return min_val <= isovalue <= max_val

def calculate_intersection_length(volume, p, q, isovalue):
    """
    Calculate the length of the intersection between the isosurface and the 
    boundary shared by voxel p and voxel q.
    """
    # Determine the shared face geometry
    # For simplicity, assume unit voxel size. The boundary is a plane.
    # The isosurface is defined by trilinear interpolation.
    
    # Get corner values for p and q
    # This function requires detailed geometric calculation of the trilinear patch intersection
    # with the shared face.
    
    # Simplified heuristic from source:
    # If there is one segment, use numerical integration.
    # If two segments (rare), divide total length by 2.
    
    # Implementation detail: [G]
    # In practice, this involves solving for the intersection of the trilinear 
    # polynomial with the plane defined by the shared face. 
    # The source notes that if two segments exist, the weight is total_length / 2.
    
    # Placeholder for complex geometric calculation
    # In a real implementation, this would use ray casting or analytical root finding
    # on the trilinear patch restricted to the shared face.
    length = compute_trilinear_intersection_length(p, q, isovalue)
    
    # Handle the rare two-segment case
    # [G] The source states: "if there are two segments... divide the total length of the two 
    # intersection segments by 2."
    # Detecting two segments requires analyzing the roots of the intersection polynomial.
    # For the purpose of this article, we assume the function returns the adjusted weight.
    
    return length

def compute_trilinear_intersection_length(p, q, isovalue):
    """
    Computes the length of the isosurface intersection on the shared face.
    Returns the weight as defined by the source.
    """
    # [G] Detailed geometric implementation omitted for brevity, but must return
    # the length of the curve segment(s) where the trilinear interpolant equals isovalue
    # on the shared face between p and q.
    # If multiple segments are found, sum their lengths and divide by 2.
    pass
```

**Parameters:**
*   **Isosurface Value:** The scalar threshold defining the surface [S1, unofficial].
*   **Label Volume Precision:** 8-bit integer, allowing up to 255 distinct surface structures [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Trilinear Ambiguity:** Trilinear interpolation can create up to two intersection segments per voxel boundary. The method handles this by dividing the total length of all segments by 2 [S1, unofficial].
*   **Open Surfaces:** Extracted surface structures are not guaranteed to be closed and cannot substitute for full volume segmentation [S1, unofficial].
*   **Staircase Artifacts:** By operating on the isosurface domain rather than a label volume, this method avoids the stair-case artifacts common in traditional volume segmentation [S1, unofficial].

## Subtopics

### Edge Weight Definition for Isosurface Consistency
This variant modifies the edge weight definition to ensure boundaries are consistent with isosurfaces across different materials, particularly in multi-material CT data. Instead of geometric intersection length, weights are assigned based on CT value thresholds relative to isovalue thresholds $v = iso(m_i, m_j)$ [S2].
*   **Definition:** $w(p, q) = w_{cut}$ if $I_p \le v \le I_q$ or $I_p \ge v \ge I_q$ for some isovalue threshold $v$; otherwise $w(p, q) = w_{join}$ [S2]. Here $I_p, I_q$ are CT values. $w_{cut}$ and $w_{join}$ are positive constants [S2].
*   **Cost Function:** The total energy is $E = w_{join} (\tau \cdot N_{cut} + N_{join})$, where $N_{cut}$ is the number of edges with weight $w_{cut}$, $N_{join}$ is the number of edges with weight $w_{join}$, and $\tau = w_{cut} / w_{join}$ is the cut-ratio parameter ($0 < \tau < 1$) [S2].
*   **Rationale:** Using standard weights $|I_p - I_q|$ fails to ensure boundaries are consistent with isosurfaces. This rule prevents misclassification by pulling voxels toward the cut direction if linked by enough $w_{cut}$ edges, even if $w_{cut} < w_{join}$ [S2].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the min-cut algorithm (Boykov-Kolmogorov). The number of nodes is proportional to the number of voxels containing the isosurface, not the total volume size [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of isosurface voxels and their 6-neighborhood connections. Significantly lower than full volume segmentation [S1, unofficial].
*   **Performance:** On a 256x256x124 brain MRI dataset (8,126,464 total voxels), the method involved 832,869 graph nodes and took 8.25 seconds [S1, unofficial]. With cropping to a region of interest, nodes reduced to 40,830 and time reduced to 0.36 seconds [S1, unofficial].
*   **Strengths:** Avoids label volume storage and associated stair-case artifacts; computationally efficient for sparse isosurfaces; integrates well with interactive rendering [S1, unofficial].
*   **Limitations:** Extracted structures are not guaranteed to be closed; limited to 255 structures due to 8-bit label volume; requires user seed selection [S1, unofficial].

## Background
This method was proposed to address the limitations of traditional volume segmentation, such as computational expense and precision issues in label volumes [S1, unofficial]. It leverages the efficiency of graph-cut algorithms, specifically the Boykov-Kolmogorov optimization method [S1, unofficial]. The approach is designed to work in conjunction with color-enhanced isosurface rendering, allowing for multi-surface-structure visualization within a single scene [S1, unofficial].

## Sources
[S1] Efficient and Effective Volume Visualization with Enhanced Isosurface Rendering, https://doi.org/10.48550/arxiv.1202.5360, fetched 2026-10-26 (unofficial)
[S2] Extraction of isosurfaces from multi-material CT volumetric data of mechanical parts, https://doi.org/10.1145/1364901.1364931, fetched 2026-11-01
