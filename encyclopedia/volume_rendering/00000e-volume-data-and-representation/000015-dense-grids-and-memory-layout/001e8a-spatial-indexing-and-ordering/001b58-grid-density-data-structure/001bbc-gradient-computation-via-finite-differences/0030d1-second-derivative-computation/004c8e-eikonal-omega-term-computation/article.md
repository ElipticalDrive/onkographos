# Eikonal Omega Term Computation

Also called: Double backprop for Eikonal term, Second-order dOmega

## Overview
Computes the gradient of the Eikonal constraint term with respect to the hash grid encoding features, enabling the enforcement of the unit-norm gradient property in neural implicit surfaces. It differs from standard first-order backpropagation by explicitly handling the second-order derivative dependencies inherent in the Eikonal loss, supporting two distinct implementation paths: a "double backprop" approach and a "second-order dOmega" approach.

## How it works
The method operates on a tetrahedral hash grid encoding (specifically TetraSDF). It computes the gradient of the Eikonal term $\mathcal{L}_{Eikonal} = \sum (\|\nabla \phi\|^2 - 1)^2$ with respect to the feature table. The implementation relies on CUDA kernels that access features via hash-based indexing and interpolate within tetrahedra.

### Inputs and Outputs
**Inputs:**
*   `x_pre`: Input coordinates tensor of shape $(N, 3)$, where $N$ is the number of points.
*   `feature_table`: The hash grid feature tensor of shape $(L, F, \text{table\_size})$ or similar flattened layout, where $L$ is the number of levels, $F$ is the feature dimension, and $\text{table\_size}$ is the hash table size.
*   `resolutions`: Array of grid resolutions for each level $L$.
*   `cube_vertices`: Array defining the cube topology.
*   `tetrahedra_index`: Array mapping cube vertices to tetrahedra.
*   `prime_factors`: Array of three prime numbers used for hash collision resolution.
*   `g_de`: Gradient tensor flowing back from the Eikonal loss with respect to the encoded field values.
*   `alpha_dn`: Scaling factor tensor for the Eikonal term.
*   `z_dalpha_dx_de`: Optional tensor containing second-order derivative information. If empty or None, the "second-order dOmega" path is not used in the specific conditional logic described in [S1, unofficial].

**Outputs:**
*   `g_feature_table`: Gradient tensor with respect to `feature_table`.
*   Other gradients (`grad_x_pre`, etc.) are returned as `None` or not computed in this specific backward pass implementation [S1, unofficial].

### Data Structures
The method uses a hash grid structure. The index into the feature table is computed using a hash function based on prime factors to minimize collisions [S1, unofficial]:
$$ h = (x \cdot P_0) \oplus (y \cdot P_1) \oplus (z \cdot P_2) $$
$$ \text{index} = h \pmod{\text{table\_size}} $$
where $P_0, P_1, P_2$ are the prime factors. If `table_size` is a power of two, a bitwise AND is used instead of modulo [S1, unofficial].

Coordinates are normalized to the grid resolution $R$ using:
$$ s = R - 1 $$
$$ p_x = x \cdot s $$
$$ g_x = \lfloor p_x \rfloor $$
$$ f_x = p_x - g_x $$
where $g_x$ is the integer grid coordinate and $f_x$ is the fractional part [S1, unofficial].

### Steps
1.  **Forward Pass (Context Saving):**
    *   Save all input tensors (`x_pre`, `feature_table`, `resolutions`, `cube_vertices`, `tetrahedra_index`, `prime_factors`, `g_de`, `alpha_dn`, `z_dalpha_dx_de`, etc.) to the autograd context.
    *   Return a scalar zero tensor of the same device and dtype as `x_pre` [S1, unofficial].

2.  **Backward Pass:**
    *   Retrieve saved tensors from context.
    *   Compute `scale` from the incoming gradient: `scale = float(grad_out.item())`.
    *   Extract dimensions $N$ (points), $L$ (levels), $F$ (features), and `table_size`.
    *   Initialize `g_feature_table` as a zero tensor of the same shape as `feature_table`.
    *   **Conditional Execution:**
        *   If `_tetra_db_double` kernel is available:
            *   Check if `z_dalpha_dx_de` is empty (`numel() == 0`).
            *   If empty, initialize $z$ as a zero tensor of shape $(N, L \cdot F)$.
            *   If not empty, set $z = (z\_dalpha\_dx\_de \cdot \text{scale}).\text{contiguous}().\text{float}()$.
            *   Call `_tetra_db_double` kernel with arguments: `x_pre`, `feature_table`, `z`, `g_de`, `(alpha_dn * scale)`, `resolutions`, `g_feature_table`, $N, L, F$, `table_size`, `cube_vertices`, `tetrahedra_index`, `prime_factors`.
        *   Else if `_tetra_db_second` kernel is available:
            *   Call `_tetra_db_second` kernel with arguments: `x_pre`, `g_de`, `(alpha_dn * scale)`, `resolutions`, `g_feature_table`, $N, L, F$, `table_size`, `cube_vertices`, `tetrahedra_index`, `prime_factors`.
        *   Else:
            *   Raise `RuntimeError` [S1, unofficial].

