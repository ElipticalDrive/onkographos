# Monte Carlo Path Tracing for Volume Rendering

Also called: MCSRender, Monte Carlo Scattering Rendering

## Overview
Monte Carlo Path Tracing for Volume Rendering (MCSRender) is a stochastic image-order rendering technique that simulates light transport within participating media by tracing random paths through the volume [S1, unofficial]. Unlike deterministic ray casting which samples equidistant intervals, this method uses distance sampling based on the maximum scattering coefficient to select scattering events, accumulating results over multiple frames via exponential moving averages to converge to the final image [S1, unofficial]. It distinguishes itself from standard direct volume ray casting by explicitly handling scattering events through random distance jumps rather than fixed-step integration, making it suitable for volumetric effects where light interacts with the medium's interior [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- Camera parameters: Origin $\mathbf{o}$, direction $\mathbf{d}$, and inverse Model-View-Projection (MVP) matrix for unprojection [S1, unofficial].
- Volume domain: Axis-aligned bounding box (AABB) defined by minimum and maximum bounds [S1, unofficial].
- Volume data: Scalar field $V(\mathbf{p})$ accessed via texture sampling [S1, unofficial].
- Transfer function: Maps scalar values to color $C$ and opacity $\alpha$ [S1, unofficial].
- Environment map: Provides incident lighting $L(\mathbf{d})$ [S1, unofficial].
- Parameters:
  - $u_{\Sigma_{max}}$: Maximum scattering coefficient for distance sampling [S1, unofficial].
  - $u_{\alpha_{corr}}$: Alpha correction factor applied to transfer function output [S1, unofficial].
  - $u_{\text{offset}}$: Random seed offset [S1, unofficial].
  - $u_{1/N}$: Inverse frame number for accumulation [S1, unofficial].

**Outputs:**
- Final pixel color $C_{final}$ accumulated over frames [S1, unofficial].

### Data Structures
The implementation relies on GPU fragment shader state and texture samplers:
- `uAccumulator`: A texture storing the running average of rendered frames [S1, unofficial].
- `uFrame`: A texture storing the current frame's raw contribution [S1, unofficial].
- `uVolume`: 3D texture for scalar volume data [S1, unofficial].
- `uEnvMap`: Cubemap or 2D texture for environment lighting [S1, unofficial].
- `seed`: A `vec2` variable initialized per pixel to drive random number generation [S1, unofficial].

### Algorithm Steps

1. **Ray Generation**: Unproject the screen position $(x, y)$ using the inverse MVP matrix to obtain the ray origin $\mathbf{v}_{\text{from}}$ and a point on the far plane $\mathbf{v}_{\text{to}}$ [S1, unofficial].
2. **Volume Intersection**: Intersect the ray with the volume's AABB to determine the entry parameter $t_{\text{start}}$ and exit parameter $t_{\text{end}}$ [S1, unofficial]. Let the segment within the volume be defined by $\mathbf{p}_{\text{start}}$ and $\mathbf{p}_{\text{end}}$. Calculate the maximum distance $D_{\text{max}} = \|\mathbf{p}_{\text{end}} - \mathbf{p}_{\text{start}}\|$ [S1, unofficial].
3. **Early Exit**: If the ray does not intersect the volume ($t_{\text{start}} \ge t_{\text{end}}$), sample the environment map at the view direction and output as color [S1, unofficial].
4. **Seed Initialization**: Initialize the random seed using the pixel position and a uniform offset:
   $$ \text{seed} = \mathbf{p}_{\text{pixel}} + \text{rand}(\mathbf{u}_{\text{offset}}) $$
   where $\text{rand}$ generates a pseudo-random value [S1, unofficial].
5. **Distance Sampling**: Call `sampleDistance` with $\mathbf{p}_{\text{start}}$, $\mathbf{p}_{\text{end}}$, and $\text{seed}$ to find the distance $d_{\text{scat}}$ to the first scattering event [S1, unofficial].
   - If $d_{\text{scat}} > D_{\text{max}}$, the ray exits the volume without scattering. Sample the environment map at the view direction and output as color [S1, unofficial].
6. **Update Position**: Update the ray origin to the scattering point:
   $$ \mathbf{p}_{\text{scat}} = \text{mix}(\mathbf{p}_{\text{start}}, \mathbf{p}_{\text{end}}, \frac{d_{\text{scat}}}{D_{\text{max}}}) $$
   [S1, unofficial].
