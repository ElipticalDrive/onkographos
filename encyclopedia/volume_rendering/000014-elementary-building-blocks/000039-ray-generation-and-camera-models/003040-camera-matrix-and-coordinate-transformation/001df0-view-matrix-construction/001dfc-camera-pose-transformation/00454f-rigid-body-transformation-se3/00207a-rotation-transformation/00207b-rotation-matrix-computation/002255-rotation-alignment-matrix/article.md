# Rotation Alignment Matrix

Also called: Differentiable Rotation Matrices

## Overview
The Rotation Alignment Matrix computes a $3 \times 3$ rotation matrix $R$ that rotates a source direction vector $v_1$ to align with a target direction vector $v_2$ such that $R v_1 = v_2$ [S1]. It solves the problem of constructing valid rotation matrices when $v_1$ and $v_2$ are parallel or identical, a case where standard Rodrigues' rotation formula fails due to division by zero in the normalized rotation axis [S1]. Unlike [Rotation Matrix Computation](2280-rotation-matrix-computation.md), which requires an explicit axis-angle pair, this method derives the rotation directly from two arbitrary vectors and remains differentiable even in degenerate cases [S1].

## How it works
**Inputs:** Two 3D vectors $v_1$ (from) and $v_2$ (to). Both are assumed to be non-zero for the general case, though specific handling exists for zero vectors [S2, unofficial].
**Output:** A $3 \times 3$ rotation matrix $R$.

**Data Structures:**
- $v_1, v_2$: 3-element vectors (float32 or float64).
- $R$: $3 \times 3$ matrix (float32 or float64).
- Intermediate scalars $\alpha, c$ and vector $\beta, v$.

**Steps:**
1. Compute the dot product $\alpha = v_1 \cdot v_2$ [S1].
2. Compute the cross product $\beta = v_1 \times v_2$ [S1].
3. Handle degenerate cases:
   - If $v_1$ is zero and $v_2$ is non-zero, or if the alignment is undefined, the behavior is implementation-dependent. However, [S2, unofficial] specifies handling for zero vectors in its specific implementation context: if the input vector is zero and the scalar component $c > 0$, return the identity matrix; if $c < 0$, return the negative identity matrix [S2, unofficial].
   - If $v_1$ and $v_2$ are parallel and point in the same direction ($v_1 = v_2$), $R$ is the identity matrix $I$ [S1].
   - If $v_1$ and $v_2$ are parallel and point in opposite directions ($v_1 = -v_2$), a 180-degree rotation is required. The axis of rotation is undefined by the cross product (which is zero). A stable implementation must select an arbitrary orthogonal axis or use a fallback rotation matrix [S1, G]. The sources do not specify the exact fallback axis selection, but note that Rodrigues' formula is undefined here [S1].
4. Compute the scalar $k = \frac{1}{1 + \alpha}$ [S1]. Note: [S2, unofficial] uses $c$ for the dot product and $k = \frac{1}{1+c}$, which is equivalent to $\frac{1}{1+\alpha}$ [S2, unofficial].
5. Construct the skew-symmetric matrix $[\beta]_x$ from $\beta$ [S1]:
   $$ [\beta]_x = \begin{bmatrix} 0 & -\beta_x & \beta_y \\ \beta_x & 0 & -\beta_z \\ -\beta_y & \beta_z & 0 \end{bmatrix} $$
   [S1]