3.  **Return:**
    *   Return gradients: `grad_x_pre=None`, `grad_feature_table=g_feature_table`, `grad_resolutions=None`, `grad_cv=None`, `grad_ti=None`, `grad_pf=None`, `grad_g_de=None`, `grad_alpha_dn=None`, `grad_z=None`, `grad_F=None`, `grad_table_size=None` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eikonal_omega_backward(grad_out, context):
    # Retrieve saved tensors
    x_pre = context.saved_tensors[0]
    feature_table = context.saved_tensors[1]
    resolutions = context.saved_tensors[2]
    cube_vertices = context.saved_tensors[3]
    tetrahedra_index = context.saved_tensors[4]
    prime_factors = context.saved_tensors[5]
    g_de = context.saved_tensors[6]
    alpha_dn = context.saved_tensors[7]
    z_dalpha_dx_de = context.saved_tensors[8]
    
    # Compute scale
    scale = float(grad_out.item())
    
    # Extract dimensions
    N, L, F, table_size = get_dimensions(feature_table, resolutions)
    
    # Initialize gradient table
    g_feature_table = torch.zeros_like(feature_table)
    
    # Check which kernel to use
    if has_kernel("_tetra_db_double"):
        if z_dalpha_dx_de.numel() == 0:
            z = torch.zeros(N, L * F, device=x_pre.device, dtype=torch.float32)
        else:
            z = (z_dalpha_dx_de * scale).contiguous().float()
            
        _tetra_db_double(
            x_pre.contiguous().float(),
            feature_table.contiguous().float(),
            z,
            g_de.contiguous().float(),
            (alpha_dn * scale).contiguous().float(),
            resolutions.contiguous(),
            g_feature_table,
            N, L, F, table_size,
            cube_vertices.contiguous(),
            tetrahedra_index.contiguous(),
            prime_factors.contiguous()
        )
    elif has_kernel("_tetra_db_second"):
        _tetra_db_second(
            x_pre.contiguous().float(),
            g_de.contiguous().float(),
            (alpha_dn * scale).contiguous().float(),
            resolutions.contiguous(),
            g_feature_table,
            N, L, F, table_size,
            cube_vertices.contiguous(),
            tetrahedra_index.contiguous(),
            prime_factors.contiguous()
        )
    else:
        raise RuntimeError("No Eikonal backward kernel available")
        
    return None, g_feature_table, None, None, None, None, None, None, None, None, None
```

### Parameters
*   `x_pre`: Input coordinates.
*   `g_de`: Gradient from Eikonal loss.
*   `alpha_dn`: Scaling factor.
*   `z_dalpha_dx_de`: Optional second-order derivative tensor.

### Edge Cases and Pitfalls
*   **Empty `z_dalpha_dx_de`:** The implementation explicitly checks if this tensor is empty to decide whether to pass a zero tensor or the actual second-order derivative data to the double backprop kernel [S1, unofficial].
*   **Kernel Availability:** The method relies on specific CUDA kernels (`_tetra_db_double` or `_tetra_db_second`). If neither is available, it raises an error [S1, unofficial].
*   **Hash Collisions:** The hash function uses prime factors to distribute indices. Collisions are handled by the underlying hash table implementation, but poor prime selection can lead to performance degradation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of points $N$, levels $L$, and features $F$. The hash lookup is $O(1)$ on average. The interpolation within tetrahedra is constant time per point per level.
*   **Memory Complexity:** Requires storing the feature table and intermediate gradients. The backward pass requires temporary storage for $z$ if using the double backprop method [S1, unofficial].
*   **Strengths:** Efficiently computes gradients for the Eikonal constraint in hash-based neural implicit representations.
*   **Limitations:** Requires specific CUDA kernel implementations. Relies on hash grid structure, which may have collision issues.

## Background
This method is part of the TetraSDF framework, which uses tetrahedral hash grids for neural implicit surface representation. The Eikonal term is crucial for ensuring the predicted field is a valid signed distance function. The "double backprop" and "second-order dOmega" approaches are specific implementations within this framework to handle the second-order derivatives required for the Eikonal loss gradient [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
