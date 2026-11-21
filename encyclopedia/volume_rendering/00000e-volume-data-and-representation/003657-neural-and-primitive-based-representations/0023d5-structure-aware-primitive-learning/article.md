# Structure-aware primitive learning

Also called: ESVR (Ellipsoid-based Sparse Volume Rendering)

## Overview
Structure-aware primitive learning represents volumetric data using a sparse set of parameterized ellipsoids, optimizing for both geometric structure and appearance [S1, unofficial]. It solves the problem of efficiently reconstructing large-scale volume data by combining structural similarity losses with adaptive primitive management, distinguishing itself from dense voxel grids and standard Gaussian splatting by explicitly modeling anisotropic ellipsoidal shapes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of ground-truth subvolumes $V_{gt}$ from a larger volume dataset.
*   **Output:** A set of ellipsoid primitives $E$, where each primitive $j$ is defined by mean $\boldsymbol{\mu}_j$, covariance $\boldsymbol{\Sigma}_j$, and intensity $I_j$.

**Data Structures:**
*   **Primitive $E_j$:** A tuple $(\boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j, I_j)$.
    *   $\boldsymbol{\mu}_j \in \mathbb{R}^3$: Center position.
    *   $\boldsymbol{\Sigma}_j \in \mathbb{R}^{3 \times 3}$: Covariance matrix defining shape and orientation.
    *   $I_j \in \mathbb{R}$: Scalar intensity value.
*   **Subvolume $V_{gt}$:** A 3D grid of intensity values.

**Parameters:**
*   $\beta$: Reconstruction loss weight for L1 term. Typical value not given in sources.
*   $d$: Densification iteration count. Not given in sources.
*   $p$: Pruning iteration count. Not given in sources.

**Algorithm:**
The method operates in epochs, traversing subvolumes sequentially.

1.  **Fetch Subvolume:** Retrieve the next ground-truth subvolume $V_{gt}$ [S1, unofficial].
2.  **Retrieve Overlaps:** Identify primitives overlapping the current subvolume: $V' \leftarrow \text{GetOverlappingEllipsoids}(E, V_{gt})$ [S1, unofficial].
3.  **Voxelize:** Render the overlapping primitives into a predicted subvolume $\hat{V}$: $\hat{V} \leftarrow \text{Voxelize}(E)$ [S1, unofficial].
4.  **Compute Loss:** Calculate reconstruction loss using a combination of L1 and DSSIM:
    $$ L_{rec} = \beta L_1(\hat{V}, V_{gt}) + (1 - \beta) L_{DSSIM}(\hat{V}, V_{gt}) $$
    where $L_1$ is mean absolute error and $L_{DSSIM}$ is structural similarity loss [S1, unofficial].
5.  **Update:** Optimize primitive parameters $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, I)$ using the Adam optimizer on $\nabla L_{rec}$ [S1, unofficial].
6.  **Prune:** Remove primitives with low importance scores: $\text{RemoveLowScore}(E)$ [S1, unofficial].
7.  **Convergence Check:** Repeat until convergence or iteration limits $d, p$ are reached [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def structure_aware_primitive_learning(volume_data, beta=0.5):
    E = initialize_ellipsoids() # [G] Initialization strategy not specified
    converged = False
    
    while not converged:
        # 1. Fetch next subvolume
        V_gt = NextSubvolume(volume_data)
        
        # 2. Retrieve overlapping primitives
        V_prime = GetOverlappingEllipsoids(E, V_gt)
        
        # 3. Voxelize to predict subvolume
        V_hat = Voxelize(E)
        
        # 4. Compute loss
        L1_loss = MeanAbsoluteError(V_hat, V_gt)
        DSSIM_loss = StructuralSimilarityIndex(V_hat, V_gt)
        L_rec = beta * L1_loss + (1 - beta) * DSSIM_loss
        
        # 5. Update parameters
        gradients = ComputeGradients(L_rec, E)
        E = AdamUpdate(E, gradients)
        
        # 6. Prune low-importance primitives
        E = RemoveLowScore(E)
        
        # 7. Check convergence
        converged = CheckConvergence(E, V_gt)
        
    return E
```

**Edge Cases:**
*   **Precision:** Covariance matrix $\boldsymbol{\Sigma}$ must remain positive semi-definite during optimization [G].
*   **Boundaries:** Subvolume boundaries require careful handling of primitives that partially overlap [S1, unofficial].

## Subtopics

### [Volume Exploration-Optimized INR](6823-volume-exploration-optimized-inr.md)
Dual-pathway INR for scientific data, using hash grids and local structural encoders.

### [Mixture-of-Experts Neural Fields](10460-mixture-of-experts-neural-fields.md)
Partitions spatial domain into subdomains with specialized expert decoders.

## Operating characteristics
*   **Complexity:** Time complexity depends on the number of overlapping primitives per subvolume and the voxelization cost. Memory usage is sparse, proportional to the number of ellipsoids rather than volume resolution [S1, unofficial].
*   **Performance:** Efficient for large-scale volumes due to sparsity. Performance scales with the number of primitives and subvolume size [S1, unofficial].
*   **Strengths:** Captures anisotropic structures better than isotropic primitives; sparse representation reduces memory.
*   **Limitations:** Requires careful tuning of pruning and densification parameters; convergence depends on initial primitive placement [G].

## Background
Originated in the paper "ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling" [S1, unofficial]. The method addresses limitations of dense volume representations by introducing structure-aware ellipsoidal primitives.

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
