# TetraHashGridEncode

Also called: Tetrahedral Hash Grid Encoding

## Overview
TetraHashGridEncode is a differentiable hash grid encoding method for 3D coordinates that utilizes tetrahedral decomposition for interpolation, implemented via CUDA extensions for forward and backward passes [S1, unofficial]. It builds on [HashGrid Encoding](3866-hashgrid-encoding.md) by replacing standard trilinear interpolation within hash-grid cells with barycentric interpolation over a fixed tetrahedral decomposition, allowing for efficient training of neural radiance fields (NeRF) on consumer GPUs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A tensor of input coordinates `input_coords` of shape $(N, 3)$, where $N$ is the number of points.
- **Output:** A tensor of encoded features `output_features` of shape $(N, L \times F)$, where $L$ is the number of resolution levels and $F$ is the number of features per level per hash entry.

**Data Structures:**
The method relies on several fixed buffers and a learnable feature table. All buffers are stored on the device (GPU).

1.  **Feature Table (`feature_table`):**
    -   Shape: $(\text{total\_features})$, where $\text{total\_features} = L \times \text{table\_size} \times F$.
    -   Type: `torch.float32`.
    -   Initialization: Uniformly in $[-10^{-4}, 10^{-4}]$ [S1, unofficial].
    -   This is the only learnable parameter (`nn.Parameter`) [S1, unofficial].

2.  **Cube Vertices (`cube_vertices`):**
    -   Shape: $(8, 3)$.
    -   Type: `torch.int32`.
    -   Values:
        $$
        \begin{bmatrix}
        0 & 0 & 0 \\
        1 & 0 & 0 \\
        0 & 1 & 0 \\
        1 & 1 & 0 \\
        0 & 0 & 1 \\
        1 & 0 & 1 \\
        0 & 1 & 1 \\
        1 & 1 & 1
        \end{bmatrix}
        $$
    -   Represents the 8 corners of a unit cube [S1, unofficial].

3.  **Tetrahedra Index (`tetrahedra_index`):**
    -   Shape: $(6, 4)$.
    -   Type: `torch.int32`.
    -   Values:
        $$
        \begin{bmatrix}
        0 & 1 & 5 & 7 \\
        0 & 1 & 3 & 7 \\
        0 & 2 & 3 & 7 \\
        0 & 2 & 6 & 7 \\
        0 & 4 & 6 & 7 \\
        0 & 4 & 5 & 7
        \end{bmatrix}
        $$
    -   Defines the 6 tetrahedra that decompose the unit cube. Each row contains 4 indices into `cube_vertices` [S1, unofficial].

4.  **Prime Factors (`prime_factors`):**
    -   Shape: $(7)$.
    -   Type: `torch.int64`.
    -   Values: $[1, 2654435761, 805459861, 3674653429, 2097192037, 1434869437, 2165219737]$ [S1, unofficial].
    -   Used in the spatial hashing function [S1, unofficial].

5.  **Resolutions (`resolutions_tensor`):**
    -   Shape: $(L)$.
    -   Type: `torch.int32`.
    -   Contains the resolution values for each of the $L$ levels [S1, unofficial].

6.  **Scalar Buffers:**
    -   `F_tensor`: Shape $(1)$, value $F$ (`num_features_per_level`) [S1, unofficial].
    -   `table_size_tensor`: Shape $(1)$, value $2^{\text{log2\_hashmap\_size}}$ [S1, unofficial].

**Parameters:**
-   `input_dim`: Dimensionality of input coordinates. Default: 3 [S1, unofficial].
-   `num_levels` ($L$): Number of hash grid levels. Default: 4 [S1, unofficial].
-   `num_features_per_level` ($F$): Number of features per level per hash entry. Default: 2 [S1, unofficial].
-   `log2_hashmap_size`: Log base 2 of the hash table size. Default: 22. Table size is $2^{22}$ [S1, unofficial].
-   `resolutions`: List of resolution values for each level. Required. Raises `AssertionError` if None [S1, unofficial].

**Algorithm Steps:**

The encoding is performed in parallel for $N$ points across $L$ levels. The kernel processes indices $idx = 0 \dots N \times L - 1$.

1.  **Index Decomposition:**
    For each thread index $idx$:
    -   Point index: $p = \lfloor idx / L \rfloor$.
    -   Level index: $level = idx \pmod L$.
    -   Retrieve coordinates: $x = \text{input\_coords}[p \times 3 + 0]$, $y = \text{input\_coords}[p \times 3 + 1]$, $z = \text{input\_coords}[p \times 3 + 2]$.
    -   Retrieve resolution: $res = \text{resolutions}[level]$.

