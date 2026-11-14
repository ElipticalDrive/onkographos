# 3D Gaussian Particle Approximation of VDB Datasets

Also called: Gaussian Particle Approximation, Gaussian Particle Rendering, VDB to Gaussian Conversion

## Overview
This method converts OpenVDB volumetric datasets into a set of non-overlapping 3D Gaussian particles to enable memory-efficient, lossy representation with Level of Detail (LOD) support [S1, unofficial]. It solves the problem of rendering large sparse volumetric data in real-time by mapping voxel regions to Gaussian primitives that are accelerated via NVIDIA OptiX ray tracing [S1, unofficial]. Unlike standard Gaussian Splatting which often uses overlapping primitives and baked colors, this approach strictly enforces non-overlapping Gaussians derived from OpenVDB leaf nodes and supports dynamic transfer functions for scientific visualization [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   An OpenVDB grid containing scalar values (density) or signed distance fields (level sets) [S1, unofficial].
*   Camera parameters (origin, direction, projection) [S1, unofficial].
*   LOD level selection (Low, Medium, High) [S1, unofficial].

**Outputs:**
*   A rendered image with accumulated color and opacity [S1, unofficial].
*   Device-side buffers containing Gaussian properties (position, covariance, opacity) [S1, unofficial].

### Data Structures

**Gaussian Model (Device-Side)**
A contiguous structure holding arrays for all Gaussians. Each Gaussian $i$ stores:
*   `position`: A 3D vector $(\mu_x, \mu_y, \mu_z)$ in world space [S1, unofficial].
*   `covariance`:
    *   For spheres: A single scalar $s$ (isotropic) [S1, unofficial].
    *   For ellipsoids: A 3D vector representing the diagonal of the covariance matrix $\Sigma$ (axis-aligned) [S1, unofficial].
*   `opacity`: A scalar value representing the average density of the voxel region [S1, unofficial]. For level sets, this may be uniform [S1, unofficial].

**Acceleration Structures (OptiX)**
*   **Geometry Acceleration Structure (GAS):** Built on the GPU using a CUDA device-side array of Axis-Aligned Bounding Boxes (AABBs). Each AABB encloses a single Gaussian particle, defined by its minimum and maximum coordinates in OpenVDB Index Space [S1, unofficial].
    *   Build Flags: `OPTIX_BUILD_FLAG_ALLOW_COMPACTION`, `OPTIX_BUILD_INPUT_TYPE_CUSTOM_PRIMITIVES` [S1, unofficial].
*   **Instance Acceleration Structure (IAS):** Stores the GAS instance with the OpenVDB Grid transform. Visibility masks are set to visible for primary and continuation rays [S1, unofficial].

**OpenVDB Grid Layout (5-4-3 Variant)**
*   **Root:** Spans the dataset.
*   **5-level (Top):** $2^5 \times 2^5 \times 2^5 = 32,768$ children [S1, unofficial].
*   **4-level (Intermediate):** $2^4 \times 2^4 \times 2^4 = 4,096$ children [S1, unofficial].
*   **3-level (Leaf):** $2^3 \times 2^3 \times 2^3 = 512$ voxels [S1, unofficial].
*   **Masks:**
    *   *Child Mask:* 1 bit per child to mark active sub-trees [S1, unofficial].
    *   *Value Mask:* 1 bit per child to mark "Tiles" (homogeneous regions storing a single value) [S1, unofficial].

### Numbered Steps

1.  **Data Generation (CPU):**
    *   Retrieve all leaf nodes from the OpenVDB Grid Tree using the OpenVDB API [S1, unofficial].
    *   Apply LOD rules to determine Gaussian properties for each leaf or cluster:
        *   **Low LOD:** Each leaf node becomes a single Gaussian reflecting the distribution of active voxels [S1, unofficial].
        *   **Medium LOD:** Dense leaf nodes merge into $4 \times 4 \times 4$ groups; sparse nodes merge into $2 \times 2 \times 2$ groups. Remaining voxels become individual Gaussians [S1, unofficial].
        *   **High LOD:** Sparse leaf nodes contribute one Gaussian per active voxel; dense leaf nodes merge into $2 \times 2 \times 2$ blocks [S1, unofficial].
    *   For each Gaussian, calculate:
        *   **Position ($\mu$):** Centroid of the voxel region or bounding box [S1, unofficial].
        *   **Opacity:** Average scalar value of the voxel region [S1, unofficial].
        *   **Covariance:** Diagonal vector of the $3 \times 3$ covariance matrix (axis-aligned) or scalar for spheres [S1, unofficial].
    *   Note: Tiled leaf nodes (marked by Value Mask) are excluded from Gaussian creation as their value is stored in the parent node [S1, unofficial].

2.  **Data Transfer (CPU to GPU):**
    *   Allocate contiguous device-side buffers using `cudaMalloc` for opacity, position, and covariance [S1, unofficial].
    *   Copy data using `cudaMemcpy` [S1, unofficial].
    *   Store device pointers in a `GaussianModel` object [S1, unofficial].

3.  **Acceleration Structure Build (GPU):**
    *   Build the GAS using the array of AABBs derived from Gaussian extents in Index Space [S1, unofficial].
    *   Build the IAS with the OpenVDB Grid transform [S1, unofficial].
    *   Add `GaussianModel` to the OptiX Shader Binding Table (SBT) [S1, unofficial].

4.  **Ray Generation:**
    *   Shoot primary rays from the camera using the `__raygen__` program [S1, unofficial].

5.  **Intersection Test (`__intersection__`):**
    *   Use the primitive index to retrieve the Gaussian AABB [S1, unofficial].
    *   Perform ray-box intersection with the world AABB [S1, unofficial].
    *   If intersected:
        *   Retrieve Gaussian center $\mu$ and inverse covariance $\Sigma^{-1}$ [S1, unofficial].
        *   Transform the ray into the Gaussian's local space using $\Sigma^{-1}$ [S1, unofficial].
        *   Solve the quadratic equation for the ellipsoid/sphere intersection to find entry ($t_0$) and exit ($t_1$) distances [S1, unofficial].
        *   Store $t_0$ and $t_1$ in the payload and exit the intersection shader early [S1, unofficial].

6.  **Ray Marching and Density Evaluation:**
    *   Step along the ray from $t_0$ to $t_1$ using a predefined step size [S1, unofficial].
    *   For each sample point $x_k$:
        *   Compute difference $\delta = x_k - \mu$ [S1, unofficial].
        *   Compute squared Mahalanobis distance $D^2 = \delta^\top \Sigma^{-1} \delta$ [S1, unofficial].
        *   Evaluate density $\rho_k = \exp(-D^2 / 2)$ [S1, unofficial].

7.  **Absorption and Transmittance Update:**
    *   Compute volume factor: $V_f = 1 / \det(\Sigma)$ [S1, unofficial].
    *   Compute absorption: $A_k = \rho_k \cdot \Delta t \cdot V_f$, where $\Delta t$ is the step size (distance between sample points) [S1, unofficial].
    *   Update color: $C_{new} = C_{old} + T \cdot c(\sigma_k) \cdot A_k$, where $c(\sigma_k)$ is the scattering color derived from opacity via a transfer function [S1, unofficial].
    *   Update transmittance: $T_{new} = T_{old} \cdot \exp(-A_k)$ [S1, unofficial].

8.  **Ray Continuation:**
    *   Advance ray origin: $o_{new} = o + d \cdot (t_1 - t_0)$ [S1, unofficial].
    *   Recursively trace the next Gaussian using `optixTrace` with `RAY_TYPE_RADIANCE` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Gaussian Model Data Structures
class Gaussian:
    position: Vector3  # World space center
    covariance_diag: Vector3  # Diagonal of Sigma (axis-aligned)
    opacity: float  # Scalar density value

class GaussianModel:
    positions: Array[Vector3]
    covariances: Array[Vector3]
    opacities: Array[float]

# OptiX Intersection Program
def intersection_program(primitive_index, ray_origin, ray_direction):
    # Retrieve Gaussian properties
    gauss = GaussianModel.primitive_at(primitive_index)
    
    # Retrieve AABB for this Gaussian
    aabb = GaussianModel.aabb_at(primitive_index)
    
    # Ray-AABB intersection
    if not ray_box_intersect(ray_origin, ray_direction, aabb):
        return  # No intersection
    
    # Transform ray to Gaussian local space
    # Note: Sigma is diagonal, so inverse is element-wise reciprocal
    inv_sigma = Vector3(1.0/gauss.covariance_diag.x, 
                        1.0/gauss.covariance_diag.y, 
                        1.0/gauss.covariance_diag.z)
    
    # Transform ray origin and direction
    local_origin = ray_origin - gauss.position
    local_direction = ray_direction  # Assuming no rotation for axis-aligned
    
    # Solve quadratic equation for ellipsoid intersection
    # Equation: (d^T S^-1 d) t^2 + 2(d^T S^-1 o) t + (o^T S^-1 o) = 0
    # Where S^-1 is the diagonal matrix of inv_sigma
    a = dot(local_direction, local_direction * inv_sigma)
    b = 2.0 * dot(local_direction * inv_sigma, local_origin)
    c = dot(local_origin * inv_sigma, local_origin) - 1.0  # Unit sphere/ellipsoid boundary
    
    discriminant = b*b - 4*a*c
    if discriminant < 0:
        return  # No intersection with ellipsoid
    
    sqrt_disc = sqrt(discriminant)
    t0 = (-b - sqrt_disc) / (2.0 * a)
    t1 = (-b + sqrt_disc) / (2.0 * a)
    
    # Store in payload
    payload.t0 = t0
    payload.t1 = t1
    payload.gauss_index = primitive_index
```

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Recursive Ray Marching Loop (Conceptual)
def recursive_ray_trace(ray, t_min, t_max, max_depth, current_depth):
    if current_depth >= max_depth:
        return background_color
    
    # Initialize accumulators
    color = Vector3(0, 0, 0)
    transmittance = 1.0
    
    # Intersect with world AABB
    t0, t1 = t_min, t_max
    if not intersect_world_aabb(ray, t0, t1):
        return background_color
    
    # Iterate through Gaussians in the AABB
    # Note: In OptiX, this is handled by the traversal engine and hit programs
    # The following represents the logic within the hit/continuation phase
    
    gauss = get_gaussian_from_payload()
    
    # Compute density at sample points along the segment [t0, t1]
    # Simplified: Single sample or stepped march
    step_size = (t1 - t0) / num_samples
    for i in range(num_samples):
        t = t0 + i * step_size
        x_k = ray.origin + ray.direction * t
        
        # Density Evaluation
        delta = x_k - gauss.position
        d_sq = dot(delta * gauss.inv_covariance, delta)
        density = exp(-0.5 * d_sq)
        
        # Absorption
        volume_factor = 1.0 / determinant(gauss.covariance_matrix)
        absorption = density * step_size * volume_factor
        
        # Radiance Accumulation
        scattering_color = transfer_function(gauss.opacity)
        color += transmittance * scattering_color * absorption
        
        # Transmittance Update
        transmittance *= exp(-absorption)
        
        if transmittance < epsilon:
            break
            
    # Advance ray and recurse
    new_origin = ray.origin + ray.direction * t1
    new_ray = Ray(new_origin, ray.direction)
    
    # Recursive call
    next_color = recursive_ray_trace(new_ray, t1, t_max, max_depth, current_depth + 1)
    
    return color + transmittance * next_color
```

### Parameters
*   **Grid Layout:** 5-4-3 variant is used for all explanations and rendering samples [S1, unofficial].
*   **Build Flags:** `OPTIX_BUILD_FLAG_ALLOW_COMPACTION`, `OPTIX_BUILD_INPUT_TYPE_CUSTOM_PRIMITIVES` [S1, unofficial].
*   **Ray Type:** `RAY_TYPE_RADIANCE` for all `optixTrace` calls [S1, unofficial].
*   **Max Recursion Depth:** Typically limited by OptiX register constraints; practical limit observed around 10 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Non-overlapping Constraint:** The implementation uses non-overlapping Gaussians because OpenVDB leaf nodes are strictly non-overlapping cubic regions. This limits the model to three overarching refinement levels and introduces lossiness [S1, unofficial].
*   **NanoVDB Limitation:** NanoVDB does not support device-side traversal of the Grid Tree or device-side iterators/accessors. This creates a bottleneck for random access required in ray-BVH or ray-particle intersections. Therefore, NanoVDB cannot be used for 3D Gaussian generation or rendering in this pipeline; OpenVDB must be used for generation [S1, unofficial].
*   **NanoVDB HDDA Incompatibility:** NanoVDB's built-in HDDA transmittance algorithm only works with regular float grid volumes. It is invalid for overlapping and variable-size particle data with custom ray tracing pipelines [S1, unofficial].
*   **OptiX Register Constraint:** Propagating state via recursive `optixTrace` calls is limited by OptiX’s 8-payload register constraint, causing sub-optimal accumulation and inconsistent resetting of payload values [S1, unofficial].
*   **Transfer Functions:** Unlike other Gaussian splatting implementations that bake color into Gaussians, this renderer does not, supporting transfer functions similar to traditional volume rendering via colormap lookups [S1, unofficial].

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** Scales with the number of Gaussians. For the 'dragon' dataset, High LOD uses 546MB and 23.3M Gaussians, while Low LOD uses 174.4MB and 7.4M Gaussians [S1, unofficial].
*   **Time:** Rendering performance is consistent above 150 FPS for all tested datasets (bunny_cloud, explosion, fire, utahteapot, venusstatue, dragon) across all LODs, even with millions of Gaussians [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Real-time performance via hardware-accelerated ray tracing (OptiX); memory efficiency through sparse VDB representation and LOD; support for dynamic transfer functions [S1, unofficial].
*   **Limitations:** Lossy representation due to non-overlapping constraint; limited to three LOD levels; inability to use NanoVDB for generation; register constraints on recursive tracing [S1, unofficial].

**When to Use:**
*   Scientific visualization of large sparse volumetric datasets where real-time performance is critical and lossy approximation is acceptable [S1, unofficial].

**When Not to Use:**
*   Photorealistic rendering requiring overlapping primitives and baked colors [S1, unofficial].
*   Scenes requiring fine-grained, non-voxel-aligned Gaussian distributions [S1, unofficial].

## Background

**Origin:**
The method was introduced in the paper "3D Gaussian Particle Approximation of VDB Datasets: A Study for Scientific Visualization" [S1, unofficial]. It builds on the popularity of Gaussian approximation in SciVis and the success of Gaussian Splatting in stereoscopic image to 3D scene conversion [S1, unofficial].

**History and Development:**
The approach integrates OpenVDB for memory-efficient storage and OptiX for hardware acceleration. It was developed to address the limitations of traditional scalar-field volume rendering and mesh-based surface rendering by encoding both similarly using Gaussians [S1, unofficial].

**Usage:**
Used in NVIDIA OptiX for forward ray marching through Gaussian particles [S1, unofficial].

## Sources

[S1] 3D Gaussian Particle Approximation of VDB Datasets: A Study for Scientific Visualization, https://www.emergentmind.com/papers/2504.04857, fetched 2026-10-29 (unofficial)
