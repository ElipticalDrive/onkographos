# Monte Carlo Path Tracing for Volume Rendering

Also called: MCSRender, Monte Carlo Scattering Rendering

## Overview
Monte Carlo Path Tracing for Volume Rendering (MCSRender) is a stochastic method for simulating light transport within participating media by sampling scattering events along rays rather than discretizing the path into fixed steps [S1, unofficial]. Unlike deterministic ray marching which integrates the volume rendering equation via quadrature, this method uses distance sampling to find interaction points and accumulates results over multiple frames using an exponential moving average to converge to the solution [S1, unofficial]. It is distinguished by its ability to handle complex scattering geometries and anisotropic lighting via environment maps without requiring a precomputed light field or voxelized light transport data [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Volume data structure containing scalar density values and associated optical properties (color, opacity) via a transfer function.
- Camera parameters: origin $\mathbf{o}$, direction $\mathbf{d}$, and projection matrix.
- Volume bounding box (cube) defined in world space.
- Environment map for distant lighting.
- Parameters: $\sigma_{max}$ (maximum scattering coefficient), $\alpha_{corr}$ (alpha correction factor), $\mathbf{u}_{scat}$ (scattering direction vector), $\mathbf{u}_{offset}$ (seed offset).

**Outputs:**
- Final pixel color $C_{final}$ accumulated over $N$ frames.

### Data Structures
- **Ray**: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- **Bounding Box Intersection**: Returns $t_{min}$ and $t_{max}$ (denoted as `tbounds.x` and `tbounds.y` in source) representing the entry and exit points of the ray with the volume cube [S1, unofficial].
- **Accumulator Buffer**: A texture or buffer storing the running average color per pixel, initialized to zero [S1, unofficial].
- **Frame Buffer**: Stores the color contribution of the current frame [S1, unofficial].

### Algorithm Steps
1.  **Ray Generation**: Unproject the screen coordinates to world space to obtain the ray origin $\mathbf{o}$ and a point on the far plane to define the direction $\mathbf{d}$. Compute the ray direction vector $\mathbf{d} = \mathbf{v}_{ray\_to} - \mathbf{v}_{ray\_from}$ and normalize it [S1, unofficial].
2.  **Bounding Box Intersection**: Intersect the ray with the volume's bounding cube. If the ray does not intersect the volume ($t_{min} \ge t_{max}$), sample the environment map at direction $\mathbf{d}$ and output as the background color [S1, unofficial].
3.  **Segment Definition**: Calculate the valid segment $[t_{start}, t_{end}]$ within the volume. Compute the maximum distance $L = t_{end} - t_{start}$ [S1, unofficial].
4.  **Seed Initialization**: Generate a random seed for the current pixel using the pixel position and a uniform offset: $\mathbf{s} = \mathbf{p}_{pixel} + \text{rand}(\mathbf{u}_{offset})$ [S1, unofficial].
5.  **Distance Sampling**: Call `sampleDistance` to find the distance $d_{scat}$ to the first scattering event along the ray segment $[t_{start}, t_{end}]$.
    -   If $d_{scat} > L$, the ray exits the volume without scattering. Sample the environment map at the current direction and output as color [S1, unofficial].
    -   Otherwise, update the ray origin to the scattering point: $\mathbf{o}_{new} = \mathbf{o} + d_{scat} \cdot \mathbf{d}$ [S1, unofficial].
6.  **Secondary Ray Setup**: Define the scattering direction $\mathbf{d}_{scat}$ (often derived from $\mathbf{u}_{scat}$ or sampled from a phase function, though the source specifies a uniform direction vector parameter) [S1, unofficial]. Intersect this new ray with the volume bounding box to find the new segment $[t'_{min}, t'_{max}]$ and length $L'$ [S1, unofficial].
7.  **Lighting and Transmittance**:
    -   Sample the volume color $C_{vol}$ at $\mathbf{o}_{new}$ using the transfer function [S1, unofficial].
    -   Sample the environment map $C_{env}$ at direction $\mathbf{d}_{scat}$ [S1, unofficial].
    -   Calculate transmittance $T$ from $\mathbf{o}_{new}$ to the exit of the volume along $\mathbf{d}_{scat}$ using `sampleTransmittance` [S1, unofficial].
8.  **Color Computation**: Compute the frame color contribution:
    $$ C_{frame} = C_{vol} \times C_{env} \times T $$
    [S1, unofficial].
9.  **Accumulation**: Update the accumulator buffer using an exponential moving average:
    $$ C_{acc}^{(new)} = C_{acc}^{(old)} + (C_{frame} - C_{acc}^{(old)}) \frac{1}{k} $$
    where $k$ is the current frame number [S1, unofficial].
10. **Reset**: After rendering is complete or for a new sequence, reset the accumulator to zero [S1, unofficial].

### Distance Sampling (`sampleDistance`)
This function determines the distance to the next scattering event using inhomogeneous medium sampling [S1, unofficial].

**Inputs:**
- Ray segment start `from`, end `to`.
- Seed $\mathbf{s}$.
- $\sigma_{max}$: Maximum scattering coefficient.
- $\alpha_{corr}$: Alpha correction factor.

**Steps:**
1.  Initialize $dist = 0.0$.
2.  Compute $invSigmaMax = 1.0 / \sigma_{max}$ and $invMaxDistance = 1.0 / L$ [S1, unofficial].
3.  Loop:
    a. Generate random value $r_1$ from seed.
    b. Update distance: $dist \leftarrow dist - \ln(1.0 - r_1) \cdot invSigmaMax$ [S1, unofficial].
    c. If $dist > L$, break and return $dist$ (no scattering within segment) [S1, unofficial].
    d. Calculate sampling position: $\mathbf{p} = \text{mix}(\text{from}, \text{to}, dist \cdot invMaxDistance)$ [S1, unofficial].
    e. Sample volume opacity $\alpha_{sample}$ from the transfer function at $\mathbf{p}$ and apply correction: $\alpha_{corr\_val} = \alpha_{sample} \cdot \alpha_{corr}$ [S1, unofficial].
    f. Generate random value $r_2$ from seed.
    g. If $r_2 < \alpha_{corr\_val} \cdot invSigmaMax$, break and return $dist$ (scattering event occurred) [S1, unofficial].
4.  Return $dist$ [S1, unofficial].

### Transmittance Sampling (`sampleTransmittance`)
This function estimates the transmittance along a segment by sampling opacity values [S1, unofficial].

**Inputs:**
- Ray segment start `from`, end `to`.
- Seed $\mathbf{s}$.
- $\sigma_{max}$, $\alpha_{corr}$.

**Steps:**
1.  Initialize $transmittance = 1.0$ [S1, unofficial].
2.  Loop (similar to `sampleDistance`):
    a. Generate random $r_1$.
    b. Update $dist \leftarrow dist - \ln(1.0 - r_1) \cdot invSigmaMax$.
    c. If $dist > L$, break [S1, unofficial].
    d. Calculate position $\mathbf{p}$ and sample corrected opacity $\alpha_{corr\_val}$ [S1, unofficial].
    e. Generate random $r_2$.
    f. If $r_2 < \alpha_{corr\_val} \cdot invSigmaMax$, break [S1, unofficial].
    g. Update transmittance: $transmittance \leftarrow transmittance \cdot (1.0 - \alpha_{corr\_val} \cdot invSigmaMax)$ [S1, unofficial].
3.  Return $transmittance$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_distance(from_point, to_point, max_dist, sigma_max, alpha_corr, seed):
    """
    Samples the distance to the next scattering event.
    Returns distance to scattering event, or max_dist if none found.
    """
    dist = 0.0
    inv_sigma_max = 1.0 / sigma_max
    inv_max_dist = 1.0 / max_dist
    
    while True:
        # Generate random number from seed
        r1 = random.random() # Assuming seed manages RNG state
        
        # Update distance using exponential distribution
        dist -= math.log(1.0 - r1) * inv_sigma_max
        
        if dist > max_dist:
            return max_dist
            
        # Calculate sampling position
        t = dist * inv_max_dist
        sampling_pos = mix(from_point, to_point, t)
        
        # Sample volume opacity and apply correction
        alpha_sample = sample_volume_opacity(sampling_pos)
        alpha_corr_val = alpha_sample * alpha_corr
        
        # Russian Roulette / Acceptance test
        r2 = random.random()
        if r2 < alpha_corr_val * inv_sigma_max:
            return dist

def sample_transmittance(from_point, to_point, max_dist, sigma_max, alpha_corr, seed):
    """
    Estimates transmittance along the segment.
    """
    transmittance = 1.0
    inv_sigma_max = 1.0 / sigma_max
    inv_max_dist = 1.0 / max_dist
    
    while True:
        r1 = random.random()
        dist = 0.0 # Reset dist for this specific loop logic if needed, 
                   # but source implies a loop similar to sampleDistance 
                   # accumulating transmittance. 
                   # Note: The source description is slightly ambiguous on 
                   # whether dist accumulates or resets. 
                   # Based on "Loop similarly... update transmittance", 
                   # we assume a similar stepping mechanism.
        
        # Re-initialize dist for the loop as per source structure
        # However, standard transmittance sampling usually integrates 
        # along the path. The source says: "Loop similarly... update transmittance *= ..."
        # We follow the source's iterative step logic.
        
        # To match source exactly:
        # The source describes a loop that finds points and updates transmittance.
        # It does not explicitly return a distance, but a transmittance value.
        
        # Let's reconstruct the loop based on "Loop similarly to sampleDistance"
        # but updating transmittance.
        
        # Since the source doesn't specify the number of steps or termination 
        # for transmittance other than "similarly", we assume it samples 
        # until the end of the segment or a scattering event stops it?
        # Actually, transmittance is usually an integral. 
        # The source code snippet implies:
        # Loop:
        #   dist -= log(...) * invSigmaMax
        #   if dist > maxDist: break
        #   alpha = ...
        #   if r2 < alpha * invSigmaMax: break  <-- This looks like finding a scattering event
        #   transmittance *= 1.0 - alpha * invSigmaMax
        
        # This logic suggests it samples *between* scattering events or 
        # estimates transmittance by sampling opacity at random distances.
        
        # Re-reading source: "In sampleTransmittance... Loop similarly... 
        # update transmittance *= 1.0 - alphaSample * invSigmaMax"
        
        # We will implement the loop as described.
        
        dist = 0.0
        while True:
            r1 = random.random()
            dist -= math.log(1.0 - r1) * inv_sigma_max
            
            if dist > max_dist:
                break
                
            t = dist * inv_max_dist
            sampling_pos = mix(from_point, to_point, t)
            alpha_sample = sample_volume_opacity(sampling_pos)
            alpha_corr_val = alpha_sample * alpha_corr
            
            r2 = random.random()
            if r2 < alpha_corr_val * inv_sigma_max:
                break
                
            transmittance *= (1.0 - alpha_corr_val * inv_sigma_max)
            
    return transmittance

def mix(a, b, t):
    return a + (b - a) * t

def sample_volume_opacity(pos):
    # Placeholder for actual volume lookup and transfer function
    return 0.5

def sample_volume_color(pos):
    # Placeholder for color lookup
    return [1.0, 1.0, 1.0]

def sample_environment_map(direction):
    # Placeholder for environment map lookup
    return [1.0, 1.0, 1.0]

def render_pixel(pixel_pos, camera_origin, camera_direction, volume_bounds, 
                 sigma_max, alpha_corr, u_offset, u_scat_direction, frame_number):
    
    # 1. Ray Generation
    ray_from = camera_origin
    ray_to = camera_origin + camera_direction
    
    # 2. Intersection with Volume Cube
    tbounds = intersect_ray_cube(ray_from, ray_to, volume_bounds)
    
    if tbounds.x >= tbounds.y:
        return sample_environment_map(camera_direction)
    
    t_start = tbounds.x
    t_end = tbounds.y
    max_distance = t_end - t_start
    
    # 3. Seed Initialization
    seed = pixel_pos + random.uniform(u_offset, u_offset) # Simplified seed gen
    
    # 4. Distance Sampling
    dist = sample_distance(ray_from, ray_to, max_distance, sigma_max, alpha_corr, seed)
    
    if dist > max_distance:
        return sample_environment_map(camera_direction)
    
    # 5. Update Origin
    from_point = mix(ray_from, ray_to, dist / max_distance)
    
    # 6. Secondary Ray Setup
    # Source uses uScatteringDirection. In a full path tracer, this would be 
    # sampled from a phase function. Here we use the uniform parameter.
    scattering_dir = u_scat_direction
    
    # Intersect secondary ray with volume to get new segment
    # Note: The source intersects the scattering direction from the new 'from' point.
    # We assume a new ray from 'from_point' in 'scattering_dir'.
    sec_ray_from = from_point
    sec_ray_to = from_point + scattering_dir * 1000.0 # Arbitrary large distance for intersection
    
    sec_tbounds = intersect_ray_cube(sec_ray_from, sec_ray_to, volume_bounds)
    
    if sec_tbounds.x >= sec_tbounds.y:
        return sample_environment_map(scattering_dir)
        
    sec_t_start = sec_tbounds.x
    sec_t_end = sec_tbounds.y
    sec_max_distance = sec_t_end - sec_t_start
    
    # 7. Lighting and Transmittance
    diffuse_color = sample_volume_color(from_point)
    light_color = sample_environment_map(scattering_dir)
    
    # Transmittance from from_point to exit of volume along scattering_dir
    # The source calls sampleTransmittance(from, to, seed). 
    # 'from' is the scattering point. 'to' is the exit point.
    transmittance = sample_transmittance(from_point, sec_ray_to, sec_max_distance, sigma_max, alpha_corr, seed)
    
    # 8. Color Computation
    frame_color = [
        diffuse_color[0] * light_color[0] * transmittance,
        diffuse_color[1] * light_color[1] * transmittance,
        diffuse_color[2] * light_color[2] * transmittance
    ]
    
    # 9. Accumulation
    acc_color = get_accumulator(pixel_pos)
    inv_frame_number = 1.0 / frame_number
    
    final_color = [
        acc_color[0] + (frame_color[0] - acc_color[0]) * inv_frame_number,
        acc_color[1] + (frame_color[1] - acc_color[1]) * inv_frame_number,
        acc_color[2] + (frame_color[2] - acc_color[2]) * inv_frame_number
    ]
    
    set_accumulator(pixel_pos, final_color)
    
    return final_color
```

### Parameters
- `uSigmaMax`: Uniform float, maximum scattering coefficient used for distance sampling [S1, unofficial].
- `uAlphaCorrection`: Uniform float, correction factor for alpha values from transfer function [S1, unofficial].
- `uScatteringDirection`: Uniform vec3, direction for secondary ray scattering [S1, unofficial].
- `uOffset`: Uniform float, offset for random seed generation [S1, unofficial].
- `uInvFrameNumber`: Uniform float, inverse of the current frame number for exponential moving average accumulation [S1, unofficial].

### Edge Cases and Pitfalls
- **No Intersection**: If the ray does not intersect the volume cube, the method outputs the environment map color directly [S1, unofficial].
- **No Scattering**: If `sampleDistance` returns a distance greater than the segment length, the method outputs the environment map color [S1, unofficial].
- **Precision**: The use of `log(1.0 - seed.x)` requires `seed.x` to be strictly less than 1.0 to avoid `log(0)` [S1, unofficial].
- **Alpha Correction**: The `uAlphaCorrection` factor is critical for balancing the stochastic sampling against the transfer function's opacity values [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The complexity depends on the number of samples required for convergence. Each frame requires one ray traversal with stochastic distance sampling. The number of steps in `sampleDistance` and `sampleTransmittance` is probabilistic but bounded by the optical thickness of the medium.
- **Memory Complexity**: Requires storage for the volume data, environment map, and an accumulator buffer per pixel [S1, unofficial].
- **Performance**: Converges slowly due to the stochastic nature of Monte Carlo integration. Requires multiple frames to reduce noise [S1, unofficial].
- **Strengths**: Handles complex lighting and scattering without precomputation. Flexible to different phase functions and transfer functions [S1, unofficial].
- **Limitations**: High variance (noise) in early frames. Sensitive to the choice of $\sigma_{max}$ and $\alpha_{corr}$ [S1, unofficial].

## Background
The method is described in the context of the `vpt-conductor` project, which implements a volume path tracer using Monte Carlo scattering rendering techniques [S1, unofficial]. It utilizes a fragment shader-based approach for accumulation and ray traversal [S1, unofficial].

## Sources
[S1] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