7. **Scattering Direction**: Determine the scattering direction $\mathbf{d}_{\text{scat}}$. In the cited implementation, this is provided as a uniform parameter $u_{\text{ScatteringDirection}}$ [S1, unofficial].
8. **Secondary Ray Intersection**: Intersect a new ray starting at $\mathbf{p}_{\text{scat}}$ with direction $\mathbf{d}_{\text{scat}}$ against the volume AABB to find the new exit point $\mathbf{p}_{\text{new\_end}}$ [S1, unofficial].
9. **Sample Properties**:
   - Sample the volume color at $\mathbf{p}_{\text{scat}}$: $C_{\text{diffuse}} = \text{sampleVolumeColor}(\mathbf{p}_{\text{scat}})$ [S1, unofficial].
   - Sample the environment lighting in the scattering direction: $L_{\text{env}} = \text{sampleEnvironmentMap}(\mathbf{d}_{\text{scat}})$ [S1, unofficial].
10. **Transmittance Calculation**: Compute the transmittance from $\mathbf{p}_{\text{scat}}$ to $\mathbf{p}_{\text{new\_end}}$ using `sampleTransmittance` [S1, unofficial].
11. **Color Computation**: Calculate the final color contribution:
    $$ C_{\text{final}} = C_{\text{diffuse}} \times L_{\text{env}} \times T $$
    where $T$ is the transmittance [S1, unofficial].
