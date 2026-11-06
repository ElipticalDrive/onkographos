# Structure-aware primitive learning

Also called: ESVR (Ellipsoid-based Sparse Volume Rendering)

## Overview
Structure-aware primitive learning represents volumetric data using a sparse set of parameterized ellipsoids, optimizing for structural similarity rather than just pixel-wise error. It differs from dense voxel-based methods by explicitly modeling anisotropic structures through covariance matrices, and from isotropic Gaussian splatting by using ellipsoidal primitives that can capture elongated features more efficiently [S1, unofficial].

## How it works
**Inputs:** Ground-truth volume data $V_{gt}$, initial set of ellipsoids $E$.
**Outputs:** Optimized set of ellipsoids $E$.

**Data Structures:**
Each ellipsoid primitive $j \in E$ is defined by:
- Mean position $\boldsymbol{\mu}_j \in \mathbb{R}^3$
- Covariance matrix $\boldsymbol{\Sigma}_j \in \mathbb{R}^{3 \times 3}$ (symmetric positive-definite)
- Intensity/Color $I_j \in \mathbb{R}$ (or $\mathbb{R}^3$ for color)

**Algorithm:**
The method operates in iterations. In each iteration, the volume is processed sequentially in subvolumes.

1. **Fetch Subvolume:** Retrieve the next ground-truth subvolume $V_{gt}$ from the input data [S1, unofficial].
2. **Retrieve Overlapping Primitives:** Identify the subset of ellipsoids $E'$ that overlap with $V_{gt}$ [S1, unofficial].
3. **Voxelize:** Render the overlapping ellipsoids into a voxel grid $\hat{V}$ to predict the subvolume [S1, unofficial]. This involves projecting each ellipsoid onto the grid cells it intersects.
4. **Compute Loss:** Calculate the reconstruction loss $L_{rec}$ using a weighted combination of L1 loss and Structural Similarity (DSSIM) loss [S1, unofficial]:
   $$ L_{rec} = \beta L_1(\hat{V}, V_{gt}) + (1 - \beta) L_{DSSIM}(\hat{V}, V_{gt}) $$
   where $\beta$ is the reconstruction loss weight [S1, unofficial].
5. **Update Parameters:** Update the parameters $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, I)$ of the overlapping ellipsoids using the Adam optimizer on the gradient of $L_{rec}$ [S1, unofficial].
6. **Pruning:** Periodically, compute an importance score for each ellipsoid (e.g., based on fractional anisotropy or contribution to loss) and remove primitives with low scores [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def structure_aware_primitive_learning(V_gt_data, E_initial, beta=0.5, max_iters=1000):
    E = E_initial
    i = 0
    while i < max_iters:
        # 1. Fetch next sequential ground-truth subvolume
        V_gt = NextSubvolume(V_gt_data)
        
        # 2. Retrieve ellipsoids overlapping the current subvolume
        E_overlapping = GetOverlappingEllipsoids(E, V_gt)
        
        # 3. Voxelize: Predict subvolume from ellipsoids
        V_hat = Voxelize(E_overlapping)
        
        # 4. Compute reconstruction loss
        L1_loss = MeanAbsoluteError(V_hat, V_gt)
        DSSIM_loss = StructuralSimilarityIndex(V_hat, V_gt)
        L_rec = beta * L1_loss + (1 - beta) * DSSIM_loss
        
        # 5. Update primitive parameters using Adam optimizer
        gradients = ComputeGradients(L_rec, E_overlapping)
        E_overlapping = AdamUpdate(E_overlapping, gradients)
        
        # 6. Prune primitives with low importance scores (periodic)
        if i % prune_interval == 0:
            E = RemoveLowScore(E)
            
        i += 1
    return E
```

**Parameters:**
- $\beta$: Weight for L1 loss vs DSSIM loss. Typical value not given in sources.
- `prune_interval`: Frequency of pruning. Not given in sources.

**Edge Cases:**
- **Degenerate Covariance:** Ensure $\boldsymbol{\Sigma}_j$ remains positive-definite during updates. [G]
- **Boundary Conditions:** Handle ellipsoids partially outside the volume bounds. [G]

## Subtopics

## Operating characteristics
**Time Complexity:** Depends on the number of overlapping ellipsoids per subvolume and the voxelization cost. Sparse representation reduces computation compared to dense volumes.
**Memory Complexity:** Proportional to the number of ellipsoids, which is typically much smaller than the number of voxels in the volume.
**Strengths:** Efficient representation of anisotropic structures; structural similarity loss preserves edges and textures better than L1 alone.
**Limitations:** Requires careful initialization and pruning; covariance matrix updates can be unstable.

## Background
Originated in the paper "ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling" [S1, unofficial]. The method addresses the need for efficient and structurally accurate volume representation by moving beyond isotropic primitives.

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
