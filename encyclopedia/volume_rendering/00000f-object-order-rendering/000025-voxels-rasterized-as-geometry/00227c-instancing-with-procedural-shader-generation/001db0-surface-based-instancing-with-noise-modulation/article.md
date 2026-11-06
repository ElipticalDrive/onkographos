# Surface-based Instancing with Noise Modulation

Also called: Instanced surface points, Surface primitive instancing

## Overview
Surface-based instancing with noise modulation generates high-density point clouds from parametric surface primitives to approximate volumetric effects, such as smoke or fire, at interactive or near-real-time frame rates. It solves the problem of rendering complex, noisy volumetric boundaries without the memory overhead of dense voxel grids by distributing millions of points over a surface mesh. It differs from standard surface rendering by treating the surface as a distribution of discrete particles rather than a continuous triangle mesh, and from pure particle systems by deriving point positions and attributes from a structured surface parameterization ($s,t$) rather than independent physical simulation.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of mesh pieces (surface primitives), each defined by a grid of vertices with attributes (position, normal, velocity, thickness, density). Parameters include `numPoints` (points per primitive), `densScale`, `dispScale`, `dispAmplitude`, `instanceRadius`, and noise fractal objects.
*   **Output:** A buffer of instanced points (particles) with world-space positions, density vectors, and radii, ready for rasterization.

**Data Structures:**
*   **Mesh Attributes:** Stored per mesh piece. Includes `numCols`, `numRows`, and vertex attributes accessed via interpolation macros.
*   **Point Attributes:** Stored per generated point. Includes `position` (Vector3), `density` (Vector3), and `radius` (Float).
*   **Noise Objects:** Fractal noise evaluators (`densFractal` for scalar, `dispFractal` for vector).

**Numbered Steps:**
1.  **Initialization:** For each mesh piece in the input geometry:
    *   Update mesh-specific attributes.
    *   Determine grid dimensions: $N_{cols}$ and $N_{rows}$.
    *   Calculate total points: $N_{points} = N_{cols} \times N_{rows}$.
    *   Initialize a random number generator (e.g., `Rand48`) with a seed.
2.  **Point Generation Loop:** For each instance index $i$ from $0$ to $N_{points}-1$:
    *   **Random Local Position:** Generate a random vector $P_{ls} = (x, y, z)$ where $x, y, z \sim U[0, 1]$.
    *   **Surface Parameterization:** Convert local space to surface parameters $s$ and $t$:
        $$ s = x \times (N_{cols} - 1) $$
        $$ t = y \times (N_{rows} - 1) $$
    *   **Attribute Interpolation:** Interpolate base attributes at $(s, t)$ using the surface primitive's interpolation method (e.g., bilinear or spline). Let these be:
        *   $D_{base}$: Base density vector.
        *   $P_{ws}$: World space position.
        *   $V_{ws}$: World space velocity.
        *   $N_{ws}$: World space normal (normalized).
        *   $T_{ws}$: Tangent vector ($\frac{\partial P}{\partial s}$).
        *   $B_{ws}$: Bitangent vector ($\frac{\partial P}{\partial t}$).
        *   $H$: Thickness.
    *   **Thickness Offset:** Offset the position along the normal based on the random $z$ component:
        $$ P_{ws} \leftarrow P_{ws} + \text{fit01}(z, -1.0, 1.0) \times N_{ws} \times H $$
        Where $\text{fit01}(v, a, b)$ maps $v \in [0,1]$ to $[a,b]$.
    *   **Density Noise Modulation (Optional):** If `doDensNoise` is enabled:
        *   Calculate noise lookup position: $P_{noise} = P_{ls} / S_{dens}$, where $S_{dens}$ is `densScale`.
        *   Evaluate scalar noise: $n = \text{Noise}(P_{noise})$.
        *   Calculate edge fade: $f = \text{edgeFade}(x, y, z)$.
        *   Update density: $D_{final} = D_{base} \times (n + f)$.
    *   **Displacement Noise (Optional):** If `doDispNoise` is enabled:
        *   Calculate noise lookup position: $P_{noise} = P_{ls} / S_{disp}$, where $S_{disp}$ is `dispScale`.
        *   Evaluate vector noise: $\vec{d} = \text{NoiseVec}(P_{noise})$.
        *   Displace position along tangent, bitangent, and normal:
            $$ P_{ws} \leftarrow P_{ws} + \vec{d}_x \times T_{ws} \times H \times A_{disp} $$
            $$ P_{ws} \leftarrow P_{ws} + \vec{d}_y \times B_{ws} \times H \times A_{disp} $$
            $$ P_{ws} \leftarrow P_{ws} + \vec{d}_z \times N_{ws} \times H \times A_{disp} $$
            Where $A_{disp}$ is `dispAmplitude`.
    *   **Finalize Instance:**
        *   Set point position to $P_{ws}$.
        *   Set point density to $D_{final}$.
        *   Set point radius to $R_{inst}$ (`instanceRadius`).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_instancing(meshes, params):
    points = PointBuffer()
    rng = Random48(params.seed)
    
    for mesh in meshes:
        # Update mesh attributes
        num_cols = mesh.num_cols
        num_rows = mesh.num_rows
        num_points = num_cols * num_rows
        
        # Pre-calculate derivatives if not stored
        # dPds(s,t) and dPdt(s,t) are assumed available via mesh interface
        
        for i in range(num_points):
            # 1. Random local space position
            ls_p = Vector3(rng.nextf(), rng.nextf(), rng.nextf())
            
            # 2. Surface parameters
            s = ls_p.x * (num_cols - 1)
            t = ls_p.y * (num_rows - 1)
            
            # 3. Interpolate attributes
            # Note: SURFACE_INST_INTERP handles the specific interpolation (bilinear/spline)
            base_density = mesh.interpolate("density", s, t)
            ws_pos = mesh.interpolate("wsP", s, t)
            ws_vel = mesh.interpolate("wsVelocity", s, t)
            ws_normal = mesh.interpolate("wsNormal", s, t).normalized()
            ws_tangent = mesh.derivative_s(s, t) # dPds
            ws_bitangent = mesh.derivative_t(s, t) # dPdt
            thickness = mesh.interpolate("thickness", s, t)
            
            # 4. Offset along normal
            z_factor = fit01(ls_p.z, -1.0, 1.0)
            ws_pos += z_factor * ws_normal * thickness
            
            # 5. Density Noise
            if params.do_dens_noise:
                ns_lookup = ls_p / params.dens_scale
                noise_val = params.dens_fractal.eval(ns_lookup)
                fade = edge_fade(ls_p.x, ls_p.y, ls_p.z)
                final_density = base_density * (noise_val + fade)
            else:
                final_density = base_density
                
            # 6. Displacement Noise
            if params.do_disp_noise:
                ns_lookup = ls_p / params.disp_scale
                disp_vec = params.disp_fractal.eval_vec(ns_lookup)
                amp = params.disp_amplitude
                ws_pos += disp_vec.x * ws_tangent * thickness * amp
                ws_pos += disp_vec.y * ws_bitangent * thickness * amp
                ws_pos += disp_vec.z * ws_normal * thickness * amp
            
            # 7. Set attributes
            points.set_position(i, ws_pos)
            points.set_vector_attr("density", i, final_density)
            points.set_float_attr("radius", i, params.instance_radius)
            
    return points

