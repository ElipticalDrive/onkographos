# Opacity Pruning

Also called: Opacity Based Pruning, Opacity and Scale Pruning

## Overview
Opacity Pruning is a density control strategy in 3D Gaussian Splatting that removes Gaussian primitives with low opacity values, as they contribute minimally to scene reconstruction [S1]. It serves as a primary mechanism for eliminating "floaters" and redundant primitives generated during optimization [S1]. Unlike gradient-based pruning, which relies on optimization gradient magnitude, Opacity Pruning relies solely on the opacity attribute magnitude [S1].

## How it works
**Inputs:**
- A set of $N$ 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$ [G].
- Each Gaussian $g_i$ has attributes: position $\mathbf{x}_i$, scale $\mathbf{s}_i$, rotation $\mathbf{r}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$ [G].
- A pruning threshold $\tau_\alpha$ [G].
- Optional: A visibility mask $M$ (binary image) and depth information $D$ [S2, unofficial].

**Outputs:**
- A pruned set of Gaussians $\mathcal{G}' \subset \mathcal{G}$ containing only those Gaussians deemed significant [G].

**Data Structures:**
- The Gaussian set is typically stored in a flat array or structure-of-arrays (SoA) layout for efficient rasterization [G]. Pruning updates this structure by compacting the array or marking entries as inactive [G].

**Steps:**
1. **Evaluate Opacity:** For each Gaussian $g_i$, retrieve its current opacity value $\alpha_i$ [S1].
2. **Apply Threshold:** Compare $\alpha_i$ against the threshold $\tau_\alpha$ [S1].
   $$ \text{keep}_i = (\alpha_i > \tau_\alpha) $$
3. **Optional Visibility Masking:** If a visibility mask $M$ is provided, calculate a visible mask and prune Gaussians projected onto masked pixels [S2, unofficial].
   $$ \text{keep}_i = \text{keep}_i \land \neg \text{projected\_onto\_mask}(g_i, M) $$
4. **Optional Depth/Distance Pruning:** Use depth information to prune Gaussians located in free space or control density according to distance [S2, unofficial].
5. **Remove Pruned Gaussians:** Discard all $g_i$ where $\text{keep}_i$ is false [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def opacity_pruning(gaussians, threshold, visibility_mask=None, depth_threshold=None, camera_intrinsics=None, camera_pose=None):
    """
    Prunes Gaussians based on opacity, and optionally visibility and depth.
    
    Args:
        gaussians: List of Gaussian objects with attributes .opacity, .position, .scale.
        threshold: Float, opacity threshold below which Gaussians are pruned.
        visibility_mask: Optional 2D array, 1 where visible, 0 where masked.
        depth_threshold: Optional float, max depth for valid Gaussians.
        camera_intrinsics: Optional 3x3 matrix for projection.
        camera_pose: Optional 4x4 matrix for camera-to-world transform.
        
    Returns:
        List of pruned Gaussians.
    """
    pruned_gaussians = []
    
    for g in gaussians:
        # Step 1 & 2: Opacity check
        if g.opacity <= threshold:
            continue
            
        # Step 3: Visibility mask check (if provided)
        if visibility_mask is not None:
            # Project Gaussian center to 2D image plane using standard pinhole camera model
            if camera_intrinsics is not None and camera_pose is not None:
                world_pos = (camera_pose[:3, :3] @ g.position.reshape(3, 1)).flatten() + camera_pose[:3, 3]
                cam_pos = camera_intrinsics @ world_pos
                u, v = cam_pos[0] / cam_pos[2], cam_pos[1] / cam_pos[2]
                # Check if projection falls within masked (invalid) regions
                if not (0 <= u < visibility_mask.shape[1] and 0 <= v < visibility_mask.shape[0]):
                    continue
                if visibility_mask[int(v), int(u)] == 0:
                    continue
            else:
                # Fallback: skip visibility check if camera parameters are missing
                pass
                    
        # Step 4: Depth check (if provided)
        if depth_threshold is not None:
            if camera_pose is not None:
                world_pos = (camera_pose[:3, :3] @ g.position.reshape(3, 1)).flatten() + camera_pose[:3, 3]
                depth = world_pos[2] # Assuming Z-forward camera space
                if depth > depth_threshold:
                    continue
            else:
                # Fallback: skip depth check if camera parameters are missing
                pass
                    
        pruned_gaussians.append(g)
            
    return pruned_gaussians
```

**Parameters:**
- $\tau_\alpha$: Opacity threshold. Not given in the sources.
- Visibility Mask: Binary image indicating valid viewing regions [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Floaters:** Low opacity Gaussians are often "floaters" (noise artifacts), so pruning them is generally beneficial for visual quality [S1].
- **Learnable Masking:** Hard thresholding can be suboptimal. Some methods use a learnable binary mask with a straight-through estimator to dynamically determine pruning, combined with a sparsity-inducing loss [S1].
- **Occlusion Sensitivity:** Removing Gaussians with low opacity may eliminate essential geometry if the Gaussian is occluded or viewed from an angle where its contribution is naturally low, rather than being insignificant [G].

## Subtopics

### [Moderate Pruning](15413-moderate-pruning.md)
Combines opacity thresholds with a multi-view, loss-driven contribution score to prevent the removal of geometrically significant but low-opacity Gaussians.

### [Adaptive Opacity Pruning](15576-adaptive-opacity-pruning.md)
Uses a percentile-based adaptive threshold constrained by an upper limit, rather than a fixed value, to dynamically adjust pruning during training.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians, as it requires a single pass to evaluate opacity [G].
- **Memory Complexity:** $O(N)$ for the Gaussian set; pruning reduces this by removing inactive entries [G].
- **Strengths:** Effective at removing noise and floaters; simple to implement [S1].
- **Limitations:** Can remove valid geometry if the threshold is too high or if Gaussians are occluded [G].
- **When to Use:** As a baseline pruning step, often in combination with gradient-based pruning to ensure both noise and redundancy are removed [S1].

## Background
Opacity Pruning originated as a simple heuristic in 3D Gaussian Splatting optimization to manage the explosion of Gaussian primitives during densification [G]. It was notably employed in methods like "Trimming the Fat" and ELMGS, often in conjunction with gradient-based pruning to improve compression and visual quality [S1]. Variations include learnable opacity masking (CompGS) and volume-based masking (Compact 3D), which integrate pruning into the training loop via differentiable masks [S1].

## Sources
[S1] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
[S2] GauStudio: A Modular Framework for 3D Gaussian Splatting and Beyond, https://doi.org/10.48550/arXiv.2403.19632, fetched 2026-10-30 (unofficial)