6. Compute the outer product $\beta^T \beta$ (a $3 \times 3$ matrix) [S1].
7. Assemble $R$ using the formula:
   $$ R = \alpha I + [\beta]_x + \frac{1}{1 + \alpha} \beta^T \beta $$
   [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rotation_alignment_matrix(v1: np.ndarray, v2: np.ndarray) -> np.ndarray:
    """
    Computes a rotation matrix R such that R @ v1 = v2.
    Handles the case where v1 == v2 (identity) and v1 == -v2 (180 deg rotation).
    Differentiable with respect to v1 and v2.
    
    Args:
        v1: (3,) array-like, source vector.
        v2: (3,) array-like, target vector.
        
    Returns:
        (3, 3) ndarray, rotation matrix.
    """
    v1 = np.asarray(v1, dtype=np.float64)
    v2 = np.asarray(v2, dtype=np.float64)
    
    # Dot product alpha
    alpha = np.dot(v1, v2)
    
    # Cross product beta
    beta = np.cross(v1, v2)
    
    # Handle parallel cases
    # If v1 and v2 are identical (or very close), return identity
    if np.allclose(v1, v2):
        return np.eye(3)
    
    # If v1 and v2 are opposite, cross product is zero.
    # We need a 180 degree rotation around an axis orthogonal to v1.
    if np.allclose(v1, -v2):
        # Find an orthogonal vector to v1
        # Pick an arbitrary vector not parallel to v1
        if not np.allclose(v1, [1, 0, 0]):
            arbitrary = np.array([1.0, 0.0, 0.0])
        else:
            arbitrary = np.array([0.0, 1.0, 0.0])
        
        # Orthogonal axis via cross product
        axis = np.cross(v1, arbitrary)
        if np.linalg.norm(axis) < 1e-8:
            # Fallback if v1 is parallel to arbitrary (shouldn't happen with above logic)
            axis = np.array([0.0, 1.0, 0.0])
        
        axis = axis / np.linalg.norm(axis)
        
        # Rodrigues' formula for 180 degrees: R = I + 2 * [axis]_x^2
        # Or simply: R = 2 * axis * axis^T - I
        R = 2.0 * np.outer(axis, axis) - np.eye(3)
        return R

    # General case
    # k = 1 / (1 + alpha)
    k = 1.0 / (1.0 + alpha)
    
    # Skew-symmetric matrix of beta
    beta_x = np.array([
        [0.0, -beta[2], beta[1]],
        [beta[2], 0.0, -beta[0]],
        [-beta[1], beta[0], 0.0]
    ])
    
    # Outer product beta^T beta
    beta_outer = np.outer(beta, beta)
    
    # R = alpha * I + [beta]_x + k * (beta^T beta)
    R = alpha * np.eye(3) + beta_x + k * beta_outer
    
    return R
```

**Parameters:**
- None. The method is fully determined by inputs $v_1, v_2$.

**Edge Cases and Pitfalls:**
- **Parallel Vectors:** Rodrigues' formula involves a normalized rotation axis. When $v_1 = v_2$, the rotation axis is undefined (norm is zero), making the standard formula undefined [S1]. The derived formula $R = \alpha I + [\beta]_x + \frac{1}{1 + \alpha} \beta^T \beta$ remains valid and differentiable in this limit, yielding the identity matrix [S1].
- **Anti-Parallel Vectors:** When $v_1 = -v_2$, $\alpha = -1$, leading to division by zero in $k = \frac{1}{1+\alpha}$ [S1]. The cross product $\beta$ is also zero. The formula breaks down. A separate case handling (as in the pseudocode) is required [S1, G].
- **Zero Vectors:** If $v_1$ or $v_2$ is zero, the direction is undefined. [S2, unofficial] provides specific handling for zero vectors in its context, but generally, these inputs are invalid for rotation alignment [S2, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$. The method involves a constant number of vector operations (dot, cross, outer product) and matrix additions.
- **Memory Complexity:** $O(1)$. Requires storage for a few 3D vectors and a 3x3 matrix.
- **Strengths:** Differentiable everywhere except the anti-parallel case (which can be handled with a smooth interpolation or a separate branch). It avoids the singularity of Rodrigues' formula at $v_1 = v_2$ [S1].
- **Limitations:** The anti-parallel case ($v_1 = -v_2$) is a singularity where the rotation axis is undefined. The gradient is discontinuous or undefined here without additional smoothing [S1].
- **When to use:** When differentiable rotation between two vectors is needed, especially in differentiable rendering pipelines [S1].
- **When not to use:** When $v_1$ and $v_2$ are anti-parallel and a smooth gradient is required without branching.

## Background
The method was introduced by Felix Petersen, Amit Bermano, Oliver Deussen, and Daniel Cohen-Or in 2019 as part of "Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer" [S1]. It was designed to enable differentiable rendering by providing a differentiable way to align normals or directions, overcoming the non-differentiability of standard Rodrigues' rotation formula at the identity rotation [S1].

## Sources
[S1] Reparameterizing discontinuous integrands for differentiable rendering, https://doi.org/10.1145/3355089.3356510, fetched 2026-10-26
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
