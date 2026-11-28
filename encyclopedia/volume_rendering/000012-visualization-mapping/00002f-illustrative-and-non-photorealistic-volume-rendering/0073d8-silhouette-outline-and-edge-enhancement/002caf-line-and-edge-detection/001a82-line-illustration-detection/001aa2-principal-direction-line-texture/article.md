# Principal Direction Line Texture

Also called: 3D line texture, Interrante line illustration, Interrante's line illustration technique, Principal Curvature Streamlines, Principal Direction Line Illustration

## Overview
Principal Direction Line Texture is a non-photorealistic volume rendering technique that visualizes isosurfaces by advecting particles along the principal curvature directions of the volume data [S1, unofficial]. It solves the problem of communicating surface shape and topology in volumetric datasets by generating line textures that are geometrically invariant and perceptually effective [S2]. Unlike pixel-based edge detection methods, this approach is viewpoint-independent and can visualize multiple isosurfaces simultaneously by treating particle paths as a 3D line texture [S1, unofficial].

## How it works
**Inputs:** A volumetric dataset $V$ and one or more isosurface threshold values.
**Outputs:** A rendered image containing line textures representing the principal curvature directions on the specified isosurfaces.

**Data Structures:**
*   **Volume Data:** A 3D grid of scalar values.
*   **Principal Direction Field:** A vector field where each voxel contains two orthogonal unit vectors ($\mathbf{d}_1, \mathbf{d}_2$) representing the principal directions, derived from the Hessian matrix of the volume data.
*   **Particle System:** A set of points located on the isosurfaces.

**Steps:**
1.  **Calculate Principal Directions and Curvatures:** Compute the Hessian matrix for each voxel in the volume. The eigenvectors of the Hessian matrix correspond to the principal directions of curvature, and the eigenvalues correspond to the principal curvatures [S1, unofficial].
2.  **Initialize Particles:** Place an evenly distributed set of tiny opaque particles on the isosurfaces defined by the threshold values [S1, unofficial]. The distribution must be uniform to ensure consistent line density [S1, unofficial].
3.  **Advect Particles:** Advect the particles through the vector field defined by the principal directions. This involves tracing streamlines over the volumetric surface using the principal direction field as the velocity field [S1, unofficial].
4.  **Render Line Texture:** Treat the paths (trajectories) of the advected particles as a 3D line texture. These lines are rendered to show the shape of the identified isosurfaces [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def principal_direction_line_texture(volume, thresholds):
    # Step 1: Calculate principal directions and curvatures
    # Compute Hessian matrix H for each voxel
    # H = [[d2V/dx2, d2V/dxdy, d2V/dxdz],
    #      [d2V/dydx, d2V/dy2, d2V/dydz],
    #      [d2V/dzdx, d2V/dzdy, d2V/dz2]]
    # Compute eigenvalues and eigenvectors of H
    # Eigenvectors e1, e2 are the principal directions
    # Eigenvectors are normalized to unit length
    principal_directions = compute_hessian_eigenvectors(volume)
    
    # Step 2: Place evenly distributed particles on isosurfaces
    particles = []
    for threshold in thresholds:
        # Find voxels where volume value is close to threshold
        # Distribute particles evenly on the resulting surface
        surface_points = extract_isosurface_points(volume, threshold)
        particles.extend(evenly_distribute(surface_points))
        
    # Step 3: Advect particles along principal directions
    trajectories = []
    for particle in particles:
        path = [particle.position]
        current_pos = particle.position
        # Trace streamline using principal direction field
        # Integrate dx/dt = d1(x) or d2(x) depending on choice
        # Use numerical integration (e.g., Runge-Kutta)
        for step in range(num_steps):
            # Get principal direction at current_pos
            d1, d2 = principal_directions[current_pos]
            # Choose direction (e.g., d1)
            direction = d1
            # Advance position
            current_pos = integrate_step(current_pos, direction, step_size)
            path.append(current_pos)
            # Stop if out of bounds or volume ends
            if not is_inside_volume(current_pos):
                break
        trajectories.append(path)
        
    # Step 4: Render trajectories as lines
    render_lines(trajectories)
```

**Parameters:**
*   **Particle Distribution:** Evenly distributed on isosurfaces [S1, unofficial].
*   **Step Size:** Not given in the sources.
*   **Number of Steps:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Algorithmic Efficiency:** The technique requires significant computation for Hessian calculation and advection, which must be addressed for NPR applicability [S1, unofficial].
*   **Artistic Line Quality:** Showing line direction alone is not necessarily sufficient for an aesthetically pleasing rendering; additional artistic controls may be needed [S1, unofficial].
*   **Precision:** Numerical integration errors in advection can lead to inaccurate streamline tracing, especially in regions of high curvature variation [G].

## Subtopics

### Principal Curvature Streamlines
A specific implementation where streamlines are strictly aligned with principal curvature directions. It is chosen when geometric invariance and shape communication are prioritized over artistic style. See [Principal Direction Line Texture](773-principal-direction-line-texture.md).

### Hatching-Based Volume Rendering
Uses lines of varying thickness and orientation to simulate hatching. It is chosen for artistic styles like copper plate. See [Hatching-Based Volume Rendering](774-hatching-based-volume-rendering.md).

## Operating characteristics
*   **Time Complexity:** High, due to Hessian computation ($O(N)$ for $N$ voxels) and particle advection ($O(P \cdot S)$ for $P$ particles and $S$ steps).
*   **Memory Complexity:** $O(N)$ for storing the volume and principal direction field.
*   **Strengths:** Viewpoint independence, ability to visualize multiple isosurfaces simultaneously, geometric invariance, and strong perceptual communication of surface shape [S1, unofficial][S2].
*   **Limitations:** Algorithmic efficiency issues, potential lack of aesthetic appeal without additional artistic controls, and computational cost of Hessian calculation [S1, unofficial].
*   **When to Use:** When the goal is to clearly communicate the shape and topology of isosurfaces in a volumetric dataset, especially in scientific visualization where geometric accuracy is important.
*   **When Not to Use:** When real-time performance is critical or when artistic style is the primary concern without regard to geometric invariance.

## Background
The technique was introduced by Interrante in 1997 [S1, unofficial]. It is based on the psychological literature suggesting that lines in the principal directions of curvature communicate surface shape better than lines in other directions [S1, unofficial][S2]. Girshick et al. cited this method for its psychological basis and geometrical invariance [S2]. The method has been used in various NPR volume visualization techniques, often in conjunction with other methods like tone shading and silhouette enhancement [S2].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] State of the Art Non-Photorealistic Rendering (NPR) Techniques, https://doi.org/10.2312/localchapterevents/tpcg/tpcg06/089-098, fetched 2026-10-26
