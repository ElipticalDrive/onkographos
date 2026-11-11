# Octree-GS

Also called: LOD-Structured 3D Gaussians, Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians

## Overview
Octree-GS is a Level-of-Detail (LOD) structured method for 3D Gaussian Splatting that dynamically selects appropriate detail levels from a multi-scale Gaussian primitive set organized in a sparse octree [S3]. It solves the problem of inconsistent rendering performance and excessive memory usage in large-scale scenes by adapting the number of primitives to viewing distance and scene complexity [S3]. Unlike concurrent methods that require post-processing or manual threshold tuning, Octree-GS achieves LOD effects in a single end-to-end training round using an accumulative LOD strategy, where rendering at level $K$ includes all primitives from levels $0$ to $K$ [S3].

## How it works
### Inputs and Outputs
**Inputs:**
- Sparse Structure-from-Motion (SfM) points.
- Training images with known camera poses and intrinsics.
- Hyperparameters: base voxel size $\delta$, outlier discard ratio $r_d$, visibility threshold $\tau_v$, gradient threshold $\tau_g$, growth difficulty factor $\beta$, bias increment $\epsilon$, densification interval $T$, and progress training weight $\omega$ [S3].

**Outputs:**
- An octree-structured set of anchors, each associated with a LOD level $L_j$ and a learnable LOD bias $\Delta L_j$.
- Gaussian primitives (explicit or neural) attached to selected anchors.

### Data Structures
**Anchor:**
- Position: Center of a voxel in the octree grid.
- LOD Level: Integer $L_j \in [0, K-1]$.
- LOD Bias: Learnable scalar $\Delta L_j$.
- Gaussians: A set of $k$ Gaussian primitives associated with the anchor.

**Octree Grid:**
- Hierarchical structure with $K$ layers.
- Layer $L$ has voxel size $V_L = \delta / 2^L$ [S3].
- LOD 0 is the coarsest level (largest voxels); higher $L$ implies finer detail (smaller voxels) [S3].

### Algorithm Steps

#### 1. Anchor Initialization
1. Calculate the distance $d_{ij}$ between each training camera center $i$ and each SfM point $j$ [S3].
2. Determine $d_{max}$ and $d_{min}$ as the $r_d$-th largest and smallest distances, respectively, to discard outliers. Typical $r_d = 0.999$ [S3].
3. Calculate the number of octree layers $K$:
   $$ K = \lfloor \log_2(d_{max} / d_{min}) \rceil + 1 $$
   where $\lfloor \cdot \rceil$ denotes the nearest integer rounding operator [S3].
4. Construct $K$ layers of sparse uniform voxel grids. The voxel size for layer $L$ is:
   $$ V_L = \delta / 2^L $$
   where $\delta$ is the base voxel size for LOD 0 [S3].
5. Initialize anchors at the center of each voxel in the grids. Assign initial LOD level $L_j = L$ for anchors in layer $L$. Set initial $\Delta L_j = 0$ [S3].

#### 2. Progressive Training with Grow-and-Prune
Training is divided into a progressive phase and a densification phase. Densification occurs every $T$ iterations (default $T=100$) [S3].

**Anchor Growing:**
1. Calculate the average accumulated gradient of spawned Gaussian primitives, $\nabla_g$ [S3].
2. If $\nabla_g > \tau_g$ (default $0.0002$), the primitive is significant. If it resides in an empty voxel, convert it into a new anchor [S3].
3. Assign LOD level using the 'next-level' growth operator:
   - Define threshold for level $L$ as $\tau_{gL} = \tau_g \cdot 2^{\beta L}$ (default $\beta = 0.2$) [S3].
   - If $\nabla_g > \tau_{g(L+1)}$, promote the anchor to level $L+1$.
   - If $\tau_{gL} < \nabla_g < \tau_{g(L+1)}$, keep the anchor at level $L$ [S3].
4. Adjust LOD bias $\Delta L_j$ for high-complexity regions:
   - Calculate anchor gradient $\nabla_v$ (average gradient of spawned primitives for anchor $j$) [S3].
   - If $\nabla_v > \tau_{gL} \cdot 0.25$, update $\Delta L_j \leftarrow \Delta L_j + \epsilon$ (default $\epsilon = 0.01$) [S3].

**Anchor Pruning:**
1. Compute the average opacity of Gaussians generated over the last $T$ iterations [S3].
2. Remove anchors with low average opacity [S3].
3. Calculate 'view-frequency' for each anchor: the probability that the anchor is selected in training view frustums [S3].
4. Remove anchors with view-frequency below visibility threshold $\tau_v$ (typical values: $0.7$ for small-scale, $0.01$ for large-scale, $0.2$ for multi-scale) [S3].

**Progressive Training Strategy:**
1. Run initial training for $10k$ iterations with weight $\omega = 1.5$ [S3].
2. Extend densification by an additional $10k$ iterations [S3].
3. Total iterations: $40k$ (densification ends at $20k$) for small-scale scenes; $100k$ (densification ends at $50k$) for large-scale scenes [S3].

