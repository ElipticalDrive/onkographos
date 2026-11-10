# Valley Cell-Based Clustering Transfer Function Design

Also called: Automated TF Design with Valley Cell-Based Clustering, VCC-TF

## Overview
This method automatically generates opacity transfer functions for volume rendering by clustering 2D density plots (value vs. gradient magnitude) into "valley cells" using topological persistence, and mapping these clusters to Gaussian transfer functions with depth-aware opacity [S1]. It solves the problem of manual transfer function design by determining the number and shape of features without user-specified bin counts, while providing a hierarchical structure for interactive refinement [S1]. Unlike [Histogram-Based Transfer Function Generation](6182-histogram-based-transfer-function-generation.md) which requires fixed bins and lacks simplification guidelines, this method uses cell persistence to automatically simplify the parameter space and remove noise [S1].

## How it works
**Inputs:**
*   Scalar volume data $V$.
*   Feature space dimensions (typically 2D: value and gradient magnitude) [S1].
*   User-defined minimum opacity $\alpha_{min}$ [S1].
*   Persistence threshold $\tau$ (typically 1% of maximal persistence) [S1].

**Outputs:**
*   A set of Gaussian Transfer Functions (GTFs), each defined by mean $\mu$, covariance $\Sigma$, maximum opacity $\alpha_{max}$, and a bounding box [S1].
*   A hierarchical merge tree of valley cells [S1].

**Data Structures:**
1.  **2D Density Plot:** A grid representing the continuous scatterplot of features, generated via Continuous Scatterplots (CSPs) [S1].
2.  **Valley Cell:** A connected region in the density plot bounded by ridges (local maxima) and valleys (local minima). Each cell has:
    *   `persistence`: The minimum persistence of its boundary edges.
    *   `size` ($s_i$): A metric of the cell's area or volume in feature space.
    *   `max`: The density value at the cell's peak.
    *   `mean` ($\mu$): The position of the maximum density or cluster center.
    *   `covariance` ($\Sigma$): The density covariance within the cell.
    *   `bounding_box`: Axis-aligned bounds for the cell.
3.  **Merge Tree:** A hierarchy where nodes are valley cells. Edges represent merges. The tree is built by removing boundaries with persistence below $\tau$ [S1].

**Steps:**

1.  **Generate Density Plot:** Construct a 2D density plot of the volume data features using CSPs [S1].
2.  **Identify Valley Cells:** Decompose the density plot into valley cells using Morse complex theory. Each cell represents a potential feature [S1].
3.  **Compute Cell Persistence:** For each cell, calculate its persistence as the smallest persistence of its boundaries. This reflects the degree of separation of the corresponding features [S1].
4.  **Construct Hierarchy:** Build a merge tree by removing boundaries between adjacent cells based on persistence. Remove boundaries with persistence below the threshold $\tau$ to simplify the tree [S1].
5.  **Calculate Opacity Range:** For each node $i$ in the tree, compute the initial opacity range $\alpha_i$ using the formula:
    $$ \alpha_i = \alpha_{min} + \frac{W}{g(d_i) \cdot s_i \cdot p_{ci}} $$
    where:
    *   $s_i$ is the feature size.
    *   $p_{ci}$ is the cell persistence.
    *   $W$ is a normalization weight that normalizes $1/(s_i \cdot p_{ci})$ into $[0, 1]$ [S1].
    *   $g(d_i)$ is a depth-based control function:
        $$ g(d_i) = \frac{d_i - d_{min}}{d_{max} - d_{min}} $$
        where $d_i$ is the depth of node $i$ in the hierarchy, and $d_{min}, d_{max}$ are the min/max depths in the tree [S1].
6.  **Generate Gaussian Transfer Function (GTF):** For each cell, define a GTF:
    $$ \alpha(\mathbf{x}) = \alpha_{max} e^{-\frac{1}{2} (\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x}-\boldsymbol{\mu})} $$
    where:
    *   $\alpha_{max}$ is the computed opacity range $\alpha_i$.
    *   $\boldsymbol{\Sigma}^{-1}$ is the inverse of the density covariance in the valley cell.
    *   $\boldsymbol{\mu}$ is the position of the maximum density (default) or the cluster center [S1].
