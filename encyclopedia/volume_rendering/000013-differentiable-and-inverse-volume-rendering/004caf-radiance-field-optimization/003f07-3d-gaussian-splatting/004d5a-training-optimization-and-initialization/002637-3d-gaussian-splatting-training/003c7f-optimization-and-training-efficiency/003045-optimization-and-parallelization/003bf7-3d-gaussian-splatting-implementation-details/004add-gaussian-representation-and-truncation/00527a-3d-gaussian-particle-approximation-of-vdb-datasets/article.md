# 3D Gaussian Particle Approximation of VDB Datasets

Also called: Gaussian Particle Approximation, Gaussian Particle Rendering, VDB to Gaussian Conversion

## Overview
This method converts OpenVDB volumetric datasets into a set of non-overlapping 3D Gaussian particles to enable memory-efficient, lossy representation with Level of Detail (LOD) support [S1, unofficial]. It solves the problem of rendering large sparse volumetric data in real-time by mapping voxel regions to Gaussian primitives that are compatible with hardware-accelerated ray tracing pipelines [S1, unofficial]. Unlike standard Gaussian Splatting which often uses overlapping primitives for photorealism, this approach strictly enforces non-overlapping Gaussians derived from OpenVDB's non-overlapping leaf nodes, limiting the model to specific refinement levels but ensuring deterministic spatial partitioning [S1, unofficial]. It differs from [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md) by treating Gaussians as volumetric particles with explicit density evaluation rather than surface splats, and from [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md) by using a tree-based generation process rather than direct voxel-to-gaussian mapping without LOD clustering.

## How it works

### Inputs and Outputs
**Inputs:**
*   An OpenVDB grid containing volumetric or level-set data [S1, unofficial].
*   A chosen Level of Detail (LOD) mode: Low, Medium, or High [S1, unofficial].
*   Camera parameters (origin, direction) for ray generation [S1, unofficial].

**Outputs:**
*   A set of 3D Gaussian particles, each defined by position, opacity, and covariance [S1, unofficial].
*   Rendered color images via forward ray marching [S1, unofficial].

### Data Structures

**OpenVDB Grid Layout (5-4-3 Variant):**
The grid uses a fixed-depth tree structure [S1, unofficial]:
*   **Root Node:** Spans the dataset.
*   **5-Level Nodes:** Top-level nodes with resolution $2^5 \times 2^5 \times 2^5 = 32,768$ children [S1, unofficial].
*   **4-Level Nodes:** Intermediate nodes with resolution $2^4 \times 2^4 \times 2^4 = 4,096$ children [S1, unofficial].
*   **3-Level Nodes (Leaf Nodes):** Contain $2^3 \times 2^3 \times 2^3 = 512$ voxels each [S1, unofficial].
*   **Child Mask:** 1 bit per child at each level to mark if the sub-tree is active. Stored in 64-bit integers (e.g., 4,096 bits for a 4-level node) [S1, unofficial].
*   **Value Mask:** 1 bit per child to mark if the sub-tree is a "Tile" (homogeneous region storing a single value) [S1, unofficial].

**GaussianModel (Device-Side):**
A parent class containing contiguous device-side arrays allocated via `cudaMalloc` [S1, unofficial]:
*   **Opacity:** Scalar array (float) [S1, unofficial].
*   **World Position:** 3D vector array (float3) [S1, unofficial].
*   **Covariance:** Array of 3 floats (float3) for ellipsoids or 1 float (float) for spheres [S1, unofficial].
*   **HitGroupRecord:** Structure containing a pointer to `GaussianModel` and shader bindings [S1, unofficial].

**Acceleration Structures (OptiX):**
*   **Geometry Acceleration Structure (GAS):** Built on the GPU using a CUDA device-side array of Axis-Aligned Bounding Boxes (AABBs). Each AABB encloses a single Gaussian particle, defined by min/max coordinates in OpenVDB Index Space [S1, unofficial].
    *   Build Flags: `OPTIX_BUILD_FLAG_ALLOW_COMPACTION`, `OPTIX_BUILD_INPUT_TYPE_CUSTOM_PRIMITIVES` [S1, unofficial].
*   **Instance Acceleration Structure (IAS):** Stores the GAS instance with the OpenVDB Grid transform. Visibility masks are set for all Gaussians to be visible for primary and continuation rays [S1, unofficial].

### Numbered Steps

1.  **Data Generation (CPU):**
    *   Retrieve all leaf nodes from the OpenVDB Grid Tree using the OpenVDB API [S1, unofficial].
    *   For each leaf node (or cluster based on LOD), derive Gaussian properties:
        *   **Position:** Center in world space, derived from the centroid of the voxel region or bounding box [S1, unofficial].
        *   **Opacity:** Average scalar value of the voxel region to preserve local material characteristics [S1, unofficial].
        *   **Covariance:** Diagonal vector of the 3x3 covariance matrix (axis-aligned) or a single scalar for isotropic spheres [S1, unofficial].
    *   *LOD Specifics:*
        *   **Low LOD:** Each leaf node becomes a single Gaussian reflecting the distribution of active voxels [S1, unofficial].
        *   **Medium LOD:** Dense leaf nodes merge into $4\times4\times4$ groups; sparse leaf nodes merge into $2\times2\times2$ groups. Remaining voxels become individual Gaussians [S1, unofficial].
        *   **High LOD:** Sparse leaf nodes contribute one Gaussian per active voxel; dense leaf nodes merge into $2\times2\times2$ blocks [S1, unofficial].
    *   *Note:* Tiled leaf nodes (marked by Value Mask) are excluded from Gaussian creation as their value is stored in the parent node [S1, unofficial].

