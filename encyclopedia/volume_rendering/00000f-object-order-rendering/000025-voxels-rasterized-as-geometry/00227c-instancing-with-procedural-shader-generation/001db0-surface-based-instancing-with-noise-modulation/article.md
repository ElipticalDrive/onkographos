# Surface-based Instancing with Noise Modulation

Also called: Instanced surface points, Surface primitive instancing

## Overview
Surface-based Instancing with Noise Modulation is a technique for rendering volumetric primitives by generating a dense cloud of instanced points on a parametric surface, where point density and position are modulated by procedural noise functions [S1, unofficial]. It solves the problem of creating smooth, organic volumetric appearances (such as smoke or fire) from discrete surface meshes by distributing millions of sample points across the geometry [S1, unofficial]. It differs from sibling methods like [3D Gaussian Splatting via Instancing](1586-3d-gaussian-splatting-via-instancing.md) by using scalar/vector noise to drive density and displacement rather than learned Gaussian parameters, and from [Procedural Particle Rendering](2926-procedural-particle-rendering.md) by explicitly mapping noise to surface parameter space ($s,t$) rather than world space [S1, unofficial].

## How it works
**Inputs:**
- A mesh geometry composed of pieces, where each piece has attributes: `density` (Vector), `wsP` (World Space Position, Vector), `wsVelocity` (Vector), `wsNormal` (Vector), `thickness` (Float), and grid dimensions `numCols`, `numRows` [S1, unofficial].
- Noise configuration: `doDensNoise` (Boolean), `doDispNoise` (Boolean), `densScale` (Float), `dispScale` (Float), `dispAmplitude` (Float), `densFractal` (Scalar Noise Evaluator), `dispFractal` (Vector Noise Evaluator) [S1, unofficial].
- Instancing parameters: `numPoints` (Integer, typically ~400 million), `instanceRadius` (Float), `seed` (Integer) [S1, unofficial].

**Outputs:**
- A point cloud with attributes: `position` (Vector), `density` (Vector), `radius` (Float) [S1, unofficial].

**Data Structures:**
- `Meshes::CPtr meshes`: Container of mesh pieces.
- `AttrVisitor`: Iterator for mesh and point attributes.
- `Imath::Rand48`: Random number generator state [S1, unofficial].
- `SURFACE_INST_INTERP`: Macro/function for bilinear interpolation of attributes on a quad-connected mesh patch given $s,t$ coordinates [S1, unofficial].

**Steps:**
1. Iterate over each mesh piece in the input geometry.
2. Update mesh-specific attributes (`updateSurfAttrs`) and point attributes for the current piece (`updatePointAttrs`) [S1, unofficial].
3. Calculate the grid dimensions: `numCols` and `numRows` for the current piece [S1, unofficial].
4. Initialize a random number generator `rng` with the configured `seed` [S1, unofficial].
5. Loop for `i` from 0 to `m_surfAttrs.numPoints - 1`:
    a. Generate a random local space position `lsP` using `rng.nextf()` for x, y, and z components (uniform in [0,1]) [S1, unofficial].
    b. Compute surface parameters:
       $$s = lsP.x \times (numCols - 1)$$
       $$t = lsP.y \times (numRows - 1)$$
    c. Interpolate instance attributes at $(s,t)$ using `SURFACE_INST_INTERP`:
       - `instanceDensity` (Vector)
       - `instanceWsP` (Vector)
       - `instanceWsV` (Vector)
       - `wsN` (Vector, normalized)
       - `wsDPds` (Vector, tangent derivative)
       - `wsDPdt` (Vector, bitangent derivative)
       - `thickness` (Float)
    d. Offset the world space position along the normal:
       $$instanceWsP += \text{fit01}(lsP.z, -1.0, 1.0) \times wsN \times thickness$$
       Where `fit01` maps the z-component from [0,1] to [-1,1] [S1, unofficial].
    e. If `doDensNoise` is true:
       - Calculate noise lookup position: $nsLookupP = lsP / densScale$
       - Evaluate scalar noise: $noise = densFractal.eval(nsLookupP)$
       - Calculate edge fade: $fade = edgeFade(lsP.x, lsP.y, lsP.z)$
       - Update density: $instanceDensity *= noise + fade$
    f. If `doDispNoise` is true:
       - Calculate noise lookup position: $nsLookupP = lsP / dispScale$
       - Evaluate vector noise: $disp = dispFractal.evalVec(nsLookupP)$
       - Update position by adding displacement components:
         $$instanceWsP += disp.x \times wsDPds \times thickness \times dispAmplitude$$
         $$instanceWsP += disp.y \times wsDPdt \times thickness \times dispAmplitude$$
         $$instanceWsP += disp.z \times wsN \times thickness \times dispAmplitude$$
    g. Set instance attributes:
       - Position: `instanceWsP`
       - Density: `instanceDensity`
       - Radius: `instanceRadius`
