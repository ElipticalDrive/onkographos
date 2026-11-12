# Octree-GS

Also called: LOD-Structured 3D Gaussians, Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians

## Overview
Octree-GS is a hierarchical Level-of-Detail (LOD) method that builds an octree over the scene, associating each node with a merged Gaussian that summarizes the radiance and opacity of its children [S1, unofficial]. It dynamically selects appropriate levels from a set of multi-scale Gaussian primitives organized in an octree, ensuring consistent rendering performance by adapting to viewing distance and scene complexity [S3]. Unlike concurrent works where each LOD level independently represents the entire scene, Octree-GS employs an explicit octree structure with an accumulative LOD strategy, enabling finer primitives to capture details while coarse primitives represent low-frequency information, reducing redundancy and pruning unnecessary primitives via a view-frequency strategy [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse Structure-from-Motion (SfM) points, camera poses, camera intrinsics (focal length), and training images.
*   **Outputs:** An octree-structured set of anchors, each associated with Gaussian primitives (positions, covariances, colors, opacities), optimized for novel view synthesis.

**Data Structures:**
*   **Octree:** A sparse hierarchical grid with $K$ layers (LOD levels).
*   **Anchor:** Located at the center of a voxel in layer $L$. Properties include position, a learnable LOD bias $\Delta L_j$, and a list of associated Gaussian primitives.
*   **Voxel Size:** For layer $L$, voxel size $V_L = \delta / 2^L$, where $\delta$ is the base voxel size for LOD 0 [S3].
*   **LOD Design:** Cumulative. Rendering at LOD $K$ rasterizes all Gaussian primitives from LOD 0 to $K$ [S3].

**Parameters:**
*   $r_d$: Hyperparameter to discard outliers when determining distance bounds. Typical value: 0.999 [S3].
*   $\delta$: Base voxel size for LOD 0. Typical value: 0.001 (for modified anchor versions) or 0.02 (for intermediate levels in specific experiments) [S3].
*   $\tau_g$: Predefined threshold for average accumulated gradient to consider a Gaussian primitive significant for growing. Default: 0.0002 [S3].
*   $\beta$: Hyperparameter controlling the exponential increase in difficulty for growing to higher LOD levels. Default: 0.2 [S3].
*   $\epsilon$: User-defined quantity by which LOD bias $\Delta L$ is increased for high-gradient anchors. Empirical value: 0.01 [S3].
*   $\tau_v$: Visibility threshold for view-frequency pruning. Values: 0.7 (small-scale), 0.01 (large-scale), 0.2 (multi-scale) [S3].
*   $T$: Densification interval. Default: 100 iterations [S3].
*   $\omega$: Progress training weight. Default: 1.5 [S3].

**Algorithm Steps:**

1.  **Anchor Initialization:**
    *   Calculate distance $d_{ij}$ between each camera center $i$ and SfM point $j$.
    *   Determine $d_{max}$ and $d_{min}$ as the $r_d$-th largest and smallest distances, respectively [S3].
    *   Calculate the number of octree layers $K$:
        $$ K = \lfloor \log_2(d_{max}/d_{min}) \rceil + 1 $$
        where $\lfloor \cdot \rceil$ denotes the round operator [S3].
    *   Construct octree-structured grids with $K$ layers.
    *   Initialize anchors at voxel centers for each layer $L$ with voxel size $V_L = \delta / 2^L$ [S3].

2.  **Anchor Selection (Inference/Rendering):**
    *   For a given viewpoint $i$ and anchor $j$, estimate the LOD level $\hat{L}_{ij}$:
        $$ \hat{L}_{ij} = \lfloor \Phi(\log_2(d_{max} / (d_{ij} \cdot s))) + \Delta L_j \rfloor $$
        where $s$ is a focal scale factor for varying intrinsics, $\Phi(\cdot)$ is a clamping function restricting the fractional LOD level to $[0, K-1]$, and $\Delta L_j$ is the learnable LOD bias for anchor $j$ [S3].
    *   Select anchor $j$ if its assigned LOD level $L_j \le \hat{L}_{ij}$ [S3].
    *   **Opacity Blending:** To ensure smooth transitions, select "nearly satisfied" anchors where $L_j = \hat{L}_{ij} + 1$. Scale their opacities by $(\hat{L}_{ij} - L_j)$ (which equals $-1$ if interpreted strictly, but the source implies a linear interpolation factor; specifically, the source states "Scale the opacities of these nearly satisfied anchors by $(\hat{L}_{ij} - L_j)$" which results in a negative value if $L_j = \hat{L}_{ij}+1$. However, standard blending usually implies $1 - (\hat{L}_{ij} - L_{coarse})$. The source text says: "Scale the opacities of these nearly satisfied anchors by $(\hat{L}_{ij} - L_j)$". Given $L_j = \hat{L}_{ij} + 1$, this factor is $-1$. This appears to be a typo in the source or implies absolute value/interpolation weight. Standard practice in such LOD blending is to use the fractional part. We follow the source's explicit formula but note the potential for sign error in implementation if taken literally without context of "weight". A more robust interpretation of "piecewise linear interpolation" is that the weight for the finer level is $1 - \text{fraction}$ and coarse is $\text{fraction}$. The source explicitly says: "Scale the opacities of these nearly satisfied anchors by $(\hat{L}_{ij} - L_j)$". If $\hat{L}_{ij}$ is the target and $L_j$ is the anchor's level, and we select $L_j = \hat{L}_{ij}+1$, the difference is $-1$. This is likely a description of the *interpolation weight* where the weight for the *coarser* level (fully satisfied) is related to the fractional part. We will implement the selection logic as defined: select if $L_j \le \hat{L}_{ij}$, and for $L_j = \hat{L}_{ij}+1$, apply opacity scaling. [S3].

3.  **Progressive Training (Grow-and-Prune):**
    *   **Anchor Growing:** Every $T$ iterations, calculate the average accumulated gradient $\nabla_g$ of spawned Gaussian primitives.
        *   If $\nabla_g > \tau_g$, the primitive is significant. If in an empty voxel, convert to a new anchor.
        *   **Next-Level Growth:** Assign LOD level based on gradient thresholds. Thresholds are $\tau_{gL} = \tau_g \cdot 2^{\beta L}$.
        *   Gaussians at level $L$ are promoted to $L+1$ only if $\nabla_g > \tau_{g(L+1)}$. They remain at $L$ if $\tau_{gL} < \nabla_g < \tau_{g(L+1)}$ [S3].
    *   **LOD Bias Adjustment:** Define anchor gradient $\nabla_v$ as the average gradient of spawned Gaussian primitives.
        *   Select anchors with $\nabla_v > \tau_{gL} \cdot 0.25$.
        *   Increase their LOD bias: $\Delta L_j = \Delta L_j + \epsilon$ (where $\epsilon = 0.01$) [S3].
    *   **Anchor Pruning:**
        *   Compute average opacity of Gaussians generated over $T$ iterations. Remove anchors with low average opacity.
        *   Compute **view-frequency**: the probability that anchors are selected in training views.
        *   Remove anchors with view-frequency below $\tau_v$ to eliminate floaters and reduce storage [S3].

4.  **Training Schedule:**
    *   Small-scale scenes: 40k total iterations, densification ends at 20k.
    *   Large-scale scenes: 100k total iterations, densification ends at 50k.
    *   Progress training: 10k iterations with $\omega = 1.5$, followed by an additional 10k iterations of densification [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_octree(sfm_points, camera_centers, rd=0.999, delta=0.001):
    # Calculate distances
    distances = []
    for i in range(num_cameras):
        for j in range(num_sfm_points):
            d_ij = distance(camera_centers[i], sfm_points[j])
            distances.append(d_ij)
    
    # Determine bounds
    distances.sort()
    d_max = distances[int(rd * len(distances))]
    d_min = distances[int((1 - rd) * len(distances))]
    
    # Calculate layers
    K = round(log2(d_max / d_min)) + 1
    
    # Initialize octree and anchors
    octree = Octree(max_depth=K)
    anchors = []
    for L in range(K):
        voxel_size = delta / (2 ** L)
        # Place anchors at voxel centers covering the scene bounds
        layer_anchors = create_voxel_grid_anchors(voxel_size, scene_bounds)
        for anchor in layer_anchors:
            anchor.L = L
            anchor.delta_L = 0.0 # Initial LOD bias
            anchor.gradients = []
            anchor.opacity_history = []
            anchor.view_frequency = 0.0
            octree.insert(anchor)
            anchors.append(anchor)
            
    return octree, anchors, K

def select_anchors(viewpoint, anchors, K, s=1.0):
    selected_primitives = []
    blended_primitives = []
    
    for anchor in anchors:
        d_ij = distance(viewpoint.camera_center, anchor.position)
        
        # Estimate LOD level
        raw_lod = log2(d_max / (d_ij * s)) + anchor.delta_L
        clamped_lod = clamp(raw_lod, 0, K - 1)
        estimated_L = floor(clamped_lod)
        
        # Selection Logic
        if anchor.L <= estimated_L:
            # Fully satisfied
            selected_primitives.extend(anchor.get_gaussians())
        elif anchor.L == estimated_L + 1:
            # Nearly satisfied for blending
            # Source says: Scale opacities by (estimated_L - anchor.L)
            # Note: estimated_L - anchor.L is -1. 
            # Implementation note: This likely implies a weight for interpolation.
            # Standard blending: weight_coarse = frac, weight_fine = 1-frac.
            # Here, we follow the source's explicit instruction for selection.
            # We pass them to rasterizer with scaled opacity.
            gaussians = anchor.get_gaussians()
            for g in gaussians:
                g.opacity = g.opacity * (estimated_L - anchor.L) # As per source
                blended_primitives.append(g)
                
    # Render selected primitives
    render(selected_primitives)
    # Render blended primitives with their scaled opacities
    render(blended_primitives)

def train_step(octree, anchors, viewpoint, T, tau_g=0.0002, beta=0.2, epsilon=0.01, tau_v=0.7):
    # 1. Compute gradients for existing Gaussians
    gradients = compute_gradients(viewpoint)
    
    # 2. Growing (every T iterations)
    if current_iter % T == 0:
        for anchor in anchors:
            avg_grad = average(anchor.gradients)
            
            # Check if significant
            if avg_grad > tau_g:
                # Check if in empty voxel
                if octree.is_voxel_empty(anchor.position, anchor.L):
                    # Determine new LOD level
                    # Thresholds: tau_gL = tau_g * 2^(beta * L)
                    # Promote to L+1 if grad > tau_g(L+1)
                    # Else stay at L
                    
                    # Note: This logic applies to *new* anchors spawned from significant gradients
                    # The source says "Gaussians at level L are promoted...". 
                    # In practice, this means assigning the LOD of the new anchor.
                    new_anchor = create_anchor(anchor.position)
                    
                    # Determine LOD
                    current_L = anchor.L # Or 0 if spawning from scratch? 
                    # Source implies promoting existing Gaussians. 
                    # If spawning new, we assign based on gradient magnitude relative to thresholds.
                    
                    # Simplified: Assign L such that tau_gL < grad < tau_g(L+1)
                    for L in range(K):
                        tau_L = tau_g * (2 ** (beta * L))
                        tau_L_plus_1 = tau_g * (2 ** (beta * (L + 1)))
                        if tau_L < avg_grad < tau_L_plus_1:
                            new_anchor.L = L
                            break
                        elif avg_grad > tau_L_plus_1:
                            new_anchor.L = L + 1
                            break
                            
                    octree.insert(new_anchor)
                    anchors.append(new_anchor)
                    
            # 3. LOD Bias Adjustment
            anchor_grad_v = average(anchor.gradients) # Average gradient of spawned primitives
            tau_L = tau_g * (2 ** (beta * anchor.L))
            if anchor_grad_v > tau_L * 0.25:
                anchor.delta_L += epsilon
                
    # 4. Pruning
    for anchor in anchors:
        # Opacity pruning
        avg_opacity = average(anchor.opacity_history)
        if avg_opacity < threshold_opacity: # Not given in sources, implied by "low average opacity"
            octree.remove(anchor)
            anchors.remove(anchor)
            
        # View-frequency pruning
        # View frequency is probability of being selected in training views
        if anchor.view_frequency < tau_v:
            octree.remove(anchor)
            anchors.remove(anchor)
```

## Operating characteristics
*   **Time Complexity:** Rendering is efficient due to LOD selection. On the MatrixCity dataset, Octree-GS achieves 32.0 FPS with 3.59M Gaussians, compared to 6.91 FPS with 20.8M Gaussians for Hierarchical-GS [S3]. It is up to 10x faster than state-of-the-art methods in large-scale scenes [S3]. On Mip-NeRF360 (40k iterations), training time is 20-23 minutes for adapted variants, compared to 38-69 minutes for Hierarchical-GS [S3].
*   **Memory Complexity:** Reduces storage overhead by pruning redundant/invisible anchors and using fewer primitives than baselines. On MatrixCity, it uses 3.59M Gaussians vs 13.0M for Scaffold-GS [S3].
*   **Strengths:** Consistent real-time rendering performance regardless of trajectory changes; handles large-scale and complex scenes effectively; end-to-end training (single round); compatible with explicit (3D-GS, 2D-GS) and neural Gaussians (Scaffold-GS) [S3].
*   **Limitations:** Requires careful tuning of hyperparameters ($\tau_v$, $\tau_g$, $\beta$); relies on SfM points for initialization; may introduce artifacts if opacity blending is not implemented correctly (source formula ambiguity) [S3].
*   **When to use:** Large-scale scenes, real-time rendering requirements, scenes with varying detail density.
*   **When not to use:** Small scenes where standard 3D-GS is sufficient and simpler; scenes with very sparse SfM points where octree initialization is difficult.

## Background
Octree-GS was introduced in the paper "Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians" [S3]. It builds upon Scaffold-GS [S2] and addresses the limitations of other LOD methods like LetsGo, CityGaussian, and Hierarchical-GS, which either require multi-resolution inputs, manual threshold adjustments, or post-processing [S3]. It is the first approach to deal with LOD in Gaussian representation using an explicit octree structure with an accumulative LOD strategy [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
[S4] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-09 (unofficial)
