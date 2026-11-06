# Self-Organizing Map Dimensional Reduction for Transfer Functions

Also called: Dimensional reduction for TF design, Multi-dimensional Transfer Function Design using Self-Organizing Maps, SOM-based TF design, Voxel Signature Mapping

## Overview
This method simplifies the design of multi-dimensional (nD) transfer functions by reducing the dimensionality of voxel signatures using Self-Organizing Maps (SOMs) [S1]. It maps nD voxel signatures to a 2D map space, allowing the user to define optical properties in the 2D space, which is then composed with the dimensional reduction function to form the actual nD transfer function [S1]. Unlike explicit nD TF definition, this approach uses a simplified 2D interaction space, making exploration easier as dimensionality increases [S1]. It performs non-discrete classification, representing uncertainty, unlike volume segmentation techniques [S1].

## How it works
**Inputs:**
*   Volume data (scalar or multivariate).
*   User-defined voxel signature attributes (e.g., scalar value, gradient magnitude, directional second derivative, or statistical measures like mean, standard deviation, cubic root of third-order moment from a subvolume centered at the voxel) [S1].
*   Weights for each attribute in the signature vector [S1].
*   Map topology type (Kohonen grid or Spherical) and size [S1].
*   Learning rate parameters $\eta$ [S1].

**Outputs:**
*   A 2D map (Kohonen or Spherical) with weight vectors.
*   A mapping of every voxel to a 2D coordinate (Best Matching Unit) in the map space [S1].
*   A 2D transfer function defined by the user (e.g., Gaussian opacity functions, automatic color mapping) [S1].
*   The final rendered volume [S1].

**Data Structures:**
1.  **Voxel Signature Vector:** An nD vector $\vec{T}$ containing normalized attribute values for a voxel.
2.  **SOM Grid:**
    *   **Kohonen:** A 2D grid of cells. Each cell $C_i$ has a weight vector $\tilde{W}(C_i)$ of dimension n. Coordinates are integer 2D indices.
    *   **Spherical:** A mesh built by subdividing an icosahedron. Each node has a weight vector $\tilde{W}$ of dimension n. Topological distance is the number of edges in the shortest path between nodes [S1].
3.  **Map Coordinates:** For each voxel, a 2D coordinate corresponding to its Best Matching Unit (BMU) in the SOM grid [S1].

**Parameters:**
*   **Learning Rate ($\eta$):** Constant between 0 and 1. Typical values: $\eta=0.3$ for Stage 1, $\eta=0.5$ for Stage 2 [S1].
*   **Map Size:**
    *   Kohonen: At least 20x20 cells; typical implementation uses 32x32 cells [S1].
    *   Spherical: At least seven subdivisions by icosahedron edge; typical implementation uses ten subdivisions [S1].
*   **Attribute Weights:** Used in the weighted Euclidean distance metric.
    *   Statistical variables (mean, std dev, cubic root 3rd moment): 1.3, 1.0, 0.7 respectively [S1].
    *   Scalar and first/second derivative values: 1.0, 1.0, 0.7 respectively [S1].
    *   Hurricane dataset (wind speed, pressure, temperature): 1.0, 1.0, 0.5 respectively [S1].
*   **Distance Metric:** Weighted Euclidean distance [S1].

**Algorithm Steps:**

1.  **Preprocessing:**
    *   Extract nD voxel signatures from volume data based on user-selected attributes [S1].
    *   Normalize the signatures [S1].

2.  **Background Removal (Optional):**
    *   Identify background voxels (e.g., air) in exterior regions [S1].
    *   Use region growing with exterior background voxels as seeds to identify and remove background voxels from the training set if desired [S1].

3.  **SOM Initialization:**
    *   Initialize weight vectors $\tilde{W}(C_i)$ for all cells/nodes in the SOM grid with random values between 0 and 1. The dimension of $\tilde{W}$ must match the dimension of the input voxel signatures [S1].

4.  **SOM Training Stage 1 (Global Structure):**
    *   Set learning rate $\eta = 0.3$ [S1].
    *   Present non-background voxel signatures as training cases $\vec{T}$ in random order [S1].
    *   For each training case $\vec{T}$:
        *   Find the Best Matching Unit (BMU) $C_{BMU}$: the cell/node with the minimum weighted Euclidean distance to $\vec{T}$ [S1].
        *   Update weights for all cells/nodes $C_i$ using the Gaussian neighborhood function $nf1(td)$:
            $$ a = \eta \times nf1(td(C_{BMU}, C_i)) $$
            $$ \tilde{W}_{t+1}(C_i) = (1-a) \times \tilde{W}_t(C_i) + a \times \vec{T} $$
            where $td$ is the topological distance (Euclidean distance between integer 2D coordinates for Kohonen, number of edges for Spherical) [S1].
        *   The neighborhood function is:
            $$ nf1(td) = \exp\left(-\frac{td^2}{3}\right) $$
    *   Repeat for a sufficient number of steps (e.g., 3,000,000 steps for 32x32 map) [S1].

