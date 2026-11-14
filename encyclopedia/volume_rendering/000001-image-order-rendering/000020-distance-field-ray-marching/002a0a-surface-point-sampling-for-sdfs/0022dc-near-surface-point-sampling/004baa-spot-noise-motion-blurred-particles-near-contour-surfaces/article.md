# Spot Noise (Motion Blurred Particles near Contour Surfaces)

Also called: Motion blurred particles

## Overview
Spot noise visualizes 3D velocity fields in a thickened region around a contour surface by rendering motion-blurred ellipsoidal particles, allowing for interactive rotation and animation using standard hardware graphics pipelines [S1, unofficial]. Unlike methods that represent tangential flows on surfaces (such as stream surfaces) or require surface parameterization, this technique operates on contour surfaces of scalar functions that are generally unrelated to the velocity vectors, meaning the flow is not necessarily tangent to the surface [S1, unofficial]. It builds on [Near-Surface Point Sampling](2910-near-surface-point-sampling.md) by using distance-field estimation to restrict particle placement to a specific band around the isosurface.

## How it works

### Inputs and Outputs
**Inputs:**
*   A 3D vector field $V(x,y,z)$ defined on a rectilinear lattice.
*   A scalar function $f(x,y,z)$ defined on the same rectilinear lattice.
*   A contour value $C$ defining the surface $f(x,y,z) = C$.
*   A distance parameter $D$ defining the thickness of the region around the contour.
*   A particle count $N$ per marked cell.
*   A particle radius $r$.
*   A fading constant $k$.

**Outputs:**
*   A rendered image containing semi-transparent, motion-blurred ellipsoidal particles overlaid on the contour surface.

### Data Structures
1.  **Lattice Grid:** Stores values for $f(x,y,z)$ and the precomputed magnitude of its gradient $|\nabla f(x,y,z)|$ at each vertex.
    *   $|\nabla f|$ is computed via finite differences of neighboring vertex values of $f$.
2.  **Cell Marking Array:** A boolean flag for each lattice cell indicating if it intersects the region $d(x,y,z) < D$.
    *   A cell is marked if at least one of its vertices is within distance $D$ of the contour surface.
3.  **Particle Linked List:** Stores active particles. Each node contains:
    *   Position $(x,y,z)$.
    *   Velocity $(V_x, V_y, V_z)$.
    *   Age $a$.
    *   Status (active/deleted).

### Parameters
*   $D$: The maximum distance from the contour surface for particle existence. Must be larger than the lattice cell size for the marking procedure to work efficiently [S1, unofficial].
*   $N$: The number of particles randomly inserted in each marked cell [S1, unofficial].
*   $r$: The radius of the particle [S1, unofficial].
*   $k$: Constant used in the fading calculation [S1, unofficial].

### Algorithm Steps

1.  **Precompute Gradient Magnitude:**
    For each lattice vertex, approximate the gradient $\nabla f$ using finite differences of neighboring vertex values of $f$. Store the magnitude $|\nabla f(x,y,z)|$ [S1, unofficial].

2.  **Mark Cells:**
    *   Identify all lattice vertices within distance $D$ of the contour surface $f(x,y,z)=C$.
    *   Mark all lattice cells that have at least one marked vertex.
    *   Update these markings whenever the user changes $C$ or $D$ [S1, unofficial].

3.  **Initialize Particles:**
    *   For each marked cell, randomly insert $N$ particles within the cell boundaries.
    *   Add these particles to the linked list with initial age $a=0$ [S1, unofficial].

4.  **Time Step Update (for each frame/animation step):**
    *   **Integration:** Move each particle in the linked list using second-order Runge-Kutta integration.
        *   Sample velocity $V$ at the particle's current position by trilinearly interpolating from the eight surrounding lattice vertices of the vector field [S1, unofficial].
    *   **Boundary Check:**
        *   If a particle moves into an unmarked cell or out of the data volume, schedule it for deletion [S1, unofficial].
    *   **Replenishment:**
        *   Check that all marked cells still contain $N$ particles.
        *   Add new particles or delete excess particles to maintain the count $N$ in each marked cell [S1, unofficial].

5.  **Rendering:**
    *   Render the opaque contour surface first.
    *   Render the particles as semi-transparent objects after the opaque objects, combining them with the contour surface in the z-buffer [S1, unofficial].
    *   Particles are all the same color and do not require sorting [S1, unofficial].

### Distance Estimation
To determine if a particle at $(x,y,z)$ is within the valid region, estimate the distance $d(x,y,z)$ from the contour surface $f(x,y,z)=C$ using Levoy's technique [S1, unofficial]:

