# Slice Artifact Reduction
Also called: Ray start jittering with solidity check, Wood grain artifact reduction

## Overview
Slice Artifact Reduction is a technique to mitigate "wood grain" artifacts (visible slice boundaries) in direct volume ray casting by jittering the ray start position and performing a local solidity check [S1, unofficial]. It sets itself apart from standard ray casting by adding a stochastic offset to the ray origin and a binary search-like verification step upon first voxel opacity detection, trading a small increase in sampling overhead for significant visual improvement without requiring global step-size reduction [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volume data, ray start position, ray direction (step vector), noise texture, threshold for opacity.
- Output: Final fragment color with reduced slice artifacts.

**Data Structures:**
- `Noise`: A 2D texture (e.g., 256x256) providing stochastic values.
- `stepVector`: The displacement vector per ray-casting step.
- `Threshold`: The opacity threshold for considering a voxel "solid".

**Steps:**
1. **Jitter Ray Start:** Offset the ray start position along the view direction using a value from the noise texture.
   $$ pos_{start} = rayStart + stepVector \times Noise(gl\_FragCoord.xy / 256).r $$
2. **Ray Casting Loop:** Iterate along the ray. For each position `pos`:
   a. Sample volume value $V = lookup(pos)$.
   b. If $V > Threshold$:
      i. Perform Solidity Check:
         - Move back by half-step: $s = -stepVector \times 0.5$.
         - Update position: $pos \leftarrow pos + s$.
         - Sample $V$.
         - If $V > Threshold$, move forward by quarter-step ($s \leftarrow s \times 0.5$).
         - Else, move backward by quarter-step ($s \leftarrow s \times -0.5$).
         - Update position: $pos \leftarrow pos + s$.
         - Sample $V$ again.
      ii. If the re-sampled $V > Threshold$, proceed with lighting calculations and break the loop.
   c. Advance ray: $pos \leftarrow pos + stepVector$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slice_artifact_reduction(ray_start, step_vector, noise_texture, threshold, lookup, normal_matrix, light_position, diffuse_material):
    # Jitter the ray start position
    noise_val = noise_texture.sample(ray_start.xy / 256.0).r
    pos = ray_start + step_vector * noise_val
    
    dst = vec4(0.0)
    ray_length = initial_ray_length # Assumed known context
    
    while dst.a < 1.0 and ray_length > 0:
        V = lookup(pos)
        
        if V > threshold:
            # Solidity check
            s = -step_vector * 0.5
            pos += s
            V = lookup(pos)
            
            if V > threshold:
                s *= 0.5
            else:
                s *= -0.5
            
            pos += s
            V = lookup(pos)
            
            if V > threshold:
                # Lighting calculation
                L = step_vector.length() # Or fixed step length
                E = lookup(pos + vec3(L, 0, 0))
                N_val = lookup(pos + vec3(0, L, 0))
                U = lookup(pos + vec3(0, 0, L))
                
                normal = normalize(normal_matrix * vec3(E - V, N_val - V, U - V))
                light = light_position
                df = abs(dot(normal, light))
                color = df * diffuse_material
                
                src = vec4(color, 1.0)
                dst = (1.0 - dst.a) * src + dst
                break
        
        pos += step_vector
        ray_length -= step_vector.length()
        
    return dst
```

**Parameters:**
- Noise texture size: 256x256 [S1, unofficial].
- Step size: Not given in the sources.

**Edge Cases:**
- Precision: Floating-point errors in position updates may affect solidity check accuracy [G].
- Boundaries: Ensure `lookup` handles out-of-bounds volume access gracefully [G].

## Subtopics

### [Ray Start Position Jittering](7198-ray-start-position-jittering.md)
A variant focusing solely on jittering the ray start position to reduce aliasing, without the subsequent solidity check. Choose when computational budget is extremely limited and only start-position variation is feasible.

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the number of steps per ray, with a constant factor increase due to the solidity check (up to 2 extra samples per voxel hit).
- Memory Complexity: $O(1)$ additional memory for the noise texture lookup.
- Performance: Reduces wood grain artifacts effectively without reducing the global step size, thus maintaining performance compared to naive high-resolution ray casting [S1, unofficial].
- Strengths: Simple to implement, effective visual improvement.
- Limitations: Does not address temporal flickering; may introduce slight positional jitter in the final image [S1, unofficial].
- Use when: Wood grain artifacts are visible and reducing step size is too costly.
- Do not use when: Temporal stability is critical without additional temporal filtering.

## Background
Originated in early direct volume ray casting implementations to address the visual artifact of visible slice boundaries, commonly referred to as "wood grain" [S1, unofficial]. It is a practical shader-level optimization discussed in early online tutorials and blogs on volumetric rendering [S1, unofficial].

## Sources
[S1] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
