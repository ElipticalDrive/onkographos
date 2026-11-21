# Connectivity-based Visibility for 3D Gaussian Splatting

Also called: 3DGS + Our Connectivity, Connectivity relationship for 3DGS

## Overview
This method replaces the standard depth-ordered blending of all overlapping points in 3D Gaussian Splatting (3DGS) with a visibility filtering step based on a connectivity relationship graph. By retrieving only visible points from the point cloud for blending, it reduces memory consumption and accelerates rendering, enabling the optimization of large-scale scenes (e.g., autonomous driving sequences with tens of millions of points) that would otherwise exceed GPU memory limits. It sets itself apart from standard 3DGS by explicitly discarding unseen points during the compositing phase, whereas standard 3DGS blends a set of depth-ordered points including many unseen ones.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point cloud (typically derived from LiDAR or SfM) with associated Gaussian attributes (position, covariance, opacity, spherical harmonics coefficients) and a camera pose.
*   **Output:** A rendered image where pixel colors are computed using only the subset of Gaussians determined to be visible via the connectivity graph.

**Data Structures:**
*   **Connectivity Graph:** A graph structure representing the spatial or topological relationships between points in the cloud. The specific construction algorithm for the graph is not detailed in the sources, but it is used to identify "visible" points relative to the camera view. [G: In typical implementations, this involves spatial hashing or k-d trees to find neighbors, but the exact graph construction is "Not given in the sources."]
*   **Visibility Mask:** A boolean array or list of indices indicating which points from the full cloud are visible.

**Steps:**
1.  **Deactivate Densification:** Disable the standard 3DGS densification (splitting/cloning) process. This is because the input LiDAR point cloud is already relatively dense, and densification would exacerbate memory issues. [S1]
2.  **Retrieve Visible Points:** Use the connectivity relationship graph to query the full point cloud and retrieve only the subset of points that are visible from the current camera viewpoint. [S1]
3.  **Blend Visible Points:** Perform the standard 3DGS alpha-blending operation using *only* the retrieved visible points. Standard 3DGS blends all depth-ordered points overlapping the pixel; this method blends only the visible subset. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_connectivity_3dgs(points, gaussians, camera, connectivity_graph):
    """
    Render a frame using connectivity-based visibility for 3DGS.
    
    Args:
        points: Array of 3D point positions (N, 3).
        gaussians: Dictionary of Gaussian attributes (covariances, opacities, SH coeffs).
        camera: Camera intrinsic and extrinsic parameters.
        connectivity_graph: Pre-built graph structure for visibility queries.
        
    Returns:
        rendered_image: Rendered RGB image.
    """
    # Step 1: Retrieve visible points using the connectivity graph
    # The graph maps camera view to a subset of point indices
    visible_indices = connectivity_graph.get_visible_points(points, camera)
    
    # Filter Gaussians to only those that are visible
    visible_gaussians = filter_gaussians(gaussians, visible_indices)
    
    # Step 2: Standard 3DGS rasterization/blending on the visible subset
    # This replaces the standard 3DGS step which processes all overlapping points
    rendered_image = standard_3dgs_rasterize(
        points[visible_indices], 
        visible_gaussians, 
        camera
    )
    
    return rendered_image

def filter_gaussians(gaussians, indices):
    """
    Filter Gaussian attributes to a specific set of indices.
    """
    filtered = {}
    for key, value in gaussians.items():
        filtered[key] = value[indices]
    return filtered
