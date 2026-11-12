# Eikonal Omega Term Computation

Also called: Double backprop for Eikonal term, Second-order dOmega

## Overview
Computes the Eikonal constraint term involving second-order derivatives of the hash grid encoding, supporting two backpropagation implementations: double backprop and second-order dOmega [S1, unofficial]. It differs from sibling methods like [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) by specifically targeting the regularization of the gradient magnitude in neural implicit representations via tetrahedral decomposition, rather than general surface curvature or divergence [S1, unofficial].

## How it works
**Inputs:**
- `x_pre`: Input coordinates tensor of shape $(N, 3)$, where $N$ is the number of points.
- `feature_table`: Hash grid feature tensor of shape $(table\_size, L, F)$, where $L$ is the number of levels and $F$ is the feature dimension.
- `resolutions`: Array of grid resolutions for each level.
- `cube_vertices`: Array defining cube vertices for tetrahedralization.
- `tetrahedra_index`: Array mapping cube vertices to tetrahedra.
- `prime_factors`: Array of prime numbers for hashing.
- `g_de`: Gradient tensor related to the encoding.
- `alpha_dn`: Scaling factor tensor for the Eikonal term.
- `z_dalpha_dx_de`: Optional tensor for second-order derivative information.

**Outputs:**
- `grad_feature_table`: Gradient of the loss with respect to the feature table.
- Other gradients (`grad_x_pre`, etc.) are returned as `None` or zero tensors as per the implementation details [S1, unofficial].

**Data Structures:**
- The hash grid uses a sparse representation indexed by a hash function based on prime factors [S1, unofficial].
- Tetrahedral decomposition is used to interpolate features within each voxel, defined by `cube_vertices` and `tetrahedra_index` [S1, unofficial].

**Steps:**
1. **Forward Pass:**
   - Save all inputs (`x_pre`, `feature_table`, `resolutions`, `cube_vertices`, `tetrahedra_index`, `prime_factors`, `g_de`, `alpha_dn`, `z_dalpha_dx_de`, `F_tensor`, `table_size_tensor`) to the context [S1, unofficial].
   - Return a scalar zero tensor of the same device and dtype as `x_pre` [S1, unofficial].

2. **Backward Pass:**
   - Retrieve saved tensors from the context [S1, unofficial].
   - Compute `scale = float(grad_out.item())` [S1, unofficial].
   - Extract dimensions $N$ (number of points), $L$ (levels), $F$ (features), and `table_size` [S1, unofficial].
   - Initialize `g_feature_table` as a zero tensor with the same shape as `feature_table` [S1, unofficial].
   - Determine the computation path based on available implementations:
     - If `_tetra_db_double` is available:
       - If `z_dalpha_dx_de` is empty (`numel() == 0`), set $z = \text{zeros}(N, L \times F)$ [S1, unofficial].
       - Otherwise, set $z = (z\_dalpha\_dx\_de \times scale).contiguous().float()$ [S1, unofficial].
       - Call `_tetra_db_double` with arguments: `x_pre`, `feature_table`, $z$, `g_de`, `(alpha_dn * scale)`, `resolutions`, `g_feature_table`, $N$, $L$, $F$, `table_size`, `cube_vertices`, `tetrahedra_index`, `prime_factors` [S1, unofficial].
     - Else if `_tetra_db_second` is available:
       - Call `_tetra_db_second` with arguments: `x_pre`, `g_de`, `(alpha_dn * scale)`, `resolutions`, `g_feature_table`, $N$, $L$, $F$, `table_size`, `cube_vertices`, `tetrahedra_index`, `prime_factors` [S1, unofficial].
     - Else raise `RuntimeError` [S1, unofficial].
   - Return gradients: `grad_x_pre=None`, `grad_feature_table=g_feature_table`, `grad_resolutions=None`, `grad_cv=None`, `grad_ti=None`, `grad_pf=None`, `grad_g_de=None`, `grad_alpha_dn=None`, `grad_z=None`, `grad_F=None`, `grad_table_size=None` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eikonal_omega_term_backward(grad_out, context):
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
    F_tensor = context.saved_tensors[9]
    table_size_tensor = context.saved_tensors[10]

    scale = float(grad_out.item())
    N, L, F, table_size = extract_dimensions(feature_table, resolutions)

    g_feature_table = torch.zeros_like(feature_table)

    if _tetra_db_double_available:
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
    elif _tetra_db_second_available:
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
        raise RuntimeError("No backprop implementation available")

    return (
        None, g_feature_table, None, None, None, None, None, None, None, None, None
    )
```

**Parameters:**
- `x_pre`: Input coordinates tensor [S1, unofficial].
- `g_de`: Gradient tensor related to the encoding [S1, unofficial].
- `alpha_dn`: Scaling factor tensor for the Eikonal term [S1, unofficial].
- `z_dalpha_dx_de`: Optional tensor for second-order derivative information. If None, treated as empty tensor [S1, unofficial].

**Edge Cases and Pitfalls:**
- If neither `_tetra_db_double` nor `_tetra_db_second` is available, the method raises a `RuntimeError` [S1, unofficial].
- Precision issues may arise if `z_dalpha_dx_de` is not handled correctly when empty [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is part of the TetraSDF framework, which uses tetrahedral decomposition for efficient neural implicit surface representation [S1, unofficial]. The Eikonal term is crucial for enforcing the Eikonal equation, which ensures that the gradient magnitude of the signed distance function is unity [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
