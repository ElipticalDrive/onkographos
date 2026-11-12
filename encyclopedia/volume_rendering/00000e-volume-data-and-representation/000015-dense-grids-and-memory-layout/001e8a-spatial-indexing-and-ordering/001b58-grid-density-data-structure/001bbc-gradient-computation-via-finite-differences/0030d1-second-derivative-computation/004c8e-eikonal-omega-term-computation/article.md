# Eikonal Omega Term Computation

Also called: Double backprop for Eikonal term, Second-order dOmega

## Overview
Computes the gradient of the Eikonal constraint term with respect to the hash grid encoding features, enabling the optimization of the neural field's gradient magnitude. It differs from standard first-order backpropagation by explicitly handling the second-order derivative dependencies inherent in the Eikonal loss ($|\nabla f|^2 \approx 1$), supporting two distinct computational paths: a "double backprop" method and a "second-order dOmega" method [S1, unofficial].

## How it works
The method operates on a trilinear hash grid encoding defined over multiple resolutions. It computes the gradient of the Eikonal loss term, which depends on the squared norm of the field gradient, with respect to the feature table. The implementation relies on CUDA kernels for performance, utilizing tetrahedral decomposition for gradient computation within each voxel.

### Inputs and Outputs
**Inputs:**
*   `x_pre`: Input coordinates tensor of shape $(N, 3)$, where $N$ is the number of points.
*   `feature_table`: The hash grid feature tensor of shape $(L, F, \text{table\_size})$, where $L$ is the number of levels, $F$ is the feature dimension, and $\text{table\_size}$ is the hash table size.
*   `resolutions`: Array of grid resolutions for each level.
*   `cube_vertices`: Vertex indices defining the cube structure.
*   `tetrahedra_index`: Indices mapping cube vertices to tetrahedra for gradient calculation.
*   `prime_factors`: Prime factors used for hash collision resolution.
*   `g_de`: Gradient tensor related to the encoding output (upstream gradient).
*   `alpha_dn`: Scaling factor tensor for the Eikonal term.
*   `z_dalpha_dx_de`: Optional tensor containing second-order derivative information. If empty or None, the method defaults to a specific path [S1, unofficial].

**Outputs:**
*   `g_feature_table`: Gradient tensor with respect to `feature_table`, same shape as input.
*   `grad_x_pre`, `grad_resolutions`, etc.: Typically `None` or not computed in this specific backward pass implementation, as the focus is on feature gradient [S1, unofficial].

### Data Structures
The method assumes the following data structures are available from the parent context:
*   **Hash Grid**: A sparse or dense feature table indexed by a hash function.
*   **Tetrahedral Decomposition**: Each voxel is decomposed into 6 tetrahedra. The `tetrahedra_index` array maps the 8 cube vertices to the specific tetrahedra used for interpolation and gradient calculation.
*   **Hash Function**: Uses prime factors to compute a unique index for coordinate $(x, y, z)$:
    $$ h = (x \cdot P_0) \oplus (y \cdot P_1) \oplus (z \cdot P_2) $$
    The final index is $h \pmod{\text{table\_size}}$ or bit-masked if table size is a power of two [S1, unofficial].

### Steps
1.  **Forward Pass (Context Saving)**:
    *   Save all input tensors (`x_pre`, `feature_table`, `resolutions`, `cube_vertices`, `tetrahedra_index`, `prime_factors`, `g_de`, `alpha_dn`, `z_dalpha_dx_de`, `F_tensor`, `table_size_tensor`) to the autograd context.
    *   Return a scalar zero tensor of the same device and dtype as `x_pre` (since this function is typically used only for its side effect of computing gradients, or as part of a larger loss computation where the direct output is not the primary scalar) [S1, unofficial].

2.  **Backward Pass Initialization**:
    *   Retrieve saved tensors from context.
    *   Compute `scale` from the upstream gradient: `scale = float(grad_out.item())`.
    *   Extract dimensions: $N$ (points), $L$ (levels), $F$ (features), $\text{table\_size}$.
    *   Initialize `g_feature_table` as a zero tensor of the same shape as `feature_table` [S1, unofficial].

3.  **Path Selection**:
    *   Check if `_tetra_db_double` (double backprop kernel) is available.
    *   Check if `_tetra_db_second` (second-order dOmega kernel) is available.

4.  **Gradient Computation**:
    *   **If `_tetra_db_double` is available**:
        *   Prepare input `z`: If `z_dalpha_dx_de` is empty (numel == 0), set $z = \mathbf{0}_{N \times L \cdot F}$. Otherwise, $z = (z\_dalpha\_dx\_de \cdot \text{scale}).\text{contiguous().float()}$.
        *   Prepare `alpha_dn_scaled`: $(\alpha\_dn \cdot \text{scale}).\text{contiguous().float()}$.
        *   Call `_tetra_db_double` with arguments: `x_pre`, `feature_table`, `z`, `g_de`, `alpha_dn_scaled`, `resolutions`, `g_feature_table`, $N, L, F, \text{table\_size}$, `cube_vertices`, `tetrahedra_index`, `prime_factors`.
    *   **Else If `_tetra_db_second` is available**:
        *   Prepare `alpha_dn_scaled`: $(\alpha\_dn \cdot \text{scale}).\text{contiguous().float()}$.
        *   Call `_tetra_db_second` with arguments: `x_pre`, `g_de`, `alpha_dn_scaled`, `resolutions`, `g_feature_table`, $N, L, F, \text{table\_size}$, `cube_vertices`, `tetrahedra_index`, `prime_factors`.
    *   **Else**:
        *   Raise `RuntimeError` indicating no supported backpropagation method is available [S1, unofficial].

