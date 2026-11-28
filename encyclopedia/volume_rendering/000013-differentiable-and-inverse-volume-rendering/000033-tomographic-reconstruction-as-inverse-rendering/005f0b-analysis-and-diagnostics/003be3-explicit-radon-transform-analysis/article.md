# Explicit Radon Transform Analysis

Also called: Radon Matrix Analysis, Sparse-View Radon Transform Null Space Analysis

## Overview
Explicit Radon Transform Analysis constructs and analyzes the explicit forward projection matrix $A$ (or operator $R_V$) for a given scanning geometry to diagnose the invertibility and stability of tomographic reconstruction [S1, S2, unofficial]. Unlike differentiable rendering which relies on gradient descent, this method explicitly exposes the linear algebra structure of the forward operator, allowing for the computation of singular values and the identification of the null space [S1, S2, unofficial].

## How it works
**Inputs:**
- Scanning geometry parameters: number of angles $n_{angles}$, maximum angle range $\theta_{max}$, number of samples per ray $n_{samples}$, and detector dimensions $v_{large}, h_{large}$ [S2, unofficial].
- Volume dimensions: $v_{small}, h_{small}$ (or 3D extent $w$) [S1, S2, unofficial].
- Optional: `sparse_svd` flag to determine if full or sparse SVD is computed [S2, unofficial].

**Outputs:**
- The explicit Radon transform matrix $A$ (size $M \times N$, where $M$ is total detector pixels and $N$ is total volume voxels) [S2, unofficial].
- Singular values $W$ of the matrix $A$ [S2, unofficial].

**Data Structures:**
- Matrix $A$: A sparse matrix where $A_{ij}$ represents the contribution of voxel $j$ to detector pixel $i$ [S2, unofficial].
- Singular Values $W$: A vector of length $\min(M, N)$ representing the decay of the operator's energy [S2, unofficial].

**Steps:**
1. **Geometry Setup:** Define the volume geometry (`vol_geom`) and projection geometry (`proj_geom`) based on input parameters $n_{angles}, \theta_{max}, n_{samples}, v_{large}, h_{large}$ [S2, unofficial].
2. **Matrix Construction:** Call the explicit Radon transform function `explicit_radon` with parameters `n_angles`, `max_angle`, `n_samples`, `v_small`, `h_small`, and `sparse_svd=False` to generate the matrix $A$ and its singular values $W$ [S2, unofficial].
3. **Spectral Analysis:** Compute the Singular Value Decomposition (SVD) of $A$ to obtain $W$ [S2, unofficial].
4. **Null Space Identification:** Identify the null space $\mathcal{N}(R_V)$ defined as:
   $$
   \mathcal{N}(R_V) = \{ \delta V \mid R_V(\delta V) = 0 \}
   $$
   where $R_V$ is the sparse-view Radon operator [S1, unofficial]. Vectors in this null space represent volumetric perturbations invisible to the projections [S1, unofficial].
5. **Visualization:** Use `imshow` to visualize the matrix $A$ to observe the sparsity pattern corresponding to the scanning geometry [S2, unofficial].

**Formulas:**
The sparse-view Radon transform is defined as the concatenation of Radon operators along $K$ projection directions $V = \{v_1, \dots, v_K\}$ [S1, unofficial]:
$$
R_V(V) = (R_{v_1}(V), \dots, R_{v_K}(V)) \in \mathbb{R}^{K \times h \times w}
$$
where $V$ is the attenuation field, $R_v$ is the line-integral operator along direction $v$, and $K$ is the number of views [S1, unofficial].

For dense views ($K \to \infty$), $R$ is injective [S1, unofficial]. For sparse views (finite $K$, e.g., $K \in \{10, 20, 25\}$), $R_V$ is strongly underdetermined and $\mathcal{N}(R_V)$ is non-trivial [S1, unofficial].