2.  **Grid Position and Fractional Part:**
    Compute the integer grid coordinates $(g_x, g_y, g_z)$ and fractional coordinates $(f_x, f_y, f_z)$ within the cell:
    $$
    g_x = \lfloor x \times res \rfloor, \quad f_x = (x \times res) - g_x
    $$
    $$
    g_y = \lfloor y \times res \rfloor, \quad f_y = (y \times res) - g_y
    $$
    $$
    g_z = \lfloor z \times res \rfloor, \quad f_z = (z \times res) - g_z
    $$
    Note: The source code uses `pos_fract_resm1`, which implies handling modulo arithmetic for wrapping or boundary conditions, but the core logic relies on the fractional parts $f_x, f_y, f_z \in [0, 1)$ [S1, unofficial].

3.  **Tetrahedron Selection:**
    Determine which of the 6 tetrahedra contains the point $(f_x, f_y, f_z)$ using the `select_tet_chain` function:
    $$
    \text{tet} = \begin{cases}
    1 & \text{if } x \ge y \text{ and } y \ge z \\
    0 & \text{if } x \ge y \text{ and } x \ge z \text{ and } y < z \\
    5 & \text{if } x \ge y \text{ and } y < z \text{ and } x < z \\
    2 & \text{if } x < y \text{ and } x \ge z \\
    3 & \text{if } x < y \text{ and } x < z \text{ and } y \ge z \\
    4 & \text{if } x < y \text{ and } x < z \text{ and } y < z
    \end{cases}
    $$
    (Note: The source code variable names in `select_tet_chain` use $x,y,z$ but operate on $f_x, f_y, f_z$ passed as arguments) [S1, unofficial].

4.  **Vertex Coordinates:**
    Retrieve the 4 vertices of the selected tetrahedron from `tetrahedra_index[tet]`. Let these indices be $v_0, v_1, v_2, v_3$.
    Compute the absolute integer grid coordinates for each vertex by adding the cube vertex offsets to the cell origin $(g_x, g_y, g_z)$:
    $$
    v_{k,x} = g_x + \text{cube\_vertices}[v_k \times 3 + 0]
    $$
    $$
    v_{k,y} = g_y + \text{cube\_vertices}[v_k \times 3 + 1]
    $$
    $$
    v_{k,z} = g_z + \text{cube\_vertices}[v_k \times 3 + 2]
    $$
    for $k \in \{0, 1, 2, 3\}$ [S1, unofficial].

5.  **Hash Indexing:**
    Compute the hash index for each vertex using `grid_index_or_hash`:
    $$
    h_k = \text{grid\_index\_or\_hash}(v_{k,x}, v_{k,y}, v_{k,z}, \text{table\_size}, res, \text{prime\_factors})
    $$
    The hashing function is defined as:
    $$
    h = (v_{k,x} \times \text{prime\_factors}[1]) \oplus (v_{k,y} \times \text{prime\_factors}[2]) \oplus (v_{k,z} \times \text{prime\_factors}[3])
    $$
    $$
    \text{index} = \begin{cases}
    h \pmod{\text{table\_size}} & \text{if table\_size is not a power of 2} \\
    h \pmod{\text{table\_size}} & \text{if table\_size is a power of 2 (optimized via bitwise AND)}
    \end{cases}
    $$
    Specifically, if `table_size` is a power of 2, it uses `h & (table_size - 1)` [S1, unofficial].

6.  **Barycentric Weights:**
    Compute the barycentric weights $(w_0, w_1, w_2, w_3)$ for the selected tetrahedron based on $(f_x, f_y, f_z)$:
    $$
    (w_0, w_1, w_2, w_3) = \begin{cases}
    (1-f_x, f_x-f_z, f_z-f_y, f_y) & \text{if tet} = 0 \\
    (1-f_x, f_x-f_y, f_y-f_z, f_z) & \text{if tet} = 1 \\
    (1-f_y, f_y-f_x, f_x-f_z, f_z) & \text{if tet} = 2 \\
    (1-f_y, f_y-f_z, f_z-f_x, f_x) & \text{if tet} = 3 \\
    (1-f_z, f_z-f_y, f_y-f_x, f_x) & \text{if tet} = 4 \\
    (1-f_z, f_z-f_x, f_x-f_y, f_y) & \text{if tet} = 5
    \end{cases}
    $$
    [S1, unofficial].