#### 3. Inference and Rendering
1. For a given viewpoint $i$ and anchor $j$, calculate distance $d_{ij}$ [S3].
2. If camera intrinsics vary, apply focal scale factor $s$ to adjust distance: $d'_{ij} = d_{ij} \cdot s$ [S3].
3. Estimate the target LOD level $\hat{L}_{ij}$:
   $$ \hat{L}_{ij} = \lfloor \Phi(\log_2(d_{max} / d'_{ij})) + \Delta L_j \rfloor $$
   where $\Phi(\cdot)$ clamps the fractional LOD level to $[0, K-1]$ [S3].
4. **Anchor Selection:**
   - Select anchor $j$ if $L_j \le \hat{L}_{ij}$ (fully satisfied) [S3].
   - Also select anchors where $L_j = \hat{L}_{ij} + 1$ (nearly satisfied) for smooth transitions [S3].
5. **Opacity Blending:**
   - For nearly satisfied anchors, scale their opacities by $(\hat{L}_{ij} - L_j)$ using piecewise linear interpolation to prevent artifacts [S3].
6. Rasterize the selected Gaussian primitives using standard 3D Gaussian Splatting alpha compositing [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def initialize_octree(sfm_points, camera_centers, delta, rd=0.999):
    """
    Initialize the octree structure and anchors.
    """
    # Calculate distances between all camera centers and SfM points
    distances = []
    for cam in camera_centers:
        for pt in sfm_points:
            d = distance(cam.position, pt.position)
            distances.append(d)
    
    # Determine d_max and d_min based on rd percentile
    distances.sort()
    n = len(distances)
    d_max = distances[int(rd * n)]
    d_min = distances[int((1 - rd) * n)]
    
    # Calculate number of layers K
    K = round(math.log2(d_max / d_max)) + 1 # Note: Source uses round operator on log2 ratio
    # Correction based on source formula: K = floor(log2(dmax/dmin)) rounded + 1
    # The source says: K = ⌊log2(dmax/dmin)⌉ + 1. ⌊·⌉ denotes round operator.
    K = round(math.log2(d_max / d_min)) + 1
    
    anchors = []
    for L in range(K):
        voxel_size = delta / (2 ** L)
        # Create sparse voxel grid for level L
        grid = SparseVoxelGrid(voxel_size=voxel_size)
        # Initialize anchors at voxel centers
        for voxel in grid.get_occupied_voxels():
            anchor = Anchor()
            anchor.position = voxel.center
            anchor.L = L
            anchor.delta_L = 0.0
            anchor.view_frequency = 0.0
            anchor.opacity_history = []
            anchors.append(anchor)
            
    return anchors, K, d_max

def select_anchors_for_view(anchors, K, d_max, camera_center, focal_scale=1.0, delta_L_j_map=None):
    """
    Select anchors for rendering based on viewing distance and LOD bias.
    """
    selected_anchors = []
    
    for anchor in anchors:
        d_ij = distance(camera_center, anchor.position)
        d_prime_ij = d_ij * focal_scale
        
        # Calculate fractional LOD level
        log_dist = math.log2(d_max / d_prime_ij)
        
        # Clamp to [0, K-1]
        if delta_L_j_map is not None:
            bias = delta_L_j_map.get(anchor.id, 0.0)
        else:
            bias = anchor.delta_L
            
        fractional_L = log_dist + bias
        clamped_L = max(0.0, min(K - 1, fractional_L))
        
        # Floor to get integer target LOD
        target_L = math.floor(clamped_L)
        
        # Selection criteria
        is_fully_satisfied = anchor.L <= target_L
        is_nearly_satisfied = anchor.L == target_L + 1
        
        if is_fully_satisfied or is_nearly_satisfied:
            opacity_scale = 1.0
            if is_nearly_satisfied:
                # Scale opacity by (target_L - L) -> (target_L - (target_L + 1)) = -1?
                # Source says: scale opacities by (L_hat_ij - L_j).
                # If L_j = L_hat_ij + 1, then L_hat_ij - L_j = -1.
                # However, opacity must be positive. The source likely implies absolute difference 
                # or the interpolation factor (1 - fraction). 
                # Re-reading source: "Scale the opacities of these nearly satisfied anchors by (L_hat_ij - L_j)."
                # This results in negative opacity if L_j > L_hat_ij. 
                # Context: "piecewise linear interpolation between adjacent levels".
                # Standard LOD blending usually uses alpha = 1 - frac. 
                # Let's stick to the literal formula but note the sign issue. 
                # Actually, if L_hat is the *target* max level, and we include L_hat+1, 
                # the weight should decrease as we go further away.
                # Let's assume the source meant |L_hat - L_j| or similar, but strictly:
                # Source text: "Scale the opacities ... by (L_hat_ij - L_j)."
                # If L_hat=2, L_j=3. Scale = -1. This is physically invalid for opacity.
                # Likely interpretation: The blending factor is (L_hat - (L_j - 1))? 
                # Or perhaps L_hat is the *current* LOD and we blend with L_hat+1?
                # Given the constraint "implement from text", we must follow the text.
                # However, "opacity" cannot be negative. 
                # Hypothesis: The source implies the weight for the *higher* LOD decreases.
                # Let's use the absolute difference or assume a typo in the source's sign, 
                # but since I cannot guess, I will implement the literal formula 
                # and add a guard [G] to clamp to 0 if negative, as negative opacity is undefined.
                scale = target_L - anchor.L
                if scale < 0:
                    scale = 0 # [G] Guard against negative opacity
                opacity_scale = scale
            else:
                opacity_scale = 1.0
                
            selected_anchors.append((anchor, opacity_scale))
            
    return selected_anchors

def train_step(anchors, gradients, T, tau_g=0.0002, beta=0.2, epsilon=0.01, tau_v=0.01):
    """
    Perform grow and prune operations every T iterations.
    """
    # Growing
    for anchor in anchors:
        # Check if anchor is in empty voxel (simplified check)
        if anchor.is_in_empty_voxel():
            avg_grad = anchor.get_average_gradient()
            if avg_grad > tau_g:
                # Determine new LOD level
                current_L = anchor.L
                threshold_next = tau_g * (2 ** (beta * (current_L + 1)))
                threshold_curr = tau_g * (2 ** (beta * current_L))
                
                if avg_grad > threshold_next:
                    anchor.L = current_L + 1
                # Else remains at current_L
                
                # Adjust bias
                anchor_grad_v = anchor.get_anchor_gradient()
                threshold_bias = tau_g * (2 ** (beta * anchor.L)) * 0.25
                if anchor_grad_v > threshold_bias:
                    anchor.delta_L += epsilon
                    
    # Pruning
    for anchor in anchors:
        # Opacity pruning
        if anchor.get_average_opacity() < threshold_opacity: # Threshold not explicitly named, implied by "low"
            anchor.mark_for_removal()
            
        # View frequency pruning
        if anchor.view_frequency < tau_v:
            anchor.mark_for_removal()
            
    return [a for a in anchors if not a.marked_for_removal()]
```

### Parameters
- $\delta$: Base voxel size for LOD 0. Typical value for intermediate level is $0.02$ [S3].
- $r_d$: Outlier discard ratio. Typical value: $0.999$ [S3].
- $\tau_g$: Gradient threshold for growing. Default: $0.0002$ [S3].
- $\beta$: Growth difficulty factor. Default: $0.2$ [S3].
- $\epsilon$: LOD bias increment. Empirical value: $0.01$ [S3].
- $\tau_v$: Visibility threshold. $0.7$ (small-scale), $0.01$ (large-scale), $0.2$ (multi-scale) [S3].
- $T$: Densification interval. Default: $100$ [S3].
- $\omega$: Progress training weight. Default: $1.5$ [S3].
- $s$: Focal scale factor for varying intrinsics [S3].

### Edge Cases and Pitfalls
- **Negative Opacity Scaling:** The formula for opacity blending of nearly satisfied anchors is given as $(\hat{L}_{ij} - L_j)$. Since $L_j = \hat{L}_{ij} + 1$, this yields $-1$. A guard is required to clamp opacity to $0$ or use absolute value [G].
- **Intrinsic Variation:** When camera intrinsics change, the observation distance must be scaled by $s$ to maintain correct footprint estimation [S3].
- **Aliasing:** Without proper opacity blending, switching between LOD levels can cause stroboscopic effects or aliasing artifacts [S3].

## Operating characteristics
- **Performance:** On the MatrixCity dataset (2k resolution), Octree-GS achieves 32.0 FPS with 3.59M Gaussians in zoom-out views, compared to 6.91 FPS for Hierarchical-GS and 8.68 FPS for Scaffold-GS [S3]. It is up to 10x faster than state-of-the-art methods in large-scale scenes [S3].
- **Training Time:** On Mip-NeRF360 (40k iterations), training takes 20-23 minutes for adapted variants, compared to 38-69 minutes for Hierarchical-GS, due to single-stage LOD construction and fewer primitives [S3].
- **Memory:** Significantly lower Gaussian counts across scales compared to baselines, ensuring consistent rendering speed and lower storage overhead [S3].
- **Strengths:** End-to-end training, no manual threshold tuning, no post-processing, robust to trajectory changes, handles multi-resolution datasets without erosion artifacts [S3].
- **Limitations:** Requires careful tuning of visibility thresholds for different scene scales; cumulative LOD strategy may render more primitives than strictly necessary if bias is not well-calibrated [S3].

## Background
Octree-GS was introduced as a response to the limitations of existing LOD methods in 3D Gaussian Splatting, such as LetsGo, CityGaussian, and Hierarchical-GS, which either require multi-resolution inputs, manual tuning, or post-processing [S3]. It builds upon the Scaffold-GS representation, introducing an explicit octree structure to decouple multi-scale Gaussian primitives into varying LOD levels [S3]. The method was published in IEEE Transactions on Pattern Analysis and Machine Intelligence [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
[S4] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
