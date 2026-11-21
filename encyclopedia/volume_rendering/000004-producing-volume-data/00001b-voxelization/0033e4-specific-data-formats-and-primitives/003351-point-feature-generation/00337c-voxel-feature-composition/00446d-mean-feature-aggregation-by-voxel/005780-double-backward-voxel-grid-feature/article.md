# Double Backward Voxel Grid Feature

Also called: Custom CUDA Double Backward, Voxel Grid Gradient of Gradient

## Overview
The Double Backward Voxel Grid Feature is a custom CUDA implementation that computes second-order gradients (gradients of gradients) for voxel grid features, specifically to support Eikonal regularization in neural rendering pipelines [S1, unofficial]. It solves the problem of excessive memory overhead and computational slowness associated with naive automatic differentiation frameworks, which construct long and complex computation graphs that cannot be optimized automatically [S1, unofficial]. Unlike its sibling [Mean Feature Aggregation by Voxel](11242-mean-feature-aggregation-by-voxel.md), which handles first-order mean aggregation, this method explicitly derives and implements the backward pass for the gradient of the voxel feature lookup, enabling efficient second-order optimization without storing intermediate gradients for the entire graph.

## How it works
**Inputs:**
- Query points $\mathbf{x}_b = (x_b, y_b, z_b)$ for batch index $b$.
- Voxel grid feature tensor $\mathbf{F}_V$ with grid size $G$ (dimensions $G \times G \times G$) and bounds $m$ (min) and $M$ (max).
- First-order gradient of the output feature w.r.t. the interpolated value, denoted $g^{(1)}\bar{\mathbf{f}}(\bar{\mathbf{x}}_b)$.
- Second-order gradient of the loss w.r.t. the first-order gradient of the query point, denoted $g^{(2)}g^{(1)}\mathbf{x}_b = (g^{(2)}g^{(1)}x_b, g^{(2)}g^{(1)}y_b, g^{(2)}g^{(1)}z_b)$.

**Outputs:**
- Second-order gradient of the loss w.r.t. the interpolated feature value, $g^{(2)}g^{(1)}\bar{\mathbf{f}}(\bar{\mathbf{x}}_b)$.
- Accumulated second-order gradient w.r.t. the voxel grid features, $g^{(2)}\mathbf{f}_{i,j,k}$.

**Data Structures:**
- The voxel grid feature $\mathbf{F}_V$ is stored as a dense tensor of shape $(G, G, G, D)$, where $D$ is the feature dimension.
- Gradients are stored in tensors of the same shape as their corresponding inputs.
- Atomic add operations are used for accumulating gradients into the voxel grid feature tensor to handle concurrent writes from multiple threads [S1, unofficial].

**Steps:**

1.  **Coordinate Transformation:**
    Transform the continuous query point $\mathbf{x}_b$ to the discrete coordinate system $\bar{\mathbf{x}}_b$. For each component (e.g., $x$):
    $$ \bar{x}_b = \frac{x_b - m}{M - m} G = s_x (x_b - m) $$
    where $s_x = \frac{G}{M - m}$ is the scaling factor. Similarly compute $s_y$ and $s_z$ (assuming isotropic grid or separate bounds per axis) and $\bar{y}_b, \bar{z}_b$ [S1, unofficial].

2.  **Forward Interpolation (Context):**
    The interpolated feature $\bar{\mathbf{f}}(\bar{\mathbf{x}}_b)$ is computed via trilinear interpolation (or similar) using weights $w_{i,j,k}(\bar{\mathbf{x}}_b)$ and features $\mathbf{f}_{i,j,k}$ from the 8 surrounding voxels:
    $$ \bar{\mathbf{f}}(\bar{\mathbf{x}}_b) = \sum_{i,j,k} w_{i,j,k}(\bar{\mathbf{x}}_b) \mathbf{f}_{i,j,k} $$
    The derivative of the weight $\frac{\partial w_{i,j,k}}{\partial \bar{\mathbf{x}}_b}$ depends on the specific interpolation scheme [S1, unofficial].

3.  **Compute First-Order Gradients (Required for Second-Order):**
    Before computing the double backward, the first-order gradients must be available.
    
    *Gradient w.r.t. query point $\mathbf{x}_b$:*
    For each dimension $d$ of the feature vector:
    $$ g^{(1)}x_b^d = \sum_{i,j,k} s_x \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{x}_b} f_{i,j,k}^d $$
    $$ g^{(1)}y_b^d = \sum_{i,j,k} s_y \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{y}_b} f_{i,j,k}^d $$
    $$ g^{(1)}z_b^d = \sum_{i,j,k} s_z \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{z}_b} f_{i,j,k}^d $$
    These are scaled by the upstream gradient $g^{(1)}\bar{\mathbf{f}}^d$ in the actual backprop flow, but the source defines the structural form of the gradient term itself [S1, unofficial]. Note: If using Straight Through Estimator (STE), these derivatives are zero [S1, unofficial].

    *Gradient w.r.t. voxel feature $\mathbf{f}_{i,j,k}$:*
    $$ g^{(1)}\mathbf{f}_{i,j,k} += g^{(1)}\bar{\mathbf{f}}(\bar{\mathbf{x}}_b) w_{i,j,k}(\bar{\mathbf{x}}_b) $$
    This accumulation is performed via atomic add [S1, unofficial].