```

**Parameters:**
*   **Densification:** Disabled (Boolean: `False`). [S1]
*   **GPU Memory:** The method is designed to fit within limited resources (e.g., 48 GB), whereas standard 3DGS fails on large datasets. [S1]

**Edge Cases and Pitfalls:**
*   **Sparse Coverage:** The method may underperform in scenes with sparse coverage (e.g., fast-moving camera paths in autonomous driving) where the connectivity graph might not adequately capture visible structure. [S1]
*   **Graph Construction:** The performance depends on the efficiency and accuracy of the connectivity graph. If the graph fails to identify visible points correctly, rendering quality degrades. [G: The specific algorithm for building the connectivity graph is not provided in the sources.]

## Subtopics

### DeSplat
Focuses on reducing the number of Gaussians by merging similar ones, rather than filtering visibility. [Link](12925-desplat.md)

### 360 Gaussian Splatting
Optimizes for omnidirectional scenes, not specifically for large-scale sparse visibility filtering. [Link](12929-360-gaussian-splatting.md)

### Data Compression and Storage Optimization
Reduces storage size of Gaussian attributes, not rendering-time visibility. [Link](12943-data-compression-and-storage-optimization.md)

### SOGS
Uses second-order anchors for better geometry, not visibility filtering. [Link](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)

### Scene Representation and Geometry Constraints
Adds geometric priors, not visibility-based blending. [Link](15421-scene-representation-and-geometry-constraints.md)

### Perceptual Wrapper
Uses perceptual loss functions, not visibility changes. [Link](15438-perceptual-wrapper.md)

### Frequency Guided Resolution Scheduler
Manages training resolution, not rendering visibility. [Link](15751-frequency-guided-resolution-scheduler.md)

### Inverse and Internal Rendering
Handles reflections and internal views, not basic visibility filtering. [Link](15764-inverse-and-internal-rendering.md)

### Gaussian Frosting
Adds a surface layer, not visibility filtering. [Link](15863-gaussian-frosting.md)

### Semantic and Language Integration
Adds semantic labels, not visibility filtering. [Link](15930-semantic-and-language-integration.md)

### 3D Difference-of-Gaussians (3D-DoG)
Uses DoG for feature extraction, not visibility. [Link](16198-3d-difference-of-gaussians-3d-dog.md)

### Hardware and Computational Acceleration
General hardware optimizations, not algorithmic visibility changes. [Link](16214-hardware-and-computational-acceleration.md)

### Pruning and Density Control
Removes Gaussians based on opacity/size, not view-dependent visibility. [Link](16215-pruning-and-density-control.md)

### QuantumGS
Uses quantum computing concepts, not visibility filtering. [Link](16253-quantumgs.md)

### Dynamic and Spacetime Modeling
Handles 4D scenes, not static visibility filtering. [Link](16342-dynamic-and-spacetime-modeling.md)

### Closed-Form Triangle Splatting (CF-Splat)
Uses triangles instead of Gaussians, not visibility filtering. [Link](16773-closed-form-triangle-splatting-cf-splat.md)

### Softmax-GS
Uses softmax for blending, not visibility filtering. [Link](16854-softmax-gs.md)

### Student’s t-Primitive Representation
Uses t-distributions, not visibility filtering. [Link](17072-student-s-t-primitive-representation.md)

### Hierarchical, Graph-Based, and Adaptive Structures
General graph structures, not specifically connectivity-based visibility for 3DGS. [Link](17106-hierarchical-graph-based-and-adaptive-structures.md)

### Variable Basis Mapping (VBM)
Changes basis functions, not visibility. [Link](17312-variable-basis-mapping-vbm.md)

### Human Gaussian Splatting
Specialized for humans, not general visibility. [Link](17349-human-gaussian-splatting.md)

### Uncertainty and Probabilistic Modeling
Adds uncertainty estimates, not visibility filtering. [Link](17380-uncertainty-and-probabilistic-modeling.md)

### SCION
Uses implicit neural representations, not Gaussian visibility. [Link](17424-scion.md)

### Optimization Efficiency and Sampling
General optimization tips, not visibility. [Link](17461-optimization-efficiency-and-sampling.md)

### On-the-Fly GS
Renders without pre-training, not visibility filtering. [Link](17509-on-the-fly-gs.md)

### View-Dependent and Appearance Modeling
Handles appearance, not visibility. [Link](17617-view-dependent-and-appearance-modeling.md)

### Optimization and Training Loop
General training loop, not visibility. [Link](17791-optimization-and-training-loop.md)

### BAD-Gaussians
Handles bad Gaussians, not visibility filtering. [Link](18036-bad-gaussians.md)

### LiteTex-GS
Uses texture-based representation, not visibility. [Link](18111-litetex-gs.md)

### Virtual Camera Optimization for Mirror Reflections
Handles mirrors, not basic visibility. [Link](18121-virtual-camera-optimization-for-mirror-reflections.md)

### Differentiable DOF Rendering
Handles depth of field, not visibility. [Link](18289-differentiable-dof-rendering.md)

### ParetoGS Continuous-Budget Decoding
Manages budget, not visibility. [Link](18388-paretogs-continuous-budget-decoding.md)

### SpotLessSplats (SLS)
Reduces splat count, not visibility filtering. [Link](18423-spotlesssplats-sls.md)

### TruncGradGS
Truncates gradients, not visibility. [Link](18675-truncgradgs.md)

### EfficientGS
General efficiency, not visibility. [Link](18681-efficientgs.md)

### Variational 3D Gaussians
Probabilistic approach, not visibility. [Link](18706-variational-3d-gaussians.md)

### Gaussian Splatting Training with SDS Loss
Uses SDS loss, not visibility. [Link](18710-gaussian-splatting-training-with-sds-loss.md)

### EnvMap-GS
Handles environment maps, not visibility. [Link](18915-envmap-gs.md)

### Thermal-Physics-Informed 3D Gaussian Splatting
Adds physics, not visibility. [Link](18935-thermal-physics-informed-3d-gaussian-splatting.md)

### Environment Map Generation from Gaussian Background
Generates env maps, not visibility. [Link](18942-environment-map-generation-from-gaussian-background.md)

### Occupancy Region Derivation & ϵ-Intersection Criterion
Uses occupancy, not connectivity visibility. [Link](19020-occupancy-region-derivation-intersection-criterion.md)

### CC-4DGS Training Schedule and Hash Encoding Configuration
Training config, not visibility. [Link](19196-cc-4dgs-training-schedule-and-hash-encoding-configuration.md)

### MAC-Splat
Uses MAC for compression, not visibility. [Link](19210-mac-splat.md)

### Gaussian Piling
Stacks Gaussians, not visibility. [Link](19322-gaussian-piling.md)

### Heat Kernel Textures (HKTex)
Uses heat kernels, not visibility. [Link](19427-heat-kernel-textures-hktex.md)

### Gaussian Splatting in Style
Style transfer, not visibility. [Link](19431-gaussian-splatting-in-style.md)

### EOGS (Earth-observation Gaussian Splatting)
Specialized for Earth observation, not general visibility. [Link](19449-eogs-earth-observation-gaussian-splatting.md)

### HDR Gaussian Splatting
Handles HDR, not visibility. [Link](19777-hdr-gaussian-splatting.md)

### Parallel Differentiable Rasterization
Parallelizes rasterization, not visibility. [Link](19784-parallel-differentiable-rasterization.md)

### Coupled Gaussian Primitives Rendering
Couples primitives, not visibility. [Link](20018-coupled-gaussian-primitives-rendering.md)

### Surface-Constrained Gaussian Splatting
Adds surface constraints, not visibility. [Link](20026-surface-constrained-gaussian-splatting.md)

### BoundedGS
Uses bounding boxes, not connectivity visibility. [Link](20068-boundedgs.md)

### COLMAP-Free 3D Gaussian Splatting
Avoids COLMAP, not visibility. [Link](20075-colmap-free-3d-gaussian-splatting.md)

### Sketch and Patch
Uses sketches, not visibility. [Link](20108-sketch-and-patch.md)

## Operating characteristics
*   **Time Complexity:** Rendering time is significantly reduced. On KITTI-4-reduced (19.4 million points), rendering time decreased from 0.046s (21.6 FPS) to 0.00089s (1114 FPS), an acceleration of more than 50 times. [S1]
*   **Memory Complexity:** Reduces GPU memory requirements by only processing visible points. Standard 3DGS failed on full KITTI-4 (approx. 36.3 million points) due to memory constraints, while this method succeeded on the same hardware (48 GB GPU). [S1]
*   **Measured Performance:** On KITTI-4-reduced, PSNR improved from 19.61 to 28.57, SSIM from 0.58 to 0.83, and LPIPS from 0.57 to 0.23. [S1]
*   **Strengths:** Enables optimization and rendering of very large scenes (tens of millions of points) that exceed standard GPU memory. Significantly faster rendering speeds.
*   **Limitations:** Performance may degrade in scenes with sparse coverage (e.g., fast-moving cameras) where the connectivity graph may not accurately capture visible structure. [S1]
*   **When to Use:** Large-scale scenes with dense initial point clouds (e.g., LiDAR scans) where standard 3DGS fails due to memory or is too slow.
*   **When Not to Use:** Scenes with sparse coverage or where densification is critical for filling gaps, as densification is deactivated. [S1]

## Background
The method was introduced in the paper "CE-NPBG: Connectivity Enhanced Neural Point-Based Graphics for Novel View Synthesis in Autonomous Driving Scenes" (2025). It addresses the scalability issues of 3D Gaussian Splatting when applied to large-scale autonomous driving datasets like KITTI-360. The authors observed that standard 3DGS blends many unseen points, leading to unnecessary computation and memory usage. By introducing a connectivity relationship graph to filter visible points, they achieved better scene fitting and scalability. [S1]

## Sources
[S1] CE-NPBG: Connectivity Enhanced Neural Point-Based Graphics for Novel View Synthesis in Autonomous Driving Scenes, https://doi.org/10.1109/cvprw67362.2025.00240, fetched 2026-10-29