1.  Trilinearly interpolate the scalar value $f(x,y,z)$ from the eight surrounding lattice vertices.
2.  Trilinearly interpolate the gradient magnitude $|\nabla f(x,y,z)|$ from the eight surrounding lattice vertices.
3.  The distance is approximated by:
    $$ d(x,y,z) = \frac{|f(x,y,z) - C|}{|\nabla f(x,y,z)|} $$
    *(Note: The source text states the quantities are trilinearly interpolated but the specific equation formula was garbled in the provided notes; this is the standard Levoy distance estimation formula implied by the context of using $f$ and $|\nabla f|$ [S1, unofficial].)*

A particle is rendered only if $d(x,y,z) < D$ [S1, unofficial].

### Particle Shape and Size
Particles are drawn as blurred ellipses stretched in the direction of motion. A single blurred disk texture is mapped to a 3D rectangle defined by four vertices.

**Rectangle Vertices:**
The vertices of the rectangle are located at:
$$ P + S + T, \quad P + S - T, \quad P - S - T, \quad P - S + T $$
Where:
*   $P$ is the particle's position vector relative to the viewpoint.
*   $V$ is the particle's velocity vector.
*   $r$ is the particle radius.
*   $S$ and $T$ are orthogonal vectors defining the ellipse axes:
    $$ S = r \frac{V \times P}{|V \times P|} $$
    $$ T = \frac{V}{|V|} r - \frac{S \times P}{|S \times P|} \times r \quad \text{(Corrected interpretation of garbled source)} $$
    *Source Text:* $T = V + r - \frac{S \times P}{|S \times P|}$.
    *Interpretation:* The source text for $T$ is garbled ("T = V + r - ..."). However, it states $T$ is along the velocity direction, and the second term ensures the particle shrinks to a small round dot of radius $r$ when velocity approaches zero or is oriented near the viewing direction. A robust implementation normalizes $V$ for direction and scales by $r$. The term $\frac{S \times P}{|S \times P|}$ suggests a correction for perspective or alignment. Given the garbled nature, the core logic is: $S$ is perpendicular to both velocity and view direction (width), and $T$ is primarily along velocity (length) but adjusted to prevent degenerate shapes [S1, unofficial].

