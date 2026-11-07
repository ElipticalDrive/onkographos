# Dynamic Particle Distribution

Also called: Particle Distribution on Isosurface, Dynamic Particle Isosurface Sampling with Contraction Correction

## Overview
Dynamic Particle Distribution is a particle-based sampling method for isosurface mesh generation that uses inter-particle forces to drive particles to local energy minima, controlling density via a sizing field $h$ and correcting for Euclidean distance approximation artifacts using a contraction factor $g$ [S1]. It builds on [Point Engine for Scalar Fields](7529-point-engine-for-scalar-fields.md) by taking the initial vertex positions and refining them into a uniformly distributed point cloud suitable for high-quality triangulation, rather than outputting a mesh directly [S1]. Unlike Marching Cubes variants which produce fixed topological structures, this method adapts particle density to the local curvature and feature size of the implicit surface [S1].

## How it works
**Inputs:**
*   Implicit scalar field $F(\mathbf{x})$ defining the isosurface $F(\mathbf{x}) = 0$.
*   Sizing field $h(\mathbf{x})$ derived from the local feature size (e.g., via medial axis transform) [S1].
*   Initial particle positions $\{\mathbf{x}_i\}$ obtained from a Marching Cubes triangulation [S1].

**Outputs:**
*   A set of particles $\{\mathbf{x}_i\}$ distributed on the isosurface with spacing controlled by $h$.
*   A triangulated mesh generated from the final particle positions using TIGHT COCONE [S1].

**Data Structures:**
*   **Particle:** A structure containing position $\mathbf{x}_i \in \mathbb{R}^3$, sizing field value $h_i$, and potential energy $E_i$.
*   **Sizing Field:** Evaluated at arbitrary positions $\mathbf{x}$ using trilinear reconstruction kernels within a voxel [S1].

**Parameters:**
*   $\beta = 0.5 / \cos(\pi/6)$ [S1]. Derived from ideal hexagonal packing where the region of influence ends at the closest two-ring neighbors [S1].
*   $g = 1.5$. A conservative empirical estimate for distance contraction correction, determined by observing histograms of triangle edge lengths relative to $h$ [S1].
*   $\varepsilon$: The sampling error bound. The contraction factor $g$ is bounded by $1 \geq g \geq (1 - \varepsilon^2/4)^{1/2}$ [S1].

**Steps:**

1.  **Initialization:** Initialize the particle system with positions from a Marching Cubes triangulation. This ensures that disconnected components of the isosurface are seeded with particles [S1].

2.  **Projection:** Project each particle onto the implicit function $F$ using Newton-Raphson gradient descent [S1].
    *   Let $\mathbf{x}_i^{(0)}$ be the initial position.
    *   Iterate: $\mathbf{x}_i^{(k+1)} = \mathbf{x}_i^{(k)} - \lambda \frac{F(\mathbf{x}_i^{(k)})}{\|\nabla F(\mathbf{x}_i^{(k)})\|^2} \nabla F(\mathbf{x}_i^{(k)})$ until $|F(\mathbf{x}_i)| < \text{tol}$ [G]. (Note: The source specifies Newton-Raphson gradient descent but does not provide the exact step size formula; standard Newton-Raphson for root finding is used here as [G]).

3.  **Force Calculation:** For each pair of particles $i, j$, calculate the scaled distance $d_{ij}$ which determines the magnitude of the inter-particle force [S1].
    *   Evaluate sizing field values $h_i$ and $h_j$ at positions $\mathbf{x}_i$ and $\mathbf{x}_j$ using trilinear reconstruction kernels [S1].
    *   Calculate scaling factor:
        $$ \alpha_{ij} = \frac{\beta}{\min(h_i, h_j)} $$
    *   Calculate scaled distance:
        $$ d_{ij} = \alpha_{ij} \|\mathbf{x}_i - \mathbf{x}_j\| $$
    *   The effective distance is symmetric: $d_{ij} = d_{ji}$ [S1].

