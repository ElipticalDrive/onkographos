# Gaussian Splashing


## Overview
Gaussian Splashing (GSP) is a unified framework that combines 3D Gaussian Splatting (3DGS) with Position-Based Dynamics (PBD) to enable realistic, interactive simulation and rendering of fluids, deformable bodies, and rigid objects from novel views [S3]. It solves the limitation of vanilla 3DGS, which simplifies light field composition without well-defined appearance properties, making it unsuitable for translucent and specular fluids [S3]. GSP distinguishes itself by enriching Gaussian kernels with material parameters (diffusiveness, specularity, roughness, normals) and aligning kernel orientation with surface normals to eliminate spiky noise from rotational deformation [S1][S2, unofficial][S3].

## How it works
GSP treats each Gaussian kernel as a discrete physical particle with volume, allowing direct simulation within the rendering pipeline [S1][S2, unofficial]. The method integrates physically based rendering (PBR) to augment dynamic surface reflections on fluids [S3].

### Inputs and Outputs
*   **Inputs**: Initial 3DGS scene representation (positions, covariances, opacities, colors); camera poses; scene geometry (meshes for solids); physical parameters for PBD (density, viscosity, surface tension).
*   **Outputs**: Rendered images from novel viewpoints; simulated particle states over time.

### Data Structures
The core data structure is the Gaussian kernel, augmented from standard 3DGS with additional fields for physics and material properties [S3]:
1.  **Position** ($\mathbf{p}$): 3D center of the Gaussian.
2.  **Covariance** ($\Sigma$): Defines the shape and orientation of the Gaussian ellipsoid.
3.  **Opacity** ($\alpha$): Transparency of the kernel.
4.  **Spherical Harmonics** ($\mathbf{c}$): View-dependent color coefficients.
5.  **Normal** ($\mathbf{n}$): Surface normal vector, aligned with the kernel's orientation for solids and approximated for fluids [S3].
6.  **Material Parameters**: Diffusiveness, specularity, roughness [S1][S2, unofficial].
7.  **Mass/Volume**: Physical properties for PBD integration [S1][S2, unofficial].

### Steps
1.  **Reconstruction and Disentanglement**: Reconstruct surfaces from input images using 3DGS. Disentangle objects that will be actively simulated (solids, fluids) from static background elements [S1][S2, unofficial][S3].
2.  **Mesh-Based Simulation Framework**: Attach Gaussians to meshes for solid objects to enable texture rendering and physical interaction [S1][S2, unofficial][S3].
3.  **Normal Alignment and Material Enrichment**:
    *   For **solids**: Augment each Gaussian kernel with a normal vector. Align the kernel’s orientation with this surface normal to refine the PBD simulation [S3].
    *   For **fluids**: Enrich Gaussians with material parameters (diffusiveness, specularity, roughness) to handle specular and reflective objects [S1][S2, unofficial].
4.  **Physics Simulation (PBD)**:
    *   Apply Position-Based Dynamics to update particle positions and velocities [S3].
    *   **Solid Constraint**: Use an anisotropy loss during 3DGS training to cap the stretching ratio, mitigating rendering artifacts induced by rotational deformation [S3].
    *   **Fluid Normal Approximation**:
        *   For fluid kernels near the surface: Approximate the normal based on surface tension [S3].
        *   For scattered fluid droplets: Estimate normal information using a depth volume rendered via the current camera pose [S3].
