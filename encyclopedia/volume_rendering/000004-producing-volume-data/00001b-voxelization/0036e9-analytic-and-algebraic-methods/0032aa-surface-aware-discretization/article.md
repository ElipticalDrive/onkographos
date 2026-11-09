# Surface-Aware Discretization

Also called: Mesh-intersection voxelization, Surface-aware voxelization

## Overview
Surface-Aware Discretization is a voxelization method that computes the effective volume of each voxel based on the intersection of the voxel with a target mesh, rather than assuming the voxel is fully filled or empty [S1, unofficial]. This approach mitigates resolution-related artifacts in volumetric additive manufacturing prints by accurately modeling the partial occupancy of voxels near the surface [S1, unofficial]. It differs from [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md) and [Analytic Voxelization](10493-analytic-voxelization.md) by using Monte Carlo ray tracing against the mesh geometry to estimate occupancy, rather than relying on distance fields or closed-form geometric integrals [S1, unofficial].

## How it works
**Inputs:**
- A target mesh (scene object) representing the surface to be voxelized.
- A voxel grid defined by resolution `res` (x, y, z dimensions) and voxel size `voxel_size`.
- The bounding box of the voxel grid.
- A sample count `sample_count` for Monte Carlo estimation.

**Outputs:**
- A tensor `self.volumes` of shape `res`, containing the effective volume of each voxel.

**Data Structures:**
- `voxel`: A tensor of `Point3f` coordinates representing the integer indices of each voxel in the grid, generated via `meshgrid`.
- `count_in`, `count_out`: Integer tensors of size `prod(res)`, tracking the number of rays hitting the inside and outside of the mesh for each voxel.
- `active`: A boolean mask indicating which voxels still require sampling.

**Parameters:**
- `sample_count`: Number of Monte Carlo samples per voxel. Default value is $2^{14}$ [S1, unofficial].

**Algorithm:**
1. **Initialization:**
   - If the film is not surface-aware, return the full voxel volume: $V = \text{prod}(\text{voxel\_size})$ for all voxels [S1, unofficial].
   - If volumes are already computed, return them [S1, unofficial].
   - Identify the target mesh shape in the scene. If not found, raise `ValueError` [S1, unofficial].
   - Create a `target_scene` containing only the target shape [S1, unofficial].
   - Initialize `self.volumes` as a tensor of ones with shape `self.m_film.data.shape` [S1, unofficial].
   - Generate voxel coordinates:
     $$
     \begin{aligned}
     x &= \text{arange}(0, \text{res}.x) \\
     y &= \text{arange}(0, \text{res}.y) \\
     z &= \text{arange}(0, \text{res}.z) \\
     z_{idx}, y_{idx}, x_{idx} &= \text{meshgrid}(z, y, x, \text{indexing}='ij') \\
     \text{voxel} &= \text{Point3f}(x_{idx}, y_{idx}, z_{idx})
     \end{aligned}
     $$
   - Initialize `count_in` and `count_out` to zero tensors of size $\text{prod}(\text{res})$ [S1, unofficial].
   - Initialize `d = 0` and `active` mask to all `True` [S1, unofficial].
   - Create an independent sampler and seed it with `0` and total samples $\text{prod}(\text{res}) \times \text{sample\_count}$ [S1, unofficial].

2. **Monte Carlo Sampling Loop:**
   - While `active` is true:
     - Generate random offset:
       $$
       \text{offset} = \text{Point3f}(\text{sampler.next\_1d}(), \text{sampler.next\_1d}(), \text{sampler.next\_1d}())
       $$
     - Compute ray origin:
       $$
       \text{pos} = \text{bbox.min} + \text{voxel\_size} \times (\text{voxel} + \text{offset})
       $$
     - Generate random direction uniformly over the sphere:
       $$
       \text{dir} = \text{warp.square\_to\_uniform\_sphere}(\text{sampler.next\_2d}())
       $$
     - Construct ray: $\text{ray} = \text{Ray3f}(\text{pos}, \text{dir})$ [S1, unofficial].
     - Check if ray origin is inside the mesh bounding box:
       $$
       \text{in\_mesh\_bbox} = \text{all}((\text{ray.o} > \text{bbox.min}) \land (\text{ray.o} < \text{bbox.max}))
       $$
     - Perform ray intersection with `target_scene`:
       $$
       \text{si} = \text{target\_scene.ray\_intersect}(\text{ray}, \text{active} = \text{active} \land \text{in\_mesh\_bbox})
       $$
     - Determine if the ray hits the inside of the mesh:
       $$
       \text{is\_inside} = \text{si.is\_valid}() \land (\text{si.shape} == \text{target\_shape}) \land (\text{dot}(\text{ray.d}, \text{si.n}) > 0)
       $$
     - Update counters:
       - If `is_inside` is true, increment `count_in` at the corresponding voxel index.
       - If `is_inside` is false (and ray is valid/active), increment `count_out` [S1, unofficial].
     - Increment `d` and update `active`:
       $$
       \text{active} \&= (d < \text{sample\_count})
       $$