4.  **Compute Second-Order Gradient w.r.t. Interpolated Feature:**
    Using the second-order gradient of the loss w.r.t. the query point gradients ($g^{(2)}g^{(1)}x_b$, etc.) and the first-order gradient of the feature ($g^{(1)}\bar{\mathbf{f}}$), compute the gradient flowing back to the output of the interpolation function:
    $$ g^{(2)}g^{(1)}\bar{f}^d(\bar{\mathbf{x}}_b) = g^{(2)}g^{(1)}x_b \sum_{i,j,k} s_x \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{x}_b} f_{i,j,k}^d + g^{(2)}g^{(1)}y_b \sum_{i,j,k} s_y \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{y}_b} f_{i,j,k}^d + g^{(2)}g^{(1)}z_b \sum_{i,j,k} s_z \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{z}_b} f_{i,j,k}^d $$
    Here, $g^{(2)}g^{(1)}x_b$ represents $\frac{\partial^2 L}{\partial (g^{(1)}x_b) \partial (g^{(1)}x_b)}$ or the relevant cross-term depending on the Eikonal loss structure. The source presents this as the gradient of Eq. S19 w.r.t. the grad output [S1, unofficial].

5.  **Compute Second-Order Gradient w.r.t. Voxel Grid Feature:**
    Accumulate the gradient into the voxel grid feature tensor $\mathbf{F}_V$. For each voxel $(i,j,k)$ and feature dimension $d$:
    $$ g^{(2)}f_{i,j,k}^d(\bar{\mathbf{x}}_b) += g^{(2)}g^{(1)}x_b \cdot g^{(1)}\bar{f}^d(\bar{\mathbf{x}}_b) \cdot s_x \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{x}_b} + g^{(2)}g^{(1)}y_b \cdot g^{(1)}\bar{f}^d(\bar{\mathbf{x}}_b) \cdot s_y \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{y}_b} + g^{(2)}g^{(1)}z_b \cdot g^{(1)}\bar{f}^d(\bar{\mathbf{x}}_b) \cdot s_z \frac{\partial w_{i,j,k}(\bar{\mathbf{x}}_b)}{\partial \bar{z}_b} $$
    This step uses atomic add operations to handle concurrent updates from different query points [S1, unofficial].

