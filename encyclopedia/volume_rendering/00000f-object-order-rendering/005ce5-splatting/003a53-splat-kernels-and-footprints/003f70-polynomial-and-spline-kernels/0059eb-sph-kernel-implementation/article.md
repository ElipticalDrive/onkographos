# SPH Kernel Implementation

Also called: SPH Kernels

## Overview
The SPH Kernel Implementation provides a polymorphic interface for evaluating Smoothed Particle Hydrodynamics (SPH) kernels, supporting both CPU and GPU execution for density calculation and voxelization [S1, unofficial]. It is distinguished from other splatting kernels by its specific focus on particle-based fluid simulation data structures, utilizing closed-form expressions for kernel values and derivatives in 3D [S1, unofficial]. Unlike general-purpose splatting kernels that may assume isotropic Gaussian or B-spline footprints, this implementation supports specific SPH families (B-splines and Wendland functions) and includes an anisotropic variant for fluid surface rendering [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Particle position $\mathbf{x}_i$, smoothing length $h$, particle mass $m$, and kernel type identifier. For anisotropic rendering, the anisotropy matrix $\mathbf{H}$ is also required [S1, unofficial][S2].
*   **Output:** Kernel value $W(\mathbf{x})$ and its spatial derivative $\nabla W(\mathbf{x})$ at a query point (e.g., voxel center) [S1, unofficial].

**Data Structures:**
The implementation uses a polymorphic interface class structure. Concrete implementations store the kernel parameters (e.g., support radius $R$) and provide methods for evaluation. The voxelization process relies on a kd-tree for neighbor search [S1, unofficial].

**Supported Kernels:**
The module implements the following kernels, each with closed-form 3D expressions for value and derivative [S1, unofficial]:
1.  Cubic B-spline
2.  Quintic B-spline
3.  Wendland C2
4.  Wendland C4
5.  Wendland C6
6.  Wendland C8

**Algorithm Steps:**
1.  **Distance Calculation:** Compute the Euclidean distance $r = \|\mathbf{x} - \mathbf{x}_i\|$ or the Mahalanobis distance $q = \| \mathbf{H} \cdot (\mathbf{x} - \mathbf{x}_i) \| / h$ for anisotropic cases [S1, unofficial][S2].
2.  **Normalization Check:** If $r > h$ (or $q > 1$ for normalized anisotropic forms), return 0.
3.  **Kernel Evaluation:** Evaluate the specific polynomial piecewise function based on the selected kernel type.
4.  **Derivative Calculation:** Compute the gradient $\nabla W$ using the closed-form derivative expression for the selected kernel [S1, unofficial].
5.  **Density Bias Correction:** Apply density bias correction based on particle mass and smoothing length if computing density [S1, unofficial].

**Formulas:**
For the standard isotropic SPH rendering kernel (commonly used for fluid surfaces), the formulation is [S2]:
$$ \hat{W}(q) = (1 - q^2)^3_+ $$
where $q = \frac{\|\mathbf{x} - \mathbf{x}_i\|}{h}$ and $(\cdot)_+ = \max(\cdot, 0)$. The normalization constant is $C_3 = \frac{64}{64\pi}$ (as cited in source text, though standard normalization for this kernel form typically implies a different constant; the source explicitly states $C_3 = \frac{64}{64\pi}$) [S2].

For the anisotropic symmetric formulation, the kernel value is computed as [S2]:
$$ \hat{W}(\|H \cdot x_{ij}\|) = \frac{1}{2} \left( \hat{W}(\|H_i \cdot x_{ij}\|) + \hat{W}(\|H_j \cdot x_{ij}\|) \right) $$
where $H$ is the anisotropy matrix, $x_{ij} = \mathbf{x}_i - \mathbf{x}_j$, and $|H|$ is the determinant of $H$ [S2].

**Parameters:**
*   **Smoothing Length ($h$):** For isotropic formulation, $h_i = \sqrt[3]{\frac{3 N_h V_i}{4\pi}}$ where $V_i$ is the rest volume and $N_h = 50$ for the cubic spline kernel [S2].
*   **Kernel Type:** Enumerated type selecting between B-spline and Wendland variants [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_sph_kernel(position_query, particle_pos, smoothing_length, kernel_type, anisotropy_matrix=None):
    """
    Evaluates the SPH kernel value and derivative.
    
    Args:
        position_query: Vector3, query point (e.g., voxel center).
        particle_pos: Vector3, particle center.
        smoothing_length: Float, h.
        kernel_type: Enum, e.g., 'CUBIC_BSPLINE', 'WENDLAND_C2'.
        anisotropy_matrix: Matrix3x3, optional. If provided, uses anisotropic formulation.
        
    Returns:
        Tuple (value, gradient)
    """
    # 1. Calculate distance metric
    if anisotropy_matrix is not None:
        # Anisotropic case: Mahalanobis distance
        diff = position_query - particle_pos
        # Note: Source [S2] uses H * x_ij. Assuming H is applied to the vector.
        # For symmetric formulation, this is handled at a higher level (see Anisotropic SPH Kernel).
        # Here we assume a single-particle evaluation with its own H.
        transformed_diff = anisotropy_matrix @ diff
        q = transformed_diff.norm() / smoothing_length
    else:
        # Isotropic case: Euclidean distance
        diff = position_query - particle_pos
        r = diff.norm()
        q = r / smoothing_length

    # 2. Compact support check
    if q > 1.0:
        return (0.0, Vector3(0, 0, 0))

    # 3. Evaluate kernel value and derivative based on type
    # Closed-form expressions are specific to each kernel.
    # Example for Rendering Kernel (1-q^2)^3_+
    if kernel_type == 'RENDERING_KERNEL':
        # Value
        val = (1.0 - q**2)**3
        # Derivative dW/dr = dW/dq * dq/dr
        # dW/dq = 3*(1-q^2)^2 * (-2q) = -6q*(1-q^2)^2
        # dq/dr = 1/h
        dW_dq = -6.0 * q * (1.0 - q**2)**2
        grad = (dW_dq / smoothing_length) * (diff / r) if r > 0 else Vector3(0,0,0)
        return (val, grad)
    
    # Other kernels (Cubic B-spline, Wendland C2, etc.) would have their own
    # closed-form branches here. [S1, unofficial]
    else:
        # Placeholder for other closed-form implementations [G]
        # Implement specific polynomial pieces for Cubic B-spline, Quintic B-spline,
        # Wendland C2, C4, C6, C8 as defined in [S1, unofficial].
        pass
```

## Subtopics

### Anisotropic SPH Kernel
An extension of the standard SPH kernel that uses an anisotropy matrix $\mathbf{H}$ to create an ellipsoidal support domain, addressing issues with fluid surface rendering where isotropic support leads to artifacts [S2]. It employs a symmetric formulation averaging the kernel evaluation of two interacting particles to ensure consistency [S2]. It is used specifically for fluid surface rendering, while the simulation itself may remain isotropic [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per kernel evaluation. Neighbor search complexity depends on the underlying data structure (kd-tree), typically $O(\log N)$ for search [S1, unofficial].
*   **Memory Complexity:** Low, as kernels are stateless functions. The kd-tree requires $O(N)$ space [S1, unofficial].
*   **Strengths:** Closed-form derivatives allow for efficient gradient-based operations (e.g., in density calculation or voxelization). Support for multiple kernel types (B-spline, Wendland) provides flexibility for different simulation requirements [S1, unofficial].
*   **Limitations:** Anisotropic kernels require non-cubic cells in spatial data structures, complicating neighborhood queries [S2]. The source notes a potential ambiguity in the normalization constant for the rendering kernel ($C_3 = \frac{64}{64\pi}$) [S2].
*   **When to use:** When visualizing SPH simulation data, particularly fluid surfaces, where accurate density estimation or smooth voxelization is required [S1, unofficial][S2].

## Background
The implementation is based on the C++ module defined in [Bös25], providing a polymorphic interface for SPH kernels [S1, unofficial]. The voxelization algorithm extends the method described in [Pri07] to support 3D voxels [S1, unofficial]. The anisotropic formulation for fluid surface rendering is based on [OVSM98] and [YT13], with symmetric gathering/scattering derived from [HK89] [S2].

## Sources
[S1] Visualizing the Cosmos: Bridging Astrophysical Simulations with Volume Rendering, https://doi.org/10.2312/visgap.20261003, fetched 2026-10-26 (unofficial)
[S2] Multi‐Level Memory Structures for Simulating and Rendering Smoothed Particle Hydrodynamics, https://doi.org/10.1111/cgf.14090, fetched 2026-10-28
