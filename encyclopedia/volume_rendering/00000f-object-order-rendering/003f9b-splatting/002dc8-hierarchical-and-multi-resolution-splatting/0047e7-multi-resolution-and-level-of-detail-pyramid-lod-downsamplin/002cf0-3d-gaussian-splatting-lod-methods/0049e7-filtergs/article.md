# FilterGS

Also called: Adaptive Shrinking, Parallel filtering structure for Gaussian scale selection, Traversal-Free Parallel Filtering

## Overview
FilterGS is a rendering acceleration method for large-scale Level-of-Detail (LoD) 3D Gaussian Splatting that eliminates inefficient serial tree traversal and reduces redundant Gaussian-tile pairs through parallel filtering and adaptive shrinking [S2, unofficial]. It differs from its siblings, such as [Hierarchical 3D Gaussian Splatting (H-3DGS)](10279-hierarchical-3d-gaussian-splatting-h-3dgs.md) and [Sequential Point Trees (SPTs)](10290-sequential-point-trees-spts.md), by replacing depth-dependent serial traversal with a traversal-free, parallel filtering structure that selects a single scale for each Gaussian via a learned network, avoiding expensive tree traversal at render time [S1, unofficial][S2, unofficial].

## How it works
FilterGS operates in two phases: a pre-rendering preprocessing phase to compute scene-adaptive parameters, and a real-time rendering phase involving parallel filtering and adaptive shrinking.

### Inputs and Outputs
**Inputs:**
*   A hierarchical LoD Gaussian tree model trained on a set of images and SfM points [S2, unofficial].
*   Camera parameters for the current view.
*   Pre-computed Global Tile Coverage (GTC) metric $\bar{G}$ and shrinking threshold $\tau$ derived from a pre-rendering pass [S2, unofficial].

**Outputs:**
*   A set of filtered, adaptively scaled 2D Gaussians ready for standard $\alpha$-blending rasterization [S2, unofficial].

### Data Structures
The method relies on the canonical recursive construction of the LoD Gaussian tree [S2, unofficial].

**Node Structure ($N_{i,j}$):**
Each node at level $i$ and index $j$ encapsulates a full Gaussian primitive with parameters:
*   Center position $\mu$
*   Covariance matrix $\Sigma$ (or rotation $q$ and scale $s$)
*   Opacity $\alpha$
*   Color $c$ (represented by spherical harmonics coefficients)
*   Ancestor Path $AP_{i,j}$: An ordered list of serial numbers of all ancestor nodes from the direct parent up to the root node [S2, unofficial].

**Tree Construction:**
For a parent node $v$ with rotation matrix $R(q_v)$ and scale $s_v$, children $u_k$ are placed using oriented offsets $o_k \in \{ \pm \frac{x_v}{2}, \pm \frac{y_v}{2}, \pm \frac{z_v}{2} \}$:
$$ \mu_{uk} = \mu_v + R(q_v) o_k $$
Children shrink geometrically:
$$ s_{uk} = \gamma s_v \quad \text{with} \quad 0 < \gamma < 1 $$
Children inherit orientation, color, and opacity:
$$ q_{uk} = q_v, \quad c_{uk} = c_v, \quad \alpha_{uk} = \alpha_v $$
where $\gamma$ is the geometric shrinking factor [S2, unofficial].

### Numbered Steps

1.  **Preprocessing (Pre-rendering Pass):**
    *   Perform a pre-rendering pass over all training views following the standard 3DGS pipeline [S2, unofficial].
    *   Compute the Key-value Pair Contribution (KPC) for Gaussian-tile pairs to quantify redundancy. KPC is defined as the expected number of pixels influenced by a Gaussian weighted by blended opacity:
        $$ K_{g_k \to t_i} = \frac{1}{n_{gs}} \sum_{j=1}^{n_{gs}} \alpha_{ij} T_{ij} $$
        where $n_{gs}$ is the number of Gaussians influencing tile $t_i$, $\alpha_{ij}$ is the opacity of Gaussian $g_k$ at pixel $j$, and $T_{ij}$ is the transmittance before $g_k$ at that pixel [S2, unofficial].
    *   Compute the Gaussian to Tile Contribution (GTC) $\bar{G}_{t_i}$ as the average KPC of all Gaussians influencing tile $t_i$:
        $$ \bar{G}_{t_i} = \frac{1}{n_{gs}} \sum_{j=1}^{n_{gs}} K_{g_j \to t_i} $$
    *   Aggregate GTC across tiles and views to derive the Global Tile Coverage $\bar{G}$ [S2, unofficial].
    *   Derive the scene-adaptive shrinking threshold $\tau = f(\bar{G})$. A low $\bar{G}$ indicates congestion, necessitating a more aggressive pruning strategy [S2, unofficial].

