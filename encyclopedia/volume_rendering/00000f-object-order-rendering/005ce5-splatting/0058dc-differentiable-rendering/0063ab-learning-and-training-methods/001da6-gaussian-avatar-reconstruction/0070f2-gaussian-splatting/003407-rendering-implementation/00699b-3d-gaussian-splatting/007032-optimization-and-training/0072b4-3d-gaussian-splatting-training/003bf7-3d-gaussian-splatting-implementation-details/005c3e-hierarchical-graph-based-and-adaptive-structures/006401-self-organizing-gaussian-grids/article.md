# Self-Organizing Gaussian Grids

Also called: Self-Organizing Gaussians

## Overview
Self-Organizing Gaussian Grids is a compact 3D scene representation that organizes the high-dimensional parameters of 3D Gaussian Splatting (3DGS) into a 2D grid with local homogeneity to drastically reduce storage size without compromising visual quality [S1]. It differs from fixed-ordering methods like Z-order curves or tri-planes by learning a scene-specific permutation of Gaussians via unsupervised competitive learning, thereby preserving topological relationships and enabling efficient image-based compression [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ 3D Gaussians $\{g_i\}_{i=1}^N$ representing a static scene, where each Gaussian $g_i$ contains position $\mathbf{x}_i$, covariance $\mathbf{\Sigma}_i$ (or rotation/scale), opacity $\alpha_i$, and spherical harmonic coefficients for color $\mathbf{c}_i$.
*   **Output:** A 2D grid structure $G$ of size $H \times W$ (where $H \times W \approx N$) containing the permuted Gaussian parameters, organized into multiple data layers (e.g., one layer for position/scale/rotation, one for opacity, one for color) that share the same 2D layout [S1, S2].

**Data Structures**
The representation consists of a 2D grid where each cell $(u, v)$ holds a reference or copy of a specific Gaussian from the original set. The grid is organized into separate channels or layers for different attribute types to facilitate standard 2D image compression codecs [S2].
*   **Grid Dimensions:** $H \times W$. The product $H \times W$ is chosen to be approximately equal to the number of Gaussians $N$.
*   **Cell Content:** Each cell stores the full parameter set of a single Gaussian, identical to the standard 3DGS structure [S1].
*   **Layers:** Distinct 2D grids are maintained for geometric attributes (position, scale, rotation) and appearance attributes (opacity, SH coefficients) [S2].

**Algorithm Steps**
1.  **Initialization:** Initialize a 2D grid of size $H \times W$ with random or uniformly sampled Gaussian parameters from the input set [S2].
2.  **Self-Organization (Competitive Learning):**
    *   Iterate through the input Gaussians.
    *   For each input Gaussian $g_i$, find the "winner" neuron (grid cell) $w$ whose current parameter vector is most similar to $g_i$ in the high-dimensional parameter space. Similarity is typically measured by Euclidean distance in the concatenated parameter space [S2].
    *   Update the winner $w$ and its topological neighbors in the grid to become more similar to $g_i$. This step enforces local homogeneity, ensuring that Gaussians with similar parameters are placed adjacent to each other in the grid [S1, S2].
    *   This process is repeated for all Gaussians until the grid converges to a stable ordering that preserves the topological structure of the input data [S2].
3.  **Smoothness Enforcement (Optional/Training Phase):** During the optimization phase, a smoothness loss can be applied to the grid to further enforce local continuity between neighboring cells, enhancing compressibility [S1].
4.  **Compression:** The resulting 2D grids for each attribute layer are compressed using standard image compression techniques (e.g., JPEG, WebP, or neural image codecs) [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def self_organizing_gaussian_grids(gaussians, grid_height, grid_width):
    """
    Organizes 3D Gaussians into a 2D grid using self-organizing map principles.
    
    Args:
        gaussians: List of N Gaussian objects, each with attributes 
                   (position, scale, rotation, opacity, sh_coeffs).
        grid_height: Height H of the output 2D grid.
        grid_width: Width W of the output 2D grid.
        
    Returns:
        grid: A 2D array of shape (H, W) containing references to Gaussians.
    """
    N = len(gaussians)
    # Initialize grid with random Gaussians from the input set
    grid = [[None for _ in range(grid_width)] for _ in range(grid_height)]
    for i in range(grid_height):
        for j in range(grid_width):
            idx = random.randint(0, N - 1)
            grid[i][j] = gaussians[idx]
            
    # Competitive learning phase
    # Iterate multiple epochs to converge the grid ordering
    for epoch in range(num_epochs):
        # Shuffle input Gaussians to ensure unbiased learning
        shuffled_gaussians = shuffle(gaussians)
        
        for g in shuffled_gaussians:
            # Find the winning neuron (grid cell)
            # Winner is the cell with the minimum distance to g in parameter space
            best_dist = infinity
            best_cell = None
            
            for r in range(grid_height):
                for c in range(grid_width):
                    current_g = grid[r][c]
                    # Calculate distance in high-dimensional parameter space
                    # This includes position, scale, rotation, opacity, and SH coefficients
                    dist = calculate_parameter_distance(g, current_g)
                    if dist < best_dist:
                        best_dist = dist
                        best_cell = (r, c)
            
            # Update the winner and its neighbors
            # This step pulls the winner and its topological neighbors 
            # closer to the input Gaussian g
            update_grid_neighbors(grid, best_cell, g, learning_rate)
            
    return grid

def calculate_parameter_distance(g1, g2):
    """
    Calculates Euclidean distance between two Gaussians in parameter space.
    """
    # Concatenate all parameters into a single vector
    vec1 = concatenate(g1.position, g1.scale, g1.rotation, g1.opacity, g1.sh_coeffs)
    vec2 = concatenate(g2.position, g2.scale, g2.rotation, g2.opacity, g2.sh_coeffs)
    return euclidean_distance(vec1, vec2)

def update_grid_neighbors(grid, winner_cell, target_gaussian, lr):
    """
    Updates the winner and its neighbors to be more similar to target_gaussian.
    """
    r, c = winner_cell
    # Define neighborhood radius (e.g., Gaussian decay)
    # Update winner
    grid[r][c] = target_gaussian # Or interpolate if using continuous values
    
    # Update neighbors based on distance from winner
    for dr in range(-neighborhood_radius, neighborhood_radius + 1):
        for dc in range(-neighborhood_radius, neighborhood_radius + 1):
            nr, nc = r + dr, c + dc
            if is_valid_cell(nr, nc):
                # Apply learning rate decay based on distance
                dist = sqrt(dr*dr + dc*dc)
                effective_lr = lr * exp(-dist*dist / (2 * sigma*sigma))
                # Update neighbor parameters towards target
                grid[nr][nc].update_towards(target_gaussian, effective_lr)
```

**Parameters**
*   **Grid Dimensions ($H \times W$):** Should be approximately equal to the number of Gaussians $N$. Typical values depend on the scene complexity [S1].
*   **Learning Rate:** Controls the step size during the competitive learning phase [S2].
*   **Neighborhood Radius:** Determines the extent of topological influence during updates [S2].
*   **Storage Reduction Factor:** Achieves a reduction of 17x to 42x in storage size for complex scenes [S1].

**Edge Cases and Pitfalls**
*   **Encoding Complexity:** The self-organization process requires a computationally expensive optimization to determine the element order, unlike fixed-ordering methods [S2].
*   **Precision:** The method relies on the assumption that perceptual redundancies exist in natural scenes, allowing permutations of Gaussian parameters to equivalently represent the scene [S1].

## Operating characteristics
*   **Time Complexity:** Encoding is computationally expensive due to the iterative competitive learning process [S2]. Rendering time is comparable to standard 3DGS as the uncompressed Gaussians use the same structure [S1].
*   **Memory Complexity:** Storage size is reduced by a factor of 17x to 42x for complex scenes [S1].
*   **Strengths:** Maintains fast rendering and high visual quality while drastically reducing storage requirements [S1]. Exploits perceptual redundancies and local homogeneity [S1].
*   **Limitations:** High encoding complexity [S2].
*   **When to Use:** When storage size is a critical constraint (e.g., resource-constrained devices) and encoding time is not a bottleneck [S1].
*   **When Not to Use:** When fast encoding is required or when the scene lacks the perceptual redundancies assumed by the method [S1].

## Background
Self-Organizing Gaussian Grids was introduced by Wieland Morgenstern, Florian Barthel, Anna Hilsmann, and Peter Eisert from Fraunhofer Heinrich Hertz Institute and Humboldt University of Berlin [S1]. The method was published in arXiv:2312.13299v2 on May 2, 2024 [S1]. It is based on the concept of Self-Organizing Maps (SOMs), an unsupervised learning model that projects high-dimensional data onto a lower-dimensional grid, preserving topological relationships through competitive learning [S2]. The method addresses the high storage requirements of 3D Gaussian Splatting, which hinders practical deployment on resource-constrained devices [S1].

## Sources
[S1] Compact 3D Scene Representation via Self-Organizing Gaussian Grids, https://doi.org/10.1007/978-3-031-73013-9_2, fetched 2026-10-26
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