6. Repeat for all mesh pieces [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_based_instancing(meshes, params):
    """
    meshes: List of mesh pieces, each with attrs: density, wsP, wsVelocity, wsNormal, thickness, numCols, numRows
    params: dict containing numPoints, instanceRadius, seed, doDensNoise, doDispNoise, 
            densScale, dispScale, dispAmplitude, densFractal, dispFractal
    """
    points = [] # Output point cloud
    
    for mesh in meshes:
        # Update attributes for this mesh piece
        update_surf_attrs(mesh)
        
        num_cols = mesh.numCols
        num_rows = mesh.numRows
        
        # Initialize RNG
        rng = Rand48(seed=params['seed'])
        
        # Pre-calculate derivatives if not available in mesh attrs
        # Note: SURFACE_INST_INTERP handles interpolation of dPds/dPdt
        
        for i in range(params['numPoints']):
            # 1. Random local space position
            ls_p = V3f(rng.nextf(), rng.nextf(), rng.nextf())
            
            # 2. Surface parameters
            s = ls_p.x * (num_cols - 1)
            t = ls_p.y * (num_rows - 1)
            
            # 3. Interpolate attributes
            # Assuming SURFACE_INST_INTERP returns the interpolated value
            instance_density = surf_inst_interp(mesh, 'density', s, t)
            instance_ws_p = surf_inst_interp(mesh, 'wsP', s, t)
            instance_ws_v = surf_inst_interp(mesh, 'wsVelocity', s, t)
            ws_n = surf_inst_interp(mesh, 'wsNormal', s, t).normalized()
            ws_dpds = surf_inst_interp(mesh, 'dPds', s, t)
            ws_dpdt = surf_inst_interp(mesh, 'dPdt', s, t)
            thickness = surf_inst_interp(mesh, 'thickness', s, t)
            
            # 4. Offset along normal
            z_offset = fit01(ls_p.z, -1.0, 1.0)
            instance_ws_p += z_offset * ws_n * thickness
            
            # 5. Density Noise
            if params['do_dens_noise']:
                ns_lookup_p = ls_p / params['dens_scale']
                noise = params['dens_fractal'].eval(ns_lookup_p)
                fade = edge_fade(ls_p.x, ls_p.y, ls_p.z)
                instance_density *= noise + fade
            
            # 6. Displacement Noise
            if params['do_disp_noise']:
                ns_lookup_p = ls_p / params['disp_scale']
                disp = params['disp_fractal'].eval_vec(ns_lookup_p)
                amp = params['disp_amplitude']
                
                instance_ws_p += disp.x * ws_dpds * thickness * amp
                instance_ws_p += disp.y * ws_dpdt * thickness * amp
                instance_ws_p += disp.z * ws_n * thickness * amp
            
            # 7. Set attributes
            points.append({
                'position': instance_ws_p,
                'density': instance_density,
                'radius': params['instance_radius']
            })
            
    return points

def fit01(val, min_val, max_val):
    """Maps value from [0,1] to [min_val, max_val]"""
    return min_val + val * (max_val - min_val)

def edge_fade(x, y, z):
    """Placeholder for edge fade function. Implementation details not provided in source."""
    # [G] Typically fades out points near the boundaries of the local space cube
    # to prevent artifacts at the edges of the primitive.
    return 1.0 
```

**Parameters:**
- `numPoints`: Number of instanced points per primitive. Typically ~400 million for smooth results at 1024x1024 resolution [S1, unofficial].
- `voxel buffer resolution`: Frustum-shaped buffer of ~1200x1200x250 used for rasterization of the instanced points [S1, unofficial].
- `densScale`, `dispScale`: Scaling factors for noise lookup coordinates [S1, unofficial].
- `dispAmplitude`: Magnitude of displacement noise [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Wasteful Evaluation:** Primitives modulated by noise are computationally wasteful because a point must be instantiated, its noise space position calculated, and the full fractal noise function evaluated before determining if it should be culled due to zero density [S1, unofficial].
- **Boundary Artifacts:** Points near the edges of the local space cube may require edge fading to prevent visual artifacts [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of mesh pieces and $P$ is the number of instanced points per piece. The cost is dominated by noise evaluation for each point if noise modulation is enabled [S1, unofficial].
- **Memory Complexity:** $O(P)$ for storing the instanced point attributes (position, density, radius) [S1, unofficial].
- **Performance:** Requires high point counts (~400 million) to achieve smooth results at standard resolutions (1024x1024) [S1, unofficial].
- **Strengths:** Produces smooth, organic volumetric appearances; flexible noise modulation for density and displacement [S1, unofficial].
- **Limitations:** High computational cost due to noise evaluation per point; potential for wasted cycles on points that are ultimately culled [S1, unofficial].
- **When to use:** When organic, noise-driven volumetric effects are needed and computational resources allow for high point counts [S1, unofficial].
- **When not to use:** When performance is critical and noise modulation is not required, or when the scene cannot support the memory bandwidth of hundreds of millions of points [S1, unofficial].

## Background
Originated in production volume rendering pipelines, specifically described in "Production Volume Rendering" (2011) [S1, unofficial]. It is used in visual effects for simulating organic volumetric phenomena like smoke and fire by instancing points on surface primitives and modulating them with Perlin noise [S1, unofficial].

## Sources
[S1] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