**Parameters:**
- `n_angles`: Number of projection angles. Typical values: 10, 20, 25, 45 [S1, S2, unofficial].
- `max_angle`: Maximum angular range. Typical values: $\pi$ (sparse, full rotation) or limited ranges (limited angle) [S2, unofficial].
- `n_samples`: Number of samples per ray [S2, unofficial].
- `v_small, h_small`: Volume dimensions [S2, unofficial].
- `v_large, h_large`: Detector dimensions [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Underdetermined Systems:** In sparse-view or limited-angle scenarios, the operator is strongly underdetermined with a non-trivial null space, meaning projection consistency alone cannot uniquely determine the density field [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import matplotlib.pyplot as plt

def explicit_radon_analysis(n_angles, max_angle, n_samples, v_small, h_small, v_large, h_large, sparse_svd=False):
    """
    Constructs the explicit Radon matrix and analyzes its properties.
    """
    # 1. Setup Geometry
    vol_geom = {'voxel_size': (1.0, 1.0, 1.0), 'volume_size': (v_small, h_small, 1), 'origin': (0, 0, 0)}
    proj_geom = {'detector_size': (h_large, v_large), 'num_pixels': (h_large, v_large),
                 'num_angles': n_angles, 'angle_range': max_angle, 'ray_spacing': 1.0}

    # 2. Matrix Construction
    # [G] The source specifies calling `explicit_radon` with these parameters to obtain A and W.
    # Standard implementation initializes a sparse matrix A of shape (M, N) and populates it
    # via ray-tracing through the volume geometry.
    M = proj_geom['num_pixels'][0] * proj_geom['num_pixels'][1]
    N = vol_geom['volume_size'][0] * vol_geom['volume_size'][1] * vol_geom['volume_size'][2]
    A = np.zeros((M, N))
    # [G] Populate A with ray-voxel intersection weights.

    # 3. Spectral Analysis
    if sparse_svd:
        # [G] Use scipy.sparse.linalg.svds for large matrices
        U, S, Vt = np.linalg.svd(A, full_matrices=False)
        W = S
    else:
        U, S, Vt = np.linalg.svd(A, full_matrices=False)
        W = S

    # 4. Null Space Identification
    null_space_threshold = 1e-6
    null_space_vectors = Vt[W < null_space_threshold, :]

    # 5. Visualization
    plt.figure()
    plt.imshow(A.reshape(proj_geom['num_pixels'][0], -1), cmap='hot')
    plt.title(f"Radon Matrix Sparsity (n={n_angles}, max_angle={max_angle})")
    plt.xlabel("Voxel Index")
    plt.ylabel("Detector Pixel Index")
    plt.show()

    return A, W, null_space_vectors
```

## Subtopics

### Sparse-View Analysis
Analyzes the operator when $K$ is small (e.g., 10-25 views) but the angular range is full ($\pi$) [S1, unofficial]. Sets it apart by isolating the effects of angular undersampling. Chosen when diagnosing missing angular data effects.

### Limited-Angle Analysis
Analyzes the operator when the angular range is restricted (e.g., $\theta_{max} < \pi$) [S2, unofficial]. Sets it apart by isolating the effects of geometric constraints. Chosen when diagnosing missing geometric data effects.

### Comparison
Compares the singular value decay for sparse angles (fixed max angle $\pi$, varying $K$) vs. limited angles (fixed $K$, varying max angle) [S2, unofficial].

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Strengths:** Provides exact diagnostic information about the forward operator's invertibility and null space [S1, unofficial].
- **Limitations:** Does not provide a reconstruction itself, only analysis [S1, S2, unofficial].

## Background
Originating from the mathematical analysis of the Radon transform in computed tomography, this method was applied to 3D Gaussian tomography to diagnose optimization drift and null-space ambiguities in sparse-view settings [S1, unofficial]. The explicit matrix construction and SVD analysis are implemented in inverse problems toolboxes like the ASTRA Toolbox [S2, unofficial].

## Sources
[S1] Projection-Volume Fidelity Divergence: Diagnosing and Controlling Optimization Drift in Sparse-View 3D Gaussian Tomography, https://doi.org/10.48550/arXiv.2606.22525, fetched 2026-10-27 (unofficial)
[S2] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
