# Self-Organizing Map Dimensional Reduction for Transfer Functions

Also called: Dimensional reduction for TF design, Multi-dimensional Transfer Function Design using Self-Organizing Maps, SOM-based TF design, Voxel Signature Mapping

## Overview
This method simplifies the design of multi-dimensional (nD) transfer functions by reducing the dimensionality of voxel signatures using Self-Organizing Maps (SOMs) [S1]. It maps nD voxel signatures to a 2D map space, allowing the user to define optical properties in the 2D space, which is then composed with the dimensional reduction function to form the actual nD transfer function [S1]. Unlike explicit nD TF definition, this method uses a simplified 2D interaction space via SOMs, making exploration and visualization of the TF domain easier as dimensionality increases [S1]. Unlike volume segmentation techniques, this method performs non-discrete classification, which can represent uncertainty [S1].

## How it works
**Inputs:**
*   Volume data $V$ with dimensions $D_x \times D_y \times D_z$.
*   Voxel signature attributes: A set of $k$ attributes per voxel (e.g., scalar value, gradient magnitude, directional second derivative, or statistical measures like mean, standard deviation, cubic root of third-order moment from a subvolume) [S1].
*   Attribute weights: A vector $\mathbf{w}_{attr}$ of size $k$ defining the importance of each attribute [S1].
*   SOM parameters: Map topology (Kohonen grid or Spherical), map size (e.g., $32 \times 32$ cells for Kohonen, 10 subdivisions for Spherical), learning rates $\eta_1=0.3$ and $\eta_2=0.5$, and training steps [S1].

**Outputs:**
*   A multidimensional transfer function $TF(\mathbf{x})$ that maps each voxel $\mathbf{x}$ to optical properties (color $C$ and opacity $\alpha$) [S1].
*   The mapping is defined by the composition of the dimensional reduction function $R: \mathbb{R}^k \to \mathbb{R}^2$ (implemented via SOM weights and RBFs) and the 2D transfer function $TF_{2D}: \mathbb{R}^2 \to \mathbb{R}^4$ (defined by the user in the map space) [S1].

**Data Structures:**
*   **Voxel Signature Vector:** $\vec{S}_i \in \mathbb{R}^k$ for each voxel $i$. Values are normalized [S1].
*   **SOM Grid:** A set of cells $C_i$. Each cell $C_i$ has a weight vector $\vec{W}(C_i) \in \mathbb{R}^k$ [S1].
    *   For Kohonen maps: 2D integer coordinates $(u, v)$ [S1].
    *   For Spherical maps: Nodes on a subdivided icosahedron [S1].
*   **Best Matching Unit (BMU):** For a signature $\vec{T}$, the BMU is the cell $C_{BMU}$ such that $d(\vec{W}(C_{BMU}), \vec{T})$ is minimized, where $d$ is the weighted Euclidean distance [S1].

**Numbered Steps:**

1.  **Preprocessing and Background Removal:**
    *   Extract $k$ attributes for each voxel to form signature vectors $\vec{S}_i$. Normalize these values [S1].
    *   Identify background voxels (e.g., air in CT/MRI) in the exterior regions. Use region growing with exterior seeds to identify background voxels. Remove them from the training set if they do not carry useful information [S1]. Let $V_{nb}$ be the set of non-background voxels [S1].

2.  **SOM Initialization:**
    *   Initialize weight vectors $\vec{W}(C_i)$ for all cells $C_i$ in the SOM with random values between 0 and 1. The dimension of $\vec{W}$ must match the dimension $k$ of the input signatures [S1].

3.  **SOM Training Stage 1 (Global Structure):**
    *   Present non-background voxel signatures $\vec{T} \in V_{nb}$ in random order.
    *   For each $\vec{T}$, find the BMU $C_{BMU}$ using weighted Euclidean distance [S1].
    *   Update weights of $C_{BMU}$ and its neighbors using the Gaussian neighborhood function $nf1(td)$ and learning rate $\eta_1 = 0.3$ [S1]:
        $$ a = \eta_1 \times nf1(td(BMU, C_i)) $$
        $$ \vec{W}_{t+1}(C_i) = (1-a) \times \vec{W}_t(C_i) + a \times \vec{T} $$
        where $td$ is the topological distance (Euclidean distance between integer 2D coordinates for Kohonen, or number of edges for Spherical) [S1].
        $$ nf1(td) = \exp\left(-\frac{td^2}{3}\right) $$
    *   Repeat for the specified number of steps (e.g., 3,000,000 steps for a $32 \times 32$ map) [S1].