5.  **Rendering**: Render the scene using the updated Gaussian kernels. Integrate PBR to dynamically synthesize specular shading along with fluid movement [S3].
6.  **Scene Editing (Optional)**: Augment the system with an image-space segmentation module to select objects of interest. Use generative AI to fill missing pixels for physics-based scene editing [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Gaussian Splashing Simulation and Rendering Loop

def gaussian_splashing_init(scene_data):
    """
    Initialize GSP from input images and scene geometry.
    """
    # 1. Reconstruct 3DGS scene
    gaussians = reconstruct_3dgs(scene_data.images)
    
    # 2. Disentangle active objects
    solid_meshes = extract_meshes(scene_data.geometry)
    fluid_particles = extract_fluid_particles(scene_data.geometry)
    
    # 3. Attach Gaussians to meshes for solids
    for mesh in solid_meshes:
        mesh_gaussians = attach_gaussians_to_mesh(mesh, gaussians)
        # Align kernel orientation with surface normal
        for g in mesh_gaussians:
            g.normal = compute_surface_normal(mesh, g.position)
            g.covariance = align_covariance_to_normal(g.covariance, g.normal)
    
    # 4. Initialize fluid Gaussians with material parameters
    for particle in fluid_particles:
        fluid_gaussian = create_gaussian(particle.position)
        fluid_gaussian.material = {
            'diffusiveness': 0.5,
            'specularity': 0.8,
            'roughness': 0.2
        }
        # Initial normal approximation based on surface tension if near surface
        if is_near_surface(particle.position, fluid_particles):
            fluid_gaussian.normal = compute_surface_tension_normal(particle.position, fluid_particles)
        else:
            # Placeholder for scattered droplet normal estimation
            fluid_gaussian.normal = estimate_normal_from_depth(particle.position, scene_data.camera_pose)
        add_to_gaussians(fluid_gaussian)

def gaussian_splashing_step(gaussians, dt, camera_pose):
    """
    Perform one step of PBD simulation and render.
    """
    # 1. PBD Simulation
    # Update positions and velocities based on PBD constraints
    update_pbd(gaussians, dt)
    
    # 2. Update Normals and Material Properties
    for g in gaussians:
        if g.is_solid:
            # Cap stretching ratio using anisotropy loss during training
            # This is typically done during optimization, not runtime step
            pass 
        elif g.is_fluid:
            # Re-approximate normal based on surface tension or depth
            if is_near_surface(g.position, get_fluid_gaussians(gaussians)):
                g.normal = compute_surface_tension_normal(g.position, get_fluid_gaussians(gaussians))
            else:
                g.normal = estimate_normal_from_depth(g.position, camera_pose)
    
    # 3. Render
    # Render using PBR with enriched material parameters
    image = render_gaussians_pbr(gaussians, camera_pose)
    return image

def estimate_normal_from_depth(position, camera_pose):
    """
    Estimate normal for scattered droplets using depth volume.
    """
    # Render depth map from current camera pose
    depth_map = render_depth(camera_pose)
    # Extract depth at position
    depth_value = get_depth_at_position(depth_map, position)
    # Compute normal from depth gradient
    normal = compute_normal_from_depth_gradient(depth_map, position)
    return normal
```

### Parameters
*   **Number of solid Gaussian kernels**: 334,815 [S3].
*   **Number of fluid Gaussian kernels**: 280,000 [S3].
*   **Anisotropy Loss Weight**: Not given in the sources.
*   **Surface Tension Coefficient**: Not given in the sources.

### Edge Cases and Pitfalls
*   **Rotational Deformation in Solids**: Vanilla 3DGS suffers from spiky noises due to rotational deformation. GSP mitigates this by aligning kernel orientation with surface normals and using an anisotropy loss to cap stretching ratio [S3].
*   **Fluid Normal Estimation**: For scattered fluid droplets, normal information is not directly available from surface tension. GSP resorts to estimating normals using a depth volume rendered via the current camera pose [S3].
*   **Vanilla 3DGS Limitations**: Vanilla 3DGS simplifies light field composition without well-defined appearance properties, making fluid rendering cumbersome. GSP addresses this by integrating PBR and normal alignment [S3].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Not given in the sources.
*   **Strengths**: Unified framework for solids, fluids, and deformable bodies; realistic surface reflections on dynamic fluids; eliminates spiky noise from rotational deformation; enables novel view synthesis and physics-based scene editing [S3].
*   **Limitations**: Relies on PBD, which may not capture complex constitutive models as accurately as SPH or FEM; normal estimation for scattered droplets depends on camera pose and depth rendering quality [S3].
*   **When to Use**: When realistic interaction between solids and fluids is required in a dynamic scene with novel view synthesis.
*   **When Not to Use**: When high-fidelity fluid physics (e.g., complex viscosity, turbulence) is more important than visual appearance and novel view synthesis.

## Background
Gaussian Splashing (GSP) was introduced by Tianjia Shao, Hongzhi Wu, Kun Zhou, Chenfanfu Jiang, and Yin Yang from the University of Utah, Zhejiang University, and UCLA [S3]. The paper was published in CVPR 2025 [S3]. GSP builds upon GaussianShader, which enriches Gaussians with material parameters [S1][S2, unofficial]. It leverages the coherence of 3DGS and PBD to manage rendering, view synthesis, and dynamics in a cohesive manner [S3]. The method addresses the limitations of vanilla 3DGS in handling translucent and specular fluids [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
