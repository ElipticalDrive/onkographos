# FilterGS

Also called: Traversal-Free Parallel Filtering, Adaptive Shrinking, Parallel filtering structure for Gaussian scale selection

## Overview
FilterGS is a rendering acceleration method for large-scale Level-of-Detail (LoD) 3D Gaussian Splatting that eliminates inefficient serial tree traversal and reduces redundant Gaussian-tile pairs through parallel filtering and adaptive shrinking [S2, unofficial]. It addresses the mismatch between discrete sampling and continuous representation by applying a traversal-free, parallel filtering structure that selects a single scale for each Gaussian via a learned network, avoiding expensive tree traversal at render time [S1, unofficial]. Unlike sibling methods that rely on serial level-by-level traversal, FilterGS completes filtering in two parallel steps regardless of tree depth, decoupling computational cost from hierarchical depth [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A hierarchical LoD Gaussian tree model (trained from SfM points and images), a camera view (position, orientation, intrinsics), and a pre-computed scene-adaptive shrinking threshold $\tau$.
*   **Outputs:** A set of filtered 2D Gaussians ready for standard $\alpha$-blending rasterization, with redundant pairs removed and influence radii adaptively shrunk.

**Data Structures**
*   **LoD Gaussian Tree:** Nodes encapsulate full Gaussian primitives.
    *   **Node Parameters:** Center position $\mu$, covariance matrix $\Sigma$, opacity $\alpha$, spherical harmonics coefficients for color $c$, rotation $q$, and scale $s$.
    *   **Hierarchy Construction:** For a parent node $v$ with rotation $R(q_v)$ and scale $s_v$, $K_v$ children are placed using oriented offsets $o_k \in \{\pm x_v/2, \pm y_v/2, \pm z_v/2\}$:
        $$ \mu_{uk} = \mu_v + R(q_v) o_k $$
        Children shrink geometrically: $s_{uk} = \gamma s_v$ with $0 < \gamma < 1$. Children inherit parent orientation, color, and opacity: $q_{uk} = q_v, c_{uk} = c_v, \alpha_{uk} = \alpha_v$ [S2, unofficial].
    *   **Ancestor Path ($AP_{i,j}$):** For every node $N_{i,j}$ at level $i$, a pre-computed ordered list of serial numbers of all ancestor nodes from the direct parent up to the root node [S2, unofficial].

**Algorithm Steps**

1.  **Preprocessing (Offline):**
    *   Perform a pre-rendering pass over all training views following the standard 3DGS pipeline.
    *   Compute the Global Tile Coverage (GTC) metric $\bar{G}$, defined as the average Gaussian-to-Tile Contribution (GTC) across all views.
    *   Derive the scene-adaptive shrinking threshold $\tau = f(\bar{G})$. A low $\bar{G}$ indicates high congestion, necessitating a more aggressive $\tau$ [S2, unofficial].

2.  **Parallel Filtering (Per Frame):**
    *   **Step 1: Radius and Leaf (R&L) Filter:**
        *   Project all Gaussians inside the view frustum into 2D Gaussians.
        *   Compute the 2D covariance $\Sigma_{2D}$ and its eigenvalues $\sigma$ (standard deviations along screen-space principal axes).
        *   Calculate the pixel radius $R_{2D} = 3\sigma$.
        *   **Retention Rule:** Retain any node where $R_{2D} \leq \tau_R$ (default $\tau_R = 3$).
        *   **Leaf Exemption:** Exempt all leaf nodes from the pruning threshold; preserve every leaf node at least once to prevent holes in the rendered image [S2, unofficial].
    *   **Step 2: Ancestor Filter:**
        *   For every node $N_{i,j}$ that passed R&L filtering, traverse its pre-computed ancestor path $AP_{i,j}$.
        *   **Culling Rule:** If any ancestor node $N_{i^*,j^*} \in AP_{i,j}$ is an internal node that passed R&L filtering, cull the descendant node $N_{i,j}$.
        *   This ensures that branches with a valid coarser node retain only a single, highest-level Gaussian node [S2, unofficial].

3.  **Adaptive Shrinking:**
    *   During Axis-Aligned Bounding Box (AABB) formation, adaptively scale the filtered Gaussians.
    *   Calculate the effective influence radius $r$ at which opacity decays to threshold $\tau$:
        $$ r = \sqrt{2\sigma_{max} \ln(\alpha_0 / \tau)} $$
        where $\sigma_{max}$ is the maximum standard deviation and $\alpha_0$ is the central opacity [S2, unofficial].
    *   Scale the Gaussians by the pre-computed factor $\tau$ to reduce redundant key-value pairs before sorting and $\alpha$-blending [S2, unofficial].

4.  **Rasterization:**
    *   Sort the filtered Gaussians by depth.
    *   Compute pixel color $C$ by blending $N$ ordered 2D Gaussians:
        $$ C = \sum_{i \in N} c_i \alpha_i T_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c_i$ is color, $\alpha_i$ is opacity, and $T_i$ is residual transmittance [S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def filtergs_render(tree, camera, tau_shrink, tau_r=3.0):
    """
    tree: Hierarchical LoD Gaussian Tree structure
    camera: Camera parameters for projection
    tau_shrink: Pre-computed scene-adaptive shrinking threshold
    tau_r: Radius threshold for R&L filter (default 3)
    """
    # 1. Project Gaussians in frustum
    in_frustum_gaussians = []
    for node in tree.all_nodes:
        if node.is_in_frustum(camera):
            # Project 3D Gaussian to 2D
            sigma_2d = project_covariance(node.sigma_3d, camera)
            node.sigma_2d = sigma_2d
            in_frustum_gaussians.append(node)

    # 2. Parallel Filtering Step 1: R&L Filter
    # Compute R2D for all nodes in parallel
    for node in in_frustum_gaussians:
        eigenvalues = compute_eigenvalues(node.sigma_2d)
        sigma_max = max(eigenvalues)
        node.r_2d = 3 * sigma_max
        node.passed_rl_filter = False

    # Apply R&L rules
    for node in in_frustum_gaussians:
        if node.is_leaf:
            node.passed_rl_filter = True
        elif node.r_2d <= tau_r:
            node.passed_rl_filter = True

    # 3. Parallel Filtering Step 2: Ancestor Filter
    # Pre-computed ancestor paths are assumed to be stored in tree nodes
    # We need to identify which internal nodes passed R&L to cull descendants
    
    # First, collect all internal nodes that passed R&L
    qualified_internal_nodes = [
        node for node in in_frustum_gaussians 
        if node.passed_rl_filter and not node.is_leaf
    ]
    
    # Create a set of qualified internal node IDs for fast lookup
    qualified_ids = {node.id for node in qualified_internal_nodes}

    # Cull descendants based on ancestor paths
    for node in in_frustum_gaussians:
        if not node.passed_rl_filter:
            continue
        
        # Check if any ancestor is a qualified internal node
        has_qualified_ancestor = False
        for ancestor_id in node.ancestor_path:
            if ancestor_id in qualified_ids:
                has_qualified_ancestor = True
                break
        
        if has_qualified_ancestor:
            node.passed_rl_filter = False # Cull this node

    # 4. Adaptive Shrinking
    final_gaussians = []
    for node in in_frustum_gaussians:
        if node.passed_rl_filter:
            # Calculate shrinking factor based on opacity decay
            # r = sqrt(2 * sigma_max * ln(alpha_0 / tau_shrink))
            # The actual implementation scales the Gaussian parameters
            # to match this effective radius or opacity threshold
            sigma_max = max(compute_eigenvalues(node.sigma_2d))
            alpha_0 = node.alpha
            
            # Compute the radius where opacity decays to tau_shrink
            # This is used to determine the scaling factor for AABB/radius
            if alpha_0 > tau_shrink:
                effective_radius = sqrt(2 * sigma_max * log(alpha_0 / tau_shrink))
                # Scale the Gaussian's influence/radius by tau_shrink factor
                # Note: The exact scaling operation depends on the rasterizer's 
                # AABB construction logic. Here we assume a scaling of the 
                # effective radius or covariance.
                node.scale_shrinking_factor = tau_shrink 
                final_gaussians.append(node)

    # 5. Rasterization
    # Sort by depth
    final_gaussians.sort(key=lambda g: g.depth)
    
    # Alpha blend
    image = zeros_like(target_image)
    for gaussian in final_gaussians:
        # Standard 3DGS rasterization step
        blend_gaussian(image, gaussian)
        
    return image
```

**Parameters**
*   $\tau_R$: Radius threshold for pruning. Default value is 3. Smaller values retain finer primitives [S2, unofficial].
*   $\tau$: Scene-adaptive shrinking threshold derived from GTC. Determined by $\lambda_G = 0.2$ scaling factor in experiments [S2, unofficial].
*   $\gamma$: Geometric shrinking factor for child node scales in the LoD tree, must satisfy $0 < \gamma < 1$ [S2, unofficial].
*   Redundancy Threshold: Key-value pairs with Key-value Pair Contribution (KPC) below 0.01 are defined as redundant [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Holes in Rendering:** Directly applying the radius pruning threshold may remove entire branches of Gaussians with uniformly large radii. The leaf exemption rule is critical to prevent this [S2, unofficial].
*   **Congestion:** In highly congested scenes, a fixed threshold (e.g., $\tau = 1/255$) is overly conservative. The scene-adaptive $\tau$ derived from GTC is necessary to mitigate congestion effectively [S2, unofficial].
*   **Precision:** The calculation of $R_{2D} = 3\sigma$ assumes a standard deviation-based cutoff. Precision in eigenvalue computation for $\Sigma_{2D}$ is required for accurate radius estimation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Filtering time scales with the total number of in-frustum Gaussians $N$ and is independent of the hierarchical depth $L$ (Depth Decoupling) [S2, unofficial].
    $$ T_{parallel} = T_{synch.} + T_{calcu.}(N) $$
    where $T_{synch.}$ is the fixed overhead of kernel launch and synchronization, and $T_{calcu.}(N)$ is the time for radius estimation and filtering [S2, unofficial].
*   **Memory Complexity:** Requires pre-computation of ancestor paths for all nodes, adding static memory overhead proportional to the number of nodes and tree depth [S2, unofficial].
*   **Measured Performance:** Achieves state-of-the-art rendering speeds across multiple large-scale datasets (tested at 1080p resolution on RTX 4090) while maintaining competitive visual quality [S2, unofficial].
*   **Strengths:** Eliminates serial traversal overhead (which can consume >60% of rendering time in LoG methods); maximizes GPU concurrency via parallel processing; reduces redundant Gaussian-tile pairs.
*   **Limitations:** Requires a pre-rendering pass to compute GTC and $\tau$; relies on the correctness of the hierarchical tree structure; performance depends on the efficiency of the parallel filtering implementation [S2, unofficial].

## Background
FilterGS was introduced to address critical challenges in large-scale Level-of-Detail 3D Gaussian Splatting, specifically inefficient serial traversal and redundant Gaussian-tile pairs [S2, unofficial]. It builds on the principles of Multi-scale filtering, which address the mismatch between discrete sampling and continuous representation by modifying the projected 2D covariance matrix [S1, unofficial]. The method was developed to improve upon serial strategies like LoG, which require steps proportional to tree depth [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] FilterGS: Traversal-Free Parallel Filtering and Adaptive Shrinking for Large-Scale LoD 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2603.23891, fetched 2026-10-29 (unofficial)
