# Screen-Space Gradient Densification Criterion

Also called: Screen-Space Gradient Densification

## Overview
This method provides a criterion for 3D primitive densification based on the mean magnitude of the gradient of the photometric loss with respect to the projected 2D screen-space center of the primitive, accumulated over all training views [S1, unofficial]. It differs from alternatives like 2DGS [Huang et al. 2024] by using an analytical T-matrix backward pass to avoid conflating depth and lateral motion, which prevents gradient cancellation and ensures the signal faithfully captures the rendering residual [S1, unofficial].

## How it works
**Inputs:**
- Photometric loss $\mathcal{L}_j$ for each training view $j$.
- Projection matrix $T$ for each view.
- Number of views $N$.

**Outputs:**
- Mean gradient score $\bar{g}_i$ for each primitive $i$.

**Data Structures:**
- $T$: A $4 \times 4$ projection matrix. Elements are denoted $T_{m,n}$ where $m$ is the row index and $n$ is the column index.
- $\boldsymbol{\mu}_i$: The projected 2D screen-space center of primitive $i$.
- $\bar{g}_i$: Scalar score for primitive $i$.

**Steps:**
1. For each primitive $i$ and each view $j$ where the primitive contributes to the rendered image:
    a. Identify the specific row index $z$ used for projection in the T-matrix backward pass.
    b. Compute the screen-space center coordinates $(u, v)$ using the T-matrix components:
       $$ u = \frac{T_{1,z}}{T_{4,z}}, \quad v = \frac{T_{2,z}}{T_{4,z}} $$
       [S1, unofficial]
    c. Compute the per-pixel contributions to the screen-space positional gradient $(g_x, g_y)$ using the chain rule through the T-matrix gradients:
       $$ g_x = \frac{\partial \mathcal{L}}{\partial T_{1,z}} \cdot T_{4,z} $$
       $$ g_y = \frac{\partial \mathcal{L}}{\partial T_{2,z}} \cdot T_{4,z} $$
       [S1, unofficial]
    d. Accumulate the gradient magnitude for primitive $i$ in view $j$. Note: The source defines the final score using the norm of the gradient with respect to $\boldsymbol{\mu}_i$, but provides the analytical components $g_x, g_y$ derived from the T-matrix. The implementation should use the provided analytical components to compute the norm $\left\| \frac{\partial \mathcal{L}_j}{\partial \boldsymbol{\mu}_i} \right\|_2$.