4.  **SOM Training Stage 2 (Refinement):**
    *   Continue presenting non-background signatures in random order.
    *   Use the modified neighborhood function $nf2(td, d)$ and learning rate $\eta_2 = 0.5$ [S1]:
        $$ a = \eta_2 \times nf2(td(BMU, C_i)) $$
        $$ \vec{W}_{t+1}(C_i) = (1-a) \times \vec{W}_t(C_i) + a \times \vec{T} $$
        where $d$ is the distance between the training case $\vec{T}$ and the winner cell's weight vector $\vec{W}(C_{BMU})$ [S1].
        $$ nf2(td, d) = \min(d, 1) \times \exp\left(-\frac{td^2}{3}\right) $$
    *   This allows signatures far from their winner cell to have more influence, preventing large homogeneous regions from dominating [S1].
    *   Repeat for the specified number of steps (e.g., 4,500,000 steps for Kohonen) [S1].

5.  **Dimensional Reduction:**
    *   Map all voxels (including background) to the 2D map space. For each voxel $i$ with signature $\vec{S}_i$, find its BMU $C_{BMU}$ in the trained SOM [S1].
    *   Assign the coordinates of $C_{BMU}$ (2D coordinates for Kohonen, or spherical coordinates projected to 2D) as the reduced signature for voxel $i$ [S1].
    *   Note: This step replaces the nD signature with 2D map coordinates. The time for this step is proportional to the number of non-background voxels, map size, and weight vector size [S1].

6.  **TF Design in 2D Space:**
    *   The user interacts with the 2D map space to define the transfer function. This is described as "dual domain interaction" [S1].
    *   **Automatic Color TF:** Assign different colors to different regions of the SOM map, which correspond to voxels with considerably different attribute values [S1].
    *   **Manual Opacity/Color TF:** The user can move Gaussian opacity functions on the map space to tune parameters and build meaningful visualizations [S1].
    *   The 2D TF maps the 2D map coordinates to optical properties $(r, g, b, \alpha)$ [S1].

7.  **Rendering:**
    *   The actual multidimensional transfer function is the composition of the dimensional reduction function (mapping nD signature to 2D map coordinates) and the 2D transfer function (mapping 2D map coordinates to optical properties) [S1].
    *   Evaluate Radial Basis Functions (RBF) for each non-background voxel to determine the transfer function response. This is solved using the Lapack library [S1].
    *   Render the volume using OpenGL and Cg. Use hardware interpolation for map coordinates and precomputed normals by default. Allow user selection of color/opacity interpolation and on-the-fly normal computation for high-quality rendering [S1].
    *   Note: Precomputed normals cannot be used for multivariate data sets (e.g., Hurricane) because the attributes lack a clear physical meaning for normal orientation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial.distance import cdist

def weighted_euclidean_distance(v1, v2, weights):
    """Calculate weighted Euclidean distance."""
    diff = (v1 - v2) * weights
    return np.sqrt(np.sum(diff**2))

def gaussian_neighborhood(td):
    """Equation 3: Gaussian neighborhood function."""
    return np.exp(-(td**2) / 3.0)

def modified_neighborhood(td, d):
    """Equation 4: Modified neighborhood function."""
    return min(d, 1.0) * np.exp(-(td**2) / 3.0)

