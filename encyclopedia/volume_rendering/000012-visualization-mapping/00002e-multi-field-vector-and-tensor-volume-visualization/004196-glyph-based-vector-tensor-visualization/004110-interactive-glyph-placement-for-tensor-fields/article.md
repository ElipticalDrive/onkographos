# Interactive Glyph Placement for Tensor Fields

Also called: Glyph Packing, Glyph Packing with Delaunay Triangulation, Glyph Packing with Delaunay Acceleration, Hlawitschka et al. Glyph Placement

## Overview
Interactive Glyph Placement for Tensor Fields is an iterative algorithm that distributes glyphs densely throughout a tensor field (volume or slice) to minimize overlap while preserving local tensor information [S1, S2]. It solves the problem of visual clutter and misleading artifacts caused by regular grid sampling, where consistent alignment or overlap creates false impressions of spatial variation [S2]. Unlike sibling methods that rely on fixed bin sizes or simple repulsion, this method uses a Delaunay triangulation acceleration structure to speed up neighbor searches and allows for arbitrary glyph shapes via a surface function $g(y)$ [S1]. It is specifically designed for interactive exploration, allowing users to move planes or regions of interest while the glyph cloud reorganizes in real-time [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Discrete tensor field data $\mathbf{D}_i$ at voxel centers.
*   Confidence mask $M$ (binary or scalar) indicating valid tensor regions [S2].
*   User parameters: Glyph scaling factor $\alpha$ and distance parameter $\gamma$ [S1, S2].
*   Optional: Initial number of glyphs $P$ or area of interest [S1].

**Outputs:**
*   A set of particle positions $\{\mathbf{p}_a\}$ representing glyph centers.
*   A Delaunay triangulation of these positions for neighbor queries [S1].

### Data Structures
1.  **Particle List:** Array of structures containing:
    *   $\mathbf{p}_a$: 3D position vector.
    *   $\mathbf{v}_a$: 3D velocity vector.
    *   $\mathbf{D}(\mathbf{p}_a)$: Interpolated tensor at position.
2.  **Spatial Bins (Kindlmann/Westin):** Dynamic lists of particle indices grouped into a 3D grid. Bin edge length $L_{bin} = 2\alpha\lambda_{max}(1+\gamma)$, where $\lambda_{max}$ is the maximal eigenvalue of the dataset [S2].
3.  **Delaunay Triangulation (Hlawitschka):** A triangulation of particle positions. Supports breadth-first search (BFS) to find neighbors within edge distance [S1].
4.  **Interpolated Field:** Continuous tensor field $\mathbf{D}(\mathbf{p})$ obtained via cubic spline interpolation (C2 field) using pre-filtering and convolution-based separable reconstruction with uniform cubic B-splines [S2].

### Initialization
The method supports two initialization strategies depending on glyph density relative to cell size [S1]:

**Variant 1: Sparse Glyphs (<1 per cell)**
1.  Calculate cell area $A[v]$ and estimated glyph area $a[v]$ for each voxel $v$.
2.  Compute density ratio $p[v] = A[v]/a[v]$.
3.  Integrate density: $p[v] = p[v-1] + p[v]$ for $v=1 \dots N_{cells}$.
4.  Normalize: $p[v] = p[v] / P$, where $P = p[N_{cells}-1]$.
5.  For $i = 0 \dots P-1$:
    *   Generate random $d \in [0, 1)$.
    *   Binary search for cell $i$ such that $p[i-1] \le d < p[i]$ (with $p[-1]=0$).
    *   Place glyph randomly within cell $i$.

**Variant 2: Dense Glyphs (>1 per cell)**
1.  Determine number of glyphs per voxel $N_v = \text{round}(p[v])$.
2.  Fill each voxel with $N_v$ glyphs using stratified sampling (e.g., Poisson-disc or interleaved sampling via hash map using cell index) to avoid regular patterns [S1].

**Alternative Initialization (Kindlmann/Westin):**
1.  Traverse dataset to find $\det_{max}$ and $\lambda_{max}$ [S2].
2.  Initialize particles at random locations.
3.  Reject candidates outside confidence mask $M$.
4.  Reject candidate $\mathbf{p}$ with probability $\det(\mathbf{D}(\mathbf{p})) / \det_{max}$ to approximate correct density [S2].

### Force Model and Energy Function
The system evolves to minimize potential energy. The force on particle $a$ from particle $b$ is derived from an energy function $E_{ab}$ [S2]:

$$ E_{ab} = \phi(|\mathbf{x}_{ab}|) \cdot g(\mathbf{D}_{ab}^{-1}, \mathbf{y}_{ab}) $$

Where:
*   $\mathbf{y}_{ab} = \mathbf{p}_a - \mathbf{p}_b$ [S1].
*   $\mathbf{x}_{ab} = \mathbf{D}_{ab}^{-1} \mathbf{y}_{ab}$ [S2].
*   $\mathbf{D}_{ab}^{-1} \approx (\mathbf{D}^{-1}(\mathbf{p}_a) + \mathbf{D}^{-1}(\mathbf{p}_b)) / 2$ (fast approximation) [S2].
*   $g(\mathbf{D}^{-1}, \mathbf{y}) = \|\mathbf{D}^{-1} \mathbf{y}\|$ is the glyph surface function (distance from center to surface along $\mathbf{y}$) [S1, S2].
*   $\phi(r)$ is the radial potential function. Its derivative $\phi'(r)$ ensures $C^1$ continuity:

$$
\phi'(r) =
\begin{cases}
0 & 0 < r < 1 \\
(r - 1)(1 + \gamma - r)^2 / \gamma^2 & 1 \le r \le 1 + \gamma \\
0 & r > 1 + \gamma
\end{cases}
$$

Where $\gamma$ is a tunable width parameter (typical value 0.5) [S1, S2]. The optimum distance is $r=1$. Glyphs farther than $1+\gamma$ exert no force [S1].

**Force Calculation:**
Assuming local tensor constancy (ignoring the third-order tensor gradient $\partial \mathbf{D}^{-1}/\partial \mathbf{y}$), the inter-particle force is:

$$ \mathbf{f}_{ab} = - \frac{\phi'(|\mathbf{x}_{ab}|)}{2\alpha |\mathbf{x}_{ab}|} \mathbf{D}_{ab}^{-1} \mathbf{x}_{ab} $$

Additional forces:
*   **Drag:** $\mathbf{f}_{drag} = -C_{drag} \frac{d\mathbf{p}}{dt}$ [S1, S2].
*   **Mask:** $\mathbf{f}_{mask} = C_{mask} \nabla M(\mathbf{p})$ pushes particles into valid regions [S2].

### Main Iteration Loop
The system solves the second-order ODE:

$$ \frac{d^2\mathbf{p}}{dt^2} + C_{drag}\frac{d\mathbf{p}}{dt} = \sum_{b \neq a} \mathbf{f}_{ab} + \mathbf{f}_{mask} $$

Using Euler integration with step size $h=0.2$ [S1, S2].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_glyphs(particles, tensor_field, mask, alpha, gamma, c_drag, c_mask, max_iterations):
    # Initialize Delaunay triangulation or Spatial Bins
    triangulation = compute_delaunay(particles)
    bins = create_spatial_bins(particles, alpha, gamma)
    
    for iteration in range(max_iterations):
        # 1. Compute Forces
        for particle in particles:
            forces = vector(0, 0, 0)
            
            # Inter-particle forces
            neighbors = get_neighbors(particle, triangulation, bins)
            for neighbor in neighbors:
                y = particle.pos - neighbor.pos
                # Approximate inverse tensor at midpoint
                D_inv_ab = (tensor_field.inverse(particle.pos) + tensor_field.inverse(neighbor.pos)) / 2.0
                x = D_inv_ab @ y
                r = norm(x)
                
                if r > 1 + gamma:
                    continue
                
                # Compute phi'(r)
                if r < 1:
                    phi_prime = 0
                elif r <= 1 + gamma:
                    phi_prime = (r - 1) * (1 + gamma - r)**2 / gamma**2
                else:
                    phi_prime = 0
                
                # Force calculation
                if r > 0:
                    factor = phi_prime / (2 * alpha * r)
                    f_ab = -factor * (D_inv_ab @ x)
                    forces += f_ab
            
            # Mask force
            if mask.gradient_exists(particle.pos):
                forces += c_mask * mask.gradient(particle.pos)
            
            # Drag force
            forces -= c_drag * particle.vel
            
            particle.acc = forces # Assuming unit mass
        
        # 2. Update Positions (Euler Integration)
        for particle in particles:
            particle.vel += particle.acc * 0.2 # Step size 0.2
            particle.pos += particle.vel * 0.2
            
            # Project to slice plane if in 2D mode [S2]
            if mode == 2D:
                particle.pos.z = slice_z

        # 3. Update Acceleration Structure
        triangulation.update(particles)
        bins.update(particles)
        
        # 4. Dynamic Addition/Removal (Optional)
        if allow_dynamic_glyphs:
            # Add glyphs in triangles with long edges
            add_glyphs_in_large_triangles(triangulation, particles)
            # Remove glyphs with high repulsion on all edges
            remove_glyphs_with_high_repulsion(triangulation, particles)
            
        # 5. Convergence Check
        if energy_change < threshold:
            break
```

### Interactive Group Movement
For interactive probing:
1.  Place a cluster of glyphs randomly around the cursor [S1].
2.  Optimize the grid while displayed [S1].
3.  When cursor moves, translate all points by the cursor delta [S1].
4.  Fix one glyph to the cursor to prevent diffusion [S1].
5.  Apply small additional forces pointing to the cursor to maintain symmetry [S1].

## Operating characteristics

**Time and Memory Complexity:**
*   **Force Computation:** $O(N)$ with spatial binning or Delaunay triangulation, where $N$ is the number of glyphs. Without acceleration, it is $O(N^2)$ [S1, S2].
*   **Triangulation Update:** $O(N \log N)$ or better depending on implementation [S1].
*   **Memory:** $O(N)$ for particle data and acceleration structures [S1, S2].

**Measured Performance:**
*   **Hlawitschka et al.:** Up to 8000 glyphs optimized interactively on a desktop PC. Complete slices take ~1 minute for first results, up to 4 minutes for stability [S1]. Convergence is fast: almost stable after ~15 steps, usually stable after several hundred steps [S1].
*   **Kindlmann et al.:** 4000 iterations in ~1.5 hours on a dual 2.0 GHz PowerPC G5 for 3D; 2000 iterations in ~8 minutes on a 1.67 GHz PowerPC G4 for 2D [S2].

**Strengths:**
*   Removes visual artifacts of regular grids [S2].
*   Delaunay triangulation avoids re-computation when planes move through a volume, enabling true interactivity [S1].
*   User parameters are intuitive: glyph scaling $\alpha$ and distance $\gamma$ [S1, S2].
*   Supports arbitrary glyph shapes via $g(y)$ [S1].

**Limitations:**
*   Tensor inverse and interpolation at midpoints are computationally expensive [S2].
*   Holes in initial seeding can remain stable due to weak forces across large distances [S1].
*   2D projection artifacts may occur if 3D forces do not cleanly separate 3D glyphs viewed from above [S2].
*   Insufficient signal regions (skull/air) require confidence masks to avoid meaningless tensor estimates [S2].

## Background

**Origin:**
The method builds upon "Glyph Packing" introduced by Kindlmann and Westin [S1, S2]. Kindlmann and Westin (2006) proposed using particle systems with anisotropic potential energy to densely pack glyphs in diffusion tensor MRI data [S2]. Hlawitschka, Scheuermann, and Hamann (2008) improved this by introducing a Delaunay triangulation acceleration structure, improved initial seeding, and interactive group movement [S1].

**History and Development:**
*   **Kindlmann and Westin (2006):** Introduced the concept of using particle systems with tensor-shaped potential energy to pack glyphs, reducing overlap and highlighting continuous features [S2].
*   **Hlawitschka et al. (2008):** Enhanced the method for interactive visualization. Key contributions included the parameter-less Delaunay acceleration structure, improved initial placement algorithms to prevent holes, and interactive cursor-following capabilities [S1].

**Usage:**
Primarily used in medical imaging, specifically Diffusion Tensor Imaging (DTI) for neurosurgery planning. It helps assess tumor extent and plan minimally damaging approaches through brain white matter by providing a dense, non-overlapping visualization of fiber orientations [S1, S2].

## Sources

[S1] Interactive Glyph Placement for Tensor Fields, https://doi.org/10.1007/978-3-540-76858-6_33, fetched 2026-10-27
[S2] Diffusion tensor visualization with glyph packing, http://lmi.bwh.harvard.edu/papers/pdfs/2006/KindlmannTVCG2006.pdf, fetched 2026-10-29
