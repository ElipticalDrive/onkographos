# Monte Carlo Path Tracing for Volume Rendering

Also called: MCSRender, Monte Carlo Scattering Rendering

## Overview
Monte Carlo Path Tracing for Volume Rendering (MCSRender) is a stochastic method that simulates light transport within participating media by randomly sampling scattering events along a ray, rather than discretizing the ray into fixed steps [S1, unofficial]. Unlike deterministic ray marching methods that integrate over equidistant intervals, this approach uses distance sampling to locate the next interaction point, allowing for efficient handling of sparse or highly heterogeneous volumes where large empty regions can be skipped [S1, unofficial]. It distinguishes itself from its siblings by replacing the fixed-step compositing loop with a probabilistic path generation loop, accumulating results over multiple frames to reduce noise [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera parameters: Origin $\mathbf{o}$, direction $\mathbf{d}$, and inverse Model-View-Projection (MVP) matrix for ray unprojection [S1, unofficial].
*   Volume domain: An axis-aligned bounding box (AABB) defining the volume extent [S1, unofficial].
*   Volume data: A scalar field $V(\mathbf{p})$ accessed via a transfer function to yield color $C$ and opacity $\alpha$ [S1, unofficial].
*   Environment map: A texture providing incident light color $L_{env}$ [S1, unofficial].
*   Parameters:
    *   $\sigma_{max}$: Maximum scattering coefficient used for distance sampling [S1, unofficial].
    *   $\alpha_{corr}$: Correction factor for alpha values from the transfer function [S1, unofficial].
    *   $\mathbf{d}_{scat}$: Uniform scattering direction vector [S1, unofficial].
    *   $\mathbf{offset}$: Offset for random seed generation [S1, unofficial].
    *   $N_{frame}$: Current frame number for accumulation [S1, unofficial].

**Outputs:**
*   Final pixel color $C_{final}$, accumulated over frames using an exponential moving average [S1, unofficial].

### Data Structures
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Seed:** A 2D vector $\mathbf{s} = (s_x, s_y)$ used for random number generation in distance sampling and Russian roulette/termination checks [S1, unofficial].
*   **Accumulator:** A buffer storing the running average of rendered frames, initialized to zero [S1, unofficial].

### Algorithm Steps

1.  **Ray Generation:**
    Unproject the screen coordinates to obtain the ray origin $\mathbf{o}$ and a point on the far plane $\mathbf{p}_{far}$. Construct the ray direction $\mathbf{d} = \mathbf{p}_{far} - \mathbf{o}$ and normalize it [S1, unofficial].

2.  **Volume Intersection:**
    Intersect the ray with the volume AABB to find entry parameter $t_{start}$ and exit parameter $t_{end}$ [S1, unofficial].
    *   If $t_{start} \ge t_{end}$, the ray misses the volume. Sample the environment map using $\mathbf{d}$ and return the color [S1, unofficial].
    *   Calculate the ray segment within the volume: $\mathbf{r}_{from} = \mathbf{o} + t_{start}\mathbf{d}$ and $\mathbf{r}_{to} = \mathbf{o} + t_{end}\mathbf{d}$.
    *   Compute the maximum distance: $D_{max} = \|\mathbf{r}_{to} - \mathbf{r}_{from}\|$ [S1, unofficial].

3.  **Initialization:**
    Initialize the random seed based on pixel position and frame offset:
    $$ \mathbf{s} = \mathbf{p}_{pixel} + \text{rand}(\mathbf{u}_{offset}) $$
    where $\text{rand}(\cdot)$ generates a uniform random vector [S1, unofficial].

4.  **Distance Sampling (Finding the Scattering Point):**
    Call `sampleDistance` to find the distance $d_{scat}$ to the first scattering event along the ray segment $[\mathbf{r}_{from}, \mathbf{r}_{to}]$.
    *   Initialize $d = 0.0$, $I_{\sigma} = 1.0 / \sigma_{max}$, and $I_{D} = 1.0 / D_{max}$ [S1, unofficial].
    *   Loop:
        1.  Generate a random value $u_1 \in [0, 1)$ from the seed.
        2.  Update distance: $d \leftarrow d - \ln(1 - u_1) \cdot I_{\sigma}$ [S1, unofficial].
        3.  If $d > D_{max}$, break the loop (no scattering within volume).
        4.  Calculate sampling position: $\mathbf{p}_{sample} = \text{mix}(\mathbf{r}_{from}, \mathbf{r}_{to}, d \cdot I_{D})$ [S1, unofficial].
        5.  Sample volume opacity: $\alpha_{sample} = T(V(\mathbf{p}_{sample})) \cdot \alpha_{corr}$ [S1, unofficial].
        6.  Generate a second random value $u_2 \in [0, 1)$ from the seed.
        7.  If $u_2 < \alpha_{sample} \cdot I_{\sigma}$, break the loop (scattering event accepted) [S1, unofficial].
    *   If the loop exits because $d > D_{max}$, sample the environment map using the scattering direction $\mathbf{d}_{scat}$ and return the color [S1, unofficial].

5.  **Shading and Transmittance:**
    *   Update the ray origin to the scattering point: $\mathbf{r}_{from} \leftarrow \mathbf{p}_{sample}$ [S1, unofficial].
    *   Sample the diffuse color at the scattering point: $C_{diff} = \text{sampleVolumeColor}(\mathbf{r}_{from})$ [S1, unofficial].
    *   Sample the light color from the environment map in the scattering direction: $C_{light} = \text{sampleEnvironmentMap}(\mathbf{d}_{scat})$ [S1, unofficial].
    *   Calculate transmittance from $\mathbf{r}_{from}$ to $\mathbf{r}_{to}$ using `sampleTransmittance` [S1, unofficial].
        *   Initialize $T = 1.0$.
        *   Loop similarly to `sampleDistance` but update transmittance: $T \leftarrow T \cdot (1 - \alpha_{sample} \cdot I_{\sigma})$ [S1, unofficial].

6.  **Color Computation:**
    Compute the frame color:
    $$ C_{frame} = C_{diff} \times C_{light} \times T $$
    [S1, unofficial].

7.  **Accumulation:**
    Update the accumulator buffer using an exponential moving average:
    $$ C_{acc} = C_{acc} + (C_{frame} - C_{acc}) \cdot \frac{1}{N_{frame}} $$
    [S1, unofficial].
    The final output color is $C_{acc}$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_path_trace_volume(ray_origin, ray_direction, volume_bounds, volume_data, env_map, sigma_max, alpha_corr, scattering_dir, offset, frame_number, accumulator):
    """
    Renders a single pixel using Monte Carlo Path Tracing for Volume Rendering.
    
    Args:
        ray_origin: vec3, camera origin
        ray_direction: vec3, normalized ray direction
        volume_bounds: AABB, min and max corners of the volume
        volume_data: function, samples scalar value V(p) at position p
        env_map: function, samples light color L(d) at direction d
        sigma_max: float, maximum scattering coefficient
        alpha_corr: float, alpha correction factor
        scattering_dir: vec3, direction for secondary ray scattering
        offset: vec2, offset for random seed
        frame_number: int, current frame index (1-based)
        accumulator: vec4, running average color buffer
        
    Returns:
        vec4, final accumulated color
    """
    # 1. Ray-Volume Intersection
    t_start, t_end = intersect_aabb(ray_origin, ray_direction, volume_bounds)
    
    if t_start >= t_end:
        return env_map(ray_direction)
    
    r_from = ray_origin + t_start * ray_direction
    r_to = ray_origin + t_end * ray_direction
    max_distance = length(r_to - r_from)
    
    # 2. Initialize Seed
    # Assuming pixel position is available as pixel_pos
    seed = pixel_pos + random_vector(offset)
    
    # 3. Distance Sampling
    dist = sample_distance(r_from, r_to, max_distance, sigma_max, volume_data, alpha_corr, seed)
    
    if dist > max_distance:
        return env_map(scattering_dir)
    
    # 4. Update Position and Sample Properties
    # Interpolate position along the ray segment
    t_sample = dist / max_distance
    p_sample = mix(r_from, r_to, t_sample)
    
    # Sample diffuse color
    c_diff = sample_volume_color(p_sample, volume_data)
    
    # Sample light color
    c_light = env_map(scattering_dir)
    
    # 5. Calculate Transmittance
    transmittance = sample_transmittance(p_sample, r_to, max_distance, sigma_max, volume_data, alpha_corr, seed)
    
    # 6. Compute Frame Color
    c_frame = c_diff * c_light * transmittance
    
    # 7. Accumulate
    inv_frame_number = 1.0 / frame_number
    c_acc = accumulator
    c_final = c_acc + (c_frame - c_acc) * inv_frame_number
    
    return c_final

def sample_distance(r_from, r_to, max_distance, sigma_max, volume_data, alpha_corr, seed):
    """
    Samples the distance to the next scattering event using distance sampling.
    """
    dist = 0.0
    inv_sigma_max = 1.0 / sigma_max
    inv_max_distance = 1.0 / max_distance
    
    while True:
        # Generate random number for distance
        u1 = random_float(seed)
        dist -= log(1.0 - u1) * inv_sigma_max
        
        if dist > max_distance:
            return dist
        
        # Sample volume at current distance
        t = dist * inv_max_distance
        p = mix(r_from, r_to, t)
        
        # Get opacity from transfer function
        alpha_sample = alpha_corr * get_alpha(volume_data(p))
        
        # Russian Roulette / Acceptance test
        u2 = random_float(seed)
        if u2 < alpha_sample * inv_sigma_max:
            return dist

def sample_transmittance(r_from, r_to, max_distance, sigma_max, volume_data, alpha_corr, seed):
    """
    Calculates transmittance along the ray segment.
    """
    transmittance = 1.0
    inv_sigma_max = 1.0 / sigma_max
    inv_max_distance = 1.0 / max_distance
    
    # Note: The source implies a loop similar to sampleDistance but for transmittance.
    # In standard distance sampling, transmittance is often computed analytically 
    # or via a separate sampling pass. Here we follow the source's iterative description.
    
    # Since the source does not specify the exact loop bounds for transmittance 
    # other than "Loop similarly", we assume it samples the same path or a parallel path.
    # However, standard practice is to compute T = exp(-sigma_t * d).
    # The source code snippet: "transmittance *= 1.0 - alphaSample * invSigmaMax"
    # suggests a discrete approximation or a specific sampling strategy.
    
    # To be implementable from the source alone, we replicate the logic structure:
    # We assume the transmittance calculation iterates over the same distance steps 
    # or uses a similar random walk. Given the ambiguity, we implement a simplified 
    # version that matches the source's variable updates.
    
    # Re-initialize for transmittance calculation along the segment
    # The source does not give a distinct start point for transmittance loop other than 'from'
    # We assume it integrates from the scattering point to the end of the volume or next event.
    
    # Based on source: "Calculate transmittance from from to to using sampleTransmittance"
    # And: "Loop similarly to sampleDistance, but update transmittance *= ..."
    
    # We will implement a loop that steps through the volume to compute T.
    # Since the exact step count isn't given, we use the distance sampling logic 
    # to determine if we hit the end, but accumulate T.
    
    # Note: This is a direct translation of the source's procedural description.
    # In a real implementation, one might use the same 'dist' found in sample_distance 
    # if the transmittance is for the path *to* the scattering point, but here it is 
    # from the scattering point *out*.
    
    # Let's assume a similar random walk for the outgoing path.
    current_dist = 0.0
    while current_dist < max_distance:
        u1 = random_float(seed)
        step = -log(1.0 - u1) * inv_sigma_max
        current_dist += step
        
        if current_dist > max_distance:
            break
            
        t = current_dist * inv_max_distance
        p = mix(r_from, r_to, t) # Note: r_from is the scattering point here
        alpha_sample = alpha_corr * get_alpha(volume_data(p))
        
        transmittance *= (1.0 - alpha_sample * inv_sigma_max)
        
    return transmittance
```

### Edge Cases and Pitfalls
*   **Precision:** The use of `log(1.0 - seed.x)` requires `seed.x` to be strictly less than 1.0 to avoid `log(0)`. Standard random number generators should be clamped or handled to ensure this [S1, unofficial].
*   **Boundary Conditions:** If the ray origin is inside the volume, $t_{start}$ should be 0 [S1, unofficial].
*   **Noise:** Since this is a Monte Carlo method, the result is noisy for low frame counts. Accumulation over many frames is required for convergence [S1, unofficial].
*   **Parameter Sensitivity:** $\sigma_{max}$ must be chosen carefully. If $\sigma_{max}$ is too small, the distance sampling will produce very large steps, potentially skipping scattering events. If it is too large, the acceptance rate ($\alpha_{sample} \cdot I_{\sigma}$) will be low, leading to many wasted samples [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** The complexity depends on the number of scattering events per ray. In dense volumes, many samples are required. In sparse volumes, the distance sampling allows skipping empty space, potentially offering better performance than fixed-step ray marching [S1, unofficial].
*   **Memory:** Requires storage for the volume data, environment map, and an accumulator buffer [S1, unofficial].

### Measured Performance
*   Not given in the sources.

### Scaling
*   Scales linearly with the number of frames for noise reduction [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Efficiently handles sparse volumes by skipping empty regions; naturally handles complex lighting via environment map integration [S1, unofficial].
*   **Limitations:** Produces noisy results that require temporal accumulation; sensitive to parameter tuning ($\sigma_{max}$) [S1, unofficial].

### When to Use
*   Use when the volume is sparse or when accurate integration of light transport from complex environments is required [S1, unofficial].
*   Do not use when deterministic, artifact-free results are needed immediately without temporal accumulation [S1, unofficial].

## Background

### Origin
The method is described in the context of the `vpt-conductor` project by UL-FRI-LGM [S1, unofficial].

### History
Not given in the sources.

### Usage
Used in real-time or near-real-time volume rendering applications where stochastic sampling is preferred for handling participating media [S1, unofficial].

## Sources

[S1] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
