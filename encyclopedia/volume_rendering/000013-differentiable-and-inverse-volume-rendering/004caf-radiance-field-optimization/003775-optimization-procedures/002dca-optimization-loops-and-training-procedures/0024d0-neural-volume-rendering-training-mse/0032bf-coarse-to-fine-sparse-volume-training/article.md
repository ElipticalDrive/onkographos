# Coarse-to-fine Sparse Volume Training

Also called: Coarse-to-fine reconstruction, Sparse volume finetuning with pruning, Training and Optimization

## Overview
This method defines the training and optimization pipeline for sparse neural radiance fields, specifically focusing on the fine-tuning stage that refines a pre-constructed sparse volume $V^g$ per scene [S1]. It solves the problem of achieving high-fidelity novel view synthesis with significantly reduced training time compared to dense per-scene optimization methods like NeRF [S1]. It is distinguished from its siblings by its reliance on a coarse-to-fine iterative pruning and subdivision strategy during optimization, rather than static initialization or global optimization over the entire dataset [S1]. It builds on [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-constructed sparse neural volume $V^g$ containing per-voxel neural features and an MLP decoder $R$; a set of captured reference images with known camera poses; ground truth pixel colors $\hat{C}$ [S1].
*   **Outputs:** An optimized sparse neural volume $V^g$ and updated MLP decoder $R$ parameters [S1].

**Data Structures:**
*   **Sparse Volume $V^g$:** A sparse mapping of 3D voxel coordinates to neural feature vectors [S1].
*   **MLP Decoder $R$:** A multi-layer perceptron that maps spatial position, view direction, and voxel features to volume density and view-dependent radiance [S1].

**Numbered Steps:**
1.  **Initialization:** Start with the sparse volume $V^g$ and decoder $R$ obtained from the pre-training phase (local reconstruction and end-to-end fusion training) [S1].
2.  **Per-Scene Optimization Loop:** Iterate for a maximum of 25,000 iterations [S1].
3.  **Rendering:** For each iteration, render novel view images from both the current global volume $V^g_t$ and the local volume $V_t$ using differentiable ray marching, supervising with ground truth [S1].
4.  **Loss Calculation:** Compute the Mean Squared Error (MSE) between rendered colors and ground truth colors [S1].
5.  **Backpropagation:** Update the per-voxel neural features in $V^g$ and the MLP decoder weights using gradient descent [S1].
6.  **Coarse-to-Fine Refinement:** Every 10,000 iterations, perform the following structural updates [S1]:
    *   **Pruning:** Remove unnecessary voxels from $V^g$ based on a threshold criterion (referenced as Eqn. 8 in the source) [S1].
    *   **Subdivision:** Subdivide each remaining voxel into 8 sub-voxels. This increases the spatial resolution of the representation [S1].
7.  **Termination:** Stop after 25,000 iterations [S1].

**Formulas:**
The optimization minimizes the rendering loss. The core loss component cited for the fusion/training phase is:

$$
\mathcal{L}_{\text{fuse}} = \sum_t \|C_t - \hat{C}\|_2^2 + \|C_t^g - \hat{C}\|_2^2
$$

Where:
*   $t$ indexes the input frames/views [S1].
*   $\hat{C}$ is the ground truth pixel color [S1].
*   $C_t$ is the pixel color rendered from the local volume $V_t$ [S1].
*   $C_t^g$ is the pixel color rendered from the global volume $V^g_t$ after fusing frame $t$ [S1].

During the specific fine-tuning phase described, the primary objective is to minimize the error between the rendered global volume and the ground truth, effectively optimizing the features to match $\hat{C}$ [S1].

**Parameters:**
*   **Optimization Iterations:** Less than 25,000 [S1].
*   **Pruning/Subdivision Interval:** Every 10,000 iterations [S1].
*   **Subdivision Factor:** 8 sub-voxels per parent voxel [S1].
*   **Training Time:** Less than 1 hour per scene on standard hardware [S1].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_sparse_volume_training(V_g, R, images, poses, max_iterations=25000, interval=10000):
    """
    Fine-tunes a sparse neural volume and MLP decoder per scene.
    
    Args:
        V_g: Sparse mapping of voxel coordinates to neural feature vectors.
        R: MLP decoder function(position, direction, feature) -> (density, radiance).
        images: List of ground truth RGB arrays.
        poses: List of camera extrinsics/intrinsics.
        max_iterations: Total optimization steps.
        interval: Steps between pruning and subdivision.
        
    Returns:
        V_g: Optimized sparse volume.
        R: Optimized MLP decoder.
    """
    for step in range(1, max_iterations + 1):
        total_loss = 0.0
        # Gradient accumulators (conceptual for pseudocode)
        grad_V_g = {}
        grad_R = {}

        for img, pose in zip(images, poses):
            # 1. Generate rays from camera pose
            rays = generate_rays_from_pose(pose, img.shape)

            # 2. Differentiable ray marching
            rendered_colors = []
            for ray in rays:
                samples = sample_along_ray(ray, num_samples=128)
                densities = []
                radiances = []
                for pos in samples:
                    feat = V_g.get(voxel_key(pos), default_zero_vector)
                    dens, rad = R(pos, ray.direction, feat)
                    densities.append(dens)
                    radiances.append(rad)
                color = volume_render_composite(densities, radiances)
                rendered_colors.append(color)

            # 3. Compute MSE loss
            loss = sum((r - g)**2 for r, g in zip(rendered_colors, img.flatten()))
            total_loss += loss

        # 4. Backpropagation and parameter update
        # Automatic differentiation updates V_g features and R weights
        V_g = sgd_update(V_g, grad_V_g, lr=0.01)
        R = sgd_update(R, grad_R, lr=0.01)

        # 5. Coarse-to-fine refinement
        if step % interval == 0:
            # Prune unnecessary voxels (source references Eqn. 8 for criterion)
            V_g = prune_voxels_by_threshold(V_g, threshold=0.01)
            # Subdivide each remaining voxel into 8 sub-voxels
            V_g = subdivide_voxels(V_g, factor=8)

    return V_g, R

def generate_rays_from_pose(pose, img_shape):
    # Standard pinhole camera projection to generate ray origins and directions
    return [(pose.origin + (u, v, 1) @ pose.inv_intrinsics) for u, v in grid(img_shape)]

def sample_along_ray(ray, num_samples):
    return [ray.origin + ray.direction * t for t in linspace(ray.near, ray.far, num_samples)]

def volume_render_composite(densities, radiances):
    # Alpha compositing: C = sum(T_i * alpha_i * c_i)
    alphas = 1.0 - exp(-densities)
    transmittances = [1.0] + [prod(1.0 - a for a in alphas[:i]) for i in range(1, len(alphas))]
    return sum(t * a * c for t, a, c in zip(transmittances, alphas, radiances))

def sgd_update(structure, grads, lr):
    return {k: v - lr * grads.get(k, 0) for k, v in structure.items()}

def prune_voxels_by_threshold(V_g, threshold):
    # Removes voxels whose density or feature magnitude falls below threshold
    # [G] Exact threshold formula from Eqn. 8 not provided in sources
    return {k: v for k, v in V_g.items() if magnitude(v) > threshold}

def subdivide_voxels(V_g, factor):
    # Splits each voxel into 8 children, initializing child features
    new_V_g = {}
    for k, v in V_g.items():
        for dx, dy, dz in product([-1, 1], repeat=3):
            new_k = tuple(k[i] + dx/2 for i in range(3))
            new_V_g[new_k] = v / 2.0 # [G] Standard feature interpolation for subdivision
    return new_V_g
```

## Subtopics

### Pre-training Phase
The initial training of the local reconstruction network and radiance field decoder using $\mathcal{L}_{\text{local}}$ and $\mathcal{L}_{\text{fuse}}$ to initialize the model before fine-tuning [S1].

### Sparse Volume Representation
The underlying sparse data structure that allows for efficient storage and dynamic modification (pruning/subdivision) of the neural features [S1].

## Operating characteristics
*   **Time Complexity:** Optimization requires less than 25,000 iterations and takes less than 1 hour per scene, significantly faster than dense per-scene methods like NeRF [S1].
*   **Memory Complexity:** Depends on the sparsity of $V^g$, with pruning reducing memory usage and subdivision increasing it [S1].
*   **Strengths:** Fast per-scene optimization and dynamic resolution refinement [S1].
*   **Limitations:** Requires a good initial reconstruction from the pre-training phase [S1].
*   **When to use:** When high-quality novel view synthesis is needed for a specific scene with limited computational resources or time constraints, and a pre-trained model is available [S1].
*   **When not to use:** Not given in the sources.

## Background
The method originates from the NeRFusion paper, which introduces a framework for fusing radiance fields for large-scale scene reconstruction [S1]. The coarse-to-fine training strategy is adapted from NSVF [S1]. It was developed to address the computational bottleneck of training neural radiance fields per scene, offering a faster alternative to pure per-scene optimization methods like NeRF [S1]. The approach is used in large-scale indoor scene reconstruction (e.g., ScanNet) and small object reconstruction (e.g., DTU) [S1].

## Sources
[S1] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
