# Motion Blur
## Overview
Motion blur simulates the integration of light over a finite shutter interval $[t_0, t_1]$, resulting in the blurring of moving objects or camera motion [S1, unofficial][S4]. In path tracing, this is achieved by sampling a random time instance for each ray, naturally accumulating motion blur across samples [S1, unofficial]. It differs from static rendering by introducing temporal variance that must be resolved via increased sampling or specific optimization strategies [S3, S4].
## How it works
### Inputs and Outputs
**Inputs:**
*   Shutter interval $[t_0, t_1]$ [S1, unofficial].
*   Per-object velocity data or interpolated transforms to evaluate positions at time $t$ [S1, unofficial].
*   Camera pose and intrinsic parameters [S1, unofficial][S4].

**Outputs:**
*   A single blurred image frame $B$ [S4].

### Data Structures
*   **Transform Data:** Per-object transformation matrices or velocity vectors indexed by time $t$ [S1, unofficial].
*   **Sample Buffer:** Accumulates weighted radiance values from multiple temporal samples [S4].

### Steps
1.  **Sample Time:** For each ray, sample a random time $t$ uniformly within the shutter interval $[t_0, t_1]$ [S1, unofficial]:
    $$ t = \text{lerp}(t_0, t_1, u) $$
    where $u \in [0, 1)$ [S1, unofficial].
2.  **Evaluate Transforms:** Compute the object transforms (position, rotation, scale) at the sampled time $t$ using the available velocity data or interpolated keyframes [S1, unofficial].
3.  **Trace Ray:** Cast the ray through the scene using the evaluated transforms and camera parameters [S1, unofficial].
4.  **Accumulate:** Weight the resulting radiance $I_t$ by a sampling weight $w_t$ and add to the pixel buffer [S4].
5.  **Finalize:** After collecting $N$ samples, compute the final pixel value as $\hat{B} = \sum_{i=0}^{N-1} w_i I_i$, where the weights satisfy $\sum_{i=0}^{N-1} w_i = 1$ [S4]. For uniform sampling, $w_i = 1/N$ [S4]. In some implementations, $w$ is a learnable parameter jointly optimized with other scene parameters [S4].

### Parameters
*   **Shutter Interval $[t_0, t_1]$:** Defines the duration of exposure [S1, unofficial].
*   **Number of Samples $N$:** Controls temporal sampling density for motion blur simulation [S4].
*   **Intermediate Moments $M$:** In Gaussian Splatting variants, $M$ clean images are rasterized and averaged to simulate camera movement during training, with $M=6$ as an example value [S3].

