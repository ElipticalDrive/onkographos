# Spot Noise (Motion Blurred Particles near Contour Surfaces)

Also called: Elliptical Spot Noise, Motion blurred particles, Surface Particles

## Overview
Spot noise visualizes 3D velocity fields in a thickened region around a contour surface by rendering motion-blurred ellipsoidal particles, allowing for interactive rotation and animation using standard hardware graphics pipelines [S2, unofficial]. Unlike methods that represent tangential flows on surfaces (such as stream surfaces) or require surface parameterization, this technique operates on contour surfaces of scalar functions that are generally unrelated to the velocity vectors, meaning the flow is not necessarily tangent to the surface [S2, unofficial]. It builds on [Near-Surface Point Sampling](2910-near-surface-point-sampling.md) by using distance-field estimation to restrict particle placement to a specific band around the isosurface [G].

## How it works

### Inputs and Outputs
**Inputs:**
*   A 3D vector field $V(x,y,z)$ defined on a rectilinear lattice [S2, unofficial].
*   A scalar function $f(x,y,z)$ defined on the same rectilinear lattice [S2, unofficial].
*   A contour value $C$ defining the surface $f(x,y,z) = C$ [S2, unofficial].
*   A distance parameter $D$ defining the thickness of the region around the contour [S2, unofficial].
*   A particle count $N$ per marked cell [S2, unofficial].
*   A particle radius $r$ [S2, unofficial].
*   A fading constant $k$ [S2, unofficial].

**Outputs:**
*   A rendered image containing semi-transparent, motion-blurred ellipsoidal particles overlaid on the contour surface [S2, unofficial].

### Data Structures
1.  **Lattice Grid:** Stores values for $f(x,y,z)$ and the precomputed magnitude of its gradient $|\nabla f(x,y,z)|$ at each vertex [S2, unofficial]. $|\nabla f|$ is computed via finite differences of neighboring vertex values of $f$ [S2, unofficial].
2.  **Cell Marking Array:** A boolean flag for each lattice cell indicating if it intersects the region $d(x,y,z) < D$ [S2, unofficial]. A cell is marked if at least one of its vertices is within distance $D$ of the contour surface [S2, unofficial].
3.  **Particle Linked List:** Stores active particles [S2, unofficial]. Each node contains position $(x,y,z)$, velocity $(V_x, V_y, V_z)$, age $a$, and status.

### Parameters
*   $D$: The maximum distance from the contour surface for particle existence. Must be larger than the lattice cell size for the marking procedure to work efficiently [S2, unofficial].
*   $N$: The number of particles randomly inserted in each marked cell [S2, unofficial].
*   $r$: The radius of the particle [S2, unofficial].
*   $k$: Constant used in the fading calculation [S2, unofficial].

### Algorithm Steps

1.  **Precompute Gradient Magnitude:**
    For each lattice vertex, approximate the gradient $\nabla f$ using finite differences of neighboring vertex values of $f$. Store the magnitude $|\nabla f(x,y,z)|$ [S2, unofficial].

2.  **Mark Cells:**
    *   Identify all lattice vertices within distance $D$ of the contour surface $f(x,y,z)=C$ [S2, unofficial].
    *   Mark all lattice cells that have at least one marked vertex [S2, unofficial].
    *   Update these markings whenever the user changes $C$ or $D$ [S2, unofficial].

3.  **Initialize Particles:**
    *   For each marked cell, randomly insert $N$ particles within the cell boundaries [S2, unofficial].
    *   Add these particles to the linked list with initial age $a=0$ [S2, unofficial].

4.  **Time Step Update (for each frame/animation step):**
    *   **Integration:** Move each particle in the linked list using second-order Runge-Kutta integration.
        *   Sample velocity $V$ at the particle's current position by trilinearly interpolating from the eight surrounding lattice vertices of the vector field [S2, unofficial].
    *   **Boundary Check:**
        *   If a particle moves into an unmarked cell or out of the data volume, schedule it for deletion [S2, unofficial].
    *   **Replenishment:**
        *   Check that all marked cells still contain $N$ particles [S2, unofficial].
        *   Add new particles or delete excess particles to maintain the count $N$ in each marked cell [S2, unofficial].