4.  **Contraction Correction:** Counteract the artifact where Euclidean distances cause two-ring neighbors to become influential, which disrupts ideal packing [S1].
    *   Inversely scale the sizing field: $h_i \leftarrow h_i / g$ for all particles [S1].
    *   This ensures that the length of virtually every edge in the final mesh is at most $2\varepsilon h_0$ [S1].

5.  **Energy Minimization:** Iteratively move particles to lower local energy states until the system reaches equilibrium [S1].
    *   The potential function induces inter-particle forces based on $d_{ij}$ [S1].
    *   The population control mechanism adds or removes particles based on energy to drive the system toward ideal hexagonal packing [S1].
    *   Stop when the change in particle positions or total energy falls below a threshold [G].

6.  **Triangulation:** Triangulate the final particle distribution using the TIGHT COCONE method [S1].
    *   TIGHT COCONE builds an initial Delaunay triangulation of the input point set [S1].
    *   Extract a manifold surface from all triangles that intersect a thickened surface built from the approximation of the points' normal vectors [S1].
    *   Apply a final selection criteria to the original triangulation to fill in holes where normals cannot be faithfully approximated [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_particle_distribution(implicit_field, sizing_field, initial_particles, beta=0.5/cos(pi/6), g=1.5, tol=1e-6):
    """
    Distributes particles on an isosurface defined by implicit_field using sizing_field.
    
    Args:
        implicit_field: Function F(x) returning scalar value.
        sizing_field: Function h(x) returning sizing field value via trilinear interpolation.
        initial_particles: List of initial 3D positions from Marching Cubes.
        beta: Scaling constant for ideal hexagonal packing.
        g: Contraction factor for distance correction.
        tol: Convergence tolerance for projection and equilibrium.
        
    Returns:
        final_particles: List of 3D positions on the isosurface.
    """
    particles = []
    
    # Step 1 & 2: Initialize and Project
    for pos in initial_particles:
        # Newton-Raphson projection onto F(x) = 0
        x = pos
        for _ in range(100): # Max iterations [G]
            f_val = implicit_field(x)
            if abs(f_val) < tol:
                break
            grad = compute_gradient(implicit_field, x) # [G]
            norm_grad_sq = dot(grad, grad)
            if norm_grad_sq < 1e-12:
                break # [G]
            x = x - (f_val / norm_grad_sq) * grad
        particles.append({'pos': x, 'h': sizing_field(x)})

    # Step 4: Contraction Correction
    for p in particles:
        p['h'] = p['h'] / g

    # Step 3 & 5: Force Calculation and Energy Minimization
    # Note: The specific potential function and force update rule are not explicitly 
    # detailed in the source beyond "inter-particle forces" and "lower local energy states".
    # The following implements the scaled distance logic described.
    
    converged = False
    while not converged:
        forces = [vec3(0,0,0) for _ in particles]
        
        for i in range(len(particles)):
            for j in range(i + 1, len(particles)):
                xi = particles[i]['pos']
                xj = particles[j]['pos']
                hi = particles[i]['h']
                hj = particles[j]['h']
                
                # Calculate scaled distance
                min_h = min(hi, hj)
                alpha_ij = beta / min_h
                dist_vec = xj - xi
                dist_norm = length(dist_vec)
                
                if dist_norm < 1e-12:
                    continue
                    
                # Scaled distance determines force magnitude
                # The source implies forces push particles to equilibrium based on d_ij
                # Standard repulsive force model for particle packing:
                # F_ij proportional to (d_ij - target_d) / d_ij * direction
                # However, the source does not give the exact force function.
                # We assume a standard repulsive force based on the scaled distance metric.
                # [G] Placeholder for specific potential derivative not provided in source.
                # Using a simple repulsive force inversely proportional to scaled distance squared
                # to simulate "pushing towards lower energy states" as described.
                force_mag = 1.0 / (alpha_ij * dist_norm) 
                force_dir = normalize(dist_vec)
                
                forces[i] += force_dir * force_mag
                forces[j] -= force_dir * force_mag
        
        # Update positions
        max_move = 0
        for i in range(len(particles)):
            move = forces[i] * step_size # [G] step_size not defined in source
            new_pos = particles[i]['pos'] + move
            # Re-project to surface if moved significantly? Source says "Iteratively move... until equilibrium"
            # It does not explicitly state re-projection after every step, only initial projection.
            # However, forces are computed on the surface.
            # [G] Assuming forces are tangential or small enough to stay on surface locally.
            particles[i]['pos'] = new_pos
            particles[i]['h'] = sizing_field(new_pos) / g # Re-evaluate h and apply contraction
            
            move_len = length(move)
            if move_len > max_move:
                max_move = move_len
                
        if max_move < tol:
            converged = True
            
    return [p['pos'] for p in particles]
```

**Subtopics:**
*   **TIGHT COCONE:** A water-tight Delaunay triangulation method used to generate the final mesh from the particle distribution. It extracts a manifold surface from triangles intersecting a thickened surface based on normal approximations and fills holes using a final selection criteria [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   The generation of medial axes and $\lambda$ grids takes approximately 1 hour per dataset on a P4 3.2GHz CPU with 2.0GB memory [S1].
*   Limiting the gradient field of $h_0$ to generate $h$ requires several minutes [S1].
*   Distributing the particle system takes 1–280 minutes depending on dataset size [S1].
*   Tessellation using TIGHT COCONE requires on the order of several minutes [S1].

**Measured Performance:**
*   **Brain (149x188x148, $\varepsilon=0.5, \delta=0.3$):** 91,702 vertices, 182,188 triangles in 41 minutes [S1].
*   **Pelvis (271x390x282, $\varepsilon=0.75, \delta=0.5$):** 4,992 vertices, 9,984 triangles in 1 minute [S1].
*   **Skull (256x256x115, $\varepsilon=0.5, \delta=0.3$):** 212,188 vertices, 424,588 triangles in 232 minutes [S1].
*   **Vessel (265x256x171, $\varepsilon=0.5, \delta=0.3$):** 287,990 vertices, 576,493 triangles in 280 minutes [S1].

**Strengths and Limitations:**
*   **Quality:** Achieves average radius ratios of 0.92–0.94, superior to modified Marching Cubes (0.66–0.67) and comparable to Advancing Front (0.93–0.94) [S1].
*   **Valence:** Produces 63.275% valence-6 vertices, compared to 44.15% for Marching Cubes and 71.75% for Advancing Front [S1].
*   **Extreme Valences:** Only 0.7% of vertices have valences <5 or >7, compared to 12.86% for Marching Cubes and 0.95% for Advancing Front [S1].
*   **Sampling Compliance:** Less than 0.004% of triangles in brain/skull meshes and less than 0.008% in vessel meshes contain an edge exceeding the required sampling length defined by $h_0$ [S1].
*   **Limitation:** The method relies on Euclidean distances rather than geodesic distances, which causes artifacts in two-ring neighborhoods. This is mitigated by the contraction factor $g$, but the approximation remains a theoretical limitation [S1].

## Background
The method was proposed in the paper "Topology, Accuracy, and Quality of Isosurface Meshes Using Dynamic Particles" [S1]. It addresses the need for high-quality isosurface meshes suitable for simulations and data interpolation, where regular triangulations and specific sampling requirements (controlled by $h$) are critical [S1]. The approach combines particle system dynamics with implicit surface representation to achieve $\varepsilon$-sampling of the isosurface [S1].

## Sources
[S1] Topology, Accuracy, and Quality of Isosurface Meshes Using Dynamic Particles, https://doi.org/10.1109/tvcg.2007.70604, fetched 2026-10-30