def train_som(signatures, map_size, num_steps_stage1, num_steps_stage2, eta1=0.3, eta2=0.5, attr_weights=None):
    """
    Train SOM on non-background voxel signatures.
    
    Args:
        signatures: Array of shape (N, k) containing normalized voxel signatures.
        map_size: Tuple (rows, cols) for Kohonen map.
        num_steps_stage1: Number of training steps for Stage 1.
        num_steps_stage2: Number of training steps for Stage 2.
        eta1: Learning rate for Stage 1.
        eta2: Learning rate for Stage 2.
        attr_weights: Array of shape (k,) with weights for each attribute.
    
    Returns:
        weights: Array of shape (map_size[0]*map_size[1], k) with trained weight vectors.
    """
    if attr_weights is None:
        attr_weights = np.ones(signatures.shape[1])
    
    n_cells = map_size[0] * map_size[1]
    k = signatures.shape[1]
    
    # Initialize weights with random values between 0 and 1
    weights = np.random.rand(n_cells, k)
    
    # Precompute cell coordinates for topological distance calculation
    cell_coords = np.array([(i // map_size[1], i % map_size[1]) for i in range(n_cells)])
    
    def get_topological_distance(idx_winner, idx_cell):
        """Calculate topological distance between two cells."""
        coord_winner = cell_coords[idx_winner]
        coord_cell = cell_coords[idx_cell]
        return np.sqrt(np.sum((coord_winner - coord_cell)**2))
    
    # Stage 1: Global Structure
    for step in range(num_steps_stage1):
        # Present signatures in random order
        idx = np.random.randint(0, signatures.shape[0])
        T = signatures[idx]
        
        # Find BMU
        distances = np.array([weighted_euclidean_distance(T, weights[i], attr_weights) for i in range(n_cells)])
        bmu_idx = np.argmin(distances)
        
        # Update weights
        for i in range(n_cells):
            td = get_topological_distance(bmu_idx, i)
            nf = gaussian_neighborhood(td)
            a = eta1 * nf
            weights[i] = (1 - a) * weights[i] + a * T
            
    # Stage 2: Refinement
    for step in range(num_steps_stage2):
        idx = np.random.randint(0, signatures.shape[0])
        T = signatures[idx]
        
        # Find BMU
        distances = np.array([weighted_euclidean_distance(T, weights[i], attr_weights) for i in range(n_cells)])
        bmu_idx = np.argmin(distances)
        d = distances[bmu_idx] # Distance to winner
        
        # Update weights
        for i in range(n_cells):
            td = get_topological_distance(bmu_idx, i)
            nf = modified_neighborhood(td, d)
            a = eta2 * nf
            weights[i] = (1 - a) * weights[i] + a * T
            
    return weights

def dimensional_reduction(signatures, weights, map_size, attr_weights=None):
    """
    Map voxel signatures to 2D map coordinates.
    
    Args:
        signatures: Array of shape (N, k) containing voxel signatures.
        weights: Array of shape (map_size[0]*map_size[1], k) with trained weight vectors.
        map_size: Tuple (rows, cols) for Kohonen map.
        attr_weights: Array of shape (k,) with weights for each attribute.
        
    Returns:
        map_coords: Array of shape (N, 2) with 2D coordinates for each voxel.
    """
    if attr_weights is None:
        attr_weights = np.ones(signatures.shape[1])
        
    n_cells = map_size[0] * map_size[1]
    N = signatures.shape[0]
    map_coords = np.zeros((N, 2))
    
    for i in range(N):
        T = signatures[i]
        # Find BMU
        distances = np.array([weighted_euclidean_distance(T, weights[j], attr_weights) for j in range(n_cells)])
        bmu_idx = np.argmin(distances)
        
        # Get 2D coordinates of BMU
        row = bmu_idx // map_size[1]
        col = bmu_idx % map_size[1]
        map_coords[i] = [row, col]
        
    return map_coords

def render_volume(volume_data, map_coords, tf_2d_func, attr_weights=None):
    """
    Render volume using the 2D transfer function and map coordinates.
    
    Args:
        volume_data: 3D array of voxel intensities (for ray casting).
        map_coords: Array of shape (N, 2) with 2D coordinates for each voxel.
        tf_2d_func: Function that takes 2D coordinates and returns (r, g, b, alpha).
        attr_weights: Not directly used here, but assumed used in map_coords calculation.
        
    Returns:
        image: Rendered 2D image.
    """
    # This is a simplified representation of the rendering process.
    # In practice, this involves ray casting through the volume.
    # For each sample point along the ray, the voxel's map coordinates are retrieved.
    # The 2D TF is evaluated at these coordinates to get optical properties.
    # These properties are used in the volume rendering integral.
    
    # Example: Precompute optical properties for all voxels
    N = map_coords.shape[0]
    optical_properties = np.zeros((N, 4))
    for i in range(N):
        optical_properties[i] = tf_2d_func(map_coords[i])
        
    # Ray casting loop would go here, using optical_properties for shading
    # ...
    
    return image
```

**Parameters with Typical Values:**
*   **Learning Rate ($\eta$):** 0.3 for Stage 1, 0.5 for Stage 2 [S1].
*   **Map Size:** At least $20 \times 20$ cells for Kohonen maps; typical implementation uses $32 \times 32$ cells [S1]. For Spherical maps, at least seven subdivisions by icosahedron edge; typical implementation uses ten subdivisions [S1].
*   **Attribute Weights:**
    *   Statistical variables (mean, std dev, cubic root 3rd moment): 1.3, 1.0, 0.7 respectively [S1].
    *   Scalar and first/second derivative values: 1.0 for scalar, 1.0 for first derivative, 0.7 for second derivative [S1].
    *   Hurricane dataset attributes: 1.0 for wind speed, 1.0 for pressure, 0.5 for temperature [S1].
*   **Distance Metric:** Weighted Euclidean distance [S1].

**Edge Cases and Pitfalls:**
*   **Background Influence:** Background voxels (e.g., air) have high occurrence and can negatively influence the map if included in training. They should be removed via region growing from exterior seeds if they do not carry useful information [S1].
*   **Homogeneous Region Dominance:** Without the modified neighborhood function (Equation 4), large homogeneous regions of the volume would dominate the map, while important regions with fewer voxels (signatures far from their winner cells) would be badly represented [S1].
*   **Dimensional Reduction Loss:** Dimensional reduction causes loss and distortion of information, making accurate quantitative analysis impossible. The method is suitable only for qualitative aspects like shape and dissimilarity [S1].
*   **Normals for Multivariate Data:** Precomputed normals cannot be used for multivariate data sets (e.g., Hurricane) because the attributes lack a clear physical meaning for normal orientation [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Training time is quadratically related to map size (number of cells) and proportional to weight vector size [S1].
    *   Winner search is linear in number of cells [S1].
    *   Weight vector size has linear influence on winner search and update time [S1].
    *   Dimensional reduction time is proportional to the number of non-background voxels, map size, and weight vector size [S1].
    *   RBF evaluation time is proportional to the number of cells and the size of weight vectors [S1].
*   **Measured Performance:**
    *   Training time for $32 \times 32$ Kohonen map: 57 seconds for 3,000,000 steps (Neigh. func 1) and 87 seconds for 4,500,000 steps (Neigh. func 2) [S1].
    *   Training time for Spherical map (10 subdivisions): 151 seconds for 3,000,000 steps (Neigh. func 1) and 264 seconds for 5,250,000 steps (Neigh. func 2) [S1].
    *   Dimensional reduction time for Engine dataset (256x2x128, 5,297,265 non-bg voxels): 140 seconds [S1].
    *   Dimensional reduction time for Tooth dataset (128x2x256, 4,193,888 non-bg voxels): 110 seconds [S1].
    *   Dimensional reduction time for Foot dataset (256^3, 4,890,753 non-bg voxels): 133 seconds [S1].
    *   Dimensional reduction time for Carp dataset (256x2x512, 32,806,605 non-bg voxels): 845 seconds [S1].
    *   Dimensional reduction time for Hurricane dataset (256x2x128, 8,388,608 non-bg voxels): 220 seconds [S1].
    *   Dimensional reduction time for Sheep heart dataset (256^3, 16,776,721 non-bg voxels): 375 seconds [S1].
    *   Spherical map dimensional reduction time is approximately 13% greater than Kohonen map time [S1].
*   **Strengths and Limitations:**
    *   **Strengths:** Simplifies nD TF design by reducing to 2D interaction space. Allows non-discrete classification representing uncertainty. Automatic color TFs can emphasize details hard to visualize with other approaches. Powerful built-in classification scheme [S1].
    *   **Limitations:** Loss and distortion of information prevents accurate quantitative analysis. Suitable only for qualitative aspects [S1]. Training and dimensional reduction can be computationally expensive for large datasets [S1].
*   **When to Use:** When designing multi-dimensional transfer functions for qualitative visualization, especially when the user needs to explore the TF domain easily and the data has complex multi-attribute signatures [S1].
*   **When Not to Use:** When accurate quantitative analysis is required, or when the data is purely multivariate without clear physical meaning for normals (if precomputed normals are desired) [S1].

## Background
*   **Origin:** The method was proposed in the paper "Design of Multi-dimensional Transfer Functions Using Dimensional Reduction" [S1].
*   **History and Development:** The paper presents a new approach for the design of multi-dimensional transfer functions that uses self-organizing maps to perform dimensional reduction of voxel attributes. It builds on the concept of SOMs for unsupervised learning and dimensional reduction [S1].
*   **Usage:** Used in a visualization framework that uses graphics hardware to provide interactive volume rendering, with a user interface based on dual domain interaction [S1]. Implemented using OpenGL, Cg, GLUT, GLUI, and Lapack library [S1].

## Sources
[S1] Design of Multi-dimensional Transfer Functions Using Dimensional Reduction, https://doi.org/10.2312/vissym/eurovis07/131-138, fetched 2026-10-26
