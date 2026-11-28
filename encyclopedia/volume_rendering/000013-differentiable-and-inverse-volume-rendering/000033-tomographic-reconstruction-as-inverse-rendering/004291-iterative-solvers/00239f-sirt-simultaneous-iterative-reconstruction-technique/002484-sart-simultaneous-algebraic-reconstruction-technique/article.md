# SART (Simultaneous Algebraic Reconstruction Technique)

Also called: Simultaneous Algebraic Reconstruction Technique, Proximal SART, ProxiSART

## Overview
SART is an iterative algebraic reconstruction technique that solves the discrete linear system $Af = b$ arising from tomographic projection data by updating the volume estimate simultaneously for all rays in a subset of projections [S2, unofficial]. It improves upon single-ray methods by normalizing the backprojection by the number of rays intersecting each voxel, which reduces streak artifacts and accelerates convergence [S2, unofficial]. It is preferred over direct methods like FDK for limited projection scenarios and over Conjugate Gradient (CG) for sparse-view settings due to better convergence properties in matrix-free implementations [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Measured projection data $b \in \mathbb{R}^m$, system matrix $A \in \mathbb{R}^{m \times n}$, initial volume estimate $f^{(0)} \in \mathbb{R}^n$, relaxation parameter $\alpha$, and number of outer iterations $T$ [S2, unofficial].
*   **Output:** Reconstructed volume $f \in \mathbb{R}^n$ [S2, unofficial].

**Data Structures:**
*   $f$: 3D volume array of size $n$ (voxels) [S2, unofficial].
*   $b$: 1D array of size $m$ (projection rays) [S2, unofficial].
*   $w_i$: 1D array of size $m$, where $w_i = \sum_k a_{ik}$ is the sum of weights for ray $i$ (ray sensitivity) [S2, unofficial].
*   $v_j$: 1D array of size $n$, where $v_j = \sum_{i \in \text{rays through } j} a_{ij}$ is the sum of weights for voxel $j$ (voxel sensitivity) [S2, unofficial].

**Algorithm Steps:**
The standard SART algorithm updates the volume by iterating through subsets of projections (often all projections or a subset $S$) [S2, unofficial]. For each subset, it performs three steps for each ray $i$:

1.  **Forward Projection:** Compute the estimated projection $\hat{b}_i^{(t)}$ for all rays $i$ in the current subset $S$ using the current volume estimate $f^{(t)}$:
    $$ \hat{b}_i^{(t)} = \sum_{k=1}^{n} a_{ik} f_k^{(t)} $$
    where $a_{ik}$ is the contribution of voxel $k$ to ray $i$ [S2, unofficial].

2.  **Correction:** Compute the correction factor $c_i^{(t)}$ for each ray $i \in S$:
    $$ c_i^{(t)} = \frac{b_i - \hat{b}_i^{(t)}}{\sum_{k=1}^{n} a_{ik}} $$
    The denominator $\sum_{k} a_{ik}$ is the total weight of the ray (sum of contributions from all voxels it passes through) [S2, unofficial]. This normalizes the residual by the ray's path length/weight [S2, unofficial].

3.  **Backprojection:** Update the volume $f$ for all voxels $j=1 \dots n$:
    $$ f_j^{(t+1)} = f_j^{(t)} + \alpha \frac{\sum_{i \in S} c_i^{(t)} a_{ij}}{\sum_{i \in S} a_{ij}} $$
    The denominator $\sum_{i \in S} a_{ij}$ is the sum of weights of the rays in subset $S$ that intersect voxel $j$ [S2, unofficial]. This normalizes the backprojected correction by the number of rays contributing to the voxel [S2, unofficial].

**Parameters:**
*   $\alpha$: Relaxation parameter. Typical values range from 0.1 to 0.8 depending on the dataset [S2, unofficial]. In proximal Chambolle-Pock implementations, $\alpha$ is set to 0.5 [S1, unofficial].
*   $T$: Number of outer iterations [S2, unofficial].
*   $S$: Subset of projections. Often $S$ is the set of all projections [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zeros(n):
    """Return a list of n zeros."""
    return [0.0] * n

def ones(n):
    """Return a list of n ones."""
    return [1.0] * n

def sart_standard(b, A, n_voxels, n_rays, alpha, T, subset_indices=None):
    """
    Standard SART reconstruction.
    
    Args:
        b: array of measured projections (size m)
        A: 2D list/matrix of size m x n (system matrix)
        n_voxels: number of voxels
        n_rays: number of rays
        alpha: relaxation parameter
        T: number of iterations
        subset_indices: list of ray indices for the subset S. If None, use all rays.
        
    Returns:
        f: reconstructed volume
    """
    # Initialize volume f^(0) = 0
    f = zeros(n_voxels)
    
    # Precompute ray weights w_i = sum_k a_ik
    ray_weights = zeros(n_rays)
    for i in range(n_rays):
        for k in range(n_voxels):
            ray_weights[i] += A[i][k]

    if subset_indices is None:
        subset_indices = list(range(n_rays))
        
    for t in range(T):
        # Step 1: Forward projection for subset S
        b_hat = zeros(len(subset_indices))
        for idx, i in enumerate(subset_indices):
            b_hat[idx] = 0.0
            for k in range(n_voxels):
                b_hat[idx] += A[i][k] * f[k]

        # Step 2: Correction for subset S
        corrections = zeros(len(subset_indices))
        for idx, i in enumerate(subset_indices):
            if ray_weights[i] > 0:
                corrections[idx] = (b[i] - b_hat[idx]) / ray_weights[i]
            else:
                corrections[idx] = 0.0

        # Step 3: Backprojection and Update
        # Compute voxel sensitivities for subset S: v_j = sum_{i in S} a_ij
        voxel_sens = zeros(n_voxels)
        for idx, i in enumerate(subset_indices):
            for j in range(n_voxels):
                voxel_sens[j] += A[i][j]

        # Compute backprojected correction: sum_{i in S} c_i * a_ij
        backprojected = zeros(n_voxels)
        for idx, i in enumerate(subset_indices):
            for j in range(n_voxels):
                backprojected[j] += corrections[idx] * A[i][j]

        # Update volume
        for j in range(n_voxels):
            if voxel_sens[j] > 0:
                f[j] = f[j] + alpha * (backprojected[j] / voxel_sens[j])

    return f
```

**Proximal SART (for Data Term in Optimization):**
When used as a proximal operator for the data fidelity term $f(f) = \|Af - p\|_2^2$ within a Chambolle-Pock scheme, the problem is transformed into an under-determined linear system by introducing auxiliary variables $y$ and $z$ [S2, unofficial]:
$$ \text{prox}_{\lambda_f}(q) = \text{argmin}_f \|Af - p\|_2^2 + \frac{1}{2\lambda}\|f - q\|_2^2 $$
This is equivalent to solving:
$$ \min_{\tilde{f}} \|\tilde{f}\|_2^2 \quad \text{subject to: } \tilde{A}\tilde{f} = \tilde{p} $$
where $\tilde{f} = [y; z]$, $\tilde{A} = [I \ \sqrt{2\lambda}A]$, and $\tilde{p} = \sqrt{2\lambda}(p - Aq)$ [S2, unofficial].
The modified SART algorithm for this proximal operator is [S2, unofficial]:

1.  **Initialization:** Set $y^{(0)} = 0$, $f^{(0)} = q$. Scale inputs: $\tilde{p} = \sqrt{2\lambda}p$, $\tilde{A} = \sqrt{2\lambda}A$ [S2, unofficial].
2.  **Outer Loop:** Iterate $t$ from 1 to $T$ [S2, unofficial].
3.  **Inner Loop:** Iterate over projections $S$ in $S_1 \dots S_N$ [S2, unofficial].
4.  **Update:**
    *   Update $y$ for indices $j$ in current projection set $S$:
        $$ y_j^{(t+1)} = y_j^{(t)} + \alpha c_j^{(t)} $$ [S2, unofficial].
    *   Compute predicted projection sum:
        $$ \hat{b}_i^{(t+1)} = \sum_k a_{ik} f_k^{(t)} + y_i^{(t)} $$ [S2, unofficial].
    *   Compute correction factor:
        $$ c_i^{(t+1)} = \frac{b_i - \hat{b}_i^{(t+1)}}{\sum_k a_{ik} + 1} $$ [S2, unofficial].
    *   Update volume $f$ for all voxels $j=1 \dots n$:
        $$ f_j^{(t+1)} = f_j^{(t)} + \alpha \frac{\sum_{i \in S} c_i^{(t+1)} a_{ij}}{\sum_{i \in S} a_{ij}} $$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** In proximal SART implementations involving patch-based Jacobians, memory usage can be high. For a volume of size $N$, storage can be 82 times the volume size (81 for Jacobian + 1 for volume) [S2, unofficial]. For a $690 \times 668 \times 776$ voxel volume, this requires 109 GB [S2, unofficial].
*   **Convergence:** SART is preferred over CG because CG suffers from numerical noise in matrix-free tomography and convergence issues with large sparse systems [S2, unofficial].

## Subtopics

### [Voxel Splatting SART/SIRT](15304-voxel-splatting-sart-sirt.md)
A GPU-accelerated implementation that treats forward/back-projection as splatting kernels, inverting the data flow to project voxels onto the detector. It is distinguished by its efficiency in parallel ray geometry (e.g., cryo-ET) and use of pre-integrated lookup tables.

### [Proximal Jointly-Optimized Reconstruction and Denoising](15324-proximal-jointly-optimized-reconstruction-and-denoising.md)
A method that integrates denoising into reconstruction using SART as the solver for the data fidelity term. It is chosen over CG for sparse-view settings to maintain consistency with raw projections.

## Operating characteristics
*   **Time Complexity:** Each iteration involves one forward projection and one backprojection. The cost is proportional to the number of voxels and rays [S2, unofficial].
*   **Memory Complexity:** Modest memory requirements compared to direct methods that require storing the full system matrix $A$ ($O(mn)$) [S2, unofficial]. SART is matrix-free, requiring only storage for the volume and projections [S2, unofficial]. However, proximal variants may require additional storage for auxiliary variables and Jacobians [S2, unofficial].
*   **Measured Performance:** On a machine with 24 cores and 48 GB RAM, SART with Kaiser-Bessel kernels (radius 2 voxels) and slice-by-slice optimization was used for reconstruction [S1, unofficial]. For sparse-view settings, SART provides better reconstruction results than FDK and CG [S2, unofficial].
*   **Strengths:** Handles limited projections better than FDK [S2, unofficial]. Incorporates prior information (e.g., TV priors) easily [S2, unofficial]. Modest memory usage [S2, unofficial].
*   **Limitations:** Slower than FDK for full-view data. Convergence can be slow for high-accuracy requirements [S2, unofficial].
*   **When to Use:** Sparse-view tomography, limited-angle tomography, or when incorporating regularization priors [S2, unofficial].
*   **When Not to Use:** Full-view, high-quality data where speed is critical (use FDK). Large-scale problems where memory for auxiliary variables in proximal methods is prohibitive [S2, unofficial].

## Background
SART was proposed by Andersen and Kak as an efficient iterative reconstruction method based on the Kaczmarz projection method [S2, unofficial]. It updates the volume for each scan view, offering better results than FDK for limited projections [S2, unofficial]. It has been widely used in computed tomography, including industrial CT and medical imaging [S1, unofficial][S2, unofficial]. Recent developments include its use as a proximal operator in Chambolle-Pock schemes for robust 3D cone beam reconstruction [S1, unofficial] and in space-time tomographic reconstruction of deforming objects [S2, unofficial].

## Sources
[S1] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
[S2] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