def fit01(v, a, b):
    return a + v * (b - a)

def edge_fade(x, y, z):
    # Implementation detail: typically smoothstep or similar to fade edges of the local cube
    # Exact formula not specified in source, assumed to be a standard edge fade function
    return 1.0 # Placeholder for actual edge fade logic [G]
```

**Parameters:**
*   `numPoints`: Number of points per primitive. Typical value: ~400 million for smooth results at 1024x1024 resolution [S1, unofficial].
*   `densScale`: Scale factor for density noise lookup [S1, unofficial].
*   `dispScale`: Scale factor for displacement noise lookup [S1, unofficial].
*   `dispAmplitude`: Amplitude of displacement noise [S1, unofficial].
*   `instanceRadius`: Radius of each instanced point [S1, unofficial].
*   `voxelBufferResolution`: Frustum-shaped buffer resolution, approx. 1200x1200x250 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Wasteful Evaluation:** Points must be instantiated, noise positions calculated, and full fractal noise functions evaluated before culling if density is zero. This is computationally expensive [S1, unofficial].
*   **Precision:** High `numPoints` requires 64-bit indices or indirect draw calls to manage memory and draw overhead.
*   **Boundary Artifacts:** Edge fade functions are required to prevent hard edges on the local space cube of each point generation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of mesh pieces and $M$ is the number of points per piece. The cost is dominated by noise evaluation, which is independent of the final rendered pixel count but dependent on the number of generated points.
*   **Memory Complexity:** $O(N \cdot M)$ for storing point attributes. With 400 million points, this requires significant VRAM (e.g., 400M points * 32 bytes/point ≈ 12.8 GB).
*   **Performance:** Enables smooth volumetric appearance at high resolutions (1024x1024) using a relatively small voxel buffer (1200x1200x250) for culling [S1, unofficial].
*   **Strengths:** High visual fidelity for noisy surfaces; decouples resolution from mesh complexity.
*   **Limitations:** High memory usage; inefficient if most points are culled due to zero density; requires careful tuning of noise scales and amplitudes.

## Background
This technique is described in "Production Volume Rendering" [S1, unofficial], which details methods for achieving high-quality volumetric effects in production environments. It builds on the principles of surface-based instancing, where primitives are generated from surface meshes rather than volumetric grids.

## Sources
[S1] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