5.  **SOM Training Stage 2 (Refinement):**
    *   Set learning rate $\eta = 0.5$ [S1].
    *   Present non-background voxel signatures in random order [S1].
    *   For each training case $\vec{T}$:
        *   Find the BMU $C_{BMU}$ [S1].
        *   Calculate the distance $d$ between $\vec{T}$ and $\tilde{W}(C_{BMU})$ [S1].
        *   Update weights for all cells/nodes $C_i$ using the modified neighborhood function $nf2(td, d)$:
            $$ a = \eta \times nf2(td(C_{BMU}, C_i), d) $$
            $$ \tilde{W}_{t+1}(C_i) = (1-a) \times \tilde{W}_t(C_i) + a \times \vec{T} $$
        *   The neighborhood function is:
            $$ nf2(td, d) = \min(d, 1) \times \exp\left(-\frac{td^2}{3}\right) $$
    *   This allows signatures far from their winner cell to have more influence, preventing large homogeneous regions from dominating [S1].
    *   Repeat for a sufficient number of steps (e.g., 4,500,000 steps for 32x32 map) [S1].

6.  **Dimensional Reduction:**
    *   For every voxel (including background) in the volume:
        *   Compute its nD signature vector $\vec{T}$ [S1].
        *   Find the BMU $C_{BMU}$ in the trained SOM grid (minimum distance to $\vec{T}$) [S1].
        *   Assign the 2D coordinates of $C_{BMU}$ to the voxel [S1].
    *   This replaces the nD signature with 2D map coordinates [S1].

7.  **TF Design in 2D Space:**
    *   The user interacts with the 2D map space to define the transfer function [S1].
    *   **Automatic Color TF:** Assign different colors to different regions of the SOM map, corresponding to voxels with considerably different attribute values [S1].
    *   **Manual Opacity TF:** Move Gaussian opacity functions on the map space to tune parameters [S1].
    *   The 2D TF maps 2D map coordinates to optical properties (color, opacity) [S1].

8.  **Rendering:**
    *   The actual nD transfer function is the composition of the dimensional reduction function (nD signature $\to$ 2D map coordinates) and the 2D TF (2D map coordinates $\to$ optical properties) [S1].
    *   Evaluate Radial Basis Functions (RBF) for each non-background voxel to determine the transfer function response. This involves solving a system of equations, typically using a library like Lapack [S1].
    *   Render the volume using hardware-accelerated rendering (e.g., OpenGL, Cg) [S1].
    *   Use hardware interpolation for map coordinates and precomputed normals by default [S1].
    *   Allow user selection of color/opacity interpolation and on-the-fly normal computation for high-quality rendering [S1].
    *   Note: Precomputed normals cannot be used for multivariate datasets (e.g., Hurricane) because attributes lack clear physical meaning for normal orientation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def weighted_euclidean_distance(vec1, vec2, weights):
    """Calculate weighted Euclidean distance between two vectors."""
    dist_sq = 0.0
    for v1, v2, w in zip(vec1, vec2, weights):
        diff = v1 - v2
        dist_sq += w * diff * diff
    return math.sqrt(dist_sq)

def gaussian_neighborhood(td):
    """Equation 3: Gaussian neighborhood function."""
    return math.exp(-(td * td) / 3.0)

def modified_neighborhood(td, d):
    """Equation 4: Modified neighborhood function."""
    return min(d, 1.0) * math.exp(-(td * td) / 3.0)

def find_bmu(voxel_signature, som_weights, weights, map_type, map_grid):
    """Find the Best Matching Unit (BMU) for a voxel signature."""
    min_dist = float('inf')
    bmu = None
    
    if map_type == "kohonen":
        # Iterate over 2D grid
        for y in range(map_grid.height):
            for x in range(map_grid.width):
                cell = map_grid.get_cell(x, y)
                dist = weighted_euclidean_distance(voxel_signature, cell.weight, weights)
                if dist < min_dist:
                    min_dist = dist
                    bmu = cell
    elif map_type == "spherical":
        # Iterate over spherical nodes
        for node in map_grid.nodes:
            dist = weighted_euclidean_distance(voxel_signature, node.weight, weights)
            if dist < min_dist:
                min_dist = dist
                bmu = node
    return bmu, min_dist

def get_topological_distance(cell1, cell2, map_type):
    """Calculate topological distance between two cells."""
    if map_type == "kohonen":
        # Euclidean distance between integer 2D coordinates
        dx = cell1.x - cell2.x
        dy = cell1.y - cell2.y
        return math.sqrt(dx*dx + dy*dy)
    elif map_type == "spherical":
        # Number of edges in shortest path (BFS)
        # Note: In implementation, this might be precomputed or approximated
        return bfs_shortest_path_length(map_grid, cell1, cell2)
    return 0

def train_som_stage1(non_bg_signatures, som_grid, weights, eta, map_type, num_steps):
    """Stage 1: Global structure training."""
    for step in range(num_steps):
        # Present training cases in random order
        # (In practice, shuffle signatures once or pick randomly each step)
        signature = random.choice(non_bg_signatures)
        
        bmu, _ = find_bmu(signature, som_grid.weights, weights, map_type, som_grid)
        
        for cell in som_grid.all_cells():
            td = get_topological_distance(bmu, cell, map_type)
            nf = gaussian_neighborhood(td)
            a = eta * nf
            
            # Update weights
            for i in range(len(cell.weight)):
                cell.weight[i] = (1 - a) * cell.weight[i] + a * signature[i]

