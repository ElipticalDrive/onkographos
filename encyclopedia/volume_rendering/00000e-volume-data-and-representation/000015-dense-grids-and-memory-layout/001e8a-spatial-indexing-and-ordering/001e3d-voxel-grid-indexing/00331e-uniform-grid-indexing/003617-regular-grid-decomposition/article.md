# Regular Grid Decomposition

Also called: Grid Decomposition

## Overview
Regular Grid Decomposition is a spatial partitioning strategy for neural volume rendering where the global scene domain is divided into a uniform 3D grid of independent neural network models, each responsible for a specific sub-region [S1, S2]. It solves the problem of representing large-scale or complex scenes by distributing the modeling capacity across multiple smaller networks rather than relying on a single global model [S1, S2]. Unlike [Compact Grid Method](9891-compact-grid-method.md), which adapts grid density to data sparsity, Regular Grid Decomposition uses a fixed, homogeneous grid structure, which simplifies the inference pipeline but requires significantly more regions to achieve comparable accuracy in non-homogeneous scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A global 3D coordinate $p = (p_x, p_y, p_z)$ typically normalized to the domain $[-1, 1]^3$; a pre-trained set of $I \times J \times K$ neural network models stored in GPU memory; grid dimensions $I, J, K$.
*   **Output:** The predicted radiance and density (or other neural field outputs) from the specific network corresponding to the sub-region containing $p$.

**Data Structures:**
*   **Model Array:** A 1D array (or list) of neural network models. The models are stored in C-order (row-major) based on their 3D grid indices $(i, j, k)$. The total number of models is $N = I \times J \times K$.
*   **Grid Dimensions:** Integers $I, J, K$ representing the number of subdivisions along the x, y, and z axes, respectively.

**Steps:**
1.  **Load Models:** All $I \times J \times K$ neural network models are loaded into GPU memory on the local workstation [S2].
2.  **Calculate Grid Indices:** For a given global query coordinate $p$, compute the integer grid indices $(i, j, k)$ using the spatial hashing function [S2]:
    $$ i = \left\lfloor \frac{p_x + 1}{2} I \right\rfloor $$
    $$ j = \left\lfloor \frac{p_y + 1}{2} J \right\rfloor $$
    $$ k = \left\lfloor \frac{p_z + 1}{2} K \right\rfloor $$
    where $\lfloor \cdot \rfloor$ denotes the integer floor operation [S2].
3.  **Clamp Indices:** To prevent out-of-bounds access, clamp the indices such that $i < I$, $j < J$, and $k < K$. This is necessary because if any coordinate component is exactly $1.0$, the floor operation yields the maximum index, which is out of bounds for 0-indexed arrays [S2].
4.  **Flatten Index:** Compute the 1D array index for the selected network using C-order flattening [S2]:
    $$ \text{index} = i + I \cdot j + I \cdot J \cdot k $$