7.  **Feature Interpolation:**
    For each feature channel $f \in \{0, \dots, F-1\}$:
    $$
    \text{output\_features}[p \times L \times F + level \times F + f] = \sum_{k=0}^{3} w_k \times \text{feature\_table}[level \times \text{table\_size} \times F + h_k \times F + f]
    $$
    [S1, unofficial].

**Backward Pass:**
The backward pass computes gradients with respect to the input coordinates (`grad_input_coords`) and the feature table (`grad_feature_table`).

1.  **Gradient Accumulation for Feature Table:**
    For each point and level, retrieve the gradient of the output `grad_output`.
    For each feature $f$:
    $$
    \text{grad\_feature\_table}[level \times \text{table\_size} \times F + h_k \times F + f] \mathrel{+}= w_k \times \text{grad\_output}[p \times L \times F + level \times F + f]
    $$
    This is performed using `atomicAdd` to handle concurrent writes from multiple threads [S1, unofficial].

2.  **Gradient for Input Coordinates:**
    The gradient with respect to the input coordinates is computed using the chain rule through the barycentric weights. The source code calls `tetra_hash_grid_Jx_times_vec_cuda` which implements the Jacobian-vector product. The Jacobian matrix $J$ for a specific tetrahedron is derived from the derivatives of the barycentric weights with respect to $f_x, f_y, f_z$, scaled by the resolution $res$.
    
    The gradient matrix $G$ for each vertex of the tetrahedron is precomputed in `bary_grad_mat`. For example, for tet 0:
    $$
    \frac{\partial w_0}{\partial f_x} = -1, \quad \frac{\partial w_1}{\partial f_x} = 1, \quad \frac{\partial w_1}{\partial f_z} = -1, \quad \frac{\partial w_2}{\partial f_y} = -1, \quad \frac{\partial w_2}{\partial f_z} = 1, \quad \frac{\partial w_3}{\partial f_y} = 1
    $$
    All other partials are 0. The gradient flows back through the hash indices and the feature table gradients. The exact implementation is in the CUDA kernel `tetra_hash_grid_encode_backward_optimized_kernel` [S1, unofficial].

3.  **Second-Order Backward (Eikonal Loss):**
    For second-order gradients required by Eikonal loss, the method calls `tetra_hash_grid_second_order_dOmega_cuda`. This computes the Hessian-vector product or similar second-order derivatives. If the kernel is not available, it raises a `RuntimeError` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def tetra_hash_grid_encode_forward(input_coords, feature_table, resolutions, table_size, prime_factors, cube_vertices, tetrahedra_index, num_levels, num_features_per_level):
    """
    Forward pass for TetraHashGridEncode.
    Note: This is a simplified CPU simulation of the logic. The actual implementation uses CUDA kernels.
    """
    N = input_coords.shape[0]
    L = num_levels
    F = num_features_per_level
    
    output_features = torch.zeros(N, L * F, dtype=torch.float32)
    
    for p in range(N):
        x, y, z = input_coords[p]
        
        for level in range(L):
            res = resolutions[level]
            
            # Grid position and fractional part
            gx = int(math.floor(x * res))
            fx = (x * res) - gx
            gy = int(math.floor(y * res))
            fy = (y * res) - gy
            gz = int(math.floor(z * res))
            fz = (z * res) - gz
            
            # Tetrahedron selection
            if fx >= fy:
                if fy >= fz:
                    tet = 1
                elif fx >= fz:
                    tet = 0
                else:
                    tet = 5
            else:
                if fx >= fz:
                    tet = 2
                elif fy >= fz:
                    tet = 3
                else:
                    tet = 4
            
            # Vertex indices in tetrahedra_index
            v_indices = tetrahedra_index[tet]
            
            # Compute absolute grid coordinates for vertices
            vertex_coords = []
            for v_idx in v_indices:
                vx = gx + cube_vertices[v_idx][0]
                vy = gy + cube_vertices[v_idx][1]
                vz = gz + cube_vertices[v_idx][2]
                vertex_coords.append((vx, vy, vz))
            
            # Hash indices
            hash_indices = []
            for vx, vy, vz in vertex_coords:
                h = (vx * prime_factors[1]) ^ (vy * prime_factors[2]) ^ (vz * prime_factors[3])
                if (table_size & (table_size - 1)) == 0:
                    idx = h & (table_size - 1)
                else:
                    idx = h % table_size
                hash_indices.append(idx)
            
            # Barycentric weights
            if tet == 0:
                w0, w1, w2, w3 = 1-fx, fx-fz, fz-fy, fy
            elif tet == 1:
                w0, w1, w2, w3 = 1-fx, fx-fy, fy-fz, fz
            elif tet == 2:
                w0, w1, w2, w3 = 1-fy, fy-fx, fx-fz, fz
            elif tet == 3:
                w0, w1, w2, w3 = 1-fy, fy-fz, fz-fx, fx
            elif tet == 4:
                w0, w1, w2, w3 = 1-fz, fz-fy, fy-fx, fx
            else: # tet == 5
                w0, w1, w2, w3 = 1-fz, fz-fx, fx-fy, fy
            
            # Interpolate features
            level_offset = level * table_size * F
            out_offset = p * L * F + level * F
            
            for f in range(F):
                acc = 0.0
                for k in range(4):
                    h_idx = hash_indices[k]
                    acc += w_k * feature_table[level_offset + h_idx * F + f]
                output_features[p, out_offset + f] = acc
                
    return output_features