5.  **Rendering:**
    *   Render the opaque contour surface first [S2, unofficial].
    *   Render the particles as semi-transparent objects after the opaque objects, combining them with the contour surface in the z-buffer [S2, unofficial].
    *   Particles are all the same color and do not require sorting [S2, unofficial].

### Distance Estimation
To determine if a particle at $(x,y,z)$ is within the valid region, estimate the distance $d(x,y,z)$ from the contour surface $f(x,y,z)=C$ using Levoy's technique [S2, unofficial]:

1.  Trilinearly interpolate the scalar value $f(x,y,z)$ from the eight surrounding lattice vertices [S2, unofficial].
2.  Trilinearly interpolate the gradient magnitude $|\nabla f(x,y,z)|$ from the eight surrounding lattice vertices [S2, unofficial].
3.  The distance is approximated by:
    $$ d(x,y,z) = \frac{|f(x,y,z) - C|}{|\nabla f(x,y,z)|} $$
    [S2, unofficial].

A particle is rendered only if $d(x,y,z) < D$ [S2, unofficial].

### Particle Shape and Size
Particles are drawn as blurred ellipses stretched in the direction of motion [S2, unofficial]. A single blurred disk texture is mapped to a 3D rectangle defined by four vertices [S2, unofficial].

**Rectangle Vertices:**
The vertices of the rectangle are located at:
$$ P + S + T, \quad P + S - T, \quad P - S - T, \quad P - S + T $$
Where:
*   $P$ is the particle's position vector relative to the viewpoint [S2, unofficial].
*   $V$ is the particle's velocity vector [S2, unofficial].
*   $r$ is the particle radius.
*   $S$ and $T$ are orthogonal vectors defining the ellipse axes:
    $$ S = r \frac{V \times P}{|V \times P|} $$
    $$ T = \frac{V}{|V|} r - \frac{S \times P}{|S \times P|} $$
    [S2, unofficial]. Here, $T$ is along the velocity direction, but the second term ensures the particle shrinks to a small round dot of radius $r$ when velocity approaches zero or is oriented near the viewing direction [S2, unofficial].