3. **Result Aggregation:**
   - After the loop, scatter results into `self.volumes`.
   - For each voxel index `idx`:
     - The total number of samples is `sample_count`.
     - The effective volume is calculated as:
       $$
       V_{idx} = \frac{\text{count\_in}[idx] \times V_{voxel}}{\text{sample\_count}}
       $$
       where $V_{voxel} = \text{prod}(\text{voxel\_size})$ [S1, unofficial].
     - Note: The source code mentions scattering into `2*idx` and `2*idx+1` for inside/outside, but the final volume calculation uses `count_in` [S1, unofficial]. The effective volume is proportional to the fraction of samples that hit the inside of the mesh.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_aware_discretization(target_scene, target_shape, res, voxel_size, sample_count=2**14):
    # Check if surface-aware is enabled
    if not surface_aware_enabled:
        return dr.full(dr.Float, dr.prod(voxel_size), shape=res)

    # If volumes already computed, return them
    if volumes_computed:
        return self.volumes

    # Identify target shape
    if target_shape not in scene.shapes:
        raise ValueError("Target shape not found in scene")

    # Create target scene with only the target shape
    target_scene = Scene(shapes=[target_shape])

    # Initialize volumes tensor
    self.volumes = dr.ones(dr.Float, res)

    # Generate voxel coordinates
    xx = dr.arange(mi.Float, res.x)
    yy = dr.arange(mi.Float, res.y)
    zz = dr.arange(mi.Float, res.z)
    z_idx, y_idx, x_idx = dr.meshgrid(zz, yy, xx, indexing='ij')
    voxel = mi.Point3f(x_idx, y_idx, z_idx)

    # Initialize counters
    count_in = dr.zeros(dr.Int, dr.prod(res))
    count_out = dr.zeros(dr.Int, dr.prod(res))
    d = 0
    active = dr.full(dr.Bool, True, shape=res)

    # Initialize sampler
    sampler = mi.Sampler()
    sampler.seed(0, dr.prod(res) * sample_count)

    # Bounding box of the voxel grid
    bbox = compute_bbox(res, voxel_size)

    # Monte Carlo loop
    while dr.any(active):
        # Generate random offset
        offset = mi.Point3f(sampler.next_1d(), sampler.next_1d(), sampler.next_1d())

        # Compute ray origin
        pos = bbox.min + voxel_size * (voxel + offset)

        # Generate random direction
        dir = mi.warp.square_to_uniform_sphere(sampler.next_2d())

        # Construct ray
        ray = mi.Ray3f(pos, dir)

        # Check if ray origin is inside mesh bounding box
        in_mesh_bbox = dr.all((ray.o > bbox.min) & (ray.o < bbox.max))

        # Perform ray intersection
        si = target_scene.ray_intersect(ray, active=active & in_mesh_bbox)

        # Determine if inside mesh
        is_inside = si.is_valid() & (si.shape == mi.ShapePtr(target_shape)) & (dr.dot(ray.d, si.n) > 0)

        # Update counters
        # Note: dr.scatter_add is used to update counts at voxel indices
        # The index is derived from the voxel coordinates
        idx = dr.ravel_index(voxel)
        count_in = dr.scatter_add(count_in, idx, dr.cast(dr.Int, is_inside))
        count_out = dr.scatter_add(count_out, idx, dr.cast(dr.Int, ~is_inside & active))

        # Increment d and update active mask
        d += 1
        active &= (d < sample_count)

    # Scatter results into self.volumes
    voxel_vol = dr.prod(voxel_size)
    for idx in range(dr.prod(res)):
        self.volumes[idx] = count_in[idx] * voxel_vol / sample_count

    return self.volumes
```

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of voxels and $S$ is the sample count per voxel. Each voxel requires $S$ ray intersections.
- **Memory Complexity:** $O(N)$ for storing the voxel grid and counters.
- **Performance:** The method is computationally expensive due to the Monte Carlo ray tracing per voxel. In the Dr. TVAM implementation, this step is part of the inverse rendering pipeline and can take significant time (e.g., 25 minutes for certain configurations) [S1, unofficial].
- **Strengths:** Accurately models partial voxel occupancy, reducing resolution artifacts in 3D prints [S1, unofficial].
- **Limitations:** High computational cost. Requires a differentiable renderer (Mitsuba) for integration into the inverse rendering framework [S1, unofficial].
- **When to use:** When high-fidelity voxelization is required for volumetric additive manufacturing, and computational resources are available.
- **When not to use:** When real-time performance is critical, or when the mesh is simple enough for analytic or SDF-based methods.

## Background
Surface-Aware Discretization was introduced in the context of Tomographic Volumetric Additive Manufacturing (TVAM) by Nicolet et al. in 2024 [S1, unofficial]. It is part of the Dr. TVAM framework, which uses physically-based differentiable rendering to optimize patterns for TVAM [S1, unofficial]. The method addresses limitations of prior discretization schemes by exploiting ray tracing to mitigate resolution-related artifacts [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
