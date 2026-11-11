# Learnable Normals Parameterization
Also called: Normal Parameterization, Sign-Direction Normal Decomposition

## Overview
A parameterization for learnable surface normals of Gaussians that uses a sign scalar and a direction vector to stabilize optimization and allow normal flipping without rotation [S1]. It solves the instability inherent in naive vector parameterization, where flipping a normal requires a 180-degree rotation that causes convergence to inaccurate local minima [S1]. It sets itself apart from sibling methods like [Normal Field Supervision via Rendering and Depth Derivatives](9627-normal-field-supervision-via-external-channel.md) by explicitly modifying the Gaussian's internal attribute representation rather than relying on external gradient supervision.

## How it works
**Inputs and Outputs:**
*   Input: A set of Gaussians, each with an associated learnable normal attribute.
*   Output: A stable, unit-length surface normal vector $\mathbf{n}_i$ for each Gaussian $i$, suitable for rendering and gradient-based optimization.

**Data Structures:**
Each Gaussian $i$ stores two learnable parameters for its normal:
1.  A scalar sign parameter $s_i \in \mathbb{R}$.
2.  A 3D direction vector $\mathbf{d}_i \in \mathbb{R}^3$.

**Steps:**
1.  Initialize $s_i$ and $\mathbf{d}_i$ for each Gaussian.
2.  During optimization, compute the normal $\mathbf{n}_i$ using the decomposition formula.
3.  Use $\mathbf{n}_i$ in the rendering pipeline (e.g., for shading or normal loss calculation).

**Formulas:**
The normal $\mathbf{n}_i$ is computed as:
$$
\mathbf{n}_i = \tanh(s_i) \cdot \frac{\mathbf{d}_i}{\|\mathbf{d}_i\|}
$$
Where:
*   $\tanh(s_i)$ maps the scalar $s_i$ to the range $(-1, 1)$, effectively acting as a learnable sign and magnitude controller.
*   $\frac{\mathbf{d}_i}{\|\mathbf{d}_i\|}$ normalizes the direction vector to unit length.
*   The resulting $\mathbf{n}_i$ has a norm strictly less than 1 (due to $\tanh$), but its direction is well-defined and continuous.

**Parameters:**
*   $s_i$: Learnable scalar. No specific initial value is given, but it must be initialized to allow both positive and negative values.
*   $\mathbf{d}_i$: Learnable 3D vector. Typically initialized randomly or based on prior geometry.

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\|\mathbf{d}_i\| = 0$, the normalization fails. In implementation, a small epsilon should be added to the denominator or $\mathbf{d}_i$ should be clamped [G].
*   **Local Minima:** Naive parameterization (simple normalized 3D vector) leads to unstable optimization because flipping the normal direction requires a 180-degree rotation [S1]. This decomposition avoids this by allowing "flipping" via the sign parameter $s_i$, which is a continuous scalar change rather than a vector rotation [S1].
*   **Gradient Stability:** The $\tanh$ function ensures gradients are bounded, preventing exploding gradients during backpropagation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal(s_i: float, d_i: list[float]) -> list[float]:
    """
    Compute the learnable normal for a Gaussian.
    
    Args:
        s_i: Learnable sign scalar.
        d_i: Learnable direction vector [x, y, z].
        
    Returns:
        Normal vector [nx, ny, nz].
    """
    # Compute magnitude of direction vector
    mag = sqrt(d_i[0]**2 + d_i[1]**2 + d_i[2]**2)
    
    # Guard against division by zero
    if mag < 1e-8:
        # Fallback to identity or zero; implementation detail [G]
        return [0.0, 0.0, 0.0]
    
    # Normalize direction
    d_norm = [d_i[0] / mag, d_i[1] / mag, d_i[2] / mag]
    
    # Compute sign factor
    sign_factor = tanh(s_i)
    
    # Compute final normal
    n_i = [sign_factor * d_norm[0], 
           sign_factor * d_norm[1], 
           sign_factor * d_norm[2]]
    
    return n_i
```

## Subtopics

### Quaternion-based Normal Enhancement
A technique to improve normal quality and stabilize vertex locations by training an additional learnable rotation parameter (quaternion) on each Gaussian on the mesh surface, instead of relying solely on mesh face normals [S2, unofficial]. It sets itself apart by explicitly learning a rotation rather than just a direction, which is crucial for non-flat surfaces where mesh face normals require a large number of triangles for accuracy [S2, unofficial]. Choose this when high-fidelity normal reconstruction on complex, non-flat geometry is required, and when occlusion errors from simple normal learning are observed [S2, unofficial]. Pitfall: Using mesh face normals directly requires a large number of triangles for non-flat surfaces; adjusting vertex positions alone to learn rotation is insufficient and causes occlusion errors [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per Gaussian for normal computation, as it involves basic arithmetic operations and a normalization.
*   **Memory Complexity:** Adds 4 floats (1 for $s_i$, 3 for $\mathbf{d}_i$) per Gaussian.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Stabilizes gradients, prevents convergence to local minima, allows easy normal flipping.
*   **Limitations:** Requires additional learnable parameters per Gaussian.
*   **When to use:** When optimizing Gaussian splats for surface reconstruction or normal rendering, especially where normal orientation is critical.
*   **When not to use:** When memory is extremely constrained and the overhead of 4 extra floats per Gaussian is prohibitive, or when normal orientation is not important.

## Background
Originated in the context of Gaussian Splatting for surface reconstruction, specifically in the work "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1]. It addresses the limitation of standard Gaussian Splatting, which does not natively contain surface normal information, by providing a robust way to learn and represent normals [S1].

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