2. Calculate the mean gradient score $\bar{g}_i$ for primitive $i$ over all $N$ views:
   $$ \bar{g}_i = \frac{1}{N} \sum_{j=1}^N \left\| \frac{\partial \mathcal{L}_j}{\partial \boldsymbol{\mu}_i} \right\|_2 $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_screen_space_gradient_score(primitive_i, views):
    """
    Computes the mean screen-space gradient score for a primitive.
    
    Args:
        primitive_i: The primitive object containing necessary state.
        views: List of view objects, each containing:
            - T: 4x4 projection matrix (list of lists or array)
            - loss: Photometric loss value
            - dL_dT: Gradient of loss w.r.t. T matrix (4x4)
            
    Returns:
        float: Mean gradient score bar_g_i
    """
    total_gradient_norm = 0.0
    count = 0
    
    for view in views:
        # Check if primitive contributed to this view
        if not primitive_i.contributed_to(view):
            continue
            
        T = view.T
        dL_dT = view.dL_dT
        
        # Identify row index z used for projection
        # The source implies z is a specific row index used in the backward pass.
        # In standard projection, z often refers to the depth-related row or the 
        # row used for homogeneous division. The source formula uses T_{1,z} and T_{2,z}
        # divided by T_{4,z}. This suggests z is the row index of the output coordinate 
        # being differentiated, or the row index in T that maps to the screen coordinate.
        # Given (u,v) = (T_{1,z}/T_{4,z}, T_{2,z}/T_{4,z}), z is likely the row index 
        # of the screen coordinate (e.g., 0 for x, 1 for y) if T is applied to homogeneous 
        # coordinates. However, the notation T_{m,n} usually means row m, col n.
        # If u = T_{1,z}/T_{4,z}, and u is x-coordinate, then T_{1,z} is likely the 
        # element in row 1, column z? Or row z, column 1?
        # Standard matrix multiplication: P = T * C. P_x = T_{0,0}*C_x + ...
        # The source notation T_{m,n} is ambiguous without context of row/col convention.
        # Assuming T_{m,n} is row m, column n.
        # If u = T_{1,z}/T_{4,z}, and u is screen x, this implies a specific structure.
        # Let's assume z is the column index corresponding to the homogeneous coordinate 
        # or the specific dimension being projected.
        # The source says "z denotes the specific row index used for projection".
        # This is contradictory to standard matrix indexing if T_{1,z} means row 1.
        # Let's stick to the formula provided:
        # g_x = dL/dT_{1,z} * T_{4,z}
        # g_y = dL/dT_{2,z} * T_{4,z}
        # We need to determine z. The source says "z denotes the specific row index".
        # If z is a row index, then T_{1,z} is invalid if 1 is row and z is col.
        # Re-reading: "T_{m,n} is the n-component of the m-th row". So T_{1,z} is row 1, col z.
        # "z denotes the specific row index used for projection". This is confusing.
        # If z is a row index, then T_{1,z} should be T_{z,1}?
        # Let's assume the source meant "z is the column index" or the notation is transposed.
        # However, we must follow the source's formula exactly.
        # "z denotes the specific row index used for projection (implied by context of T-matrix backward pass)."
        # If z is a row index, then T_{1,z} is likely a typo for T_{z,1} or similar.
        # But the formula is explicit: T_{1,z}.
        # Let's assume z is a fixed constant or derived from the view.
        # In many splatting papers, the projection involves specific rows.
        # Let's assume z is the row index corresponding to the depth buffer or similar.
        # Without more context, we cannot determine z.
        # [G] Assuming z is a known constant or derived from the view's projection matrix structure.
        # For the purpose of implementation, z must be identified.
        # Commonly, for screen space, we look at the first two rows of the projection matrix 
        # after homogeneous division.
        # Let's assume z is the row index of the screen coordinate in the projected space.
        # If u = T_{1,z}/T_{4,z}, and u is x, then T_{1,z} is the element that contributes to x.
        # This suggests z is the column index of the input coordinate (e.g., 0 for x, 1 for y, 2 for z, 3 for w).
        # But the source says "row index".
        # Let's assume the source uses 1-based indexing and z is the row index of the output.
        # If z=1 (first row), then u = T_{1,1}/T_{4,1}? No, T_{4,z} would be T_{4,1}.
        # This is ambiguous. [G] The variable z is not clearly defined in the provided text.
        # We will leave z as a parameter to be determined by the specific T-matrix implementation details.
        
        # For the sake of the pseudocode, we assume z is provided or derived.
        # Let's assume z is the row index of the screen coordinate (0 for x, 1 for y) 
        # and the formula uses T_{row, col}.
        # If u = T_{1,z}/T_{4,z}, and u is x, then T_{1,z} is row 1, col z.
        # This implies z is the column index.
        # The source says "z denotes the specific row index".
        # This is a direct contradiction.
        # We will follow the formula's structure: T_{1,z} and T_{4,z}.
        
        # Let's assume z is the column index corresponding to the homogeneous coordinate 
        # or the specific dimension.
        # In the absence of clarity, we cannot implement this step correctly.
        # [G] The definition of z is ambiguous. We assume z is the column index 
        # corresponding to the input coordinate being projected, or a fixed constant 
        # defined by the renderer's T-matrix layout.
        
        # Let's assume z is 3 (the homogeneous coordinate column) for standard projection.
        # Then u = T_{1,3}/T_{4,3}.
        # This is a common pattern in some formulations.
        
        z = 3 # [G] Assumed column index for homogeneous division based on common patterns
        
        # Compute per-pixel contributions
        # dL_dT is the gradient of loss w.r.t. the entire T matrix
        dL_dT_1_z = dL_dT[1][z] # Row 1, Col z
        dL_dT_2_z = dL_dT[2][z] # Row 2, Col z
        T_4_z = T[4][z]         # Row 4, Col z (using 0-based indexing for implementation)
        
        # Note: Source uses 1-based indexing for T_{m,n}. 
        # Implementation uses 0-based.
        # T_{1,z} -> T[0][z-1] if 1-based z.
        # Let's assume z in source is 1-based.
        # If z=3 in source, it's T[2][2] in 0-based?
        # This is too ambiguous.
        
        # Let's rewrite the formula in 0-based indexing assuming z is 0-based column index.
        # g_x = dL/dT[1][z] * T[3][z]
        # g_y = dL/dT[2][z] * T[3][z]
        
        g_x = dL_dT[1][z] * T[3][z]
        g_y = dL_dT[2][z] * T[3][z]
        
        gradient_norm = math.sqrt(g_x**2 + g_y**2)
        total_gradient_norm += gradient_norm
        count += 1
        
    if count == 0:
        return 0.0
        
    return total_gradient_norm / count
```

**Parameters:**
- Threshold for densification: Not given in the sources.
- Number of views $N$: Depends on the training dataset.

**Edge Cases and Pitfalls:**
- **Gradient Cancellation:** The method avoids gradient cancellation identified in other approaches [S1, unofficial].
- **Projection Approximation:** Avoids the discrepancy introduced by projecting 3D center gradients to the image plane, which conflates depth and lateral motion [S1, unofficial].
- **Ambiguous Indexing:** The definition of $z$ and the indexing of $T_{m,n}$ is ambiguous in the source. Implementers must clarify the row/column convention of the T-matrix and the meaning of $z$ [G].

## Subtopics

### Density-Aware Splitting
- **What it is:** A complementary step that determines whether to clone or split a primitive based on its gradient score and size.
- **Difference:** Unlike 3DGS which uses a global scale threshold, this method uses a density-aware criterion suitable for inverse rendering.
- **When to choose:** When refining primitives based on both reconstruction error and local density.
- **Link:** [Density-Aware Splitting](3183-density-aware-splitting.md)

## Operating characteristics
- **Time Complexity:** Linear in the number of views $N$ and primitives. The gradient computation requires a backward pass through the rasterizer and T-matrix for each view.
- **Memory Complexity:** Requires storing the gradient of the loss with respect to the T-matrix for each view.
- **Strengths:** Provides a principled signal for densification that accurately reflects reconstruction error in screen space. Avoids gradient cancellation and projection artifacts.
- **Limitations:** Requires access to the T-matrix gradients, which may not be available in all differentiable renderers. The ambiguity in the definition of $z$ may complicate implementation.
- **When to use:** In differentiable volume rendering or inverse rendering settings where accurate densification is critical for high-frequency details.

## Background
The method originates from the 3DSS (3D Surface Splatting for Inverse Rendering) paper [S1, unofficial]. It addresses limitations in 3DGS [Kerbl et al. 2023] and 2DGS [Huang et al. 2024] by providing a more accurate gradient-based densification criterion. The approach builds on the analytical backward pass of the T-matrix to compute screen-space gradients directly.

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