**Particle Size Factor ($b$):**
The size of the particle is proportional to a factor $b = s(d)$, where:
*   $s(d) = 1$ for small $d$ (close to surface) [S2, unofficial].
*   $s(d)$ decreases continuously to $0$ when $d$ reaches $D$ [S2, unofficial].
*   To handle birth fading, the actual size factor is:
    $$ b = \min(s(d), k \cdot a) $$
    Where $a$ is the age of the particle and $k$ is a constant [S2, unofficial]. This ensures new particles fade in at birth and old particles fade out as they are deleted [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trilinear_interpolate(grid, x, y, z):
    # Standard trilinear interpolation from 8 surrounding lattice vertices [G]
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    def v(dx, dy, dz):
        return grid[(ix+dx) % grid.shape[0], (iy+dy) % grid.shape[1], (iz+dz) % grid.shape[2]]
    c000 = v(0,0,0); c100 = v(1,0,0); c010 = v(0,1,0); c110 = v(1,1,0)
    c001 = v(0,0,1); c101 = v(1,0,1); c011 = v(0,1,1); c111 = v(1,1,1)
    c00 = c000*(1-fx) + c100*fx; c10 = c010*(1-fx) + c110*fx
    c01 = c001*(1-fx) + c101*fx; c11 = c011*(1-fx) + c111*fx
    c0 = c00*(1-fy) + c10*fy; c1 = c01*(1-fy) + c11*fy
    return c0*(1-fz) + c1*fz

def compute_gradient_magnitude(grid_f, x, y, z):
    # Finite differences of neighboring vertex values of f [G]
    dx = trilinear_interpolate(grid_f, x+1, y, z) - trilinear_interpolate(grid_f, x-1, y, z)
    dy = trilinear_interpolate(grid_f, x, y+1, z) - trilinear_interpolate(grid_f, x, y-1, z)
    dz = trilinear_interpolate(grid_f, x, y, z+1) - trilinear_interpolate(grid_f, x, y, z-1)
    return math.sqrt(dx**2 + dy**2 + dz**2)

def norm(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def normalize(v):
    n = norm(v)
    if n < 1e-9:
        return (0.0, 0.0, 0.0)
    return (v[0]/n, v[1]/n, v[2]/n)

def in_volume(pos, bounds):
    return bounds[0] <= pos[0] <= bounds[1] and bounds[2] <= pos[1] <= bounds[3] and bounds[4] <= pos[2] <= bounds[5]

def get_cell_index(pos, cell_size):
    return (int(pos[0]/cell_size), int(pos[1]/cell_size), int(pos[2]/cell_size))

def run_spot_noise(particles, marked_cells, viewpoint, scalar_field, vector_field, C, D, N, r, k, bounds, cell_size, dt):
    # 1. Update Particle Positions (Physics)
    new_particles = []
    for particle in particles:
        # Second-order Runge-Kutta integration
        v1 = (trilinear_interpolate(vector_field, *particle['pos']),
              trilinear_interpolate(vector_field, *particle['pos'][1:]),
              trilinear_interpolate(vector_field, *particle['pos'][2:])) # Simplified for 3D vector field grid
        # Correct RK2: k1 = V(p), k2 = V(p + dt/2 * k1), p_new = p + dt * k2
        temp_pos = (particle['pos'][0] + dt*v1[0]/2, particle['pos'][1] + dt*v1[1]/2, particle['pos'][2] + dt*v1[2]/2)
        v2 = (trilinear_interpolate(vector_field, *temp_pos),
              trilinear_interpolate(vector_field, temp_pos[0], temp_pos[1]+1, temp_pos[2]),
              trilinear_interpolate(vector_field, temp_pos[0], temp_pos[1], temp_pos[2]+1))
        new_pos = (particle['pos'][0] + dt*v2[0], particle['pos'][1] + dt*v2[1], particle['pos'][2] + dt*v2[2])
        particle['pos'] = new_pos
        particle['vel'] = v2
        particle['age'] += dt

        # Distance estimation
        f_val = trilinear_interpolate(scalar_field, *new_pos)
        grad_mag = compute_gradient_magnitude(scalar_field, *new_pos)
        d = abs(f_val - C) / grad_mag if grad_mag > 1e-9 else float('inf')

        # Boundary and region check
        if d >= D or not in_volume(new_pos, bounds) or get_cell_index(new_pos, cell_size) not in marked_cells:
            particle['status'] = 'deleted'
        else:
            new_particles.append(particle)

    particles = [p for p in particles if p['status'] != 'deleted']

    # 2. Replenish Particles
    # Ensure N particles per marked cell
    cell_particle_counts = {}
    for p in particles:
        cell = get_cell_index(p['pos'], cell_size)
        cell_particle_counts[cell] = cell_particle_counts.get(cell, 0) + 1
    
    for cell in marked_cells:
        count = cell_particle_counts.get(cell, 0)
        while count < N:
            # Add new particle at random position within cell
            offset = (math.random.random(), math.random.random(), math.random.random())
            new_p = {
                'pos': (cell[0]*cell_size + offset[0], cell[1]*cell_size + offset[1], cell[2]*cell_size + offset[2]),
                'vel': (0,0,0), 'age': 0.0, 'status': 'active'
            }
            particles.append(new_p)
            count += 1

    # 3. Rendering
    # Hardware rasterization: map blurred disk texture to 3D rectangle vertices [S2, unofficial]
    for particle in particles:
        P = (particle['pos'][0] - viewpoint[0], particle['pos'][1] - viewpoint[1], particle['pos'][2] - viewpoint[2])
        V = particle['vel']
        vxP = cross(V, P)
        vxP_norm = norm(vxP)

        if vxP_norm < 1e-9:
            # Degenerate case: draw as small dot
            # Hardware rasterization placeholder for degenerate rectangle [S2, unofficial]
            pass
        else:
            S = (r * vxP[0]/vxP_norm, r * vxP[1]/vxP_norm, r * vxP[2]/vxP_norm)
            SxP = cross(S, P)
            SxP_norm = norm(SxP)
            if SxP_norm < 1e-9:
                T = (r * V[0]/norm(V), r * V[1]/norm(V), r * V[2]/norm(V))
            else:
                correction = (SxP[0]/SxP_norm, SxP[1]/SxP_norm, SxP[2]/SxP_norm)
                V_norm = normalize(V)
                T = (V_norm[0]*r - correction[0], V_norm[1]*r - correction[1], V_norm[2]*r - correction[2])

            v1 = (particle['pos'][0] + S[0] + T[0], particle['pos'][1] + S[1] + T[1], particle['pos'][2] + S[2] + T[2])
            v2 = (particle['pos'][0] + S[0] - T[0], particle['pos'][1] + S[1] - T[1], particle['pos'][2] + S[2] - T[2])
            v3 = (particle['pos'][0] - S[0] - T[0], particle['pos'][1] - S[1] - T[1], particle['pos'][2] - S[2] - T[2])
            v4 = (particle['pos'][0] - S[0] + T[0], particle['pos'][1] - S[1] + T[1], particle['pos'][2] - S[2] + T[2])

            # Calculate size factor
            f_val = trilinear_interpolate(scalar_field, *particle['pos'])
            grad_mag = compute_gradient_magnitude(scalar_field, *particle['pos'])
            d = abs(f_val - C) / grad_mag if grad_mag > 1e-9 else 1e9
            s_d = max(0.0, 1.0 - d / D)
            b = min(s_d, k * particle['age'])

            # Hardware rasterization: map blurred disk texture to 3D rectangle vertices [S2, unofficial]
            pass
```

## Operating characteristics

**Time Complexity:**
*   Particle movement: $O(N_{total})$ per time step, where $N_{total}$ is the total number of particles [S2, unofficial].
*   Distance estimation: $O(1)$ per particle (trilinear interpolation) [S2, unofficial].
*   Replenishment: Depends on implementation, but checking marked cells and particle counts is $O(N_{cells})$ or $O(N_{total})$ depending on spatial indexing [S2, unofficial].

**Memory Complexity:**
*   $O(N_{total})$ for the particle linked list [S2, unofficial].
*   $O(N_{cells})$ for the cell marking array [S2, unofficial].
*   $O(V)$ for the lattice grids ($V$ is number of lattice vertices) [S2, unofficial].

**Performance:**
*   The method uses hardware rendering for interactive speed, allowing real-time rotation and animation [S2, unofficial].
*   On SGI workstations (Elanm graphics, Onyxm workstation), it achieved real-time rotation and particle motion on small data sets and interactive performance on large data sets [S2, unofficial].
*   The use of a single blurred disk texture mapped to a 3D rectangle is efficient for hardware rasterization [S2, unofficial].

**Strengths:**
*   Interactive performance [S2, unofficial].
*   Does not require surface parameterization [S2, unofficial].
*   Handles non-tangential flows naturally [S2, unofficial].
*   Simple compositing (no sorting required for same-color particles) [S2, unofficial].

**Limitations:**
*   Particles may overlap and obscure each other [S2, unofficial].
*   Requires a thickened region around the surface, which can be memory intensive if $D$ is large [S2, unofficial].
*   Requires careful handling of degenerate velocity/view alignments to prevent shape distortion [S2, unofficial].

## Background

**Origin:**
The method was proposed by Nelson Max, Roger Crawfis, and Charles Grant at Lawrence Livermore National Laboratory [S2, unofficial].

**History:**
It was presented as one of four techniques for visualizing vector fields only near surfaces, aiming to overcome the overlap and occlusion issues of traditional 3D vector icons [S2, unofficial]. The method builds on earlier work by Levoy for distance estimation and adapts motion-blurred particles for contour surfaces rather than stream surfaces [S2, unofficial].

**Usage:**
Implemented in C++ using SGI's Invent2 for the user interface [S2, unofficial]. Controls for contour value $C$ and width $D$ are provided via sliders [S2, unofficial]. It was used to visualize airflow through a HEPA filter and tornado velocity data sets [S2, unofficial].

## Sources

[S1] Applications of texture mapping to volume and flow visualization, https://digital.library.unt.edu/ark:/67531/metadc626633/?q=volume splatting, fetched 2026-10-28 (unofficial)
[S2] Visualizing 3D velocity fields near contour surfaces. Revision 1, https://digital.library.unt.edu/ark:/67531/metadc1395286/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