12. **Accumulation**: Update the frame accumulator using an exponential moving average:
    $$ C_{\text{acc}} = C_{\text{acc}} + (C_{\text{final}} - C_{\text{acc}}) \times u_{1/N} $$
    where $C_{\text{acc}}$ is read from `uAccumulator` and $u_{1/N}$ is the inverse frame number [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcs_render_pixel(pixel_pos, inv_mvp, volume_tex, env_tex, 
                     volume_bounds, u_sigma_max, u_alpha_corr, 
                     u_offset, u_inv_frame_num, u_scattering_dir):
    """
    Renders a single pixel using Monte Carlo Path Tracing for Volume Rendering.
    """
    # 1. Ray Generation
    # Unproject screen position to world space ray origin and far plane point
    ray_origin = unproject(pixel_pos, inv_mvp)
    ray_far = unproject_far_plane(pixel_pos, inv_mvp)
    
    # 2. Volume Intersection
    t_start, t_end = intersect_ray_aabb(ray_origin, ray_far, volume_bounds)
    
    if t_start >= t_end:
        # Ray misses volume, sample environment
        return sample_environment(ray_far, env_tex)
    
    # Calculate segment within volume
    p_start = ray_origin + t_start * (ray_far - ray_origin)
    p_end = ray_origin + t_end * (ray_far - ray_origin)
    max_dist = length(p_end - p_start)
    
    # 3. Seed Initialization
    # Generate random seed based on pixel position and offset
    seed = pixel_pos + random(u_offset)
    
    # 4. Distance Sampling
    # Find distance to first scattering event
    dist_scat = sample_distance(p_start, p_end, seed, u_sigma_max, max_dist)
    
    if dist_scat > max_dist:
        # No scattering within volume, sample environment
        return sample_environment(ray_far, env_tex)
    
    # 5. Update Position
    # Move to scattering point
    p_scat = mix(p_start, p_end, dist_scat / max_dist)
    
    # 6. Secondary Ray Intersection
    # Intersect scattering direction with volume bounds
    t_new_start, t_new_end = intersect_ray_aabb(p_scat, u_scattering_dir, volume_bounds)
    
    # If no intersection in scattering direction, treat as exit
    if t_new_start >= t_new_end:
        return sample_environment(u_scattering_dir, env_tex)
        
    p_new_end = p_scat + t_new_end * u_scattering_dir
    
    # 7. Sample Properties
    # Sample volume color (includes transfer function application)
    c_diffuse = sample_volume_color(p_scat, volume_tex, u_alpha_corr)
    
    # Sample environment lighting
    l_env = sample_environment(u_scattering_dir, env_tex)
    
    # 8. Transmittance Calculation
    # Calculate transmittance from scattering point to new exit
    transmittance = sample_transmittance(p_scat, p_new_end, seed, u_sigma_max, u_alpha_corr)
    
    # 9. Color Computation
    final_color = c_diffuse * l_env * transmittance
    
    # 10. Accumulation
    # Read current accumulator
    acc_color = texture(u_accumulator, pixel_pos)
    
    # Update with exponential moving average
    new_acc = acc_color + (final_color - acc_color) * u_inv_frame_num
    
    return new_acc

def sample_distance(p_start, p_end, seed, sigma_max, max_dist):
    """
    Samples the distance to the next scattering event using rejection sampling.
    """
    inv_sigma_max = 1.0 / sigma_max
    inv_max_dist = 1.0 / max_dist
    
    dist = 0.0
    while True:
        # Generate random number
        r = random(seed)
        
        # Sample distance from exponential distribution
        dist -= log(1.0 - r.x) * inv_sigma_max
        
        if dist > max_dist:
            return dist
            
        # Check opacity at sampled position
        sampling_pos = mix(p_start, p_end, dist * inv_max_dist)
        alpha_sample = sample_volume_alpha(sampling_pos) * u_alpha_corr
        
        # Rejection test
        r2 = random(seed)
        if r2.y < alpha_sample * inv_sigma_max:
            return dist

def sample_transmittance(p_start, p_end, seed, sigma_max, u_alpha_corr):
    """
    Calculates transmittance along a segment using Monte Carlo integration.
    """
    transmittance = 1.0
    max_dist = length(p_end - p_start)
    inv_sigma_max = 1.0 / sigma_max
    inv_max_dist = 1.0 / max_dist
    
    # Similar loop structure to sample_distance but accumulates transmittance
    # Note: The source implies a loop similar to sampleDistance
    # We assume a single sample or a loop for integration. 
    # Based on source: "Loop similarly to sampleDistance... transmittance *= 1.0 - alphaSample * invSigmaMax"
    
    # For simplicity and adherence to source structure, we perform one step 
    # or iterate if multiple samples are implied. The source describes a loop.
    # We will implement a single pass as per the specific "sampleTransmittance" 
    # description which updates transmittance in a loop.
    
    dist = 0.0
    while dist < max_dist:
        r = random(seed)
        dist -= log(1.0 - r.x) * inv_sigma_max
        
        if dist > max_dist:
            break
            
        sampling_pos = mix(p_start, p_end, dist * inv_max_dist)
        alpha_sample = sample_volume_alpha(sampling_pos) * u_alpha_corr
        
        transmittance *= (1.0 - alpha_sample * inv_sigma_max)
        
        r2 = random(seed)
        if r2.y < alpha_sample * inv_sigma_max:
            break
            
    return transmittance
```

### Edge Cases and Pitfalls
- **Precision**: The use of `log(1.0 - seed.x)` requires `seed.x` to be strictly less than 1.0 to avoid `log(0)`. Standard pseudo-random generators should be clamped or handled to ensure this [S1, unofficial].
- **Boundary Conditions**: If the scattering direction ray does not intersect the volume, the algorithm defaults to environment sampling. This must be handled explicitly to avoid undefined behavior [S1, unofficial].
- **Alpha Correction**: The parameter $u_{\alpha_{corr}}$ scales the opacity from the transfer function. Incorrect values can lead to over- or under-saturation of the volume [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity**: The cost per pixel is dominated by the number of random samples required to converge. Since it uses exponential moving average accumulation, the variance decreases as $1/N$ where $N$ is the number of frames [S1, unofficial]. Each frame requires one distance sampling and one transmittance calculation, involving multiple texture lookups [S1, unofficial].
- **Memory Complexity**: Requires storage for the accumulator texture (`uAccumulator`) and the current frame texture (`uFrame`), in addition to the volume and environment textures [S1, unofficial].

### Measured Performance
- Not given in the sources.

### Strengths and Limitations
- **Strengths**: Handles scattering events stochastically, potentially offering better convergence for complex lighting interactions in volumes compared to fixed-step methods [S1, unofficial].
- **Limitations**: Requires multiple frames to converge, making it unsuitable for real-time interactive applications without temporal accumulation [S1, unofficial]. Performance depends heavily on the scattering coefficient and volume density [S1, unofficial].

### When to Use
- Use when high-quality volumetric rendering with scattering is required and real-time performance is not critical [S1, unofficial].
- Use when the volume has complex lighting interactions that benefit from path tracing [S1, unofficial].

### When Not to Use
- Do not use for real-time applications requiring immediate feedback [S1, unofficial].
- Do not use for simple opacity-based volume rendering where scattering is negligible [S1, unofficial].

## Background

### Origin and History
The method is described in the context of the "vpt-conductor" project by UL-FRI-LGM [S1, unofficial]. It represents an application of Monte Carlo path tracing techniques to volume rendering, specifically focusing on scattering events within the volume [S1, unofficial].

### Usage
It is used in research and high-quality rendering contexts where the visual fidelity of volumetric scattering is prioritized over frame rate [S1, unofficial].

## Sources

[S1] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