5.  **Select and Scale:** Retrieve the network at `index`. Scale the global coordinate $p$ to the local domain of the selected network (typically $[-1, 1]^3$ relative to the sub-region) to ensure proper inference [S2].
6.  **Inference:** Perform forward pass through the selected network to obtain the output [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def regular_grid_decomposition_inference(p, models, I, J, K):
    """
    Perform inference using Regular Grid Decomposition.
    
    Args:
        p (tuple): Global 3D coordinate (px, py, pz) in [-1, 1]^3.
        models (list): List of neural network models, length I*J*K, stored in C-order.
        I (int): Grid dimension along x.
        J (int): Grid dimension along y.
        K (int): Grid dimension along z.
        
    Returns:
        tuple: Output from the selected neural network (e.g., density, color).
    """
    px, py, pz = p
    
    # Step 2: Calculate grid indices
    # Note: Using math.floor for integer floor operation
    i = math.floor((px + 1) / 2 * I)
    j = math.floor((py + 1) / 2 * J)
    k = math.floor((pz + 1) / 2 * K)
    
    # Step 3: Clamp indices to prevent out-of-bounds access
    # If coordinate is exactly 1.0, floor operation yields I, J, or K, which is out of bounds.
    i = min(i, I - 1)
    j = min(j, J - 1)
    k = min(k, K - 1)
    
    # Step 4: Flatten index (C-order)
    index = i + I * j + I * J * k
    
    # Step 5: Select network
    network = models[index]
    
    # Step 5: Scale coordinate to local domain
    # The local domain for the sub-region corresponding to index (i,j,k)
    # is typically [-1, 1]^3. The scaling maps the global p to this local space.
    # For a uniform grid, the local coordinate p_local can be derived as:
    # p_local_x = (px - (-1 + 2*i/I)) / (2/I) = (px + 1 - 2*i/I) * I / 2
    # However, the source [S2] states "global coordinate can be scaled to the networks local domain".
    # A common implementation for uniform grids maps the sub-region [-1+2*i/I, -1+2*(i+1)/I] to [-1, 1].
    # Let's implement this standard mapping.
    
    # Calculate bounds of the sub-region for i
    min_x = -1.0 + 2.0 * i / I
    max_x = -1.0 + 2.0 * (i + 1) / I
    # Avoid division by zero if I=0, though I>=1 is expected
    if max_x == min_x:
        local_px = 0.0
    else:
        local_px = 2.0 * (px - min_x) / (max_x - min_x) - 1.0
        
    min_y = -1.0 + 2.0 * j / J
    max_y = -1.0 + 2.0 * (j + 1) / J
    if max_y == min_y:
        local_py = 0.0
    else:
        local_py = 2.0 * (py - min_y) / (max_y - min_y) - 1.0
        
    min_z = -1.0 + 2.0 * k / K
    max_z = -1.0 + 2.0 * (k + 1) / K
    if max_z == min_z:
        local_pz = 0.0
    else:
        local_pz = 2.0 * (pz - min_z) / (max_z - min_z) - 1.0
        
    local_p = (local_px, local_py, local_pz)
    
    # Step 6: Inference
    output = network(local_p)
    
    return output
```

**Parameters:**
*   $I, J, K$: Grid dimensions. Typical values are not specified in the sources, but they determine the granularity of the decomposition [S1, S2].
*   Domain: $[-1, 1]^3$ [S2].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** If any component of $p$ is exactly $1.0$, the floor operation results in an index equal to the grid dimension (e.g., $i=I$), which is out of bounds. Clamping is required [S2].
*   **Memory Incoherence:** Using a regular grid for non-homogeneous scenes leads to significant memory access incoherence for network weights, lowering computational performance [S1].
*   **Accuracy vs. Complexity:** A regular grid requires significantly more regions than adaptive methods to achieve the same accuracy in scenes with non-homogeneous structure [S1].

## Subtopics

### Domain Decomposition Inference with Spatial Hashing
A specific implementation of Regular Grid Decomposition where a spatial hashing function maps global 3D query coordinates to the correct neural network model in a domain decomposition setup [S2]. It avoids looping over all networks by directly computing the array index from the global coordinate [S2]. It is chosen when efficient inference on a local workstation is required after training on a remote server [S2]. The implementation described in [S2] loops over all networks for evaluation, which is slower than custom CUDA code for parallelized network evaluation [S2].

## Operating characteristics
*   **Time Complexity:** Inference time is $O(1)$ for index calculation and $O(1)$ for network lookup, assuming constant-time network evaluation. However, naive implementations may loop over all networks, resulting in $O(N)$ time where $N$ is the number of networks [S2].
*   **Memory Complexity:** $O(N \cdot M)$, where $N = I \times J \times K$ is the number of networks and $M$ is the memory footprint of each network [S1, S2].
*   **Performance:** The method can deliver higher quality images and faster rendering than single global models if the decomposition is compatible with the Painter’s algorithm [S1]. However, performance is degraded by memory access incoherence in non-homogeneous scenes [S1].
*   **Strengths:** Eliminates the requirement to train a single large decomposition; allows for parallel inference across networks; GPU-friendly if implemented with custom CUDA code [S1, S2].
*   **Limitations:** Requires significantly more regions for the same accuracy in non-homogeneous scenes; naive implementations are slow due to looping over all networks [S1, S2].
*   **When to Use:** When the scene can be effectively partitioned into uniform sub-regions and high-quality rendering is prioritized over memory efficiency in sparse areas [S1].
*   **When Not to Use:** When the scene has highly non-homogeneous structure, as adaptive methods like [Compact Grid Method](9891-compact-grid-method.md) are more efficient [S1].

## Background
Regular Grid Decomposition is a technique used in neural volume rendering, particularly in the context of Decomposed Radiance Fields (DeRF) [S1]. It was developed to address the limitations of single global neural networks in representing large-scale scenes [S1, S2]. The method involves dividing the scene into a regular grid of neural networks, each trained on a specific sub-region [S1, S2]. The approach is compatible with the Painter’s algorithm, allowing for efficient rendering [S1]. The technique has been implemented in Python/PyTorch for fair comparison with other models [S2].

## Sources
[S1] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
[S2] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