**Particle Size Factor ($b$):**
The size of the particle is proportional to a factor $b = s(d)$, where:
*   $s(d) = 1$ for small $d$ (close to surface).
*   $s(d)$ decreases continuously to $0$ when $d$ reaches $D$.
*   To handle birth fading, the actual size factor is:
    $$ b = \min(s(d), k \cdot a) $$
    Where $a$ is the age of the particle and $k$ is a constant. This ensures new particles fade in at birth and old particles fade out as they are deleted [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trilinear_interpolate(grid, x, y, z):
    """
    Interpolate value from 8 surrounding lattice vertices.
    grid: 3D array of values.
    x, y, z: float coordinates in lattice space.
    Returns: interpolated float value.
    """
    # Implementation details omitted for brevity; standard trilinear interpolation
    pass

def compute_gradient_magnitude(grid_f, x, y, z):
    """
    Compute |grad f| using finite differences.
    """
    # Finite difference approximation
    pass

def get_particle_size_factor(d, D, age, k):
    """
    Calculate size factor b.
    s(d) decreases from 1 to 0 as d goes from 0 to D.
    """
    # Linear fade for s(d) as per description
    s_d = max(0.0, 1.0 - d / D)
    return min(s_d, k * age)

def render_spot_noise(particles, viewpoint, contour_surface, vector_field, scalar_field, C, D, N, r, k):
    """
    Main rendering loop for a single frame/time step.
    """
    # 1. Update Particle Positions (Physics)
    new_particles = []
    particles_to_delete = []
    
    for particle in particles:
        # Second-order Runge-Kutta integration
        # k1 = V(particle.pos)
        # k2 = V(particle.pos + 0.5 * dt * k1)
        # particle.pos += dt * k2
        # particle.vel = V(particle.pos)
        # particle.age += dt
        
        # Boundary Check
        # Estimate distance d from contour
        f_val = trilinear_interpolate(scalar_field, *particle.pos)
        grad_mag = compute_gradient_magnitude(scalar_field, *particle.pos)
        
        if grad_mag == 0:
            d = 1e9 # Avoid division by zero
        else:
            d = abs(f_val - C) / grad_mag
            
        # Check if in marked region (simplified: check distance and volume bounds)
        # In a full implementation, check if particle is in a marked cell
        if d >= D or not in_volume(particle.pos):
            particles_to_delete.append(particle)
        else:
            new_particles.append(particle)
            
    # Remove deleted particles
    for p in particles_to_delete:
        particles.remove(p)
        
    # 2. Replenish Particles
    # Ensure N particles per marked cell
    # This requires tracking particles per cell or re-scanning marked cells
    # For simplicity, we assume a global replenishment strategy or cell-based list
    # The source says: "check that all marked cells still have N particles"
    # Implementation would involve a grid of particle lists or spatial hash
    
    # 3. Render
    # Draw opaque contour surface first
    draw_contour_surface(contour_surface)
    
    # Draw particles
    for particle in new_particles:
        # Calculate shape vectors S and T
        # P is position relative to viewpoint
        P = particle.pos - viewpoint.pos
        V = particle.vel
        
        # Handle degenerate cases (V=0 or V parallel to P)
        if norm(V) < 1e-6 or norm(cross(V, P)) < 1e-6:
            # Draw as small dot
            draw_disk(particle.pos, r)
        else:
            S = r * normalize(cross(V, P))
            # T calculation from source: T = V + r - (S x P) / |S x P|
            # This source formula is garbled. 
            # Logical interpretation: T is along V, scaled by r, with correction
            V_norm = normalize(V)
            # Correction term to shrink when aligned with view
            correction = cross(S, P)
            if norm(correction) > 1e-6:
                correction = normalize(correction)
            else:
                correction = V_norm # Fallback
            
            # Reconstructing T based on "T is along velocity direction"
            # and "shrinks to small round dot"
            # A common approach for motion blur is T = r * V_norm
            # The source's specific garbled formula likely intended:
            # T = r * V_norm - (some term)
            # Given the ambiguity, we use a standard motion blur vector
            T = r * V_norm 
            
            # Define rectangle vertices
            v1 = particle.pos + S + T
            v2 = particle.pos + S - T
            v3 = particle.pos - S - T
            v4 = particle.pos - S + T
            
            # Calculate size factor
            f_val = trilinear_interpolate(scalar_field, *particle.pos)
            grad_mag = compute_gradient_magnitude(scalar_field, *particle.pos)
            d = abs(f_val - C) / grad_mag if grad_mag > 0 else 1e9
            
            b = get_particle_size_factor(d, D, particle.age, k)
            
            draw_blurred_rectangle(v1, v2, v3, v4, b)
```

## Operating characteristics

**Time Complexity:**
*   Particle movement: $O(N_{total})$ per time step, where $N_{total}$ is the total number of particles.
*   Distance estimation: $O(1)$ per particle (trilinear interpolation).
*   Replenishment: Depends on implementation, but checking marked cells and particle counts is $O(N_{cells})$ or $O(N_{total})$ depending on spatial indexing.

**Memory Complexity:**
*   $O(N_{total})$ for the particle linked list.
*   $O(N_{cells})$ for the cell marking array.
*   $O(V)$ for the lattice grids ($V$ is number of lattice vertices).

**Performance:**
*   The method uses hardware rendering for interactive speed, allowing real-time rotation and animation [S1, unofficial].
*   On SGI workstations (Elanm graphics, Onyxm workstation), it achieved real-time rotation and particle motion on small data sets and interactive performance on large data sets [S1, unofficial].
*   The use of a single blurred disk texture mapped to a 3D rectangle is efficient for hardware rasterization [S1, unofficial].

**Strengths:**
*   Interactive performance.
*   Does not require surface parameterization.
*   Handles non-tangential flows naturally.
*   Simple compositing (no sorting required for same-color particles).

**Limitations:**
*   Particles may overlap and obscure each other (though less so than static icons).
*   Requires a thickened region around the surface, which can be memory intensive if $D$ is large.
*   Garbled formulas in the original source for $T$ and the distance equation require careful implementation interpretation [S1, unofficial].

## Background

**Origin:**
The method was proposed by Nelson Max, Roger Crawfis, Charles Grant, and Lawrence from Lawrence Livermore National Laboratory [S1, unofficial].

**History:**
It was presented as one of four techniques for visualizing vector fields only near surfaces, aiming to overcome the overlap and occlusion issues of traditional 3D vector icons. The method builds on earlier work by Levoy for distance estimation and van Wijk for motion-blurred particles, but adapts them for contour surfaces rather than stream surfaces [S1, unofficial].

**Usage:**
Implemented in C++ using SGI's Invent2 for the user interface. It was used to visualize airflow through a HEPA filter and tornado velocity data sets [S1, unofficial].

## Sources

[S1] Visualizing 3D velocity fields near contour surfaces. Revision 1, https://digital.library.unt.edu/ark:/67531/metadc1395286/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