2.  **Data Transfer (CPU to GPU):**
    *   Copy Gaussian properties (opacity, position, covariance) to contiguous device-side buffers using `cudaMalloc` and `cudaMemcpy` [S1, unofficial].
    *   Store device pointers in a `GaussianModel` object added to the OptiX Shader Binding Table (SBT) [S1, unofficial].

3.  **Acceleration Structure Build (GPU):**
    *   Build the GAS using the array of AABBs derived from Gaussian extents in Index Space [S1, unofficial].
    *   Build the IAS with the OpenVDB Grid transform [S1, unofficial].

4.  **Ray Tracing Pipeline (GPU/OptiX):**
    *   **Ray Generation:** Shoot primary rays from the camera using the `__raygen__` program [S1, unofficial].
    *   **Intersection (`__intersection__`):**
        *   Use the primitive index to retrieve the Gaussian AABB [S1, unofficial].
        *   Perform ray-box intersection with the world AABB [S1, unofficial].
        *   If intersected, retrieve Gaussian center $\mu$ and inverse covariance $\Sigma^{-1}$ [S1, unofficial].
        *   Transform the ray into the Gaussian's local space using $\Sigma^{-1}$ [S1, unofficial].
        *   Solve the quadratic equation for entry ($t_0$) and exit ($t_1$) distances [S1, unofficial].
        *   Store $t_0$ and $t_1$ in the payload and exit early [S1, unofficial].
    *   **Ray Marching and Density Evaluation:**
        *   Step along the ray using a predefined step size based on $t_1 - t_0$ [S1, unofficial].
        *   For each sample point $x_k$, compute difference $\delta = x_k - \mu$ [S1, unofficial].
        *   Compute squared Mahalanobis distance: $D^2 = \delta^\top \Sigma^{-1} \delta$ [S1, unofficial].
        *   Evaluate density: $\rho_k = \exp(-D^2/2)$ [S1, unofficial].
    *   **Absorption and Transmittance Update:**
        *   Compute volume factor: $V_f = 1/\det(\Sigma)$ [S1, unofficial].
        *   Calculate absorption: $A_k = \rho_k \cdot \Delta t \cdot V_f$, where $\Delta t$ is the step size [S1, unofficial].
        *   Update color: $C \leftarrow C + T_k \cdot c(\sigma_k) \cdot A_k$, where $c(\sigma_k)$ is scattering color derived from opacity [S1, unofficial].
        *   Update transmittance: $T_{k+1} = T_k \cdot \exp(-A_k)$ [S1, unofficial].
    *   **Ray Continuation:**
        *   Advance ray origin: $o_{new} = o + d \cdot (t_1 - t_0)$ [S1, unofficial].
        *   Recursively trace the next Gaussian using `optixTrace` with `RAY_TYPE_RADIANCE` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Gaussian Model Data Structures
class GaussianModel:
    opacity: DeviceArray[float]
    position: DeviceArray[float3]
    covariance: DeviceArray[float3] # or float for spheres

# Recursive Ray Tracing Loop
def recursive_ray_trace(ray, t_min, t_max, max_depth):
    color = 0.0
    transmittance = 1.0
    recursion_depth = 0
    
    while recursion_depth < max_depth:
        t0, t1 = t_min, t_max
        
        # Intersect with World AABB
        if not intersect_world_aabb(ray, world_bbox, t0, t1):
            return color + transmittance * background_color
        
        # Iterate through Gaussians in the AABB
        gaussians_in_aabb = get_gaussians_in_aabb(t0, t1)
        
        for gaussian in gaussians_in_aabb:
            if ray_intersects_gaussian(ray, gaussian, t0, t1):
                # Retrieve properties
                mu = gaussian.position
                inv_cov = inverse(gaussian.covariance)
                alpha = gaussian.opacity
                
                # Density Evaluation
                # Sample points along the ray segment [t0, t1]
                step_size = (t1 - t0) / num_samples
                for i in range(num_samples):
                    t_sample = t0 + i * step_size
                    x_k = ray.origin + ray.direction * t_sample
                    delta = x_k - mu
                    
                    # Mahalanobis Distance
                    D2 = dot(delta, multiply_matrix_vector(inv_cov, delta))
                    density = exp(-D2 / 2.0)
                    
                    # Absorption
                    volume_factor = 1.0 / determinant(gaussian.covariance)
                    absorption = density * step_size * volume_factor
                    
                    # Radiance Accumulation
                    scattering_color = get_scattering_color(alpha) # Via transfer function
                    color += transmittance * scattering_color * absorption
                    transmittance *= exp(-absorption)
                
                # Ray Continuation
                ray.origin = ray.origin + ray.direction * (t1 - t0)
                
                # Recursive Trace for next Gaussian
                # Note: OptiX register constraints limit deep recursion [S1, unofficial]
                payload_result = optix_trace(RAY_TYPE_RADIANCE, ray)
                color += transmittance * payload_result
                
                break # Move to next AABB or exit if no more intersections
        
        recursion_depth += 1
        if recursion_depth > 10:
            break
            
    return color
