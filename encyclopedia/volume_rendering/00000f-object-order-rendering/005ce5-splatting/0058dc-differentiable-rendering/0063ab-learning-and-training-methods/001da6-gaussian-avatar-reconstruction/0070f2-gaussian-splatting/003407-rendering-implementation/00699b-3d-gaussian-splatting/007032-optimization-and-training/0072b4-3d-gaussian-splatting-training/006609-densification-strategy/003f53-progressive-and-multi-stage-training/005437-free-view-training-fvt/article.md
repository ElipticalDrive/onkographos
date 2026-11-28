# Free-View Training (FVT)

## Overview
Free-View Training (FVT) is a training strategy for 3D Gaussian Splatting (3DGS) that adds geometrical constraints on Gaussian localization to disentangle model performance from the specific number of input views [S1, S2]. It enables robust novel view synthesis across broader view ranges regardless of the input view count, facilitating generalizable reconstruction from long sequences [S1, S2]. Unlike standard 3DGS which optimizes for a fixed set of views, FVT supervises renderings in broader view interpolations to improve depth estimation and adaptability [S1].

## How it works
FVT operates on the Gaussian set $\mathcal{G}$ defined in [Densification Strategy](6234-densification-strategy.md). It modifies the training loop to sample multiple context views and supervise the rendering against ground truth images from these interpolated views.

**Inputs:**
- A set of training images with known camera poses.
- A sequence of frames (potentially long sequences).
- Hyperparameter $T$: the number of context views to sample per iteration.

**Outputs:**
- Optimized 3D Gaussian primitives $\mathcal{G}$ (positions, covariances, opacities, spherical harmonics coefficients).

**Data Structures:**
- $\mathcal{G}$: The set of 3D Gaussians.
- $I_{gt}$: Ground truth image for a target view.
- $I_{pred}$: Rendered image from $\mathcal{G}$ using the camera projection for the target view.

**Parameters:**
- $T$: Number of context views sampled. Typical value: 2 to 8 [S1].

**Steps:**
1. **Context View Sampling:** During each training iteration, randomly sample $T$ context views from the available training sequence. Let the set of sampled view indices be $\mathcal{V}_{ctx} = \{v_1, v_2, ..., v_T\}$ [S1].
2. **Target View Selection:** Select a target view $v_{tgt}$ from the sequence. This view serves as the reference for supervision [S1].
3. **Rendering:** Render the image $I_{pred}$ for the target view $v_{tgt}$ using the current Gaussian set $\mathcal{G}$ and the corresponding camera pose [S1].
4. **Supervision:** Compute the loss between the rendered image $I_{pred}$ and the ground truth image $I_{gt}$ of the target view. The supervision is applied on full resolution to encourage high-resolution renderings, even if intermediate feature extraction uses lower resolution maps [S1].
5. **Optimization:** Update the parameters of $\mathcal{G}$ (and potentially camera poses if unfrozen) using gradient descent to minimize the loss [S1].
6. **Iteration:** Repeat steps 1-5 for the duration of the training, ensuring the model is exposed to a diverse range of view interpolations [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def free_view_training(Gaussians, TrainingSequence, T):
    """
    Gaussians: Current set of 3D Gaussian primitives
    TrainingSequence: List of (image, camera_pose) tuples
    T: Number of context views to sample (2-8)
    """
    for iteration in range(num_iterations):
        # Step 1: Randomly sample T context views
        # Note: The specific selection of context views vs target view
        # is described as "supervise the image renderings in the broader view interpolations"
        # We assume one view is the target for loss, others inform the geometry.
        
        sampled_indices = random.sample(range(len(TrainingSequence)), T + 1)
        target_idx = sampled_indices[0]
        context_indices = sampled_indices[1:]
        
        target_image, target_pose = TrainingSequence[target_idx]
        
        # Step 2 & 3: Render the target view
        rendered_image = render(Gaussians, target_pose)
        
        # Step 4: Compute loss on full resolution
        # Loss function typically includes photometric and depth consistency terms
        # if depth priors are available, or just photometric loss otherwise.
        loss = compute_loss(rendered_image, target_image)
        
        # Step 5: Backpropagate and update Gaussians
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # Note: Densification (splitting/cloning) is handled by the parent
        # [Densification Strategy](6234-densification-strategy.md) logic,
        # potentially triggered by gradients from this FVT supervision.
```

**Edge Cases and Pitfalls:**
- **View Distribution:** If the sampled views are too similar, the geometrical constraints may be weak. The random sampling should cover the view range effectively [S1].
- **Sequence Length:** FVT is designed for long sequences. Training on very short sequences may not provide the "broader view interpolations" necessary for the strategy to disentangle view count from performance [S1, S2].
- **Resolution:** Supervision must be applied at full resolution to prevent blurring, even if feature extraction uses lower resolutions [S1].

## Operating characteristics
- **Time Complexity:** Training time increases with the number of views sampled and the sequence length, but the efficient feature extraction and aggregation in FreeSplat++ make long-sequence training feasible [S1].
- **Memory Complexity:** Depends on the number of Gaussians and the sequence length stored in memory. FreeSplat aims to reduce redundant Gaussians, potentially lowering memory usage compared to naive multi-view training [S2].
- **Performance:** FVT significantly contributes to depth estimation accuracy and robust novel view synthesis across different numbers of input views [S1, S2]. It achieves state-of-the-art performance on indoor datasets like ScanNet and Replica [S2].
- **Strengths:** Disentangles generalizability from input view count; improves depth estimation; reduces Gaussian redundancy in overlapping regions [S1, S2].
- **Limitations:** Requires careful sampling of views to ensure broad coverage; may require more training iterations to converge compared to single-view optimization [S1].

## Background
FVT was introduced in the context of generalizable 3D Gaussian Splatting, specifically in the FreeSplat and FreeSplat++ papers [S1, S2]. It addresses the limitation of standard 3DGS, which is often tied to the specific number and arrangement of input views. By training on long sequences and supervising broader view interpolations, FVT enables feed-forward large scene reconstruction without depth priors and improves inference efficiency [S2].

## Sources
[S1] FreeSplat++: Generalizable 3D Gaussian Splatting for Efficient Indoor Scene Reconstruction., https://doi.org/10.1109/tpami.2026.3665771, fetched 2026-10-26
[S2] FreeSplat: Generalizable 3D Gaussian Splatting Towards Free-View Synthesis of Indoor Scenes, https://doi.org/10.48550/arXiv.2405.17958, fetched 2026-10-29