### Edge Cases and Pitfalls
*   **Precision:** Floating-point precision in transform interpolation can cause jitter if not handled carefully [G].
*   **Boundaries:** Ensure $t$ is clamped within $[t_0, t_1]$ to avoid out-of-bounds access to transform data [G].
*   **Sharpness Assumption:** The method assumes images at all instants are sharp without inherent blur; blur only arises from temporal averaging of sharp frames [S3].
*   **Mode Restrictions:** Grid-based volumetric motion blur only affects Volumetric, Volumetric Geometry, and Volumetric Heat Haze Modes, and does not apply to Mesh, Ocean Mesh, Cap Mesh, or Isosurface Modes [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_motion_blur_frame(shutter_start, shutter_end, num_samples, scene, camera):
    """
    Renders a single frame with motion blur by sampling multiple time instances.
    
    Args:
        shutter_start (float): Start time t0 of the shutter interval.
        shutter_end (float): End time t1 of the shutter interval.
        num_samples (int): Number of temporal samples per pixel.
        scene (Scene): Scene object containing geometry and materials.
        camera (Camera): Camera object with pose and intrinsics.
        
    Returns:
        Image: Blurred image frame.
    """
    width, height = camera.resolution
    image = [[0.0, 0.0, 0.0] for _ in range(height)]
    
    for y in range(height):
        for x in range(width):
            pixel_color = [0.0, 0.0, 0.0]
            
            for s in range(num_samples):
                # Step 1: Sample random time within shutter interval
                u = random_float(0.0, 1.0)
                t = lerp(shutter_start, shutter_end, u)
                
                # Step 2: Evaluate object transforms at time t
                scene_transforms = evaluate_transforms(scene, t)
                
                # Step 3: Trace ray
                ray_origin, ray_direction = camera.get_ray(x, y, t)
                radiance = trace_ray(ray_origin, ray_direction, scene_transforms, scene)
                
                # Step 4: Accumulate (uniform weights w_i = 1/N)
                pixel_color[0] += radiance[0]
                pixel_color[1] += radiance[1]
                pixel_color[2] += radiance[2]
            
            # Step 5: Finalize
            image[y][x] = [pixel_color[0] / num_samples, 
                           pixel_color[1] / num_samples, 
                           pixel_color[2] / num_samples]
            
    return image

def lerp(a, b, t):
    """Linear interpolation between a and b."""
    return a + t * (b - a)

def random_float(min_val, max_val):
    """Generate a uniform random float in [min_val, max_val)."""
    import random
    return min_val + random.random() * (max_val - min_val)

def evaluate_transforms(scene, t):
    """Returns per-object transformation matrices or velocity vectors at time t."""
    # Standard operation: interpolate or fetch precomputed transforms for time t
    return scene.get_transforms_at(t)

def trace_ray(origin, direction, transforms, scene):
    """Performs standard ray-scene intersection and shading to return radiance."""
    # Standard operation: trace ray through scene geometry/materials
    return scene.intersect_and_shade(origin, direction, transforms)
```
## Subtopics

### [Motion Blur via Sub-frame Optimization](11034-motion-blur-via-sub-frame-optimization.md)
ExBluRF models motion blur in radiance fields by approximating the continuous integral of ray accumulation over the shutter open time as a finite sum of $N$ intermediate sub-frames, jointly optimizing sub-frame camera poses with the radiance field. It differs from prior works that model blur using global camera trajectories or light fields by optimizing each training view’s latent camera trajectory independently, allowing for arbitrary but smooth local motion.

### Ray-traced Volumetric Motion Blur
The default Volumetric Motion Blur method for Phoenix simulations [S2]. It uses velocity channel export to compute blur [S2]. The multiplier `mbmult` specifies a value that affects the strength of the motion blur and can be negative [S2]. The option `mbself` prevents self-intersection when shifted vertices penetrate opposite geometry, though the analysis is expensive [S2].

### Camera motion blur modeling (3DGS)
Simulates exposure-time camera shake by rasterizing multiple sharp images at different temporal instants and averaging them [S3]. It trains 3D Gaussians to model clean representations at various moments light reaches the sensor, rasterizes $M$ clean images from $M$ different moments, and averages them to simulate camera movement during training [S3]. It assumes images at all instants are sharp without inherent blur; blur only arises from temporal averaging of sharp frames [S3].

### Grid-based Volumetric Motion Blur
A motion blur method for volumetric simulations that produces more visible motion blur streaks, especially with faster moving fluids, by using a pre-pass and requiring more memory than the default Ray-traced method [S2]. It only affects Volumetric, Volumetric Geometry, and Volumetric Heat Haze Modes and does not apply to Mesh, Ocean Mesh, Cap Mesh, or Isosurface Modes [S2].

### Camera Motion Blur Simulation
Simulates motion blur by integrating sharp rendered images over the shutter time, weighted by the camera's photosensitive element efficiency [S4]. It uniformly samples $N$ timestamps within shutter time $\tau$, where the $i$-th keyframe is at timestamp $t = \frac{i-1}{N}\tau$ [S4]. The sampling weight $w_i$ satisfies $\sum_{i=0}^{N-1} w_i = 1$, and for uniform sampling, $w_i = 1/N$ [S4]. In this method, $w$ is a learnable parameter [S4].

### Temporal Opacity Visualization
To visualize temporal opacity along a Gaussian trajectory, set the alpha channel value of each trajectory segment based solely on the temporal opacity value, explicitly excluding the spatial opacity term $\sigma_{is}$ [S5]. Filter the visualization to only display moving objects in the scene [S5].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of temporal samples and $R$ is the cost of a single ray trace [S1, unofficial][S4].
*   **Memory Complexity:** $O(1)$ additional memory per pixel for accumulation, assuming transforms are evaluated on the fly [S1, unofficial]. Grid-based methods require additional memory for pre-pass data [S2].
*   **Strengths:** Natural appearance of motion blur; flexible integration with existing path tracers [S1, unofficial].
*   **Limitations:** Dependent on accurate velocity/transform data [S1, unofficial].
*   **When to Use:** When simulating realistic camera or object motion blur is required [S1, unofficial][S4].
*   **When Not to Use:** When real-time performance is critical and noise is unacceptable with low sample counts [G].
## Background
Motion blur in path tracing simulates camera shutter effects by sampling different temporal instances per ray [S1, unofficial]. It has been extended to real-time rendering techniques such as 3D Gaussian Splatting to handle camera motion blur during training [S3, S4].
## Sources
[S1] Camera Model in Path Tracing - Pinhole, Lens, and DOF Guide | Dev ..., https://code-note-vr.vercel.app/pathtracer-learning-camera-model, fetched 2026-10-29 (unofficial)
[S2] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
[S3] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S4] Deblur-GS: 3D Gaussian Splatting from Camera Motion Blurred Images, https://doi.org/10.1145/3651301, fetched 2026-11-08
[S5] Spacetime Gaussian Feature Splatting for Real-Time Dynamic View Synthesis, https://huggingface.co/papers/2312.16812, fetched 2026-10-30