def train_som_stage2(non_bg_signatures, som_grid, weights, eta, map_type, num_steps):
    """Stage 2: Refinement training."""
    for step in range(num_steps):
        signature = random.choice(non_bg_signatures)
        
        bmu, dist_to_bmu = find_bmu(signature, som_grid.weights, weights, map_type, som_grid)
        
        for cell in som_grid.all_cells():
            td = get_topological_distance(bmu, cell, map_type)
            nf = modified_neighborhood(td, dist_to_bmu)
            a = eta * nf
            
            # Update weights
            for i in range(len(cell.weight)):
                cell.weight[i] = (1 - a) * cell.weight[i] + a * signature[i]

def dimensional_reduction(volume_data, som_grid, weights, map_type):
    """Map all voxels to 2D map coordinates."""
    voxel_map = {} # Map voxel_id to (x, y) coordinates
    
    for voxel in volume_data:
        signature = voxel.signature
        bmu, _ = find_bmu(signature, som_grid.weights, weights, map_type, som_grid)
        
        if map_type == "kohonen":
            voxel_map[voxel.id] = (bmu.x, bmu.y)
        elif map_type == "spherical":
            # Map spherical node to 2D projection or use node index if spherical rendering
            # The source implies mapping to 2D map space, so likely a projection or 2D representation
            voxel_map[voxel.id] = bmu.get_2d_coordinates() 
            
    return voxel_map

def render_volume(volume_data, voxel_map, tf_2d, som_grid, weights, map_type):
    """Render volume using the composed TF."""
    # 1. Evaluate RBF for each non-background voxel to get TF response
    # This step typically involves solving a system of equations using Lapack
    # to determine the contribution of each SOM node to the voxel's optical properties
    # based on the distance in signature space.
    
    # 2. Compose TF: nD signature -> 2D map coord -> optical properties
    for voxel in volume_data:
        if voxel.is_background:
            continue
            
        map_coords = voxel_map[voxel.id]
        optical_props = tf_2d.evaluate(map_coords) # Color, Opacity
        
        # 3. Render using OpenGL/Cg
        # Use hardware interpolation for map coordinates
        # Use precomputed normals if scalar data, else on-the-fly normals
        render_voxel(voxel, optical_props)
```

## Operating characteristics
*   **Training Time:** Quadratically related to map size (number of cells) and proportional to weight vector size [S1]. Winner search is linear in the number of cells [S1].
    *   32x32 Kohonen map: 57 seconds for 3,000,000 steps (Stage 1) and 87 seconds for 4,500,000 steps (Stage 2) [S1].
    *   Spherical map (10 subdivisions): 151 seconds for 3,000,000 steps (Stage 1) and 264 seconds for 5,250,000 steps (Stage 2) [S1].
*   **Dimensional Reduction Time:** Proportional to the number of non-background voxels, map size, and weight vector size [S1].
    *   Engine (5.3M non-bg): 140 seconds [S1].
    *   Tooth (4.2M non-bg): 110 seconds [S1].
    *   Foot (4.9M non-bg): 133 seconds [S1].
    *   Carp (32.8M non-bg): 845 seconds [S1].
    *   Hurricane (8.4M non-bg): 220 seconds [S1].
    *   Sheep heart (16.8M non-bg): 375 seconds [S1].
    *   Spherical map dimensional reduction time is approximately 13% greater than Kohonen map time [S1].
*   **RBF Evaluation Time:** Proportional to the number of cells and the size of weight vectors [S1].
*   **Strengths:** Simplifies nD TF design by reducing it to 2D interaction; handles arbitrary dimensionality of voxel signatures; non-discrete classification represents uncertainty; automatic color TFs emphasize details [S1].
*   **Limitations:** Dimensional reduction causes loss and distortion of information, making accurate quantitative analysis impossible; suitable only for qualitative aspects like shape and dissimilarity [S1]. Precomputed normals cannot be used for multivariate data [S1].
*   **When to use:** When designing multi-dimensional transfer functions for qualitative visualization and exploration of complex data structures [S1].
*   **When not to use:** When accurate quantitative analysis of the volume data is required [S1].

## Background
*   **Origin:** Proposed in "Design of Multi-dimensional Transfer Functions Using Dimensional Reduction" (EuroVis 2007) [S1].
*   **History:** The method builds on the concept of Self-Organizing Maps (SOMs) for unsupervised learning and dimensional reduction [S1]. It addresses the difficulty of designing multi-dimensional transfer functions, which becomes unintuitive as the number of dimensions increases [S1].
*   **Usage:** Used in a visualization framework with interactive volume rendering, dual domain interaction, OpenGL, Cg, GLUT, GLUI, and Lapack [S1].

## Sources
[S1] Design of Multi-dimensional Transfer Functions Using Dimensional Reduction, https://doi.org/10.2312/vissym/eurovis07/131-138, fetched 2026-10-26
