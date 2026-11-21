# Imperfect Volumes for Global Illumination

Also called: Point-based voxelization for GI, Real-time GI using imperfect volumes

## Overview
Imperfect Volumes is a method for the real-time computation of ambient occlusion (AO) and diffuse indirect light transport in fully dynamic scenes [S1, unofficial]. It operates on a rough approximation of the entire scene created by a fast point-based voxelization algorithm, using 'imperfect volumes' where potential errors are small and do not produce objectionable errors in the final textured image [S1, unofficial]. It sets itself apart from exact voxelization methods by accepting geometric approximation errors in exchange for extreme speed and interactivity, and from screen-space methods by including geometry outside the view frustum or occluded by front objects [S1, unofficial].

## How it works
The method consists of two main phases: volume creation (voxelization) and irradiance reconstruction (lighting). The volume creation phase builds the "imperfect volume" from scene geometry. The irradiance reconstruction phase computes lighting by sampling this volume.

### Inputs and Outputs
*   **Inputs:** Scene geometry (triangles), camera parameters, volume resolution ($N^3$), number of points per triangle ($P$), number of rays per voxel ($R$), ray marching step size ($S$), and surface properties (diffuse reflectivity $\rho$, normal).
*   **Outputs:** A voxel grid storing directional visibility (for AO) or spherical harmonic coefficients (for GI), and the final rendered pixel colors.

### Data Structures
The core data structure is a 3D voxel grid of resolution $N \times N \times N$.
*   **Voxel Content:**
    *   For AO: Directional visibility. This is typically stored as a bitmask or a set of flags indicating which directions are occluded. The source notes that for AO, "directional visibility" is sampled and stored instead of incoming radiance [S1, unofficial].
    *   For GI: Spherical Harmonic (SH) coefficients representing the incident radiance distribution. The source states that radiance $\dot{L}_i$ is represented using spherical harmonics [S1, unofficial].
*   **Buffering:** Two volume buffers are used. The method samples illumination from one volume buffer and writes results to another. By alternating between these two buffers in successive passes, multiple bounces of light are computed [S1, unofficial].

### Algorithm Steps

#### Phase 1: Volume Creation (Voxelization)
1.  **Point Sampling:** For each triangle in the scene, generate $P$ random points distributed over the triangle's surface. The default value is $P=12$ [S1, unofficial].
2.  **Voxelization:** For each generated point, determine the voxel index in the $N^3$ grid that contains the point. Increment a counter or store the point in that voxel.
3.  **Ray Tracing (for GI):** For each occupied voxel, trace $R$ rays in random directions (or a fixed set of directions) to sample the environment. The default is $R=100$ [S1, unofficial].
    *   **Ray Marching:** Rays are marched through the volume in regular intervals (step size $S=1$ voxel) until an occupied cell is found or the volume extent is reached [S1, unofficial].
    *   **Self-Intersection Avoidance:** To avoid self-intersections with the voxel containing the ray origin, the ray-marching starts outside the originating voxel. An initial distance $d_s$ is skipped along each ray. For cubic voxels of size $s_v$, the skipped distance is:
        $$ d_s = \frac{\sqrt{3}}{2} s_v $$
        This value $d_s$ is the radius of the bounding sphere of the voxel [S1, unofficial].
    *   **Note:** This scheme does not avoid self-intersections with neighboring voxels potentially generated from the same polygon. These intersections are skipped during the final per-pixel irradiance reconstruction [S1, unofficial].
    *   **Storage:** For AO, store visibility (coverage). For GI, store the incident radiance distribution, approximated by spherical harmonics [S1, unofficial].