**Pitfalls and Edge Cases:**
- **STE Limitation:** If the Straight Through Estimator is used for the forward pass, the derivative of the weights $\frac{\partial w}{\partial \bar{\mathbf{x}}}$ is zero. Consequently, the gradients w.r.t. the query point and the second-order gradients w.r.t. the voxel features become zero, ignoring the normals of the voxel grid feature [S1, unofficial].
- **Lookup Result:** The feature values $\mathbf{f}_{i,j,k}$ are results of lookups. In the CUDA implementation, gradients are not backpropagated w.r.t. the lookup result itself in the second-order step because it is treated as a constant source for the interpolation derivative calculation [S1, unofficial].
- **Atomic Contention:** The use of atomic adds for gradient accumulation into the voxel grid can lead to contention and reduced performance on high-resolution grids or large batch sizes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def double_backward_voxel_grid(query_points, voxel_features, bounds_min, bounds_max, grid_size, 
                               grad_output_first_order, grad_query_point_second_order):
    """
    query_points: (B, 3) tensor, [x, y, z]
    voxel_features: (G, G, G, D) tensor
    bounds_min: scalar or (3,) tensor
    bounds_max: scalar or (3,) tensor
    grid_size: int G
    grad_output_first_order: (B, D) tensor, g^(1) f_bar
    grad_query_point_second_order: (B, 3) tensor, g^(2)g^(1) x_b
    
    Returns:
    grad_voxel_features: (G, G, G, D) tensor, accumulated g^(2) f
    """
    B, D = query_points.shape[0], voxel_features.shape[3]
    G = grid_size
    
    # Initialize gradient accumulator for voxel features
    grad_voxel_features = torch.zeros_like(voxel_features)
    
    # Scaling factors
    # Assuming isotropic bounds for simplicity, or per-axis
    scale_x = G / (bounds_max[0] - bounds_min[0])
    scale_y = G / (bounds_max[1] - bounds_min[1])
    scale_z = G / (bounds_max[2] - bounds_min[2])
    
    # Parallelize over B and D dimensions in CUDA
    # This pseudocode represents the logic executed per thread/block
    
    for b in range(B):
        x_b, y_b, z_b = query_points[b]
        g2g1_x, g2g1_y, g2g1_z = grad_query_point_second_order[b]
        g1_f_bar = grad_output_first_order[b] # Shape (D,)
        
        # 1. Transform to discrete coordinates
        x_bar = scale_x * (x_b - bounds_min[0])
        y_bar = scale_y * (y_b - bounds_min[1])
        z_bar = scale_z * (z_b - bounds_min[2])
        
        # 2. Identify surrounding voxels (i, j, k) and compute weights w and dw/dx
        # This part depends on the specific interpolation implementation (e.g., trilinear)
        # Let's assume a helper function that returns weights and derivatives for the 8 neighbors
        neighbors = get_trilinear_neighbors(x_bar, y_bar, z_bar, G)
        
        # 3. Compute sums for the double backward formula
        # We need to sum over the 8 neighbors (i,j,k)
        
        sum_term_x = 0.0
        sum_term_y = 0.0
        sum_term_z = 0.0
        
        # Accumulators for voxel feature gradients
        # Using a temporary buffer for atomic add safety if needed, 
        # but CUDA atomic add handles this directly on global memory
        
        for neighbor in neighbors:
            i, j, k = neighbor.coords
            w = neighbor.weight
            dw_dx = neighbor.dweight_dx
            dw_dy = neighbor.dweight_dy
            dw_dz = neighbor.dweight_dz
            f_val = voxel_features[i, j, k] # Shape (D,)
            
            # Contribution to the gradient w.r.t interpolated feature (g^(2)g^(1) f_bar)
            # Note: The source formula sums f_val * dw. 
            # This term is used to compute the gradient flowing back to the output of the interpolation
            # However, the output of this function is primarily the gradient w.r.t voxel features.
            # The source computes g^(2)g^(1) f_bar as an intermediate or output.
            
            # Term for g^(2)g^(1) f_bar^d
            # sum_{i,j,k} s_x * dw/dx * f^d
            sum_term_x += scale_x * dw_dx * f_val
            sum_term_y += scale_y * dw_dy * f_val
            sum_term_z += scale_z * dw_dz * f_val
            
            # Term for g^(2) f_{i,j,k}^d
            # g^(2)g^(1) x_b * g^(1) f_bar^d * s_x * dw/dx
            # This is added to the voxel feature gradient
            
            # Vectorized operation for all D dimensions
            # grad_voxel_features[i, j, k] += (g2g1_x * g1_f_bar * scale_x * dw_dx + ...)
            
            term_x = g2g1_x * g1_f_bar * scale_x * dw_dx
            term_y = g2g1_y * g1_f_bar * scale_y * dw_dy
            term_z = g2g1_z * g1_f_bar * scale_z * dw_dz
            
            grad_voxel_features[i, j, k] += term_x + term_y + term_z

    # Compute the gradient w.r.t the interpolated feature output if needed
    # g^(2)g^(1) f_bar^d = g^(2)g^(1) x_b * sum_term_x^d + ...
    # This is typically used by the upstream autograd engine or for Eikonal loss calculation
    
    return grad_voxel_features
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of query points $B$ and the number of interpolated neighbors (constant, e.g., 8 for trilinear). The complexity is $O(B \cdot D)$.
- **Memory Complexity:** $O(G^3 \cdot D)$ for storing the voxel grid and its gradients. The double backward implementation avoids storing the large computation graph of naive auto-diff, reducing memory overhead significantly [S1, unofficial].
- **Performance:** The custom CUDA kernel parallelizes over the batch dimension $B$ and feature dimension $D$. Performance is limited by memory bandwidth due to atomic add operations on the voxel grid feature tensor, especially at high resolutions [S1, unofficial].
- **Strengths:** Efficient computation of second-order gradients required for Eikonal regularization; avoids memory explosion from automatic differentiation frameworks [S1, unofficial].
- **Limitations:** Requires custom CUDA implementation; atomic contention can bottleneck performance; not applicable if Straight Through Estimator is used (gradients vanish) [S1, unofficial].

## Background
The Double Backward Voxel Grid Feature was introduced in the context of Neural Direct and Joint Inverse Rendering (NDJIR) [S1, unofficial]. It was developed to address the computational inefficiencies of using standard automatic differentiation libraries (like PyTorch or TensorFlow) for voxel grid features in neural rendering. Naive composition of functions creates a long computation graph that is difficult to optimize, leading to slow execution and high memory usage [S1, unofficial]. By implementing specific CUDA kernels for double-backward propagation, the authors achieved a more efficient and scalable solution for optimizing voxel-based representations with second-order constraints like the Eikonal equation [S1, unofficial].

## Sources
[S1] NDJIR: Neural Direct and Joint Inverse Rendering for Geometry, Lights, and Materials of Real Object, https://doi.org/10.48550/arxiv.2302.00675, fetched 2026-10-26 (unofficial)
