# Cone Marching

Also called: Cone Marching in patches, Two-pass Cone Marching

## Overview
Cone Marching is a GPU optimization technique for Signed Distance Field (SDF) ray marching that reduces redundant distance calculations by grouping rays into cones [S2, unofficial]. It addresses the problem where adjacent rays in a pixel bundle march similar distances, causing repeated evaluations of the same scene geometry [S2, unofficial]. Unlike standard ray marching which processes pixels independently, Cone Marching computes a coarse approximation of the intersection distance in a lower-resolution pass (or vertex shader) and feeds this starting distance to a fine per-pixel pass [S1, unofficial]. This sets it apart from sibling methods like [Ray Marching with Reflections](1963-ray-marching-with-reflections.md) by specifically optimizing the traversal phase before shading, rather than optimizing the shading or reflection logic itself.

## How it works
Cone Marching typically employs a two-pass architecture or a vertex-to-fragment data pipeline. The method relies on the geometric property that a bundle of rays from a camera center forms a cone, allowing the algorithm to skip steps where the cone does not intersect the scene [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Camera position ($ro$), camera direction/orientation, field of view (FOV), screen resolution, SDF scene function ($scene(p)$), maximum marching distance ($u\_maxDis$), and maximum steps ($u\_maxSteps$).
*   **Outputs:** Final ray distance (for shading) and hit position. In the two-pass variant, the first pass outputs starting distances and step counts; the second pass outputs the final color.

### Data Structures
The core data structure passed between passes (or from vertex to fragment shader) is a `March` struct [S2, unofficial]:
*   `dis` (float): Total distance travelled by the cone so far.
*   `steps` (int): Number of steps taken during the cone march.

### Parameters
*   $u\_camTanFov$: Tangent of half the camera's field of view. Calculated as $\tan(\frac{FOV}{2})$ [S2, unofficial].
*   $u\_camPlaneSubdivisions$: The number of subdivisions used to define the marching grid or vertex plane. A typical value is 32 [S2, unofficial].
*   $padding$: A multiplier applied to the cone radius to prevent artifacts at SDF edges. Typical value is 1.25 [S2, unofficial].

### Algorithm Steps

1.  **Setup Marching Plane:** Create a geometry representing the screen or marching plane. To avoid artifacts caused by rectangular aspect ratios, vertices must be evenly spaced (square) [S2, unofficial]. If using a plane of width 1 with 32 height subdivisions, the width subdivisions should be $\lfloor aspect \times 32 \rfloor$ [S2, unofficial].
2.  **Compute Ray Direction (World Space):** For each vertex in the marching plane:
    *   Transform UV coordinates to clip space.
    *   Unproject using the inverse projection matrix.
    *   Transform to world space using the camera world matrix.
    *   Normalize the resulting vector to get the cone ray direction ($crd$) [S2, unofficial].
3.  **Cone Marching Loop:** Iterate to find the approximate intersection distance:
    *   Initialize total distance $d = 0$ and step counter $i = 0$ [S2, unofficial].
    *   Calculate current position: $p = cro + d \cdot crd$, where $cro$ is the cone ray origin (camera position) [S2, unofficial].
    *   Calculate current scene distance: $cd = scene(p)$ [S2, unofficial].
    *   Calculate current cone radius: $ccr = \frac{d \cdot u\_camTanFov \cdot 2.0}{u\_camPlaneSubdivisions}$ [S2, unofficial].
    *   **Break Condition:** If $cd < ccr \cdot 1.25$ OR $d \ge u\_maxDis$, exit the loop [S2, unofficial]. The padding factor (1.25) is used instead of a small epsilon to check for intersection [S2, unofficial].
    *   **Advance:** If not broken, $d \leftarrow d + cd$ and $i \leftarrow i + 1$ [S2, unofficial].
4.  **Return Coarse Result:** Return the `March` struct containing $d$ and $i$ [S2, unofficial].
5.  **Fine Ray Marching (Fragment Pass):**
    *   Receive $vDisTravelled$ (distance) and $vSteps$ (steps) from the coarse pass [S2, unofficial].
    *   Initialize fine ray distance $d = vDisTravelled$ and loop counter $i = vSteps$ [S2, unofficial].
    *   Compute per-pixel ray direction $rd$ from UV [S2, unofficial].
    *   Iterate while $i < u\_maxSteps$:
        *   $p = ro + d \cdot rd$ [S2, unofficial].
        *   $cd = scene(p)$ [S2, unofficial].
        *   If $cd < \epsilon$ (standard epsilon) OR $d \ge u\_maxDis$, break [S2, unofficial].
        *   $d \leftarrow d + cd$ [S2, unofficial].
    *   Return final distance $d$ for shading [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structure
struct March:
    dis: float
    steps: int

# Uniforms
u_camTanFov: float
u_camPlaneSubdivisions: int
u_maxDis: float
u_maxSteps: int
u_eps: float

# Scene Function (External)
def scene(p: vec3) -> float:
    # Returns signed distance to nearest object
    pass

# Cone Marching Function (Vertex/Coarse Pass)
def cone_march(cro: vec3, crd: vec3) -> March:
    d = 0.0
    i = 0
    while i < u_maxSteps:
        p = cro + d * crd
        cd = scene(p)
        ccr = (d * u_camTanFov * 2.0) / u_camPlaneSubdivisions
        
        # Break if cone intersects scene (with padding) or max distance reached
        if cd < ccr * 1.25 or d >= u_maxDis:
            break
        
        d += cd
        i += 1
        
    return March(dis=d, steps=i)

# Fine Ray Marching Function (Fragment Pass)
def ray_march(start_dis: float, steps_taken: int, ro: vec3, rd: vec3) -> float:
    d = start_dis
    i = steps_taken
    
    while i < u_maxSteps:
        p = ro + d * rd
        cd = scene(p)
        
        # Standard epsilon check for precise intersection
        if cd < u_eps or d >= u_maxDis:
            break
            
        d += cd
        i += 1
        
    return d
```

### Pitfalls and Edge Cases
*   **Aspect Ratio Artifacts:** Using rectangular screen vertices without accounting for the camera aspect ratio causes uneven spacing and visual artifacts. Vertices must be adjusted to ensure square/evenly spaced distribution [S2, unofficial].
*   **Padding Factor:** The break condition uses a padding factor (1.25) on the cone radius. Using a standard epsilon here may cause artifacts at the edges of the SDF [S2, unofficial].
*   **Precision:** The cone radius calculation assumes a linear relationship between distance and radius. For pixels far from the screen center, using the central focal distance may be inaccurate; using the specific pixel's distance from the camera is more precise [S3, unofficial].

## Subtopics

### Cone Radius Calculation for Raymarching
A method to estimate the radius of the ray cone at any given distance to implement cone-marching and minimize steps [S3, unofficial]. It addresses the inaccuracy of exponential radius growth approximations by using a linear relationship: $radius(distance) = \frac{distance \cdot initialRadius}{focalDistance}$ [S3, unofficial]. This requires recalculating the radius at every step because the distance from the camera changes [S3, unofficial]. For pixels not in the center of the screen, using the distance of the specific pixel on the screen plane from the camera is more accurate than the central focal distance [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of SDF evaluations in the coarse phase by marching a single cone per patch/vertex instead of per pixel [S2, unofficial]. The fine phase still requires per-pixel marching but starts from a closer distance, reducing total steps [S2, unofficial].
*   **Memory Complexity:** Requires storing intermediate results (distance and steps) between passes or shader stages [S2, unofficial].
*   **Performance:** Reported to double FPS in simple scenes (e.g., 50 FPS to 100 FPS on a single CPU core for recursive algorithms, though GPU gains vary) [S1, unofficial]. The simple Three.js approach offers a good balance of complexity and gain, whereas multi-pass approaches (decreasing cone size until it equals pixel size) are theoretically ideal but more complex to implement [S2, unofficial].
*   **Strengths:** Significant reduction in redundant calculations for bundles of rays; easy to implement in vertex shaders or simple multi-pass setups [S2, unofficial].
*   **Limitations:** Requires additional passes or uniform data transfer; less effective if the scene has high-frequency details that cause cones to intersect early and frequently [S2, unofficial].

## Background
Cone Marching was popularized in the context of real-time SDF rendering and fractal visualization. It is described in "Cone Marching in less than 5 minutes," "Rendering Mandelbox fractals faster with Cone Marching," and "Cone Marching in Three.js" [S1, unofficial]. The technique was also highlighted in "DD2018: Sebastian Aaltonen - GPU based clay simulation and ray tracing tech in Claybook" [S1, unofficial]. The underlying principle of linear cone radius growth is discussed in community forums such as Stack Overflow [S3, unofficial].

## Sources
[S1] A Recursive Algorithm to Render Signed Distance Fields, https://pointersgonewild.com/2026-03-06-a-recursive-algorithm-to-render-signed-distance-fields/, fetched 2026-10-29 (unofficial)
[S2] Cone Marching in Three.js, https://medium.com/@nabilnymansour/cone-marching-in-three-js-6d54eac17ad4, fetched 2026-10-29 (unofficial)
[S3] How to calculate the viewing cone radius (ie. size of a pixel) at a distance in raymarching?, https://stackoverflow.com/questions/10492590/how-to-calculate-the-viewing-cone-radius-ie-size-of-a-pixel-at-a-distance-in, fetched 2026-10-26 (unofficial)