#### Phase 2: Irradiance Reconstruction (Lighting)
1.  **Surface Point Shift:** For each visible surface point $x$ with normal $\mathbf{n}$, shift the point outside the current voxel by moving it half a voxel along the normal. This avoids bias from the stored radiance distribution which may be affected by intersections with neighboring voxels from the same surface [S1, unofficial].
2.  **Neighbor Selection:** Identify the $N_{vox}$ closest voxels to the shifted point. For performance reasons, the implementation considers only the six nearest voxels [S1, unofficial].
3.  **Incident Radiance Estimation:** Calculate the estimated incident radiance $\dot{L}_i$ at the shifted point using the radiance of the selected voxels. The formula is:
    $$ \dot{L}_i = \frac{\sum_{j=1}^{N_{vox}} w_j \dot{L}_{ij}}{\sum_{j=1}^{N_{vox}} w_j} $$
    where:
    *   $\dot{L}_{ij}$ is the spherical harmonic representation of the radiance from the $j$-th voxel.
    *   $w_j$ is the weight for the $j$-th voxel, defined as:
        $$ w_j = \begin{cases} \cos \theta_j & \text{if } \theta_j < \pi/2 \\ 0 & \text{if } \theta_j > \pi/2 \end{cases} $$
    *   $\theta_j$ is the angle between the surface normal $\mathbf{n}$ and the direction to the $j$-th voxel. This ensures voxels behind the surface do not contribute, and voxels facing the normal contribute most [S1, unofficial].
4.  **Irradiance Calculation:** Compute the irradiance integral for the surface point. The diffusely reflected light (radiosity $B$) is given by:
    $$ B(x) = \frac{\rho(x)}{\pi} \int_{\Omega} L_i(x, \omega) \cos \theta d\omega $$
    Since $\dot{L}_i$ is represented using spherical harmonics, this integral is computed as a simple dot product with an analytically computed spherical harmonic representation of the cosine lobe directed towards the surface normal [S1, unofficial].