```

## Subtopics

### Multi-resolution Hash Encoding
A sibling method that uses multiple hash grids at different resolutions. TetraHashGridEncode also supports multi-resolution by iterating over levels, but specifically applies tetrahedral interpolation within each level's hash grid [S1, unofficial]. Choose TetraHashGridEncode when tetrahedral interpolation is preferred for its specific gradient properties or compatibility with SDF-based losses.

### Dense Grid Encoding
A sibling method that uses dense grids instead of hash grids. TetraHashGridEncode uses spatial hashing to reduce memory usage compared to dense grids, making it suitable for larger scenes or higher resolutions on consumer GPUs [S1, unofficial]. Choose Dense Grid Encoding when deterministic memory access patterns are critical and memory is not a constraint.

### ACORN
A sibling method that uses adaptive octree-based representations. TetraHashGridEncode uses a fixed hash grid structure. Choose ACORN when adaptive resolution is needed to capture fine details efficiently without allocating memory for empty space [S1, unofficial].

### RHINO
A sibling method that uses ray-based hashing. TetraHashGridEncode uses point-based hashing. Choose RHINO when the rendering pipeline is ray-driven and ray-specific optimizations are beneficial [S1, unofficial].

### HollowNeRF Saliency Grid Weighting
A sibling method that focuses on saliency-based weighting for grid updates. TetraHashGridEncode is a general-purpose encoding method. Choose HollowNeRF Saliency Grid Weighting when the goal is to prioritize training on salient regions of the scene [S1, unofficial].

### Progressive Training
A sibling method that involves training at increasing resolutions. TetraHashGridEncode supports multi-resolution encoding which can be used in conjunction with progressive training strategies. Choose Progressive Training when starting with low-resolution details and gradually adding complexity [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The forward pass is $O(N \times L)$, where $N$ is the number of points and $L$ is the number of levels. Each point requires constant-time operations (hashing, interpolation) per level. The backward pass is also $O(N \times L)$ but involves atomic operations for gradient accumulation, which may introduce contention [S1, unofficial].
-   **Memory Complexity:** The memory usage is dominated by the feature table, which is $O(L \times \text{table\_size} \times F)$. With default parameters ($L=4, \text{table\_size}=2^{22}, F=2$), this is approximately $4 \times 4 \times 10^6 \times 2 \times 4 \text{ bytes} \approx 128 \text{ MB}$ [S1, unofficial].
-   **Performance:** The method is implemented in CUDA for high performance on GPUs. It is designed for efficient training of NeRFs on consumer GPUs [S1, unofficial].
-   **Strengths:** Differentiable, efficient memory usage via hashing, supports multi-resolution, compatible with SDF-based losses (e.g., Eikonal loss) via second-order backward passes [S1, unofficial].
-   **Limitations:** Requires CUDA extensions, which may not be available on all systems. Hash collisions can occur, though the large table size mitigates this. The fixed tetrahedral decomposition may not adapt to scene complexity as well as octree-based methods [S1, unofficial].
-   **When to use:** When training NeRFs or SDFs on consumer GPUs with limited memory, and when tetrahedral interpolation is desired for its gradient properties.
-   **When not to use:** When CPU-only inference is required, or when adaptive resolution is critical for the scene structure.

## Background
TetraHashGridEncode was introduced as part of the TetraSDF project by NAVER Corp. It is designed to provide a differentiable encoding method that combines the efficiency of hash grids with the geometric properties of tetrahedral decomposition. The method is implemented via CUDA extensions for both forward and backward passes, including support for second-order gradients required by Eikonal losses [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
