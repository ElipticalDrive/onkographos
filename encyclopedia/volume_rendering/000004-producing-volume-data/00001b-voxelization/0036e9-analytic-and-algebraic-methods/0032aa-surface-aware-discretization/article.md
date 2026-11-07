# Surface-Aware Discretization

Also called: Mesh-intersection voxelization, Surface-aware voxelization

## Overview
Surface-Aware Discretization is a voxelization method that computes the effective volume of each voxel based on the intersection of the voxel with a target mesh, rather than assuming the voxel is fully filled or empty [S1, unofficial]. This approach mitigates resolution-related artifacts in volumetric additive manufacturing (TVAM) prints by providing a more accurate representation of the object's geometry within the discrete grid [S1, unofficial]. It differs from standard binary voxelization by estimating the fractional occupancy of each voxel through Monte Carlo ray tracing, distinguishing it from analytic methods that rely on closed-form geometric intersections [S1, unofficial].

## How it works
The method uses a differentiable rendering framework (specifically Dr. TVAM, built on Mitsuba 3) to estimate the volume of intersection between a voxel and a target mesh [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A target mesh (shape), a voxel grid defined by resolution ($res$) and voxel size ($voxel\_size$), and a bounding box ($bbox$) for the scene.
*   **Output:** A tensor of effective volumes for each voxel, stored in `self.volumes`.

**Data Structures:**
*   `self.volumes`: A tensor of shape `self.m_film.data.shape` (equivalent to $res.x \times res.y \times res.z$), initialized to ones.
*   `count_in`, `count_out`: Integer tensors of size `dr.prod(res)` tracking the number of rays hitting the inside and outside of the mesh, respectively.
*   `voxel`: A tensor of `mi.Point3f` coordinates representing the grid indices $(x, y, z)$.

**Parameters:**
*   `sample_count`: Number of Monte Carlo samples per voxel. Default value is $2^{14}$ (16384) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Check if the film is surface-aware. If not, return the full voxel volume: `return dr.prod(self.voxel_size)`.
    *   If volumes are already computed, return them.
    *   Identify the shape with ID 'target' in the scene. If not found, raise `ValueError`.
    *   Create a `target_scene` containing only the target shape.
    *   Initialize `self.volumes` as a tensor of ones with shape `self.m_film.data.shape`.
    *   Generate voxel coordinates using `meshgrid`:
        $$ xx = \text{arange}(res.x) $$
        $$ yy = \text{arange}(res.y) $$
        $$ zz = \text{arange}(res.z) $$
        $$ (z\_idx, y\_idx, x\_idx) = \text{meshgrid}(zz, yy, xx, \text{indexing}='ij') $$
        $$ voxel = \text{Point3f}(x\_idx, y\_idx, z\_idx) $$
    *   Initialize counters `count_in` and `count_out` to zeros with size `dr.prod(res)`.
    *   Initialize sample counter $d = 0$.
    *   Create an independent sampler and seed it with 0 and total sample count `dr.prod(res)`.

2.  **Monte Carlo Estimation Loop:**
    *   While `active` is true:
        *   Generate a random offset within the voxel:
            $$ offset = \text{Point3f}(sampler.next\_1d(), sampler.next\_1d(), sampler.next\_1d()) $$
        *   Compute the ray origin:
            $$ pos = bbox.min + voxel\_size \times (voxel + offset) $$
        *   Generate a random direction uniformly distributed over the sphere:
            $$ ray = \text{Ray3f}(pos, \text{warp.square\_to\_uniform\_sphere}(sampler.next\_2d())) $$
        *   Check if the ray origin is inside the mesh bounding box:
            $$ in\_mesh\_bbox = \text{all}((ray.o > bbox.min) \ \& \ (ray.o < bbox.max)) $$
        *   Perform ray intersection against the `target_scene`:
            $$ si = target\_scene.ray\_intersect(ray, \text{active} = \text{active} \ \& \ in\_mesh\_bbox) $$
        *   Determine if the ray hits the inside of the target mesh:
            $$ is\_inside = si.is\_valid() \ \& \ (si.shape == \text{ShapePtr}(target\_shape)) \ \& \ (\text{dot}(ray.d, si.n) > 0) $$
            *Note: `si.n` is the surface normal at the intersection point. The condition `dot(ray.d, si.n) > 0` checks if the ray is entering the volume (assuming outward-facing normals).*
        *   Increment `count_in` if `is_inside` is true, otherwise increment `count_out`.
        *   Increment $d$ and update `active` mask:
            $$ active \ \&= (d < sample\_count) $$

3.  **Result Aggregation:**
    *   After the loop, scatter results into `self.volumes`.
    *   The index for inside hits is $2 \times idx$, and for outside hits is $2 \times idx + 1$.
    *   The value is computed as:
        $$ volume = \frac{count \times voxel\_vol}{sample\_count} $$
        where $voxel\_vol = \text{dr.prod}(voxel\_size)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import drjit as dr
import mitsuba as mi

def surface_aware_discretization(target_shape, voxel_size, res, bbox, sample_count=2**14):
    # 1. Initialization
    if not is_surface_aware():
        return dr.prod(voxel_size)
    
    if volumes_computed():
        return self.volumes
    
    target_scene = Scene([target_shape])
    self.volumes = dr.ones(mi.Float, dr.prod(res))
    
    # Generate voxel grid coordinates
    xx = dr.arange(mi.Float, res.x)
    yy = dr.arange(mi.Float, res.y)
    zz = dr.arange(mi.Float, res.z)
    z_idx, y_idx, x_idx = dr.meshgrid(zz, yy, xx, indexing='ij')
    voxel = mi.Point3f(x_idx, y_idx, z_idx)
    
    count_in = dr.zeros(mi.Int, dr.prod(res))
    count_out = dr.zeros(mi.Int, dr.prod(res))
    d = 0
    active = dr.full(mi.Bool, True, dr.prod(res))
    
    sampler = mi.load_object('independent', {'seed': 0, 'sample_count': dr.prod(res) * sample_count})
    
    # 2. Monte Carlo Loop
    while dr.any(active):
        offset = mi.Point3f(sampler.next_1d(), sampler.next_1d(), sampler.next_1d())
        pos = bbox.min + voxel_size * (voxel + offset)
        ray_dir = mi.warp.square_to_uniform_sphere(sampler.next_2d())
        ray = mi.Ray3f(pos, ray_dir)
        
        in_mesh_bbox = dr.all((ray.o > bbox.min) & (ray.o < bbox.max))
        si = target_scene.ray_intersect(ray, active=active & in_mesh_bbox)
        
        is_inside = si.is_valid() & (si.shape == mi.ShapePtr(target_shape)) & (dr.dot(ray.d, si.n) > 0)
        
        count_in = dr.select(is_inside, count_in + 1, count_in)
        count_out = dr.select(~is_inside, count_out + 1, count_out)
        
        d += 1
        active = active & (d < sample_count)
        
    # 3. Aggregation
    voxel_vol = dr.prod(voxel_size)
    # Note: The source describes scattering into indices 2*idx and 2*idx+1, 
    # but the final volume is the sum of inside hits scaled by volume/sample_count.
    # The source implies count_in holds the relevant data for volume estimation.
    self.volumes = count_in * voxel_vol / sample_count
    
    return self.volumes
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on floating-point ray tracing. Degenerate meshes or extremely thin features may lead to inaccurate intersection counts [S1, unofficial].
*   **Boundaries:** The check `in_mesh_bbox` optimizes the ray intersection but assumes the bounding box tightly encloses the target shape. If the bounding box is loose, rays may miss the shape even if the voxel intersects it, leading to underestimation [S1, unofficial].
*   **Normal Orientation:** The condition `dot(ray.d, si.n) > 0` assumes consistent outward-facing normals. Inconsistent normals will cause incorrect inside/outside classification [S1, unofficial].

## Subtopics

### Signed Distance Field Voxelization
Computes the distance from each voxel to the nearest surface. Chosen when smooth gradients and distance queries are needed, rather than binary or volumetric occupancy [S1, S3].

### Analytic Voxelization
Uses closed-form geometric formulas to determine voxel occupancy. Chosen for speed and determinism when Monte Carlo noise is unacceptable [S1, S3].

## Operating characteristics
*   **Time Complexity:** $O(N \times S)$, where $N$ is the number of voxels and $S$ is the sample count per voxel. With $S=2^{14}$, this is computationally expensive compared to analytic methods [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the volume tensor and counters [S1, unofficial].
*   **Measured Performance:** In the Dr. TVAM framework, surface-aware loss computation takes approximately 3 minutes 45 seconds for a specific test case, compared to 25 minutes for non-surface-aware methods, indicating significant optimization in the rendering pipeline [S1, unofficial].
*   **Strengths:** Accurately models partial voxel occupancy, reducing stair-stepping artifacts in TVAM prints [S1, unofficial].
*   **Limitations:** Requires a differentiable renderer (Mitsuba 3), making it less portable than pure C++ voxelization algorithms. Noise from Monte Carlo sampling may require high sample counts for convergence [S1, unofficial].
*   **When to use:** When high-fidelity geometry representation is critical for volumetric additive manufacturing, and computational resources allow for Monte Carlo estimation [S1, unofficial].
*   **When not to use:** For real-time applications or when deterministic, fast voxelization is required [S1, unofficial].

## Background
Surface-Aware Discretization was introduced in the context of Tomographic Volumetric Additive Manufacturing (TVAM) by Nicolet et al. in 2024 [S1, unofficial]. It is part of the Dr. TVAM framework, which uses inverse rendering to optimize light patterns for 3D printing [S1, unofficial]. The method addresses limitations of prior discretization schemes that failed to account for partial voxel occupancy, leading to artifacts in printed objects [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
