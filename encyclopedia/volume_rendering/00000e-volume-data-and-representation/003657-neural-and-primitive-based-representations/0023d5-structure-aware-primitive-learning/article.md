# Structure-aware primitive learning


## Overview
Structure-aware primitive learning represents volumetric data using a sparse set of parameterized ellipsoids, optimizing for both geometric structure and appearance [S1, unofficial]. It solves the problem of efficiently reconstructing large-scale volume data by combining structural similarity losses with adaptive primitive management, distinguishing itself from dense voxel grids and standard Gaussian splatting by explicitly modeling anisotropic ellipsoidal shapes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of ground-truth subvolumes $V_{gt}$ from a larger volume dataset [S1, unofficial].
*   **Output:** A set of ellipsoid primitives $E$, where each primitive $j$ is defined by mean $\boldsymbol{\mu}_j$, covariance $\boldsymbol{\Sigma}_j$, and intensity $I_j$ [S1, unofficial].

**Data Structures:**
*   **Primitive $E_j$:** A tuple $(\boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j, I_j)$ [S1, unofficial].
    *   $\boldsymbol{\mu}_j \in \mathbb{R}^3$: Center position [S1, unofficial].
    *   $\boldsymbol{\Sigma}_j \in \mathbb{R}^{3 \times 3}$: Covariance matrix defining shape and orientation [S1, unofficial].
    *   $I_j \in \mathbb{R}$: Scalar intensity value [S1, unofficial].
*   **Subvolume $V_{gt}$:** A 3D grid of intensity values [S1, unofficial].

**Parameters:**
*   $\beta$: Reconstruction loss weight for L1 term [S1, unofficial].
*   $d$: Densification iteration count [S1, unofficial].
*   $p$: Pruning iteration count [S1, unofficial].

**Algorithm:**
The method operates in epochs, traversing subvolumes sequentially [S1, unofficial].
1.  **Fetch Subvolume:** Retrieve the next ground-truth subvolume $V_{gt}$ [S1, unofficial].
2.  **Retrieve Overlaps:** Identify primitives overlapping the current subvolume: $V' \leftarrow \text{GetOverlappingEllipsoids}(E, V_{gt})$ [S1, unofficial].
3.  **Voxelize:** Render the overlapping primitives into a predicted subvolume $\hat{V}$: $\hat{V} \leftarrow \text{Voxelize}(E)$ [S1, unofficial].
4.  **Compute Loss:** Calculate reconstruction loss using a combination of L1 and DSSIM:
    $$ L_{rec} = \beta L_1(\hat{V}, V_{gt}) + (1 - \beta) L_{DSSIM}(\hat{V}, V_{gt}) $$
    where $L_1$ is mean absolute error and $L_{DSSIM}$ is structural similarity loss [S1, unofficial].
5.  **Update:** Optimize primitive parameters $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, I)$ using the Adam optimizer on $\nabla L_{rec}$ [S1, unofficial].
6.  **Compute Metrics:** Compute fractional anisotropy from covariance: $FA_j \leftarrow \text{ComputeFA}(\boldsymbol{\Sigma}_j)$ [S1, unofficial].
7.  **Prune:** Remove primitives with low importance scores: $\text{RemoveLowScore}(E)$ [S1, unofficial].
8.  **Convergence Check:** Repeat until convergence or iteration limits $d, p$ are reached [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def structure_aware_primitive_learning(volume_data, beta=0.5):
    # [G] Initialization strategy not given in sources.
    E = initialize_ellipsoids()
    converged = False
    i = 0
    
    while not converged:
        # [G] Returns the next sequential ground-truth subvolume from the dataset.
        V_gt = NextSubvolume(volume_data)
        
        # [G] Returns the subset of ellipsoids in E whose bounding volumes intersect V_gt.
        V_prime = GetOverlappingEllipsoids(E, V_gt)
        
        # [G] Renders the ellipsoids into a 3D grid by accumulating their contributions at each voxel.
        V_hat = Voxelize(E)
        
        # Standard: np.mean(np.abs(V_hat - V_gt))
        L1_loss = MeanAbsoluteError(V_hat, V_gt)
        # Standard: computes SSIM between V_hat and V_gt
        DSSIM_loss = StructuralSimilarityIndex(V_hat, V_gt)
        L_rec = beta * L1_loss + (1 - beta) * DSSIM_loss
        
        # Standard: torch.autograd.grad(L_rec, E, retain_graph=True)
        gradients = ComputeGradients(L_rec, E)
        # Standard: applies Adam optimizer step to E using gradients
        E = AdamUpdate(E, gradients)
        
        # [G] Computes fractional anisotropy from covariance matrix.
        FA_j = ComputeFA(E.covariance)
        
        # [G] Removes ellipsoids whose importance score falls below a threshold.
        E = RemoveLowScore(E)
        
        # [G] Returns True if loss change or iteration count exceeds limits.
        converged = CheckConvergence(E, V_gt)
        i += 1
        
    return E
```

**Edge Cases:**
*   **Precision:** Covariance matrix $\boldsymbol{\Sigma}$ must remain positive semi-definite during optimization [G].
*   **Boundaries:** Not given in the sources.

## Subtopics

### [Volume Exploration-Optimized INR](6823-volume-exploration-optimized-inr.md)
Dual-pathway INR for scientific data, using hash grids and local structural encoders to capture spatial coherence. It sets itself apart from standard INRs by fusing multi-resolution positional encoding with local structural context via Feature-wise Linear Modulation. Choose it when image-based transfer function design requires local structural context.

### [Mixture-of-Experts Neural Fields](10460-mixture-of-experts-neural-fields.md)
Partitions spatial domain into subdomains with specialized expert decoders to model complex datasets while maintaining boundary continuity. It distinguishes itself from independent per-block INRs by guaranteeing continuity across subdomain boundaries through a shared encoder. Choose it when modeling large-scale scientific datasets with heterogeneous features.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Captures anisotropic structures better than isotropic primitives [S1, unofficial].
*   **Limitations:** Requires careful tuning of pruning and densification parameters [S1, unofficial].

## Background
Originated in the paper "ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling" [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
