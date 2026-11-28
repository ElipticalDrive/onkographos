# Eikonal Omega Term Computation

Also called: Double backprop for Eikonal term, Second-order dOmega

## Overview
Computes the Eikonal constraint term involving second-order derivatives of the hash grid encoding, supporting two backpropagation implementations: double backprop and second-order dOmega [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `x_pre`: Input coordinates tensor [S1, unofficial].
    *   `feature_table`: The hash grid feature tensor [S1, unofficial].
    *   `resolutions`: Array of grid resolutions per level [S1, unofficial].
    *   `cube_vertices`: Cube vertex indices [S1, unofficial].
    *   `tetrahedra_index`: Tetrahedron vertex indices [S1, unofficial].
    *   `prime_factors`: Hash prime factors [S1, unofficial].
    *   `g_de`: Gradient tensor related to the encoding [S1, unofficial].
    *   `alpha_dn`: Scaling factor tensor for the Eikonal term [S1, unofficial].
    *   `z_dalpha_dx_de`: Optional tensor for second-order derivative information [S1, unofficial].
    *   `F_tensor`: Feature dimension size [S1, unofficial].
    *   `table_size_tensor`: Hash table size [S1, unofficial].
*   **Outputs:**
    *   Forward: Scalar zero tensor [S1, unofficial].
    *   Backward: Gradients for `feature_table` (`g_feature_table`), `g_de`, and others (mostly `None`) [S1, unofficial].

**Data Structures:**
The method operates on tensors representing the hash grid features and input coordinates [S1, unofficial]. The hash grid uses a sparse hash table indexed by a hash function derived from coordinates and prime factors [S1, unofficial]. The spatial interpolation uses a tetrahedral decomposition of the cube, defined by `cube_vertices` and `tetrahedra_index` [S1, unofficial].

**Steps:**
1.  **Forward Pass:**
    *   Save all inputs (`x_pre`, `feature_table`, `resolutions`, `cube_vertices`, `tetrahedra_index`, `prime_factors`, `g_de`, `alpha_dn`, `z_dalpha_dx_de`, `F_tensor`, `table_size_tensor`) to the autograd context [S1, unofficial].
    *   Return a scalar zero tensor of the same device and dtype as `x_pre` [S1, unofficial].
2.  **Backward Pass:**
    *   Retrieve saved tensors from the context [S1, unofficial].
    *   Compute `scale` as `float(grad_out.item())` [S1, unofficial].
    *   Extract dimensions `N` (number of points), `L` (number of levels), `F` (feature size), and `table_size` [S1, unofficial].
    *   Initialize `g_feature_table` as a zero tensor with the same shape as `feature_table` [S1, unofficial].
    *   Determine the backpropagation mode based on available helper functions `_tetra_db_double` and `_tetra_db_second` [S1, unofficial]:
        *   **Mode 1 (Double Backprop):** If `_tetra_db_double` is available:
            *   If `z_dalpha_dx_de` is empty (numel == 0), set `z` to a zero tensor of shape `(N, L*F)` [S1, unofficial].
            *   Otherwise, set `z = (z_dalpha_dx_de * scale).contiguous().float()` [S1, unofficial].
            *   Call `_tetra_db_double` with arguments: `x_pre`, `feature_table`, `z`, `g_de`, `(alpha_dn * scale)`, `resolutions`, `g_feature_table`, `N`, `L`, `F`, `table_size`, `cube_vertices`, `tetrahedra_index`, `prime_factors` [S1, unofficial].
        *   **Mode 2 (Second-order dOmega):** If `_tetra_db_second` is available (and Mode 1 is not):
            *   Call `_tetra_db_second` with arguments: `x_pre`, `g_de`, `(alpha_dn * scale)`, `resolutions`, `g_feature_table`, `N`, `L`, `F`, `table_size`, `cube_vertices`, `tetrahedra_index`, `prime_factors` [S1, unofficial].
        *   **Error:** If neither helper is available, raise `RuntimeError` [S1, unofficial].
    *   Return gradients: `grad_x_pre=None`, `grad_feature_table=g_feature_table`, `grad_resolutions=None`, `grad_cv=None`, `grad_ti=None`, `grad_pf=None`, `grad_g_de=None`, `grad_alpha_dn=None`, `grad_z=None`, `grad_F=None`, `grad_table_size=None` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eikonal_omega_term_backward(grad_out, saved_tensors):
    # Unpack saved tensors
    x_pre, feature_table, resolutions, cube_vertices, tetrahedra_index, prime_factors = saved_tensors[:6]
    g_de, alpha_dn, z_dalpha_dx_de, F_tensor, table_size_tensor = saved_tensors[6:]

    # Compute scale
    scale = float(grad_out.item())

    # Extract dimensions
    N, L, F = feature_table.shape[0], feature_table.shape[1], feature_table.shape[2]
    table_size = table_size_tensor.item()

    # Initialize gradient for feature table
    g_feature_table = [0.0] * (N * L * F)

    # Determine mode and compute gradients
    if _tetra_db_double is not None:
        if z_dalpha_dx_de.numel() == 0:
            z = [0.0] * (N * L * F)
        else:
            z = [val * scale for val in z_dalpha_dx_de]

        _tetra_db_double(
            x_pre, feature_table, z, g_de,
            [a * scale for a in alpha_dn], resolutions, g_feature_table,
            N, L, F, table_size, cube_vertices, tetrahedra_index, prime_factors
        )
    elif _tetra_db_second is not None:
        _tetra_db_second(
            x_pre, g_de,
            [a * scale for a in alpha_dn], resolutions, g_feature_table,
            N, L, F, table_size, cube_vertices, tetrahedra_index, prime_factors
        )
    else:
        raise RuntimeError("No backpropagation implementation available")

    return (None, g_feature_table, None, None, None, None, None, None, None, None, None)

def _tetra_db_double(x, features, z, g_de, alpha_scale, res, grad_feat, N, L, F, table_size, cv, ti, pf):
    # Double backpropagation: accumulates second-order gradient contributions into grad_feat
    for p in range(N):
        for level in range(L):
            idx = p * L + level
            x_val, y_val, z_val = x[p*3], x[p*3+1], x[p*3+2]
            res_val = res[level]
            # Compute fractional coordinates and grid indices
            fx = (x_val * (res_val - 1)) % 1.0
            gx = int((x_val * (res_val - 1)) % res_val)
            fy = (y_val * (res_val - 1)) % 1.0
            gy = int((y_val * (res_val - 1)) % res_val)
            fz = (z_val * (res_val - 1)) % 1.0
            gz = int((z_val * (res_val - 1)) % res_val)
            # Hash grid lookup and tetrahedral interpolation gradient scatter
            for f in range(F):
                feat_idx = (gx * res_val + gy) * res_val + gz
                hash_idx = (feat_idx * pf[0] ^ feat_idx * pf[1] ^ feat_idx * pf[2]) % table_size
                grad_feat[hash_idx * F + f] += z[idx * F + f] * alpha_scale[idx]

def _tetra_db_second(x, g_de, alpha_scale, res, grad_feat, N, L, F, table_size, cv, ti, pf):
    # Second-order dOmega: accumulates direct second-order derivative gradients into grad_feat
    for p in range(N):
        for level in range(L):
            idx = p * L + level
            x_val, y_val, z_val = x[p*3], x[p*3+1], x[p*3+2]
            res_val = res[level]
            # Compute fractional coordinates and grid indices
            fx = (x_val * (res_val - 1)) % 1.0
            gx = int((x_val * (res_val - 1)) % res_val)
            fy = (y_val * (res_val - 1)) % 1.0
            gy = int((y_val * (res_val - 1)) % res_val)
            fz = (z_val * (res_val - 1)) % 1.0
            gz = int((z_val * (res_val - 1)) % res_val)
            # Hash grid lookup and second-order gradient scatter
            for f in range(F):
                feat_idx = (gx * res_val + gy) * res_val + gz
                hash_idx = (feat_idx * pf[0] ^ feat_idx * pf[1] ^ feat_idx * pf[2]) % table_size
                grad_feat[hash_idx * F + f] += g_de[idx * F + f] * alpha_scale[idx]
```

**Parameters:**
*   `x_pre`: Input coordinates tensor [S1, unofficial].
*   `g_de`: Gradient tensor related to the encoding [S1, unofficial].
*   `alpha_dn`: Scaling factor tensor for the Eikonal term [S1, unofficial].
*   `z_dalpha_dx_de`: Optional tensor for second-order derivative information. If None, treated as empty tensor [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty `z_dalpha_dx_de`:** If the optional tensor is empty, it must be explicitly replaced with a zero tensor of the correct shape before passing to `_tetra_db_double` [S1, unofficial].
*   **Missing Implementations:** If neither `_tetra_db_double` nor `_tetra_db_second` is available, the method fails with a runtime error [S1, unofficial].
*   **Data Types:** Inputs to the gradient accumulation kernels are cast to `float` and made contiguous [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is part of the TetraSDF framework, which uses tetrahedral decomposition for efficient interpolation in hash grid encodings [S1, unofficial]. It supports two backpropagation strategies for computing the Eikonal term's gradient: double backpropagation and a second-order derivative approach [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
