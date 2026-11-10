# Point Wisp Voxelization

Also called: Steps to Grow Point Wisps, Wisp algorithm

## Overview
Point Wisp Voxelization is a procedural method for generating volumetric smoke data by distributing points around a "guide particle" using fractal noise and baking them into a voxel grid [S1, unofficial]. It solves the problem of creating detailed, organic-looking smoke volumes from sparse particle data, distinguishing itself from standard point-splatting by using correlated random walks and radial displacement to form cohesive "wisps" rather than isolated blobs [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Guide particle position: $(x_{particle}, y_{particle}, z_{particle})$ [S1, unofficial].
    *   Guide particle size: $particle\_size$ [S1, unofficial].
    *   Guide particle density: $particle\_density$ [S1, unofficial].
    *   Guide particle color: $particle\_color$ [S1, unofficial].
    *   Clump exponent: $clump$ [S1, unofficial].
    *   Number of dots per particle: Not given in the sources.
    *   Noise function: $fBm(x, y, z)$, representing fractal Brownian motion (fractal Perlin noise) [S1, unofficial].
*   **Outputs:** A voxel grid where voxels are updated with density and color contributions from generated dots [S1, unofficial].

**Data Structures**
The voxel grid stores density ($\rho$) and color ($c$) for each voxel. When baking a dot at position $(xdot, ydot, zdot)$, the algorithm updates the 8 nearest voxels using a weighting factor $\omega_{abc}$ [S1, unofficial]. The update rule is:
$$ \rho_{abc}^{+} = \rho_{abc} + \rho_{dot} \omega_{abc} $$
$$ c_{abc}^{+} = c_{abc} + c_{dot} \omega_{abc} $$
where $\rho_{dot}$ and $c_{dot}$ are the density and color of the dot, and $\omega_{abc}$ is the weight for voxel $(a,b,c)$ [S1, unofficial]. The exact formula for $\omega_{abc}$ is not given in the sources, but it implies an anti-aliased, distance-based contribution (smearing) [S1, unofficial].

**Parameters**
*   $clump$: An exponent used in the radial displacement calculation to control the clumping of the noise [S1, unofficial]. Typical values are not given in the sources.

**Steps**
1.  Initialize the guide particle parameters: $(x_{particle}, y_{particle}, z_{particle})$, $particle\_size$, $particle\_density$, and $particle\_color$ [S1, unofficial].
2.  For each dot to be generated for this guide particle:
    a.  Generate a random position $(x, y, z)$ uniformly between -1 and 1:
        $$ x = 2.0 \cdot drand48() - 1 $$
        $$ y = 2.0 \cdot drand48() - 1 $$
        $$ z = 2.0 \cdot drand48() - 1 $$
        where $drand48()$ is a random number generator returning a value in $[0, 1)$ [S1, unofficial].
    b.  Move the random position to the unit sphere by normalizing the vector:
        $$ radius = \sqrt{x^2 + y^2 + z^2} $$
        $$ xsphere = x / radius $$
        $$ ysphere = y / radius $$
        $$ zsphere = z / radius $$
    c.  Displace the point radially from the unit sphere using fractal Brownian motion:
        $$ radial\_disp = pow(fabs(fBm(x, y, z)), clump) $$
        $$ xsphere *= radial\_disp $$
        $$ ysphere *= radial\_disp $$
        $$ zsphere *= radial\_disp $$
    d.  Map the displaced sphere coordinates to the guide particle's coordinate system:
        $$ xdot = x_{particle} + xsphere \cdot particle\_size $$
        $$ ydot = y_{particle} + ysphere \cdot particle\_size $$
        $$ zdot = z_{particle} + zsphere \cdot particle\_size $$
    e.  Displace the dot position again using 3D vector fractal noise:
        $$ xfsn = fBm(xsphere, ysphere, zsphere) $$
        $$ yfsn = fBm(xsphere + 0.1, ysphere + 0.1, zsphere + 0.1) $$
        $$ zfsn = fBm(xsphere - 0.1, ysphere - 0.1, zsphere - 0.1) $$
        $$ xdot += xfsn $$
        $$ ydot += yfsn $$
        $$ zdot += zfsn $$
    f.  Bake the dot into the voxel grid at $(xdot, ydot, zdot)$ with $particle\_density$ and $particle\_color$ [S1, unofficial].
3.  Repeat step 2 for the required number of dots per guide particle.
4.  Repeat the entire process for the next guide particle.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_wisp_voxelization(guide_particles, voxel_grid, fBm, clump):
    """
    guide_particles: List of tuples (x_particle, y_particle, z_particle, particle_size, particle_density, particle_color)
    voxel_grid: Object with method update_voxels(x, y, z, density, color)
    fBm: Function implementing fractal Brownian motion
    clump: Float exponent for radial displacement
    """
    for x_particle, y_particle, z_particle, particle_size, particle_density, particle_color in guide_particles:
        # Number of dots per particle is not specified in sources [G]
        num_dots = 1000 
        for _ in range(num_dots):
            # Step 2a: Generate random position between -1 and 1
            x = 2.0 * random_uniform() - 1.0
            y = 2.0 * random_uniform() - 1.0
            z = 2.0 * random_uniform() - 1.0
            
            # Step 2b: Move to unit sphere
            radius = math.sqrt(x*x + y*y + z*z)
            # Guard against division by zero [G]
            if radius == 0:
                continue
            xsphere = x / radius
            ysphere = y / radius
            zsphere = z / radius
            
            # Step 2c: Displace radially from sphere using fractal sum
            fbm_val = fBm(x, y, z)
            radial_disp = math.pow(math.fabs(fbm_val), clump)
            xsphere *= radial_disp
            ysphere *= radial_disp
            zsphere *= radial_disp
            
            # Step 2d: Map to guide particle coordinate
            xdot = x_particle + xsphere * particle_size
            ydot = y_particle + ysphere * particle_size
            zdot = z_particle + zsphere * particle_size
            
            # Step 2e: Displace again with 3D fractal sum noise
            xfsn = fBm(xsphere, ysphere, zsphere)
            yfsn = fBm(xsphere + 0.1, ysphere + 0.1, zsphere + 0.1)
            zfsn = fBm(xsphere - 0.1, ysphere - 0.1, zsphere - 0.1)
            
            xdot += xfsn
            ydot += yfsn
            zdot += zfsn
            
            # Step 2f: Bake dot to voxel grid
            voxel_grid.update_voxels(xdot, ydot, zdot, particle_density, particle_color)
```

**Edge Cases and Pitfalls**
*   **Zero Radius:** If the random point $(x,y,z)$ is exactly $(0,0,0)$, normalization fails. A guard is required [G].
*   **Noise Implementation:** The $fBm$ function must be consistent across the radial and vector displacement steps. The vector noise uses offset inputs $(\pm 0.1)$ to ensure the three components are not identical, creating a 3D vector field [S1, unofficial].
*   **Baking Weights:** The exact weighting function $\omega_{abc}$ for anti-aliasing is not defined in the sources. Implementers must choose a standard splatting kernel (e.g., Gaussian or linear) [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of guide particles and the number of dots per particle. Each dot requires multiple noise evaluations and vector operations.
*   **Memory Complexity:** Depends on the voxel grid size. The algorithm itself uses constant memory per dot.
*   **Strengths:** Produces organic, wispy structures that resemble smoke or fire better than simple point splatting. The use of fractal noise allows for multi-scale detail.
*   **Limitations:** The method is procedural and may lack precise control over large-scale volume shape compared to simulation-based methods. The "clump" parameter requires tuning.
*   **When to use:** When generating background or secondary volumetric effects where performance is critical and exact physical accuracy is not required.
*   **When not to use:** When precise control over the volume's macroscopic shape is needed, or when coupling with fluid simulation data.

## Background
The method was presented by Jason Tessen in a report titled "Production Volume Rendering" from Clemson University, dated November 24, 2008 [S1, unofficial]. It is part of a pipeline for generating volume data for production rendering, specifically focusing on efficient voxelization techniques.

## Sources
[S1] PDF Production Volume Rendering - Clemson University, https://jtessen.people.clemson.edu/reports/papers_files/ProdVolRender.pdf, fetched 2026-10-27 (unofficial)
