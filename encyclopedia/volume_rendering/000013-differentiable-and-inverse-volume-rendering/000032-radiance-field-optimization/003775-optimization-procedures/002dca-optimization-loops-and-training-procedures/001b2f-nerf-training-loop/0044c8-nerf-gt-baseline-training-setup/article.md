# NeRF-GT Baseline Training Setup
Also called: NeRF-GT, HDR-NeRF Baseline
## Overview
NeRF-GT is a baseline training procedure for synthesizing High Dynamic Range (HDR) novel views using Neural Radiance Fields. It addresses the problem of encoding high-dynamic-range scenes by training the radiance field directly on HDR input views, rather than Low Dynamic Range (LDR) inputs. It differs from standard NeRF training [NeRF Training Loop](970-nerf-training-loop.md) by operating in the HDR domain and differs from LDR-based variants [Training Configuration](2332-training-configuration.md) by requiring tone-mapping of predictions for loss computation to handle the high dynamic range.
## How it works
**Inputs and Outputs:**
- Input: A set of $N$ HDR images $\{I_i^{HDR}\}_{i=1}^N$ with known camera poses.
- Output: A trained NeRF model $\mathcal{F}$ capable of rendering HDR views.

**Data Structures:**
The NeRF model $\mathcal{F}$ takes a 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input and outputs a color $\mathbf{c}$ and density $\sigma$. The color $\mathbf{c}$ is assumed to be in HDR linear space.

**Steps:**
1. **Training Setup:** Initialize the NeRF model. Use the HDR images as the ground truth targets for the training rays.
2. **Ray Marching:** For each training image, generate rays and compute the volume rendering integral to predict the HDR pixel color $\mathbf{C}_{pred}$.
3. **Tone-Mapping:** Apply a tone-mapping operator $T(\cdot)$ to the predicted HDR color $\mathbf{C}_{pred}$ to convert it to LDR space: $\mathbf{C}_{pred}^{LDR} = T(\mathbf{C}_{pred})$.
4. **Loss Computation:** Apply the same tone-mapping operator $T(\cdot)$ to the ground truth HDR image pixel $\mathbf{C}_{gt}^{HDR}$ to obtain $\mathbf{C}_{gt}^{LDR} = T(\mathbf{C}_{gt}^{HDR})$. Compute the loss $\mathcal{L}$ by comparing $\mathbf{C}_{pred}^{LDR}$ and $\mathbf{C}_{gt}^{LDR}$ (e.g., using L1 or L2 loss).
5. **Optimization:** Update the NeRF parameters using gradient descent.

**Pitfalls:**
- Training directly on HDR views can fail to encode all scene areas well due to the high dynamic range, even when using tone-mapped colors for loss calculation [S1]. The vast range of intensities can make it difficult for the network to converge uniformly across bright and dark regions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_gt(hdr_images, camera_poses, nerf_model, tone_mapper, optimizer):
    """
    Train NeRF directly on HDR views.
    """
    for epoch in range(num_epochs):
        for hdr_image, pose in zip(hdr_images, camera_poses):
            # 1. Generate rays for the current image
            rays = generate_rays(hdr_image.shape, pose)
            
            # 2. Predict HDR colors via volume rendering
            # ray_marching returns HDR color C_pred
            c_pred_hdr = ray_marching(rays, nerf_model)
            
            # 3. Tone-map prediction to LDR
            c_pred_ldr = tone_mapper(c_pred_hdr)
            
            # 4. Tone-map ground truth HDR to LDR
            c_gt_ldr = tone_mapper(hdr_image)
            
            # 5. Compute loss
            loss = compute_loss(c_pred_ldr, c_gt_ldr)
            
            # 6. Backpropagate and update
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```
## Subtopics
## Operating characteristics
- **Complexity:** Same as standard NeRF, $O(N \cdot M)$ per image where $N$ is rays and $M$ is samples.
- **Strengths:** Provides an upper bound for HDR synthesis methods by utilizing full HDR information during training [S1].
- **Limitations:** Difficult to ensure all areas of a scene are encoded well due to the high dynamic range [S1].
- **When to use:** When HDR input data is available and a baseline for HDR synthesis performance is needed.
- **When not to use:** When only LDR inputs are available, or when training stability is a primary concern without additional regularization.
## Background
The NeRF-GT baseline was introduced in the context of HDR-NeRF [S1]. It serves as a comparison point for methods that train on LDR inputs or use other strategies to handle dynamic range. The authors note that while HDR views can be considered the upper bound, direct training presents challenges [S1].
## Sources
[S1] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