7.  **Apply Bounding Box:** Compute the bounding box for the valley cell. Set $\alpha(\mathbf{x}) = 0$ for any region outside this bounding box to prevent contribution to other features [S1].
8.  **User Interaction:** Allow the user to manually adjust opacity values for each cell. Users can merge uninteresting features (e.g., small cells at low densities) or split cells to refine the visualization [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vcc_tf(volume_data, alpha_min, persistence_threshold_pct):
    # Step 1: Generate 2D Density Plot using CSPs
    # Features: value and gradient magnitude
    density_plot = generate_continuous_scatterplot(volume_data)
    
    # Step 2: Identify Valley Cells
    # Decompose density plot into connected regions
    valley_cells = decompose_into_valley_cells(density_plot)
    
    # Step 3: Compute Cell Persistence
    for cell in valley_cells:
        # Persistence is the minimum persistence of its boundaries
        cell.persistence = min(boundary.persistence for boundary in cell.boundaries)
        # Compute size and covariance
        cell.size = compute_cell_size(cell)
        cell.covariance = compute_density_covariance(cell)
        cell.mean = find_density_maximum(cell) # Default: position of max
        # cell.mean = compute_cluster_center(cell) # Alternative option
        
    # Step 4: Construct Hierarchy
    # Sort boundaries by persistence
    boundaries = get_all_boundaries(valley_cells)
    boundaries.sort(key=lambda b: b.persistence)
    
    merge_tree = MergeTree()
    current_cells = list(valley_cells)
    
    # Remove boundaries below threshold to simplify
    max_persistence = max(c.persistence for c in valley_cells)
    threshold = max_persistence * persistence_threshold_pct
    
    for boundary in boundaries:
        if boundary.persistence < threshold:
            # Merge adjacent cells
            cell_a, cell_b = boundary.cells
            merged_cell = merge_cells(cell_a, cell_b)
            merged_cell.persistence = min(cell_a.persistence, cell_b.persistence)
            # Update tree structure
            merge_tree.add_merge(cell_a, cell_b, merged_cell)
            current_cells.remove(cell_a)
            current_cells.remove(cell_b)
            current_cells.append(merged_cell)
            
    # Step 5: Calculate Opacity Range
    # Determine depth range for depth-based control
    depths = [node.depth for node in merge_tree.all_nodes()]
    d_min = min(depths)
    d_max = max(depths)
    
    # Compute normalization weight W
    # W normalizes 1/(s_i * p_ci) into [0, 1]
    raw_opacities = [1.0 / (c.size * c.persistence) for c in merge_tree.all_nodes()]
    W = 1.0 / max(raw_opacities) if max(raw_opacities) > 0 else 1.0
    
    for node in merge_tree.all_nodes():
        # Depth-based control function
        g_di = (node.depth - d_min) / (d_max - d_min) if d_max != d_min else 1.0
        # Opacity formula
        node.alpha_max = alpha_min + (W / (g_di * node.size * node.persistence))
        
    # Step 6 & 7: Generate GTFs with Bounding Boxes
    transfer_functions = []
    for node in merge_tree.all_nodes():
        # Compute bounding box for the valley cell
        bbox = compute_bounding_box(node)
        
        # Define GTF
        def gtf(x):
            # Check if x is outside bounding box
            if not is_inside_bbox(x, bbox):
                return 0.0
            # Gaussian formula
            diff = x - node.mean
            exponent = -0.5 * diff.T @ node.covariance.inv() @ diff
            return node.alpha_max * math.exp(exponent)
            
        transfer_functions.append({
            'gtf': gtf,
            'bbox': bbox,
            'color': node.color # Assigned via color scheme
        })
        
    return transfer_functions, merge_tree

def merge_cells(cell_a, cell_b):
    # Create new cell representing the union
    merged = Cell()
    merged.boundaries = (cell_a.boundaries | cell_b.boundaries) - {boundary_between_a_b}
    merged.density_values = cell_a.density_values + cell_b.density_values
    # Recalculate mean and covariance for merged region
    merged.mean = compute_weighted_mean(cell_a.mean, cell_b.mean, cell_a.size, cell_b.size)
    merged.covariance = compute_pooled_covariance(cell_a, cell_b)
    return merged
```

**Parameters:**
*   **Persistence Threshold:** Typically set to 1% of the maximal persistence. This usually results in fewer than 10 remaining clusters [S1].
*   **Mean ($\mu$):** Default is the position of the maximum density in the valley cell. Alternative option is the center of the cluster [S1].
*   **Minimum Opacity ($\alpha_{min}$):** User-specified base opacity [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** Small features may be occluded by larger features. The opacity formula includes feature size ($s_i$) and persistence ($p_{ci}$) to mitigate this, and depth-based control ($g(d_i)$) helps distinguish features at different hierarchy levels [S1].
*   **Precision:** The density plot generation uses an error threshold (e.g., 50) and resolution (e.g., 256x256) which affects accuracy [S1].
*   **Boundary Conditions:** The bounding box ensures that the Gaussian function does not contribute to other features, preventing leakage [S1].

## Subtopics

### Hierarchical Feature Representation
A multi-level feature structure created by successive persistence-driven merging of valley cells, enabling interactive exploration through feature splitting, merging, and opacity/color assignment [S1]. It allows the user to interactively merge or split features at any level of the hierarchy, with opacity determined by persistence and size [S1]. Unlike methods requiring a fixed number of clusters or regular widget shapes, this hierarchy adapts to arbitrary feature counts/shapes and incorporates user prior knowledge without manual cluster specification [S1].

### Valley Cell-Based Clustering Transfer Function Generation
Generation of a transfer function by assigning colors to valley cells using a qualitative color scheme [S1]. When merging cells in the hierarchy, the merged cell is colored with the color of the cell having the larger maximum. When splitting cells, the original color of the split cell is restored [S1].

## Operating characteristics
*   **Time Complexity:** Density plot generation is the most expensive step, taking less than 1 hour for a 256x256 resolution with an error threshold of 50 on an Intel Core 2 Duo E6320 1.8 GHz CPU [S1]. Clustering takes approximately 10 seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On an Intel Core 2 Duo E6320 1.8 GHz CPU, 2.0 GB RAM, NVIDIA GeForce GTX 260 (256 MB VRAM), rendering frame rate is approximately 25 fps for fewer than 10 valley cells [S1]. Performance decreases as the number of valley cells increases because GTF-based volume rendering is evaluated directly on the GPU [S1].
*   **Strengths:** Automatically determines the number of features; uses cell persistence for parameter space simplification; allows interactive exploration via hierarchy; handles arbitrary feature shapes [S1].
*   **Limitations:** Rendering performance degrades with more valley cells; density plot generation is slow [S1].
*   **When to use:** When automatic classification of volume data is needed without prior knowledge of the number of features, and interactive refinement is desired [S1].
*   **When not to use:** When real-time performance with a large number of features is required, or when the user prefers manual binning [S1].

## Background
The method was proposed by Y. Wang et al. in 2012 [S1]. It builds on the principle that features tend to form peaks of the density function [S1]. It uses Continuous Scatterplots (CSPs) for density estimation, which considers interpolation between data voxels and conservation of physical variables, as opposed to kernel density estimation which estimates distribution from statistical uncertainty [S1]. The approach is based on the theory of Morse complex to decompose the 2D density plot into valley cells [S1]. It has been used on medical data (CT Teeth, CT facial deformity) and simulation data (Supernova, Turbulent slip surface) to reveal structures like enamel, dentine, pulp, lesions, and vortex tubes [S1].

## Sources
[S1] Automating Transfer Function Design with Valley Cell‐Based Clustering of 2D Density Plots, https://doi.org/10.1111/j.1467-8659.2012.03122.x, fetched 2026-11-01