```

### Parameters
*   **Grid Layout Variant:** '5-4-3' is used for all explanations and rendering samples [S1, unofficial].
*   **Build Flags:** `OPTIX_BUILD_FLAG_ALLOW_COMPACTION`, `OPTIX_BUILD_INPUT_TYPE_CUSTOM_PRIMITIVES` [S1, unofficial].
*   **Ray Type:** `RAY_TYPE_RADIANCE` for all `optixTrace` calls, indicating a purely forward-ray-marching approach [S1, unofficial].
*   **Max Recursion Depth:** Typically limited by OptiX register constraints; implementation breaks if depth > 10 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Non-overlapping Constraint:** The implementation uses non-overlapping Gaussians because OpenVDB leaf nodes are strictly non-overlapping cubic regions. This limits the model to three overarching refinement levels and introduces lossiness [S1, unofficial].
*   **NanoVDB Limitation:** NanoVDB does not support device-side traversal of the Grid Tree or device-side iterators/accessors. This creates a bottleneck for random access required in ray-BVH or ray-particle intersections. Therefore, NanoVDB cannot be used for 3D Gaussian generation or rendering in this pipeline; OpenVDB must be used for generation [S1, unofficial].
*   **NanoVDB HDDA Incompatibility:** NanoVDB's built-in HDDA transmittance algorithm only works with regular float grid volumes. It is invalid for overlapping and variable-size particle data with custom ray tracing pipelines [S1, unofficial].
*   **OptiX Register Constraint:** Propagating state via recursive `optixTrace` calls is limited by OptiX’s 8-payload register constraint, causing sub-optimal accumulation and inconsistent resetting of payload values [S1, unofficial].
*   **Transfer Functions:** Unlike other Gaussian splatting implementations that bake color into Gaussians, this renderer does not, supporting transfer functions similar to traditional volume rendering via colormap lookups [S1, unofficial].

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** Depends on LOD. High LOD uses significantly more memory than Low LOD. For the Dragon dataset, High LOD uses 546MB (23.3M Gaussians), while Low LOD uses 174.4MB (7.4M Gaussians) [S1, unofficial].
*   **Time:** Rendering performance is consistently above 150 FPS for all tested datasets (bunny_cloud, explosion, fire, utahteapot, venusstatue, dragon) across all LODs, even with millions of Gaussians [S1, unofficial].

**Scaling:**
*   Performance scales with the number of Gaussians, but hardware-accelerated AABB intersections in OptiX mitigate the cost of large particle counts [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering of large sparse volumes; compatibility with standard Gaussian mixture rendering pipelines (conceptually); support for transfer functions; unified representation of volumes and level-sets [S1, unofficial].
*   **Limitations:** Lossy representation due to non-overlapping constraint; limited to three LOD levels; NanoVDB incompatibility; OptiX register constraints on recursion depth [S1, unofficial].

**When to use it:**
*   Scientific visualization (SciVis) applications requiring real-time rendering of large sparse volumetric data [S1, unofficial].
*   When hardware-accelerated ray tracing (OptiX) is available [S1, unofficial].

**When not to use it:**
*   Photorealistic rendering requiring overlapping Gaussians and complex scattering [S1, unofficial].
*   Environments where NanoVDB is the preferred format for GPU memory efficiency, as it is incompatible with this pipeline [S1, unofficial].

## Background

**Origin:**
The method was introduced in a study for Scientific Visualization exploring the integration of OpenVDB and NVIDIA OptiX [S1, unofficial]. It builds on the popularity of 3D Gaussian approximation and mixture models in SciVis, referencing earlier work by Jang et al. [11] and the success of Gaussian Splatting by Kerbl et al. [12] [S1, unofficial].

**History and Development:**
The approach was developed to leverage OpenVDB's built-in memory compaction and hardware acceleration features while providing Level of Detail support [S1, unofficial]. The authors attempted to use NanoVDB for GPU-side generation but found it insufficient due to lack of device-side iterators, leading to the decision to use OpenVDB for CPU-side generation and transfer to GPU buffers [S1, unofficial].

**Usage:**
Used in NVIDIA OptiX library for forward ray marching through Gaussian particles [S1, unofficial]. Applied to datasets like bunny_cloud, explosion, fire, utahteapot, venusstatue, and dragon [S1, unofficial].

## Sources

[S1] 3D Gaussian Particle Approximation of VDB Datasets: A Study for Scientific Visualization, https://www.emergentmind.com/papers/2504.04857, fetched 2026-10-29 (unofficial)
