# PAC-NeRF

Also called: Physics Augmented Continuum Neural Radiance Fields

## Overview
PAC-NeRF is a method for geometry-agnostic system identification that estimates both the unknown 3D geometry and physical parameters (e.g., Young’s modulus, viscosity) of highly dynamic objects from multi-view videos [S1, unofficial]. It differs from standard dynamic NeRFs by not learning dynamics from scratch; instead, it enforces continuum mechanics conservation laws using a differentiable Material Point Method (MPM) [S1, unofficial]. This hybrid Eulerian-Lagrangian approach allows for robust handling of topology changes and frictional contacts, which are difficult for mesh-based methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed multi-view video frames $\{\hat{C}(r, t_i)\}$ of a dynamic scene, where $r$ is a ray and $t_i$ is time.
*   **Output:** An explicit geometric representation (volume density $\sigma$) and physical parameters (e.g., material constants) that minimize the difference between rendered and observed images.

**Data Structures:**
The method uses a hybrid Eulerian-Lagrangian representation [S1, unofficial]:
1.  **Eulerian Grid:** A voxel grid stores the view-independent volume density $\sigma(\mathbf{x}, t)$ and view-dependent color $\mathbf{c}(\mathbf{x}, \boldsymbol{\omega}, t)$ [S1, unofficial]. This grid is discretized using the efficient voxel representation from Sun et al. (2022) [S1, unofficial].
2.  **Lagrangian Particles:** A set of particles advects the field quantities. Each particle carries state information used in the MPM simulation [S1, unofficial].
3.  **Physical Parameters:** Learnable parameters characterizing the material, such as Young’s modulus, fluid viscosity, and friction angles [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the Eulerian voxel NeRF for density $\sigma$ and color $\mathbf{c}$ [S1, unofficial]. Initialize Lagrangian particles and physical parameters.
2.  **Particle Binding:** Bind renderable quantities (density and color) from the Eulerian grid to Lagrangian particles using a sampling scheme [S1, unofficial].
3.  **Physics Simulation:** Simulate particle dynamics using a differentiable Material Point Method (MPM) [S1, unofficial]. The velocity field $\mathbf{v}$ evolves according to momentum conservation:
    $$ \frac{D\mathbf{v}}{Dt} = \frac{1}{\rho}\nabla \cdot \mathbf{T} + \mathbf{g} $$
    where $\rho$ is physical density, $\mathbf{T}$ is the internal Cauchy stress tensor, and $\mathbf{g}$ is gravity [S1, unofficial].
4.  **Grid Update (G2P):** Convert particle states back to the Eulerian grid via Grid-to-Particle (G2P) transforms [S1, unofficial]. This step enforces the conservation laws for density and color:
    $$ \frac{D\sigma}{Dt} = 0, \quad \frac{D\mathbf{c}}{Dt} = 0 $$
    where $\frac{D\phi}{Dt} = \frac{\partial \phi}{\partial t} + \mathbf{v} \cdot \nabla \phi$ is the material derivative [S1, unofficial].
5.  **Rendering:** Render images from the updated Eulerian voxel NeRF. For a ray $r(s)$ with $s \in [s_{min}, s_{max}]$, the pixel color $C(r, t)$ is:
    $$ C(r, t) = \int_{s_n}^{s_f} T(s, t) \sigma(r(s), t) \mathbf{c}(r(s), \boldsymbol{\omega}, t) ds + \mathbf{c}_{bg} T(s_f, t) $$
    where $T(s, t) = \exp\left(-\int_{s_n}^{s} \sigma(r(\bar{s}), t) d\bar{s}\right)$ [S1, unofficial].
6.  **Loss Computation:** Compute the rendering loss against ground truth video frames:
    $$ L_{render} = \frac{1}{N} \sum_{i=0}^{N-1} \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \| C(r, t_i) - \hat{C}(r, t_i) \|^2 $$
    where $N$ is the number of frames and $\mathcal{R}$ is the set of rays [S1, unofficial].
7.  **Optimization:** Optimize geometry ($\sigma, \mathbf{c}$) and physical parameters via gradient-based optimization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pac_nerf_train(video_frames, initial_params):
    # 1. Initialize Eulerian Grid and Lagrangian Particles
    voxel_grid = init_voxel_nerf()
    particles = init_particles(voxel_grid)
    physical_params = init_physical_params(initial_params)
    
    for step in range(num_training_steps):
        # 2. Bind Grid to Particles (Sampling)
        particles.bind_from_grid(voxel_grid)
        
        # 3. Differentiable MPM Simulation
        # Evolve velocity using momentum conservation
        velocity = simulate_mpm(particles, physical_params)
        
        # 4. Grid-to-Particle (G2P) / Particle-to-Grid (P2G)
        # Advect fields using material derivative conservation
        voxel_grid.update_from_particles(particles, velocity)
        
        # 5. Render Images
        rendered_frames = []
        for frame_idx, ray_set in enumerate(video_frames.rays):
            color = volume_render(voxel_grid, ray_set)
            rendered_frames.append(color)
            
        # 6. Compute Loss
        loss = compute_mse_loss(rendered_frames, video_frames.gt_colors)
        
        # 7. Backpropagate and Optimize
        gradients = compute_gradients(loss, [voxel_grid, physical_params])
        update_parameters(gradients)
        
    return voxel_grid, physical_params
```

**Edge Cases and Pitfalls:**
*   **Topology Changes:** Standard mesh-based methods fail here. PAC-NeRF uses MPM, which naturally handles topology changes and frictional contacts [S1, unofficial].
*   **Unknown Geometry:** The method does not require watertight mesh representations or known object geometries, unlike many system identification methods [S1, unofficial].

## Operating characteristics
*   **Performance:** PAC-NeRF accelerates NeRF rendering by two orders of magnitude compared to standard dynamic NeRFs due to the Eulerian voxel representation [S1, unofficial].
*   **Data Efficiency:** By using a prescriptive physics model (MPM) rather than learning dynamics from scratch, it is more data-efficient and generalizable [S1, unofficial].
*   **Applicability:** Validated on elastic bodies, plasticine, sand, and Newtonian/non-Newtonian fluids [S1, unofficial].
*   **Limitations:** Requires multi-view video input. The complexity of the MPM simulation adds computational overhead per step compared to pure neural dynamics, though rendering is faster [S1, unofficial].

## Background
PAC-NeRF was introduced to address the challenge of inferring geometric and physical properties from visual observations without assuming object geometry or topology [S1, unofficial]. It builds on the concept of Neural Radiance Fields (NeRF) [S1, unofficial] and the Material Point Method (MPM) for differentiable physics simulation [S1, unofficial]. Previous dynamic NeRF variants (e.g., D-NeRF) learned dynamics from scratch, requiring significant data and lacking interpretability [S1, unofficial]. PAC-NeRF integrates these fields to enable system identification for a wide range of continuum materials [S1, unofficial].

## Sources
[S1] PAC-NeRF: Physics Augmented Continuum Neural Radiance Fields for Geometry-Agnostic System Identification, https://arxiv.org/abs/2303.05512v1, fetched 2026-10-26 (unofficial)
