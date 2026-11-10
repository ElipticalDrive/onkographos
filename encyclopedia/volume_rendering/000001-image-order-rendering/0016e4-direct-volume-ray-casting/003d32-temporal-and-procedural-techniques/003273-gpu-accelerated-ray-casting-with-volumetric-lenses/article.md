# GPU-accelerated ray casting with volumetric lenses

Also called: Magic Lens Implementation, Ray casting with lens distortion, Volumetric lens ray casting

## Overview
This method implements interactive focus+context volume rendering by applying spatially varying ray distortions (lenses) to standard direct volume ray casting [S1]. It solves the problem of visualizing specific regions of interest within large volumetric datasets without occlusion, using GPU fragment programs to compute non-perspective ray trajectories [S1]. Unlike standard ray casting which assumes a fixed camera projection, this approach dynamically modifies ray directions or origins based on lens parameters (magnification, position, size) to create magnifier, angular, or feature-based effects [S1].

## How it works
The method extends the core ray casting loop by replacing the standard perspective projection with custom ray generation logic. It requires a GPU supporting loops, early termination, and branching [S1].

**Inputs:**
*   3D texture containing volume data (density, gradient, color) [S1].
*   Lens parameters: `lens_active` (boolean), `lens_position` (vec2, normalized 0.0–1.0), `lens_size` (vec2, normalized), `lens_magnification` (float) [S2, unofficial].
*   Volume bounding box geometry [S1].

**Outputs:**
*   Final rendered image with lens effects applied [S1].

**Data Structures:**
*   Shader uniforms for lens state [S2, unofficial].
*   Fragment shader outputs for intermediate passes (ray direction, t-front, t-back) [S1].

**Algorithm Steps:**
The rendering is performed in a multi-pass fragment program structure. For magnifier and angular lenses, four passes are used [S1]. For feature-based lenses, an additional pass is inserted; for sampling-rate-based lenses, the first pass is replaced [S1].

1.  **Pass 1: RayDirection**
    Calculate the ray direction for each fragment based on the viewport and lens parameters. Determine if the ray passes through the lens, hits a feature of interest, or calculate the distance to the lens center to enable specific rendering effects [S1].
    *   *Note:* Standard OpenGL front/back face drawing cannot be used because ray directions are not always orthogonal or perspective [S1].

2.  **Pass 1+ (Feature-based only): RayLensBorder**
    Compute the intersection of the ray with the lens boundary [S1].

3.  **Pass 1* (Sampling-rate-based only): RayStartPoints**
    Calculate the starting points for rays, replacing the standard RayDirection pass [S1].

4.  **Pass 2: RayTfrontback**
    Compute the intersection points of each ray with the bounding box of the volumetric object. Store the distances from the ray start point to the front and back intersection points as $t_{front}$ and $t_{back}$ [S1]. These values, combined with the ray direction and viewport parameters, define the intersection segment [S1].

5.  **Pass 3: RayCasting**
    Implement the ray traversal algorithm for a given step size. Access volume data (density, gradient, or color) from 3D textures. Use early termination when transmittance is low or opacity is high [S1, S1 (parent)].

6.  **Pass 4: Rendering**
    Composite the accumulated color and opacity into the final image [S1].

**Pseudocode:**
The following pseudocode illustrates the core logic for a magnifier lens, adapted from Cg fragment program concepts [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms
u_lens_active: bool
u_lens_position: vec2  # Normalized screen coords [0,1]
u_lens_size: vec2      # Normalized size [0,1]
u_lens_magnification: float

# Fragment Shader Main Entry
void main() {
    vec2 fragCoord = gl_FragCoord.xy / resolution;
    vec3 rayDir;
    vec3 rayOrigin;
    
    # Pass 1: RayDirection
    if (u_lens_active) {
        # Check if fragment is within lens area
        vec2 center = u_lens_position;
        vec2 halfSize = u_lens_size / 2.0;
        vec2 distFromCenter = fragCoord - center;
        
        if (abs(distFromCenter.x) < halfSize.x && abs(distFromCenter.y) < halfSize.y) {
            # Inside lens: Apply magnification
            # Map screen coords to lens-local coords, scale, then map back
            vec2 localCoord = distFromCenter / halfSize; # [-1, 1]
            vec2 magnifiedLocal = localCoord * u_lens_magnification;
            vec2 magnifiedScreen = magnifiedLocal * halfSize + center;
            
            # Calculate ray direction based on magnified screen position
            # Assuming standard perspective projection for base ray
            rayDir = calculateRayDirection(magnifiedScreen);
        } else {
            # Outside lens: Standard ray direction
            rayDir = calculateRayDirection(fragCoord);
        }
    } else {
        rayDir = calculateRayDirection(fragCoord);
    }
    
    # Pass 2: RayTfrontback
    # Compute intersection with volume bounding box
    # Returns t_front and t_back distances from ray origin
    float t_front, t_back;
    computeBoxIntersection(rayOrigin, rayDir, t_front, t_back);
    
    # Pass 3: RayCasting
    float t = t_front;
    vec3 color = vec3(0.0);
    float alpha = 0.0;
    
    # Step size determined by resolution and volume size
    float stepSize = (t_back - t_front) / max_steps;
    
    while (t < t_back && alpha < 1.0) {
        vec3 samplePos = rayOrigin + t * rayDir;
        float density = sample3DTexture(volumeTex, samplePos);
        
        # Transfer function and compositing
        vec3 sampleColor = applyTransferFunction(density);
        float sampleAlpha = computeAlpha(density);
        
        # Front-to-back compositing
        color += sampleColor * sampleAlpha * (1.0 - alpha);
        alpha += sampleAlpha * (1.0 - alpha);
        
        t += stepSize;
    }
    
    # Pass 4: Rendering
    gl_FragColor = vec4(color, alpha);
}
```

**Edge Cases and Pitfalls:**
*   **Non-Orthogonal Rays:** Standard OpenGL front/back face drawing techniques fail because ray directions are not always orthogonal or perspective. Custom intersection calculations are required [S1].
*   **GPU Features:** Requires support for loops, early termination, and branching in fragment shaders [S1].
*   **Performance:** Large datasets may require downsampling or adjusted `max_steps` to maintain interactivity [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of pixels. The multi-pass approach adds constant overhead per frame.
*   **Memory Complexity:** Dominated by the 3D texture storage of the volume data [S1].
*   **Performance:** Heavily dependent on GPU capabilities (loops, branches) [S1]. Large volumes may require downsampling for smoother performance on less powerful systems [S2, unofficial].
*   **Strengths:** Interactive focus+context visualization; no pre-computation for lens effects; infinite resolution for procedural components if combined with procedural techniques [S1, S1 (parent)].
*   **Limitations:** Complex ray intersection logic; requires custom shader implementation for each lens type [S1].

## Background
The technique was introduced as the "Magic Volume Lens," an interactive focus+context technique for volume rendering [S1]. It was developed to address the occlusion problems in large volumetric datasets by allowing users to inspect specific regions via magnification or angular distortion [S1]. The implementation relies on GPU fragment programs, leveraging features available in GPUs such as the NVIDIA GeForce 6800 at the time of publication [S1].

## Sources
[S1] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