2.  **Parallel Filtering Step 1 (R&L Filter):**
    *   Project all Gaussians inside the view frustum into 2D Gaussians [S2, unofficial].
    *   For every in-frustum Gaussian, compute the 2D covariance $\Sigma_{2D}$ and its eigenvalues $\sigma$ representing standard deviations along screen-space principal axes [S2, unofficial].
    *   Define the pixel radius $R_{2D} = 3\sigma$ [S2, unofficial].
    *   Retain any node where $R_{2D} \le \tau_R$, where $\tau_R$ is the radius threshold (default 3) [S2, unofficial].
    *   Exempt all leaf nodes from the pruning threshold to prevent holes in the rendered image; preserve every leaf node at least once [S2, unofficial].

3.  **Parallel Filtering Step 2 (Ancestor Filter):**
    *   For every node $N_{i,j}$ that passed R&L filtering, traverse its pre-computed ancestor path $AP_{i,j}$ [S2, unofficial].
    *   If any ancestor node $N_{i^*,j^*}$ (where $N_{i^*,j^*} \in AP_{i,j}$) is an internal node that passed R&L filtering, cull the descendant node $N_{i,j}$ [S2, unofficial].
    *   This ensures that branches with a valid coarser node retain only a single, highest-level Gaussian node [S2, unofficial].

4.  **Adaptive Shrinking:**
    *   During Axis-Aligned Bounding Box (AABB) formation, adaptively scale the filtered Gaussians [S2, unofficial].
    *   Compute the radius $r$ at which opacity decays to threshold $\tau$:
        $$ r = \sqrt{2\sigma_{max} \ln(\alpha_0 / \tau)} $$
        where $\sigma_{max}$ is the maximum standard deviation and $\alpha_0$ is the central opacity [S2, unofficial].
    *   Scale the Gaussians by the pre-computed factor $\tau$ (controlled by scaling factor $\lambda_G$, default 0.2) to reduce redundant key-value pairs before sorting and $\alpha$-blending [S2, unofficial].

