# Structure-aware primitive learning

Also called: ESVR (Ellipsoid-based Sparse Volume Rendering)

## Overview
Structure-aware primitive learning represents volumetric data using a sparse set of parameterized ellipsoids, optimizing for structural similarity rather than just pixel-wise error [S1, unofficial]. It differs from dense voxel-based methods by explicitly modeling anisotropic structures through covariance matrices, and from isotropic Gaussian splatting by using ellipsoidal primitives that can capture elongated features more efficiently [S1, unofficial].

## How it works
**Inputs:** Ground-truth volume data $V_{gt}$ and an initial set of ellipsoids $E$ [S1, unofficial].
**Outputs:** Optimized set of ellipsoids $E$ [S1, unofficial].

**Data Structures:**
Each ellipsoid primitive $j \in E$ is defined by a mean position $\boldsymbol{\mu}_j \in \mathbb{R}^3$, a symmetric positive-definite covariance matrix $\boldsymbol{\Sigma}_j \in \mathbb{R}^{3 \times 3}$, and an intensity or color value $I_j \in \mathbb{R}$ (or $\mathbb{R}^3$ for color) [S1, unofficial].

**Algorithm:**
The method operates in iterations, processing the volume sequentially in subvolumes [S1, unofficial].
1. **Fetch Subvolume:** Retrieve the next ground-truth subvolume $V_{gt}$ from the input data [S1, unofficial].
2. **Retrieve Overlapping Primitives:** Identify the subset of ellipsoids $E'$ that overlap with $V_{gt}$ [S1, unofficial].
3. **Voxelize:** Render the overlapping ellipsoids into a voxel grid $\hat{V}$ to predict the subvolume [S1, unofficial]. This involves projecting each ellipsoid onto the grid cells it intersects [S1, unofficial].
4. **Compute Loss:** Calculate the reconstruction loss $L_{rec}$ using a weighted combination of L1 loss and Structural Similarity (DSSIM) loss [S1, unofficial]:
   $$ L_{rec} = \beta L_1(\hat{V}, V_{gt}) + (1 - \beta) L_{DSSIM}(\hat{V}, V_{gt}) $$
   where $\beta$ is the reconstruction loss weight [S1, unofficial].
5. **Update Parameters:** Update the parameters $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, I)$ of the overlapping ellipsoids using the Adam optimizer on the gradient of $L_{rec}$ [S1, unofficial].
6. **Pruning:** Periodically, compute an importance score for each ellipsoid (e.g., based on fractional anisotropy or contribution to loss) and remove primitives with low scores [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def structure_aware_primitive_learning(V_gt_data, E_initial, beta=0.5, max_iters=1000, prune_interval=10):
    E = list(E_initial)
    subvol_idx = 0
    i = 0
    while i < max_iters:
        # 1. Fetch next sequential ground-truth subvolume
        V_gt = V_gt_data[subvol_idx]
        subvol_idx += 1
        if subvol_idx >= len(V_gt_data):
            break

        # 2. Retrieve ellipsoids overlapping the current subvolume
        E_overlapping = [p for p in E if p.bbox.overlaps(V_gt.bounds)]

        # 3. Voxelize: Predict subvolume from ellipsoids
        V_hat = initialize_grid(V_gt.shape)
        for p in E_overlapping:
            for cell in V_gt.cells_intersecting(p.bbox):
                V_hat[cell] += ellipsoid_value(p.mu, p.Sigma, cell.center)

        # 4. Compute reconstruction loss
        L1_loss = mean_absolute_error(V_hat, V_gt)
        # Structural similarity index computation is a standard metric [G]
        DSSIM_loss = structural_similarity_index(V_hat, V_gt)
        L_rec = beta * L1_loss + (1 - beta) * DSSIM_loss

        # 5. Update primitive parameters using Adam optimizer
        # Gradient computation and Adam parameter update are standard optimization steps [G]
        gradients = compute_gradients(L_rec, E_overlapping)
        E_overlapping = adam_update(E_overlapping, gradients)

        # 6. Prune primitives with low importance scores (periodic)
        if i % prune_interval == 0:
            E = remove_low_score_primitives(E)

        i += 1
    return E

def initialize_grid(shape):
    return [0.0] * (shape[0] * shape[1] * shape[2])

def ellipsoid_value(mu, Sigma, center):
    # Standard ellipsoid/Gaussian projection value [G]
    diff = center - mu
    return np.exp(-0.5 * diff.T @ np.linalg.inv(Sigma) @ diff)

def mean_absolute_error(V_hat, V_gt):
    return np.mean(np.abs(V_hat - V_gt))

def structural_similarity_index(V_hat, V_gt):
    # Standard DSSIM metric [G]
    pass

def compute_gradients(loss, primitives):
    # Standard autograd/numerical gradient step [G]
    pass

def adam_update(primitives, gradients, lr=1e-3):
    # Standard Adam optimizer step [G]
    pass

def remove_low_score_primitives(E):
    # Computes fractional anisotropy from covariance and removes low-score primitives [S1, unofficial]
    return [p for p in E if compute_fa(p.Sigma) > threshold]

def compute_fa(Sigma):
    # Standard fractional anisotropy calculation from covariance eigenvalues [S1, unofficial]
    pass
```

**Parameters:**
- $\beta$: Weight for L1 loss vs DSSIM loss. Typical value not given in the sources.
- `prune_interval`: Frequency of pruning. Not given in the sources.

**Edge Cases:**
- **Degenerate Covariance:** Ensure $\boldsymbol{\Sigma}_j$ remains positive-definite during updates [G].
- **Boundary Conditions:** Handle ellipsoids partially outside the volume bounds [G].

## Subtopics

## Operating characteristics
**Time Complexity:** Depends on the number of overlapping ellipsoids per subvolume and the voxelization cost [S1, unofficial]. Sparse representation reduces computation compared to dense volumes [S1, unofficial].
**Memory Complexity:** Proportional to the number of ellipsoids, which is typically much smaller than the number of voxels in the volume [S1, unofficial].
**Strengths:** Efficient representation of anisotropic structures; structural similarity loss preserves edges and textures better than L1 alone [S1, unofficial].
**Limitations:** Requires careful initialization and pruning; covariance matrix updates can be unstable [S1, unofficial].

## Background
Originated in the paper "ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling" [S1, unofficial]. The method addresses the need for efficient and structurally accurate volume representation by moving beyond isotropic primitives [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