5.  **Multiple Bounces:** To compute indirect lighting bounces, repeat the volume creation and irradiance reconstruction steps, alternating between the two volume buffers [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_imperfect_volume(scene_triangles, volume_resolution, points_per_triangle, rays_per_voxel):
    """
    Creates an imperfect volume from scene geometry.
    
    Args:
        scene_triangles: List of triangles (each with 3 vertices).
        volume_resolution: Integer N, defining an N x N x N grid.
        points_per_triangle: Number of random points to sample per triangle.
        rays_per_voxel: Number of rays to trace per occupied voxel.
        
    Returns:
        volume_buffer: A 3D array of shape (N, N, N) storing voxel data.
    """
    N = volume_resolution
    voxel_size = 1.0 / N  # Assuming normalized world space [-0.5, 0.5] or [0, 1]
    # Note: The exact world space bounds are not specified, assuming standard normalization.
    
    # Initialize volume buffer. 
    # For AO: store visibility flags. For GI: store SH coefficients.
    # Here we assume a generic structure that can hold either.
    volume_buffer = initialize_volume(N) 
    
    # Phase 1: Voxelization
    for triangle in scene_triangles:
        # Sample P random points on the triangle
        points = sample_points_on_triangle(triangle, points_per_triangle)
        
        for point in points:
            # Determine voxel index for the point
            voxel_idx = world_to_voxel_index(point, N)
            
            # Mark voxel as occupied
            mark_voxel_occupied(volume_buffer, voxel_idx)
            
            # If GI mode, trace rays from this voxel
            if mode == GI:
                trace_rays_from_voxel(volume_buffer, voxel_idx, rays_per_voxel, voxel_size)
                
    return volume_buffer

def trace_rays_from_voxel(volume_buffer, voxel_idx, rays_per_voxel, voxel_size):
    """
    Traces rays from a voxel to sample incident radiance.
    """
    voxel_center = voxel_index_to_world(voxel_idx, voxel_size)
    
    # Skip distance to avoid self-intersection with the originating voxel
    # ds = (sqrt(3)/2) * sv
    skip_distance = (math.sqrt(3) / 2) * voxel_size
    
    for i in range(rays_per_voxel):
        # Generate random direction
        direction = random_unit_vector()
        
        # Start ray outside the originating voxel
        ray_origin = voxel_center + direction * skip_distance
        ray_direction = direction
        
        # Ray march
        current_pos = ray_origin
        while current_pos is within volume bounds:
            # Sample volume at current_pos
            sample = sample_volume(volume_buffer, current_pos)
            
            if sample.is_occupied:
                # Store incident radiance/visibility in the originating voxel
                store_radiance_in_voxel(volume_buffer, voxel_idx, sample, direction)
                break
            
            # Step forward by one voxel
            current_pos += ray_direction * voxel_size

def reconstruct_irradiance(surface_point, surface_normal, volume_buffer, volume_resolution):
    """
    Computes the final irradiance for a surface point using the imperfect volume.
    """
    voxel_size = 1.0 / volume_resolution
    
    # Shift surface point outside the current voxel along the normal
    shifted_point = surface_point + surface_normal * (0.5 * voxel_size)
    
    # Find N closest voxels (implementation uses N=6)
    closest_voxels = find_closest_voxels(shifted_point, volume_buffer, num_neighbors=6)
    
    # Estimate incident radiance using weighted average of closest voxels
    numerator = 0.0
    denominator = 0.0
    
    for voxel in closest_voxels:
        # Calculate angle between surface normal and direction to voxel
        direction_to_voxel = voxel.center - shifted_point
        direction_to_voxel = normalize(direction_to_voxel)
        cos_theta = dot(surface_normal, direction_to_voxel)
        
        # Weight calculation
        if cos_theta > 0:
            weight = cos_theta
        else:
            weight = 0.0
            
        # Get SH representation of radiance from voxel
        voxel_radiance_sh = voxel.sh_coefficients
        
        numerator += weight * voxel_radiance_sh
        denominator += weight
        
    if denominator > 0:
        estimated_radiance_sh = numerator / denominator
    else:
        estimated_radiance_sh = zero_sh()
        
    # Compute irradiance as dot product with cosine lobe SH
    cosine_lobe_sh = compute_cosine_lobe_sh(surface_normal)
    irradiance = dot_product_sh(estimated_radiance_sh, cosine_lobe_sh)
    
    return irradiance

def compute_radiosity(surface_point, surface_normal, volume_buffer, volume_resolution, diffuse_reflectivity):
    """
    Computes the final radiosity (diffuse reflected light) for a surface point.
    """
    irradiance = reconstruct_irradiance(surface_point, surface_normal, volume_buffer, volume_resolution)
    
    # B(x) = (rho(x) / pi) * irradiance
    radiosity = (diffuse_reflectivity / math.pi) * irradiance
    
    return radiosity
```

### Parameters
*   **Points per triangle ($P$):** Default 12. Range 6-19. Higher values improve quality but increase computation [S1, unofficial].
*   **Rays per voxel ($R$):** Default 100 [S1, unofficial].
*   **Volume Resolution ($N$):** Default $64^3$ [S1, unofficial].
*   **Ray Marching Step Size ($S$):** Default 1 voxel [S1, unofficial].
*   **Number of Nearest Voxels ($N_{vox}$):** 6 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Self-Intersections:** The initial distance skip scheme avoids self-intersections with the originating voxel but not with neighboring voxels from the same polygon. These are handled during irradiance reconstruction by shifting the point and using weights [S1, unofficial].
*   **Insufficient Resolution:** Low volume resolution causes loss of small-scale details like contact shadows [S1, unofficial].
*   **Precision:** The method relies on floating-point arithmetic for ray marching and SH calculations. Precision issues are not explicitly detailed, but the use of SH implies potential approximation errors in high-frequency lighting [S1, unofficial].
*   **Dynamic Scenes:** The method is designed for fully dynamic scenes, but the volume must be rebuilt every frame or when geometry changes significantly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Volume creation scales sub-linearly with the number of triangles and linearly with volume size [S1, unofficial]. Performance is mostly independent of the final image resolution [S1, unofficial].
*   **Memory Complexity:** Depends on the volume resolution ($N^3$) and the data stored per voxel (visibility flags or SH coefficients).
*   **Measured Performance:** Measured on an ATI Radeon 3650. The method maintains interactive framerates even for complex scenes (up to 100k triangles) [S1, unofficial]. AO is much faster than GI because the ray-marched distance is shorter and only coverage information is fetched [S1, unofficial].
*   **Strengths:** Real-time performance, handles dynamic scenes, includes geometry outside the view frustum, simple implementation.
*   **Limitations:** Approximate geometry (imperfections), loss of small-scale details at low resolution, limited to diffuse lighting (as per the source's focus on AO and diffuse GI).

## Background
The method was presented in the paper "Global Illumination Using Imperfect Volumes" at GRAPP 2011 [S1, unofficial]. It builds on the concept of voxelization for real-time lighting, extending it to handle indirect illumination by using spherical harmonics to represent radiance distributions. The term "imperfect volumes" refers to the acceptance of geometric approximation errors in the voxelization process to achieve real-time performance.

## Sources
[S1] Global Illumination Using Imperfect Volumes, https://www.scitepress.org/PublishedPapers/2011/33186/, fetched 2026-10-29 (unofficial)
