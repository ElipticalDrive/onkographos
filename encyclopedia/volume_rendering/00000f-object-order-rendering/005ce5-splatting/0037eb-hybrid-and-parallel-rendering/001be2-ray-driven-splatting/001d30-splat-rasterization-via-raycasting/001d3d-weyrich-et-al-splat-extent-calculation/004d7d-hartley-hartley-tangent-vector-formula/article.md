# Hartley-Hartley Tangent Vector Formula

Also called: HH formula, HH technique, Householder (HH) Formula, Householder formula for tangential vector orthogonalization, Householder transformation

## Overview
The Hartley-Hartley (HH) Tangent Vector Formula computes tangent and bitangent vectors from a given surface normal vector using Householder reflections [S1]. It solves the problem of establishing a local orthonormal reference frame for splat orientation in point cloud rendering without requiring neighborhood data [S1]. Unlike the Covariance Matrix (CM) technique, which is shape-aware and rotationally invariant, HH is non-shape aware and extrinsic, meaning stroke directions depend on the embedding space rather than the point cloud shape [S1]. It is chosen when computational efficiency is prioritized over local geometric coherence, particularly for noisy, under-sampled, or sparse data where CM fails [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A unit normal vector $\mathbf{n} = [n_x, n_y, n_z]^T$ [S1].
*   **Output:** A tangent vector $\mathbf{t}$ and a bitangent (binormal) vector $\mathbf{b}$, such that $\{\mathbf{n}, \mathbf{t}, \mathbf{b}\}$ form an orthonormal basis [S1].

**Data Structures:**
*   The vectors are represented as 3-component floating-point arrays [G].
*   The calculation is typically performed in a geometry shader to avoid storing extra vertices [S1].

**Steps:**
1. Evaluate the sign of the first component of the normal vector, $n_x$ [S1].
2. Compute the tangent vector $\mathbf{t}$ based on the sign of $n_x$ [S1].
3. Compute the bitangent vector $\mathbf{b}$ based on the sign of $n_x$ [S1].

**Formulas:**
The source provides conditional component formulas derived from collinearity and orthogonality conditions of the Householder matrix [S1]. The exact LaTeX in the source is garbled, but the structure relies on conditional logic based on $n_x$ to avoid numerical cancellation [S1]. A standard Householder vector $\mathbf{v}$ for reflecting $\mathbf{n}$ to a basis vector (e.g., $\mathbf{e}_1$) is $\mathbf{v} = \mathbf{n} + \text{sgn}(n_x)\mathbf{e}_1$ [G]. The tangent and bitangent are then derived as the second and third columns of the Householder reflection matrix $H = I - 2\frac{\mathbf{v}\mathbf{v}^T}{\mathbf{v}^T\mathbf{v}}$, ensuring $\mathbf{t}$ and $\mathbf{b}$ are orthogonal to $\mathbf{n}$ and to each other [G]. The source confirms the computational cost is 1 order operation, 3 summations/subtractions, 10–12 multiplications, and 1 division [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hh_tangent_bitangent(n_x, n_y, n_z):
    """
    Computes tangent and bitangent vectors from a unit normal vector
    using the Hartley-Hartley (Householder) formula.
    
    Args:
        n_x, n_y, n_z: Components of the unit normal vector n.
        
    Returns:
        t: Tangent vector [tx, ty, tz]
        b: Bitangent vector [bx, by, bz]
    """
    # Step 1: Evaluate sign of n_x
    if n_x >= 0:
        sign_nx = 1.0
    else:
        sign_nx = -1.0

    # Construct Householder vector v = n + sign(n_x) * e_1
    v_x = n_x + sign_nx
    v_y = n_y
    v_z = n_z
    
    # Compute squared norm of v
    v_dot_v = v_x * v_x + v_y * v_y + v_z * v_z
    
    # Compute scaling factor 2 / (v^T v)
    factor = 2.0 / v_dot_v
    
    # Compute tangent vector (second column of H = I - factor * v v^T)
    tx = -factor * v_x * v_y
    ty = 1.0 - factor * v_y * v_y
    tz = -factor * v_y * v_z
    
    # Compute bitangent vector (third column of H)
    bx = -factor * v_x * v_z
    by = -factor * v_y * v_z
    bz = 1.0 - factor * v_z * v_z
    
    return [tx, ty, tz], [bx, by, bz]
```

**Parameters:**
*   The calculation is performed in the geometry shader [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The formula is designed to be continuous and smooth in most of its domain [S1].
*   **Degenerate Input:** If the normal vector is zero, the method fails [G]. The source assumes unit normal vectors [S1].
*   **Assumption:** The advantage of real-time computation holds under the assumption that normal vectors have been estimated by a more efficient technique than the CM method [S1].

## Operating characteristics
*   **Time Complexity:** The HH formula calculations sum up to 1 order operation, 3 summations/subtractions, 10–12 multiplications, and 1 division [S1]. This is much less demanding than solving the eigenvalue problem for a 3x3 matrix required by the Covariance Matrix (CM) technique [S1].
*   **Memory Complexity:** It requires no additional storage for neighborhood data, unlike CM which needs neighboring points [S1]. It is performed in the geometry shader to avoid storing three extra vertices per point, which is more effective for large datasets [S1].
*   **Strengths:**
    *   Computationally cheap, allowing for real-time tangent computation [S1].
    *   Works with noisy, under-sampled, or sparse data with holes, where CM fails [S1].
    *   Applicable to diverse input types: depth maps, implicit surfaces, meshes, and point clouds [S1].
    *   Produces "virtual paint brush strokes" that flow smoothly and continuously, providing local and global coherence [S1].
*   **Limitations:**
    *   Non-shape aware: Tangent directions depend on the embedding space, not the local point cloud shape [S1].
    *   Extrinsic: Rotating the point cloud changes the stroke directions [S1].
    *   Uses an isotropic radius, ignoring local anisotropic density [S1].
*   **When to use:** Use when rendering noisy, sparse, or low-quality data, or when computational efficiency is critical and shape-aware orientation is not required [S1].
*   **When not to use:** Use CM instead when shape-aware, rotationally invariant orientation is needed and the data is dense and high-quality [S1].

## Background
*   **Origin:** The method is described in "Stroke-based splatting: an efficient multi-resolution point cloud visualization technique" [S1]. It is based on the Householder formula for orthogonalization [S1].
*   **History:** The technique was compared to Eberly (EB), Square Plate (SQP), and Covariance Matrix (CM) techniques [S1]. It was chosen for its analyticity, continuity, and low computational cost [S1].
*   **Usage:** It is used in stroke-based splatting to provide virtual paint brush strokes that carry first-order differential information of the estimated surface normal plane [S1].

## Sources
[S1] Stroke-based splatting: an efficient multi-resolution point cloud visualization technique, https://doi.org/10.1007/s00371-017-1420-7, fetched 2026-10-26
[S2] To Splat Straight with Crooked Points: Rendering Noisy Meshes and Point Clouds using Coherent Tangent Vector Fields, https://doi.org/10.2312/egp.20211028, fetched 2026-10-26 (unofficial)
