# Training Optimization
Also called: Joint Optimization of Network Weights, Feature Grids, and Latent Codes

## Overview
This method jointly optimizes neural network weights, feature grids, and shape-specific latent codes to reconstruct 3D geometry and appearance from input data [S1]. It distinguishes itself from standard implicit field training by treating the latent codes as learnable parameters alongside the network and grid, enabling a single shared network to represent multiple distinct shapes without retraining the encoder or network weights for each instance [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training data consisting of $J$ shapes, each represented by depth maps with object masks and known camera poses [S1].
*   **Outputs:** Optimized SDF network weights $\Theta_s$, DDF network weights $\Theta_d$, feature grids $M$, and latent codes $Z$ [S1].

**Data Structures:**
*   $\Theta_s$: Learnable parameters of the Signed Distance Function (SDF) model [S1].
*   $\Theta_d$: Learnable parameters of the Directional Difference Function (DDF) model [S1].
*   $M$: Feature grids, defined as $M = (M_s, M_d)$ [S1].
    *   $M_s$: SDF feature grids, $M_s = \{M^s_i \mid i \in \{xy, yz, zx\}\}$ [S1].
    *   $M_d$: DDF feature grids, $M_d = \{M^d_i \mid i \in \{pxpy, \dots, rzrx\}\}$ [S1].
*   $Z$: Set of latent codes for the $J$ training shapes, $Z = \{z_i \mid i = 1 \dots J\}$ [S1].

**Loss Function:**
The total loss combines several components with weights $w_s, w_d, w_\sigma, w_{tv}, w_{ts}, w_l$ [S1]:
$$ \mathcal{L}_{total} = w_s \mathcal{L}_{SDF} + w_d \mathcal{L}_{DDF} + w_\sigma \mathcal{L}_{\sigma} + w_{tv} \mathcal{L}_{TV} + w_{ts} \mathcal{L}_{TS} + w_l \mathcal{L}_{latent} $$
where $\mathcal{L}_{SDF}$ and $\mathcal{L}_{DDF}$ are the SDF and DDF losses, $\mathcal{L}_{\sigma}$ is the ray hit loss, $\mathcal{L}_{TV}$ is the total variation regularizer, $\mathcal{L}_{TS}$ is the TrackSDF regularizer, and $\mathcal{L}_{latent}$ is the latent code regularizer [S1]. Specific silhouette losses are defined as [S1]:
$$ \mathcal{L}_{S_{s_+}}(m) = \| \langle s, m \rangle \|_1 $$
$$ \mathcal{L}_{S_{s_-}}(m) = \| |\langle s, 1-m \rangle| - \tau \|_1 $$

**Algorithm:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize Theta_s, Theta_d, M, Z randomly or via standard initialization [G]
for iteration in range(max_iterations): # Not given in sources.
    j = random.randint(0, J - 1) # Not given in sources.
    z_j = Z[j]
    depth, mask, pose = training_data[j]
    # Sample points x along rays from the camera pose [G]
    points = sample_points_along_rays(pose, depth, mask)
    sdf_pred, ddf_pred = query_networks(points, Theta_s, Theta_d, M, z_j)
    loss = compute_total_loss(sdf_pred, ddf_pred, points, depth, mask, z_j)
    update_parameters(Theta_s, Theta_d, M, Z, loss)
```

**Inference (Reconstruction):**
For a new single-view depth map, the method keeps $\Theta_s, \Theta_d, M$ fixed [S1]. It optimizes only the latent code $z$ to minimize the reconstruction loss [S1]. No sphere tracing is required during optimization [S1].

## Subtopics

### [Factor Fields Optimization](22964-factor-fields-optimization.md)
Factor Fields Optimization is a training method for neural and primitive-based volume representations that jointly optimizes network weights, feature grids, and latent codes. It models signals as a product of factor fields, regularizing the solution via stochastic dropout to encourage sparsity and prevent co-adaptation of features. It distinguishes itself by using a structured factorization (basis and coefficient factors) with DCT-initialized bases, enabling efficient joint optimization of multiple signals by sharing projection and basis parameters while keeping coefficient factors distinct.

## Operating characteristics
Not given in the sources.

## Background
Originated in the context of fast inverse rendering and 3D reconstruction from single-view depth maps [S1]. The method leverages an autodecoder framework where the network acts as a prior, and latent codes capture shape-specific variations [S1].

## Sources
[S1] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
