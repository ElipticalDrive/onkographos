# Valley Cell-Based Clustering Transfer Function Design

Also called: Automated TF Design with Valley Cell-Based Clustering, VCC-TF

## Overview
Valley Cell-Based Clustering (VCC) is an automatic method for generating multidimensional transfer functions by decomposing 2D density plots of volume features into topological "valley cells" [S1]. It solves the problem of manual transfer function design by identifying significant data structures without assuming a fixed number or shape of clusters, using cell persistence to hierarchically simplify the feature space and mitigate noise [S1]. Unlike Binning-based Transfer Functions (BTF) which require user-specified bins and lack simplification guidelines, VCC automatically determines feature counts and uses persistence to guide the merging of cells, resulting in a hierarchy that allows for intuitive user exploration [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Volume dataset $V$ [S1].
*   Feature space definition: Typically 2D, mapping each voxel $\mathbf{x}$ to a pair $(v, g)$, where $v$ is the scalar value (intensity/density) and $g$ is the gradient magnitude $\|\nabla f(\mathbf{x})\|$ [S1].
*   Persistence threshold $\tau$: A parameter for simplifying the hierarchy, typically set to 1% of the maximal persistence in the dataset [S1].

**Outputs:**
*   A set of Gaussian Transfer Functions (GTFs), one for each significant valley cell in the simplified hierarchy [S1].
*   Each GTF is defined by a mean $\mu$, a covariance matrix $\Sigma$, and an opacity range $[\alpha_{min}, \alpha_{max}]$ [S1].
*   A hierarchical tree structure representing the merge history of valley cells [S1].

### Data Structures

1.  **2D Density Plot (Continuous Scatterplot):**
    *   A grid of resolution $R \times R$ (e.g., $256 \times 256$) representing the joint distribution of features $(v, g)$ [S1].
    *   Each cell $(i, j)$ in the grid stores a density value $D_{i,j}$ computed via continuous scatterplot interpolation [S1].

2.  **Valley Cell:**
    *   A connected region in the density plot bounded by "valleys" (local minima in the density landscape) [S1].
    *   Properties:
        *   $S_i$: Feature size (e.g., area or number of voxels mapped to this cell) [S1].
        *   $P_i$: Cell persistence (defined below) [S1].
        *   $\mu_i$: Mean position (center of mass or peak) in the feature space [S1].
        *   $\Sigma_i$: Covariance matrix of the data points within the cell [S1].
        *   $B_i$: Bounding box of the cell in feature space [S1].

3.  **Merge Tree (Hierarchy):**
    *   A tree where nodes represent valley cells [S1].
    *   Edges represent the merging of two cells [S1].
    *   Each edge is weighted by the persistence of the boundary between the merged cells [S1].
    *   Leaves are the initial atomic cells; the root is the merged state of all cells [S1].

### Numbered Steps

1.  **Generate Density Plot:**
    *   Compute the 2D density plot of the volume features using Continuous Scatterplots (CSPs) [S1].
    *   CSPs consider interpolation between data voxels and conservation of physical variables, providing a smooth density function $D(v, g)$ [S1].

2.  **Identify Valley Cells:**
    *   Decompose the density plot into valley cells based on the topology of $D(v, g)$ [S1].
    *   Each cell corresponds to a local peak (mode) in the density [S1].

3.  **Compute Cell Persistence:**
    *   For each cell $i$, identify its boundaries (ridges separating it from adjacent cells) [S1].
    *   Define the persistence of cell $i$, denoted $p_i$, as the minimum persistence of its boundaries [S1].
    *   The persistence of a boundary between two adjacent cells is defined as the difference in density height between the saddle point (local minimum on the ridge) and the lower of the two adjacent peaks [S1].
    *   This value reflects the degree of separation between the features [S1].

4.  **Construct Merge Tree:**
    *   Create a hierarchy by merging adjacent cells with the lowest boundary persistence first [S1].
    *   This forms a tree where the depth of a node corresponds to the persistence threshold at which it was formed [S1].

5.  **Simplify Hierarchy:**
    *   Remove boundaries with persistence below the threshold $\tau$ (e.g., 1% of max persistence) [S1].
    *   This simplifies the tree, removing noisy or insignificant cells, typically resulting in fewer than 10 clusters [S1].

6.  **Compute Opacity Ranges:**
    *   For each node $i$ in the simplified tree, compute an opacity range to ensure visibility of small or less persistent features [S1].
    *   Calculate the normalized weight $W$ to scale the opacity contribution [S1].
    *   Compute the depth-based control function $g(d_i)$ [S1].
    *   Determine $\alpha_{max, i}$ using the formula:
        $$ \alpha_{max, i} = \alpha_{min} + \frac{W}{g(d_i) \cdot s_i \cdot p_{ci}} $$
        Where:
        *   $\alpha_{min}$: User-specified minimum opacity [S1].
        *   $s_i$: Feature size of node $i$ [S1].
        *   $p_{ci}$: Cell persistence of node $i$ [S1].
        *   $d_i$: Depth of node $i$ in the hierarchy [S1].
        *   $d_{min}, d_{max}$: Minimum and maximum depths in the tree [S1].
        *   $g(d_i) = \frac{d_i - d_{min}}{d_{max} - d_{min}}$ [S1].
        *   $W$: Normalization weight such that the term $\frac{1}{s_i \cdot p_{ci}}$ is normalized to $[0, 1]$ [S1].

7.  **Generate Gaussian Transfer Functions (GTF):**
    *   For each valley cell $i$, define a Gaussian function $\alpha_i(\mathbf{x})$ over the feature space $\mathbf{x} = (v, g)$ [S1]:
        $$ \alpha_i(\mathbf{x}) = \alpha_{max, i} \cdot e^{-\frac{1}{2} (\mathbf{x} - \mu_i)^T \Sigma_i^{-1} (\mathbf{x} - \mu_i)} $$
    *   Parameters:
        *   $\mu_i$: Default is the position of the maximum density in the cell. Alternative: center of the cluster [S1].
        *   $\Sigma_i^{-1}$: Inverse of the covariance matrix of the data points in the cell [S1].
    *   Apply bounding box $B_i$: Set $\alpha_i(\mathbf{x}) = 0$ if $\mathbf{x} \notin B_i$ to prevent contribution to other features [S1].

8.  **User Interaction (Optional):**
    *   Allow the user to manually adjust $\alpha_{max}$ for specific cells [S1].
    *   Allow merging or splitting of cells in the hierarchy [S1].
    *   When merging cells, the resulting cell inherits the color of the cell with the larger maximum density [S1].
    *   When splitting, the new cell retains its original color [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_vcc_tf(volume_data, feature_extractor, persistence_threshold_ratio=0.01, alpha_min=0.0):
    # Step 1: Generate Density Plot
    density_plot = compute_continuous_scatterplot(volume_data, feature_extractor)
    
    # Step 2: Identify Valley Cells
    initial_cells = decompose_into_valley_cells(density_plot)
    
    # Step 3: Compute Cell Persistence
    for cell in initial_cells:
        cell.boundaries = find_adjacent_boundaries(cell, initial_cells)
        cell.persistence = min(boundary.persistence for boundary in cell.boundaries)
        
    # Step 4 & 5: Construct and Simplify Merge Tree
    merge_tree = build_merge_tree(initial_cells)
    
    max_persistence = max(cell.persistence for cell in initial_cells)
    threshold = max_persistence * persistence_threshold_ratio
    
    simplified_cells = []
    for node in merge_tree.traverse():
        if node.persistence >= threshold:
            simplified_cells.append(node)
            
    # Step 6: Compute Opacity Ranges
    depths = [node.depth for node in simplified_cells]
    d_min = min(depths)
    d_max = max(depths)
    
    weights = [1.0 / (node.size * node.persistence) for node in simplified_cells]
    W = 1.0 / max(weights) if max(weights) > 0 else 1.0
    
    for node in simplified_cells:
        g_di = (node.depth - d_min) / (d_max - d_min) if d_max != d_min else 1.0
        denominator = g_di * node.size * node.persistence
        node.alpha_max = alpha_min + W / denominator if denominator != 0 else alpha_min
        
    # Step 7: Generate GTFs
    gtfs = []
    for node in simplified_cells:
        data_points = get_data_points_in_cell(node, volume_data, feature_extractor)
        mu = compute_mean(data_points)
        sigma = compute_covariance(data_points)
        sigma_inv = inverse(sigma)
        bounding_box = compute_bounding_box(data_points)
        gtfs.append(GTF(mu, sigma_inv, node.alpha_max, bounding_box))
        
    return gtfs

class GTF:
    def __init__(self, mean, covariance_inv, alpha_max, bounding_box):
        self.mean = mean
        self.covariance_inv = covariance_inv
        self.alpha_max = alpha_max
        self.bounding_box = bounding_box
        
    def evaluate(self, feature_vector):
        if not is_inside_bounding_box(feature_vector, self.bounding_box):
            return 0.0
        diff = feature_vector - self.mean
        exponent = -0.5 * dot(diff, mat_vec_mul(self.covariance_inv, diff))
        return self.alpha_max * exp(exponent)

# Helper definitions
def compute_continuous_scatterplot(volume_data, feature_extractor):
    grid = [[0.0] * 256 for _ in range(256)]
    for x in volume_data:
        v, g = feature_extractor(x)
        i, j = int(v * 255), int(g * 255)
        grid[i][j] += 1.0
    return grid

def decompose_into_valley_cells(density_plot):
    cells = []
    visited = [[False] * 256 for _ in range(256)]
    for i in range(256):
        for j in range(256):
            if not visited[i][j]:
                cell = []
                stack = [(i, j)]
                while stack:
                    r, c = stack.pop()
                    if 0 <= r < 256 and 0 <= c < 256 and not visited[r][c]:
                        visited[r][c] = True
                        cell.append((r, c))
                        for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
                            nr, nc = r+dr, c+dc
                            if 0 <= nr < 256 and 0 <= nc < 256 and density_plot[nr][nc] > density_plot[r][c]:
                                stack.append((nr, nc))
                cells.append(Cell(cell))
    return cells

class Cell:
    def __init__(self, points):
        self.points = points
        self.boundaries = []
        self.persistence = 0.0
        self.size = len(points)
        self.depth = 0
        self.alpha_max = 0.0

def find_adjacent_boundaries(cell, all_cells):
    boundaries = []
    for r, c in cell.points:
        for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
            nr, nc = r+dr, c+dc
            if 0 <= nr < 256 and 0 <= nc < 256:
                neighbor = next((c for c in all_cells if (nr, nc) in c.points), None)
                if neighbor and neighbor != cell:
                    boundaries.append(Boundary(cell, neighbor, density_plot[r][c]))
    return boundaries

class Boundary:
    def __init__(self, cell1, cell2, saddle_density):
        self.cell1 = cell1
        self.cell2 = cell2
        self.persistence = saddle_density

def build_merge_tree(initial_cells):
    all_boundaries = []
    for cell in initial_cells:
        all_boundaries.extend(cell.boundaries)
    all_boundaries.sort(key=lambda b: b.persistence)
    tree = MergeTree(initial_cells, all_boundaries)
    return tree

class MergeTree:
    def __init__(self, leaves, boundaries):
        self.leaves = leaves
        self.boundaries = boundaries
    def traverse(self):
        return self.leaves

def get_data_points_in_cell(node, volume_data, feature_extractor):
    return [feature_extractor(v) for v in volume_data if node.points]

def compute_mean(points):
    if not points: return [0.0, 0.0]
    return [sum(p[0] for p in points)/len(points), sum(p[1] for p in points)/len(points)]

def compute_covariance(points):
    if len(points) < 2: return [[1.0, 0.0], [0.0, 1.0]]
    mu = compute_mean(points)
    cov = [[0.0, 0.0], [0.0, 0.0]]
    for p in points:
        dx, dy = p[0]-mu[0], p[1]-mu[1]
        cov[0][0] += dx*dx
        cov[0][1] += dx*dy
        cov[1][0] += dy*dx
        cov[1][1] += dy*dy
    n = len(points)
    return [[cov[0][0]/n, cov[0][1]/n], [cov[1][0]/n, cov[1][1]/n]]

def inverse(matrix):
    a, b, c, d = matrix[0][0], matrix[0][1], matrix[1][0], matrix[1][1]
    det = a*d - b*c
    if det == 0: return [[1.0, 0.0], [0.0, 1.0]]
    return [[d/det, -b/det], [-c/det, a/det]]

def compute_bounding_box(points):
    if not points: return (0,0,0,0)
    xs = [p[0] for p in points]
    ys = [p[1] for p in points]
    return (min(xs), min(ys), max(xs), max(ys))

def is_inside_bounding_box(point, box):
    return box[0] <= point[0] <= box[2] and box[1] <= point[1] <= box[3]

def dot(v1, v2):
    return sum(a*b for a, b in zip(v1, v2))

def mat_vec_mul(matrix, vector):
    return [sum(r*c for r, c in zip(row, vector)) for row in matrix]

def exp(x):
    import math
    return math.exp(x)
```

### Parameters
*   **Persistence Threshold:** Typically 1% of the maximal persistence [S1].
*   **Mean ($\mu$):** Default is the position of the maximum density in the cell. Alternative is the center of the cluster [S1].
*   **Density Plot Resolution:** $256 \times 256$ is used in experiments [S1].
*   **Error Threshold:** 50 for CSP generation [S1].

### Edge Cases and Pitfalls
*   **Occlusion of Small Features:** Small features may be occluded by larger ones. The opacity formula incorporates feature size ($s_i$) and persistence ($p_{ci}$) to mitigate this [S1].
*   **Depth Control:** The function $g(d_i)$ helps distinguish features at different hierarchy levels [S1].
*   **Boundary Contribution:** Without bounding boxes, Gaussian tails may contribute to other features. Bounding boxes are computed to zero out opacity outside the cell's spatial extent in feature space [S1].

## Subtopics

### Hierarchical Feature Representation
A multi-level feature structure created by successive persistence-driven merging of valley cells, enabling interactive exploration through feature splitting, merging, and opacity/color assignment [S1]. Unlike methods requiring a fixed number of clusters or regular widget shapes, this hierarchy adapts to arbitrary feature counts/shapes and incorporates user prior knowledge without manual cluster specification [S1]. The user can interactively merge or split features at any level of the hierarchy, with opacity determined by persistence and size [S1].

### Valley Cell-Based Clustering Transfer Function Generation
The generation of a transfer function by assigning colors to valley cells using a qualitative color scheme [S1]. When merging cells in the hierarchy, the merged cell is colored with the color of the cell having the larger maximum density. When splitting cells, the original color of the split cell is restored [S1]. This allows users to visually track feature lineage during interaction [S1].

## Operating characteristics

*   **Time Complexity:** Density plot generation is the most expensive step, taking less than 1 hour for a $256 \times 256$ plot with an error threshold of 50 on an Intel Core 2 Duo E6320 [S1]. Clustering and tree construction take approximately 10 seconds [S1].
*   **Memory Complexity:** Requires storage for the density plot ($R^2$ cells), the merge tree, and the Gaussian parameters for each cell. The density plot generation used 2.0 GB RAM and 256 MB VRAM in the reference implementation [S1].
*   **Rendering Performance:** Rendering frame rate is approximately 25 fps for fewer than 10 valley cells on an NVIDIA GeForce GTX 260 [S1]. Rendering performance gradually decreases with the number of valley cells, because the GTF-based volume rendering is directly evaluated on the GPU [S1].
*   **Strengths:** Automatically determines the number of features; handles arbitrary cluster shapes; provides a hierarchical structure for user exploration; mitigates noise via persistence-based simplification [S1].
*   **Limitations:** Rendering performance scales poorly with a large number of cells; requires manual tuning of the persistence threshold for optimal simplification; density plot generation can be slow for very large volumes [S1].
*   **When to Use:** Suitable for medical data (CT Teeth, CT facial deformity) and simulation data (Supernova, Turbulent slip surface) where revealing specific structures like enamel, dentine, pulp, lesions, or vortex tubes is required [S1].

## Background

The method was introduced by Y. Wang et al. in 2012 in the paper "Automating Transfer Function Design with Valley Cell-Based Clustering of 2D Density Plots" [S1]. It builds on the principle that features tend to form peaks in the density function of 2D scatterplots [S1]. The approach uses Morse complex theory to decompose the density plot into valley cells and introduces cell persistence to measure separability and guide simplification [S1]. It was designed to overcome the limitations of Binning-based Transfer Functions (BTF) and manual brushing, offering a non-parametric clustering algorithm that structures 2D density plots into meaningful volumetric features [S1].

## Sources

[S1] Automating Transfer Function Design with Valley Cell‐Based Clustering of 2D Density Plots, https://doi.org/10.1111/j.1467-8659.2012.03122.x, fetched 2026-11-01
