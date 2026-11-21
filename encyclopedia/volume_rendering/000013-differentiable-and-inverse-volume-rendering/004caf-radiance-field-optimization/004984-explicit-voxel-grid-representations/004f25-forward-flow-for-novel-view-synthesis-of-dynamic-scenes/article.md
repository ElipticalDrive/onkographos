# Forward Flow for Novel View Synthesis of Dynamic Scenes

Also called: ForwardFlowDNeRF

## Overview
Forward Flow is a dynamic novel view synthesis method that replaces backward flow deformation with forward flow to warp a canonical radiance field to target time steps, ensuring smooth and continuous motion modeling [S1]. Unlike backward flow methods that map points from current time to canonical space (causing discontinuity for fixed 3D positions), forward flow maps canonical space to current time, yielding smooth flow norms for the same 3D position along the timeline [S1]. It sets itself apart by using explicit voxel grids for the canonical field and a differentiable warping process involving average splatting and inpainting to handle mapping artifacts [S1].

## How it works
**Inputs:** A set of multi-view RGB images with known camera poses and timestamps $t$.
**Outputs:** Rendered novel views at arbitrary time steps $t$.

**Data Structures:**
1.  **Canonical Radiance Field Grid ($V_{grid}^{Can}$):** A 3D voxel grid storing volume density and color features for the canonical space. Represented as discrete values to enable explicit warping [S1].
2.  **Deformation Feature Grid ($V_{grid}^{Df}$):** A 3D voxel grid storing deformation features in canonical space.
3.  **Lightweight MLP ($F_\theta$):** A small neural network that takes deformation features and time as input to predict flow vectors [S1].

**Numbered Steps:**

1.  **Canonical Representation:** Represent the static canonical radiance field using the voxel grid $V_{grid}^{Can}$ [S1]. This allows explicit spatial warping, unlike continuous MLP-based NeRFs [S1].

2.  **Forward Flow Estimation:** Model the canonical trajectory field using the deformation feature grid $V_{grid}^{Df}$ and the lightweight MLP $F_\theta$. For a 3D point $p$ in canonical space at time $t$, the forward flow vector $V_{flow}$ is computed as:
    $$ V_{flow} = F_\theta(V_{grid}^{Df}(p), t) $$
    This estimates the displacement from canonical space to time $t$ [S1].

3.  **Differential Forward Warping:** Warp the canonical radiance field $V_{grid}^{Can}$ to time $t$ to obtain $V_{grid}^t$. This step addresses two mapping issues inherent in forward warping:
    *   **Many-to-One Mapping:** Multiple canonical points map to the same target location, causing occlusion/overlap artifacts. This is resolved using an **average splatting operation** [S1].
    *   **One-to-Many Mapping:** One canonical point maps to multiple target locations, causing holes/gaps in the warped field. This is resolved using an **inpaint network** [S1].
    
    The warped field is first splatted to get $V_{grid}^t$, then inpainted to get $V_{grid}^{t, Inp}$ [S1].

4.  **Volume Rendering:** Render novel views at time $t$ via standard volume rendering on the inpainted radiance field $V_{grid}^{t, Inp}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_flow_render(canonical_grid, deformation_grid, mlp_flow, inpaint_net, camera_pose, time_t):
    """
    Renders a novel view at time_t using forward flow warping.
    
    Args:
        canonical_grid: 3D VoxelGrid storing density and color in canonical space.
        deformation_grid: 3D VoxelGrid storing deformation features in canonical space.
        mlp_flow: Lightweight MLP predicting flow vectors from deformation features and time.
        inpaint_net: Neural network for inpainting holes in warped grid.
        camera_pose: Camera extrinsics and intrinsics for the target view.
        time_t: Float, the target time step.
        
    Returns:
        Rendered RGB image.
    """
    # Step 1: Sample canonical points along rays
    # For each pixel, cast a ray through the camera pose.
    # Sample points p_canon along the ray in canonical space.
    points_canon = sample_points_along_ray(camera_pose)
    
    # Step 2: Estimate Forward Flow
    # Retrieve deformation features for each canonical point
    def_features = deformation_grid.query(points_canon)
    
    # Predict flow vectors using the lightweight MLP
    # V_flow = F_theta(def_features, time_t)
    flow_vectors = mlp_flow(def_features, time_t)
    
    # Step 3: Warp Canonical Grid to Time t
    # Calculate target positions in the warped grid
    points_target = points_canon + flow_vectors
    
    # Handle Many-to-One: Average Splatting
    # Map canonical values to target positions, averaging if multiple points map to same voxel
    warped_grid = average_splat(canonical_grid, points_canon, points_target)
    
    # Handle One-to-Many: Inpainting
    # The splatting may leave holes where no canonical point mapped to a target voxel.
    # Use the inpaint network to fill these gaps.
    inpainted_grid = inpaint_net(warped_grid)
    
    # Step 4: Volume Rendering
    # Render the final image from the inpainted grid at time t
    # Standard differentiable volume rendering equation
    image = volume_render(inpainted_grid, camera_pose)
    
    return image
```

**Parameters:**
*   $F_\theta$: Lightweight MLP network. Size not given in the sources [S1].
*   Grid Resolution: Not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Many-to-One Mapping:** Multiple canonical points map to the same target location. Naive warping causes occlusion/overlap artifacts. Solved by average splatting [S1].
*   **One-to-Many Mapping:** One canonical point maps to multiple target locations. Naive warping causes holes/gaps. Solved by inpaint network [S1].
*   **Discontinuity in Backward Flow:** Backward flow methods suffer from non-smooth, discontinuous flow fields for fixed 3D positions along the timeline, leading to distortion in the canonical space. Forward flow avoids this by ensuring smooth flow norms [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Outperforms existing methods in both novel view rendering and motion modeling [S1].
*   **Strengths:** Smooth and continuous motion modeling; explicit recovery of surface point trajectories; avoids discontinuities of backward flow [S1].
*   **Limitations:** Requires explicit voxel grid representation (vs. continuous MLP); needs additional inpaint network for hole filling [S1].
*   **When to use:** When smooth motion modeling is critical and backward flow discontinuities are problematic.
*   **When not to use:** When memory is extremely constrained and voxel grids are too large, or when implicit representations are preferred for simplicity.

## Background
Forward Flow was proposed in 2023 [S1]. It addresses the limitations of backward flow fields in dynamic NeRFs, which are non-smooth and discontinuous, making them difficult to fit with smooth motion models like MLPs [S1]. The method builds on the idea of canonical radiance fields but introduces forward warping with voxel grids and differentiable warping operations [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