5.  **Rendering:**
    *   Render the image by blending the filtered and scaled 2D Gaussians:
        $$ C = \sum_{i \in N} c_i \alpha_i T_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c_i$ is color, $\alpha_i$ is opacity, and $T_i$ is residual transmittance [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def filtergs_render(gaussian_tree, camera, precomputed_tau, tau_R=3.0, lambda_G=0.2):
    """
    FilterGS rendering pipeline.
    
    Args:
        gaussian_tree: Hierarchical LoD Gaussian tree structure.
        camera: Camera parameters for current view.
        precomputed_tau: Scene-adaptive shrinking threshold derived from GTC.
        tau_R: Radius threshold for R&L filter (default 3).
        lambda_G: Scaling factor for shrinking (default 0.2).
        
    Returns:
        Rendered image.
    """
    # Step 1: Project Gaussians in frustum
    in_frustum_gaussians = project_to_2d(gaussian_tree, camera)
    
    # Step 2: Parallel R&L Filter
    # Compute 2D covariance and eigenvalues for all in-frustum Gaussians
    # This is done in parallel across all levels
    retained_nodes = []
    for node in in_frustum_gaussians:
        sigma_2d = compute_2d_covariance_eigenvalues(node)
        sigma_max = max(sigma_2d)
        R_2D = 3 * sigma_max
        
        # Retain if radius is small enough OR if it is a leaf node
        if R_2D <= tau_R or node.is_leaf():
            retained_nodes.append(node)
            
    # Step 3: Parallel Ancestor Filter
    # Pre-computed ancestor paths AP are used to cull descendants of qualified ancestors
    final_nodes = []
    for node in retained_nodes:
        # Check if any ancestor in the path was retained as an internal node
        has_retained_ancestor = False
        for ancestor_id in node.ancestor_path:
            ancestor = gaussian_tree.get_node(ancestor_id)
            # Check if ancestor was retained by R&L filter and is internal
            if ancestor.is_internal() and is_retained_by_rl_filter(ancestor):
                has_retained_ancestor = True
                break
        
        if not has_retained_ancestor:
            final_nodes.append(node)
            
    # Step 4: Adaptive Shrinking
    # Scale Gaussians based on central opacity and precomputed tau
    shrunk_gaussians = []
    for node in final_nodes:
        alpha_0 = node.opacity
        sigma_max = max(node.covariance_eigenvalues)
        
        # Calculate radius where opacity decays to threshold
        # Note: The formula uses ln(alpha_0 / tau). 
        # If alpha_0 <= tau, the radius might be undefined or zero; handle edge case.
        if alpha_0 > precomputed_tau:
            r = sqrt(2 * sigma_max * log(alpha_0 / precomputed_tau))
            # Apply shrinking factor lambda_G to the scale/radius
            node.scale *= (1.0 - lambda_G) # [G] Implementation detail: scaling logic inferred from context
            shrunk_gaussians.append(node)
        else:
            # If opacity is already below threshold, it contributes negligibly
            # [G] Conservative approach: keep but mark as low priority, or cull.
            # Sources say "adaptively scale... to reduce redundant pairs".
            # We assume scaling reduces effective radius.
            node.scale *= (1.0 - lambda_G)
            shrunk_gaussians.append(node)
            
    # Step 5: Standard Rasterization
    # Sort and blend
    image = alpha_blend(shrunk_gaussians, camera.resolution)
    return image

def is_retained_by_rl_filter(node):
    """Helper to check if a node passed R&L filter."""
    # In a real implementation, this would be a boolean flag set during Step 2
    return node.rl_filter_passed
```

### Parameters
*   $\gamma$: Geometric shrinking factor for child node scales in the LoD tree, must satisfy $0 < \gamma < 1$ [S2, unofficial].
*   $\tau_R$: The radius threshold for pruning. Default value is 3. Smaller values retain finer primitives [S2, unofficial].
*   $\lambda_G$: Scaling factor controlling the aggressiveness of shrinking. Default value is 0.2 [S2, unofficial].
*   Redundancy Threshold: Key-value pairs with KPC below 0.01 are defined as redundant [S2, unofficial].
*   Resolution: All renderings are performed at 1080p resolution [S2, unofficial].

### Edge Cases and Pitfalls
*   **Holes in Rendered Image:** Directly applying the radius pruning criterion may incorrectly remove entire branches composed of Gaussians with uniformly large radii. This is mitigated by exempting leaf nodes from the pruning threshold [S2, unofficial].
*   **Congestion:** Low GTC indicates Gaussian congestion, where a tile contains many Gaussians with negligible individual contributions. The scene-adaptive threshold $\tau$ must be derived carefully to avoid over-pruning in sparse regions or under-pruning in congested ones [S2, unofficial].
*   **Fixed Threshold Inadequacy:** Using a fixed threshold (e.g., $\tau = 1/255$ as in FlashGS) is overly conservative for congested scenes, leaving many redundant Gaussians untouched. The adaptive approach is necessary for large-scale scenes [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Filtering time scales with the total number of Gaussians $N$ and is independent of the hierarchical depth $L$ (Depth Decoupling) [S2, unofficial].
    $$ T_{parallel} = T_{synch.} + T_{calcu.}(N) $$
    where $T_{synch.}$ is the fixed overhead of kernel launch and synchronization, and $T_{calcu.}(N)$ is the time for radius estimation and filtering of all $N$ in-frustum Gaussians [S2, unofficial].
*   **Comparison to Serial Methods:** Unlike LoG strategy which uses serial level-by-level traversal requiring steps proportional to tree depth ($T_{serial} = \sum_{\ell=0}^{L-1} (T_{calcu.}(n_\ell) + T_{synch.})$), FilterGS completes parallel filtering in just two steps regardless of tree depth [S2, unofficial].
*   **Performance:** FilterGS achieves state-of-the-art rendering speeds while maintaining competitive visual quality across multiple large-scale datasets [S2, unofficial]. It significantly reduces Gaussian preparation time $t_f$ prior to rasterization [S2, unofficial].
*   **Strengths:** Eliminates traversal overhead, maximizes GPU concurrency, reduces redundant Gaussian-tile pairs.
*   **Limitations:** Requires a pre-rendering pass to compute GTC and $\tau$. Relies on the accuracy of the learned network or heuristic for scale selection [S1, unofficial][S2, unofficial].

## Background
FilterGS was introduced as a method to address critical challenges in large-scale LoD 3D Gaussian Splatting, specifically inefficient serial traversal consuming over 60% of rendering time and redundant Gaussian-tile pairs [S2, unofficial]. It builds on the hierarchical Gaussian tree model where nodes encapsulate full Gaussian primitives, enabling direct rasterization without additional pre-rendering decoding [S2, unofficial]. The method was developed to improve upon existing LoD-based approaches like H-3DGS and LoG by introducing a traversal-free parallel filtering structure [S1, unofficial][S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] FilterGS: Traversal-Free Parallel Filtering and Adaptive Shrinking for Large-Scale LoD 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2603.23891, fetched 2026-10-29 (unofficial)
