# OmniSDF

Also called: OmniSDF: Scene Reconstruction Using Omnidirectional Signed Distance Functions and Adaptive Binoctrees

## Overview
OmniSDF is a neural 3D reconstruction method for static indoor and outdoor scenes captured by omnidirectional video with small circular sweep baselines [S1]. It estimates geometry as a signed distance field (SDF) within an adaptive spherical binoctree to handle large depth ranges and narrow baselines where traditional ray-crossing methods fail [S1]. Unlike its sibling [NeuS](6060-neus.md), which uses a fixed coordinate space, OmniSDF adapts the spatial subdivision dynamically during optimization, allowing it to refine incorrect initial depth estimates and reduce geometry error by 38% compared to fixed binoctree approaches [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** An omnidirectional video sequence captured in a small circular sweep; pre-trained spherical depth estimation network; Structure-from-Motion (SfM) poses [S1].
*   **Outputs:** A reconstructed 3D surface mesh and view-dependent appearance [S1].

**Data Structures**
The core data structure is the **adaptive spherical binoctree**, composed of spherical voxels called **sphoxels** [S1].
*   **Sphoxel Layout:** Each sphoxel stores its boundary in spherical coordinates as a tuple $[r_{min}, r_{max}, \theta_{min}, \theta_{max}, \phi_{min}, \phi_{max}]$ and its unique index [S1].
*   **Subdivision:** The binoctree subdivides a sphere in radial ($r$), polar ($\theta$), and azimuthal ($\phi$) directions [S1]. Binary subdivision is applied specifically in the radial direction to prevent sphoxel elongation as radius $r$ increases [S1]. The structure is bound by near and far spheres [S1].
*   **Adaptivity:** The tree dynamically subdivides from coarse to fine granularity through iterative optimization, focusing ray samples and memory on areas with more detail [S1]. Leaf nodes are fine-tuned to prioritize sampling in areas closer to observed surfaces [S1].

**Neural Networks**
Two neural networks are used [S1]:
1.  **SDF Network ($F_{\theta}^{SDF}$):** Takes a 3D point $\mathbf{x}$ and outputs the signed distance $d$ to the surface and a 256-dimensional feature vector $\zeta$ [S1].
2.  **Appearance Network ($F_{\theta}^{c}$):** Takes the 3D point $\mathbf{x}$, the surface normal $\mathbf{n}$ (derived from the SDF gradient), the view direction $\mathbf{v}$, and the feature vector $\zeta$ to estimate view-dependent appearance color $c$ [S1].

**Algorithm Steps**
1.  **Preprocessing:** Estimate camera poses using SfM (e.g., OpenVSLAM) [S1]. Estimate per-frame initial depth using a pre-trained spherical depth estimation network [S1]. Construct an initial binoctree from these depth estimates [S1].
2.  **Initialization:** Initialize the neural networks ($F_{\theta}^{SDF}$ and $F_{\theta}^{c}$) and the binoctree structure [S1].
3.  **Iterative Optimization:**
    a.  **Tree Traversal:** Perform a breadth-first search to manage the inconsistent number of child nodes in the adaptive binoctree [S1].
    b.  **Intersection Test:** Use a custom intersection test method to accommodate the uneven shapes of sphoxels [S1].
    c.  **Sampling:** Sample points along rays $\mathbf{x}(t) = \mathbf{o} + t\mathbf{v}$ using step-wise sampling strategies guided by the binoctree [S1].
    d.  **Network Forward Pass:** Query $F_{\theta}^{SDF}$ for $d$ and $\zeta$, and $F_{\theta}^{c}$ for $c$ [S1].
    e.  **Rendering:** Compute unbiased, occlusion-aware weights $w(t)$ from the SDFs and accumulate color [S1].
    f.  **Loss and Update:** Optimize the networks and the binoctree structure (subdividing leaf nodes) using an image-based reconstruction loss [S1].
4.  **Refinement:** Repeat step 3 until convergence, allowing the binoctree to adapt to achievable depth accuracy and optimization depth [S1].

**Formulas**
The surface $\mathcal{S}$ is defined as the zero-level set of the SDF $f$ [S1]:
$$ \mathcal {S} = \left \{ {\mathbf {x} \in {\mathbb {R}^3}|f\left ( \mathbf {x} \right ) = 0} \right \} $$
The rendered color $C$ for a ray with origin $\mathbf{o}$ and direction $\mathbf{v}$ is [S1]:
$$ C\left ( {{\mathbf {o}},{\mathbf {v}}} \right ) = \int _{\textup {near}}^{\textup {far}} {w\left ( t \right )c\left ( {{\mathbf {x}}\left ( t \right ),{\mathbf {n}}\left ( t \right ),{\mathbf {v}},\zeta \left ( t \right )} \right )dt} $$
where $w(t)$ are weights derived from the SDF, $\mathbf{x}(t) = \mathbf{o} + t\mathbf{v}$, and $\mathbf{n}(t)$ is the normal at $\mathbf{x}(t)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def omnisdf_render(ray_origin, ray_direction, binoctree, sdf_net, color_net):
    """
    Renders a single ray using the adaptive spherical binoctree.
    """
    # 1. Traverse binoctree to find intersecting sphoxels
    # Use breadth-first search to handle inconsistent child counts
    intersecting_sphoxels = binoctree.bfs_traverse(ray_origin, ray_direction)
    
    # 2. Sort sphoxels by distance from ray origin
    intersecting_sphoxels.sort(key=lambda s: s.r_min)
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    t_current = ray_origin.distance_to_origin() # or near plane
    
    for sphoxel in intersecting_sphoxels:
        # 3. Intersection test for sphoxel boundaries
        t_enter, t_exit = sphoxel.intersect_ray(ray_origin, ray_direction)
        
        # 4. Sample points within the sphoxel
        # Number of samples depends on sphoxel size and desired resolution
        num_samples = sphoxel.calculate_sample_count()
        t_samples = linspace(t_enter, t_exit, num_samples)
        
        for t in t_samples:
            x = ray_origin + t * ray_direction
            
            # 5. Query SDF network
            d, zeta = sdf_net.forward(x)
            
            # 6. Compute normal from SDF gradient (finite difference or auto-diff)
            n = compute_normal(sdf_net, x)
            
            # 7. Compute weight w(t) from SDF (unbiased, occlusion-aware)
            # Implementation detail: Wang et al.'s method [S1]
            w = compute_weight(d)
            
            # 8. Query Appearance network
            c = color_net.forward(x, n, ray_direction, zeta)
            
            # 9. Volume rendering accumulation
            alpha = 1.0 - exp(-w) # Simplified opacity model based on weight
            accumulated_color += alpha * c * (1.0 - accumulated_alpha)
            accumulated_alpha += alpha * (1.0 - accumulated_alpha)
            
            if accumulated_alpha > 0.99:
                break
                
    return accumulated_color

def omnisdf_train(video_frames, binoctree, sdf_net, color_net):
    """
    Main training loop with adaptive binoctree subdivision.
    """
    # Preprocessing: Initialize binoctree from pre-trained depth estimates
    binoctree.initialize_from_depth(video_frames)
    
    for epoch in range(num_epochs):
        for frame in video_frames:
            # Get camera pose for the frame
            pose = get_pose(frame)
            
            for ray in frame.rays:
                # Render the ray
                rendered_color = omnisdf_render(ray.origin, ray.direction, binoctree, sdf_net, color_net)
                target_color = frame.pixel_color
                
                # Compute loss
                loss = mse(rendered_color, target_color)
                
                # Backpropagate to update networks
                loss.backward()
                
            # Adaptive Subdivision Step
            # Subdivide leaf nodes based on optimization depth and accuracy
            binoctree.adaptive_subdivide(sdf_net)
            
            # Clear gradients
            sdf_net.zero_grad()
            color_net.zero_grad()
```

**Edge Cases and Pitfalls**
*   **Sphoxel Elongation:** The binoctree must use binary subdivision in the radial direction to prevent sphoxels from becoming excessively elongated at large radii [S1].
*   **Uneven Shapes:** Standard ray-sphere or ray-box intersection tests may fail; a custom intersection test is required to accommodate the irregular shapes of sphoxels in the binoctree [S1].
*   **Initial Depth Errors:** Unlike fixed TSDF fusion methods, OmniSDF can refine incorrect initial depth estimates because it uses a neural implicit function [S1]. However, poor initial estimates may slow convergence [S1].
*   **Narrow Baseline:** The method is specifically designed for small circular sweep baselines where ray crossings are difficult to find; it may not be optimal for large baseline stereo setups [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly stated, but adaptive subdivision and breadth-first traversal add overhead compared to fixed grids [S1].
*   **Memory Complexity:** More memory-efficient than Cartesian voxel grids for unbounded scenes because it focuses sampling on plausible regions and uses spherical subdivision [S1].
*   **Performance:** Reduces geometry error by 38% on average compared to existing binoctree-based reconstruction methods [S1]. Improves convergence speed and detail recovery over Cartesian grids [S1].
*   **Strengths:** Handles large depth ranges and narrow baselines effectively; adapts to scene complexity; refines initial depth errors [S1].
*   **Limitations:** Requires pre-trained depth estimation and SfM for initialization; complex tree traversal and intersection logic [S1].
*   **When to use:** Indoor/outdoor static scene reconstruction from omnidirectional video with small circular sweep baselines [S1].
*   **When not to use:** Dynamic scenes; large baseline stereo; scenes where Cartesian symmetry is dominant [S1].

## Background
OmniSDF was introduced by Hakyeong Kim, Andreas Meuleman, Hyeonjoong Jang, James Tompkin, and Min H. Kim from KAIST and INRIA/Brown University [S1]. It was published in CVPR 2024 [S1]. The method builds on [NeuS](6060-neus.md) for implicit surface rendering and extends the binoctree concept from Jang et al. [8] by making it adaptive and neural-implicit [S1]. It addresses the challenges of egocentric omnidirectional video, where narrow baselines and large depth ranges make traditional reconstruction difficult [S1].

## Sources
[S1] OmniSDF: Scene Reconstruction Using Omnidirectional Signed Distance Functions and Adaptive Binoctrees, https://doi.org/10.1109/cvpr52733.2024.01912, fetched 2026-10-26