5.  **Return**:
    *   Return gradients for all inputs. Only `grad_feature_table` is non-None (`g_feature_table`). All others are `None` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eikonal_omega_backward(grad_out, x_pre, feature_table, resolutions, 
                           cube_vertices, tetrahedra_index, prime_factors, 
                           g_de, alpha_dn, z_dalpha_dx_de, F_tensor, 
                           table_size_tensor, _tetra_db_double=None, 
                           _tetra_db_second=None):
    """
    Computes gradients for the Eikonal Omega term.
    """
    # 1. Retrieve saved tensors (simulated)
    # In a real autograd engine, these are retrieved from ctx.saved_tensors
    
    # 2. Compute scale
    scale = float(grad_out.item())
    
    # 3. Extract dimensions
    N = x_pre.shape[0]
    L = feature_table.shape[0]
    F = feature_table.shape[1]
    table_size = feature_table.shape[2]
    
    # 4. Initialize output gradient
    g_feature_table = torch.zeros_like(feature_table)
    
    # 5. Path Selection and Execution
    if _tetra_db_double is not None:
        # Prepare z
        if z_dalpha_dx_de.numel() == 0:
            z = torch.zeros(N, L * F, device=x_pre.device, dtype=torch.float32)
        else:
            z = (z_dalpha_dx_de * scale).contiguous().float()
            
        # Prepare alpha_dn_scaled
        alpha_dn_scaled = (alpha_dn * scale).contiguous().float()
        
        # Call double backprop kernel
        _tetra_db_double(
            x_pre.contiguous().float(),
            feature_table.contiguous().float(),
            z,
            g_de.contiguous().float(),
            alpha_dn_scaled,
            resolutions,
            g_feature_table,
            N, L, F, table_size,
            cube_vertices.contiguous(),
            tetrahedra_index.contiguous(),
            prime_factors.contiguous()
        )
        
    elif _tetra_db_second is not None:
        # Prepare alpha_dn_scaled
        alpha_dn_scaled = (alpha_dn * scale).contiguous().float()
        
        # Call second-order dOmega kernel
        _tetra_db_second(
            x_pre.contiguous().float(),
            g_de.contiguous().float(),
            alpha_dn_scaled,
            resolutions,
            g_feature_table,
            N, L, F, table_size,
            cube_vertices.contiguous(),
            tetrahedra_index.contiguous(),
            prime_factors.contiguous()
        )
        
    else:
        raise RuntimeError("No supported backpropagation method for Eikonal Omega term.")
        
    # 6. Return gradients
    return (
        None,       # grad_x_pre
        g_feature_table, # grad_feature_table
        None,       # grad_resolutions
        None,       # grad_cube_vertices
        None,       # grad_tetrahedra_index
        None,       # grad_prime_factors
        None,       # grad_g_de
        None,       # grad_alpha_dn
        None,       # grad_z_dalpha_dx_de
        None,       # grad_F_tensor
        None        # grad_table_size_tensor
    )
```

### Parameters and Typical Values
*   `scale`: Derived from upstream gradient.
*   `N`: Number of sample points. Typical values depend on batch size and sampling strategy (e.g., 10,000 to 1,000,000).
*   `L`: Number of hash levels. Typical values: 4-16.
*   `F`: Feature dimension. Typical values: 2-4.
*   `table_size`: Hash table size. Typical values: $2^{14}$ to $2^{20}$.
*   `resolutions`: Grid resolutions per level. Typical values: $16, 32, 64, 128, 256, 512$.

### Edge Cases and Pitfalls
*   **Empty `z_dalpha_dx_de`**: If the tensor is empty, it must be explicitly replaced with a zero tensor of shape $(N, L \cdot F)$ before passing to the kernel. Failure to do so may cause kernel launch errors or incorrect gradients [S1, unofficial].
*   **Kernel Availability**: The method requires either `_tetra_db_double` or `_tetra_db_second` to be registered. If neither is available, the operation fails. This is a common pitfall in custom CUDA extension setups [S1, unofficial].
*   **Precision**: Inputs are cast to `float32` before kernel execution. Ensure upstream gradients are compatible with this precision [S1, unofficial].
*   **Boundary Handling**: The underlying tetrahedral interpolation assumes coordinates are within the valid range. Out-of-bounds coordinates should be handled by the parent sampling method (e.g., clamping or zeroing) before calling this backward pass [G].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot L)$, where $N$ is the number of points and $L$ is the number of levels. Each point accesses $L$ hash tables.
*   **Memory Complexity**: $O(\text{table\_size} \cdot F)$ for the feature table. The gradient computation requires temporary storage for $z$ and scaled inputs, proportional to $N \cdot L \cdot F$.
*   **Performance**: Highly optimized for GPU execution via CUDA kernels. Performance scales with the number of threads and memory bandwidth. The choice between `_tetra_db_double` and `_tetra_db_second` may affect performance based on the specific hardware and the sparsity of second-order derivative information [S1, unofficial].
*   **Strengths**: Enables efficient optimization of the Eikonal constraint in neural fields. Supports two distinct backpropagation strategies for flexibility.
*   **Limitations**: Requires custom CUDA kernels. Not available in standard autograd frameworks without extension. Dependent on the availability of specific kernel implementations [S1, unofficial].

## Background
The Eikonal Omega term computation is part of the optimization pipeline for Neural Implicit Representations, specifically those using hash grid encodings like Instant NGP or TetraSDF. The Eikonal loss enforces the property that the gradient magnitude of the signed distance field (SDF) is approximately 1 everywhere. Computing the gradient of this loss with respect to the hash grid features requires careful handling of the second-order derivatives, as the loss depends on the squared gradient norm. The "double backprop" and "second-order dOmega" methods were introduced to efficiently compute these gradients without explicitly constructing the full Hessian matrix, leveraging the structure of the tetrahedral interpolation [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
