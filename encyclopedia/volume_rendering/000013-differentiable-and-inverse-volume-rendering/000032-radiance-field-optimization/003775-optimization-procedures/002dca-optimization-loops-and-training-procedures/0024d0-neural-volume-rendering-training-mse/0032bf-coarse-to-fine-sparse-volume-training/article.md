# Coarse-to-fine Sparse Volume Training

Also called: Coarse-to-fine reconstruction, Sparse volume finetuning with pruning, Training and Optimization

## Overview
This method defines the training and optimization pipeline for sparse neural radiance fields, specifically focusing on the fine-tuning stage that refines a pre-constructed sparse volume $V^g$ per scene. It solves the problem of achieving high-fidelity novel view synthesis with significantly reduced training time compared to dense per-scene optimization methods like NeRF [S1]. It is distinguished from its siblings by its reliance on a coarse-to-fine iterative pruning and subdivision strategy during optimization, rather than static initialization or global optimization over the entire dataset [S1]. It builds on [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-constructed sparse neural volume $V^g$ containing per-voxel neural features and an MLP decoder $R$; a set of captured reference images with known camera poses; ground truth pixel colors $\hat{C}$.
*   **Outputs:** An optimized sparse neural volume $V^g$ and updated MLP decoder $R$ parameters.

**Data Structures:**
*   **Sparse Volume $V^g$:** A hash table or sparse grid mapping 3D voxel coordinates to neural feature vectors. The structure supports dynamic pruning (removal of voxels) and subdivision (splitting one voxel into 8 children).
*   **MLP Decoder $R$:** A multi-layer perceptron that maps spatial position, view direction, and voxel features to volume density and view-dependent radiance.

**Numbered Steps:**
1.  **Initialization:** Start with the sparse volume $V^g$ and decoder $R$ obtained from the pre-training phase (local reconstruction and end-to-end fusion training) [S1].
2.  **Per-Scene Optimization Loop:** Iterate for a maximum of 25,000 iterations [S1].
3.  **Rendering:** For each iteration, render novel view images from both the current global volume $V^g_t$ and the local volume $V_t$ (if applicable in the broader pipeline context, though fine-tuning primarily targets $V^g$) using differentiable ray marching [S1].
4.  **Loss Calculation:** Compute the Mean Squared Error (MSE) between rendered colors and ground truth colors [S1].
5.  **Backpropagation:** Update the per-voxel neural features in $V^g$ and the MLP decoder weights using gradient descent [S1].
6.  **Coarse-to-Fine Refinement:** Every 10,000 iterations, perform the following structural updates [S1]:
    *   **Pruning:** Remove unnecessary voxels from $V^g$ based on a threshold criterion (referenced as Eqn. 8 in the source, typically involving density or feature magnitude thresholds) [S1].
    *   **Subdivision:** Subdivide each remaining voxel into 8 sub-voxels. This increases the spatial resolution of the representation [S1].
7.  **Termination:** Stop after 25,000 iterations or when convergence is reached [S1].

**Formulas:**
The optimization minimizes the rendering loss. While the pre-training phase uses specific local and fused losses, the fine-tuning stage generally minimizes the pixel-wise reconstruction error [S1]. The core loss component cited for the fusion/training phase is:

$$
\mathcal{L}_{\text{fuse}} = \sum_t \|C_t - \hat{C}\|_2^2 + \|C_t^g - \hat{C}\|_2^2
$$

Where:
*   $t$ indexes the input frames/views.
*   $\hat{C}$ is the ground truth pixel color.
*   $C_t$ is the pixel color rendered from the local volume $V_t$.
*   $C_t^g$ is the pixel color rendered from the global volume $V^g_t$ after fusing frame $t$ [S1].

During the specific fine-tuning phase described, the primary objective is to minimize the error between the rendered global volume and the ground truth, effectively optimizing the features to match $\hat{C}$ [S1].

**Parameters:**
*   **Optimization Iterations:** Less than 25,000 [S1].
*   **Pruning/Subdivision Interval:** Every 10,000 iterations [S1].
*   **Subdivision Factor:** 8 sub-voxels per parent voxel [S1].
*   **Training Time:** Less than 1 hour per scene on standard hardware [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on differentiable ray marching, requiring careful handling of floating-point precision during gradient backpropagation through the sparse data structure [S1].
*   **Boundaries:** Pruning must be handled carefully to avoid removing voxels that contribute to visible geometry, potentially causing holes in the reconstruction. The source notes that the initial reconstruction is "very good," implying the pruning threshold is set conservatively [S1].
*   **Degenerate Input:** If the initial sparse volume is too sparse or inaccurate, the fine-tuning may fail to converge to high quality. The method assumes a reasonable initial reconstruction from the pre-training phase [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_sparse_volume_training(V_g, R, images, poses, max_iterations=25000, interval=10000):
    """
    Fine-tunes a sparse neural volume and MLP decoder per scene.
    
    Args:
        V_g: Sparse volume data structure (hash map/grid) with initial features.
        R: MLP decoder function.
        images: List of ground truth images.
        poses: List of camera poses corresponding to images.
        max_iterations: Total optimization steps.
        interval: Steps between pruning and subdivision.
        
    Returns:
        V_g: Optimized sparse volume.
        R: Optimized MLP decoder.
    """
    for step in range(1, max_iterations + 1):
        # 1. Render from current global volume
        # Differentiable ray marching through V_g and R
        rendered_colors = []
        for img, pose in zip(images, poses):
            # Sample rays based on pose
            rays = generate_rays(pose, img.shape)
            # Query V_g for features at ray points
            features = query_sparse_volume(V_g, rays)
            # Decode radiance and density
            radiance, density = R(rays.positions, rays.directions, features)
            # Volume rendering to get pixel color
            color = volume_render(radiance, density, rays)
            rendered_colors.append(color)
        
        # 2. Compute Loss
        # Sum of squared errors between rendered and ground truth
        loss = 0.0
        for rendered, ground_truth in zip(rendered_colors, images):
            loss += mse_loss(rendered, ground_truth)
        
        # 3. Backpropagate
        # Update V_g features and R weights
        gradients = compute_gradients(loss, V_g, R)
        update(V_g, R, gradients)
        
        # 4. Coarse-to-fine refinement
        if step % interval == 0:
            # Prune unnecessary voxels
            # Note: Eqn. 8 from source defines the pruning criterion
            # Typically based on density or feature magnitude threshold
            prune_voxels(V_g, threshold=0.01) # [G] Threshold value not explicitly given in source, assumed small
            
            # Subdivide each remaining voxel into 8 sub-voxels
            subdivide_voxels(V_g, factor=8)
            
    return V_g, R
```

## Subtopics

### Pre-training Phase
The initial training of the local reconstruction network and radiance field decoder using $\mathcal{L}_{\text{local}}$ and $\mathcal{L}_{\text{fuse}}$ to initialize the model before fine-tuning. [S1]

### Sparse Volume Representation
The underlying data structure (hash grid) that allows for efficient storage and dynamic modification (pruning/subdivision) of the neural features. [S1]

## Operating characteristics
*   **Time Complexity:** Significantly faster than dense per-scene optimization methods like NeRF. The total training time is less than 1 hour for less than 25,000 iterations [S1].
*   **Memory Complexity:** Depends on the sparsity of the volume. Pruning reduces memory usage by removing empty or low-density voxels, while subdivision increases it by creating more voxels. The net effect is a manageable memory footprint for large scenes compared to dense grids [S1].
*   **Measured Performance:** Produces high-quality rendering results comparable to denser methods but with much shorter optimization times [S1].
*   **Strengths:** Fast per-scene optimization; ability to refine resolution dynamically; leverages pre-trained features for better initialization.
*   **Limitations:** Requires a good initial reconstruction from the pre-training phase; performance depends on the effectiveness of the pruning and subdivision heuristics.
*   **When to use:** When high-quality novel view synthesis is needed for a specific scene with limited computational resources or time constraints, and a pre-trained model is available.
*   **When not to use:** When no pre-training data is available, or when the scene geometry is extremely complex and requires more than 25k iterations to converge (though the source claims 25k is usually sufficient) [S1].

## Background
The method originates from the NeRFusion paper, which introduces a framework for fusing radiance fields for large-scale scene reconstruction. The coarse-to-fine training strategy is adapted from NSVF (Neural Sparse Voxel Fields) [S1, S24]. It was developed to address the computational bottleneck of training neural radiance fields per scene, offering a faster alternative to pure per-scene optimization methods like NeRF [S1]. The approach is used in large-scale indoor scene reconstruction (e.g., ScanNet) and small object reconstruction (e.g., DTU) [S1].

## Sources
[S1] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
