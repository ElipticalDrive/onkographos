# Cone Marching


## Overview
Cone Marching is an optimization for distance-field ray marching that groups bundles of rays into a cone to skip repeated Signed Distance Function (SDF) calculations [S1, unofficial]. It differs from standard ray marching by performing a coarse, vertex-shader-based march of these cones to advance the ray origin and step count before the fragment shader resumes fine-grained marching [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera position ($ro$), camera ray direction ($rd$), camera field of view tangent ($u\_camTanFov$), plane subdivision count ($u\_camPlaneSubdivisions$), maximum distance ($u\_maxDis$), and maximum steps ($u\_maxSteps$).
*   **Outputs:** A `March` struct containing the total distance travelled ($dis$) and the number of steps taken ($steps$), passed from the vertex shader to the fragment shader.

**Data Structures**
*   **March Struct:**
    *   `float dis`: Total distance travelled by the cone.
    *   `int steps`: Number of steps taken during the cone march.

**Algorithm Steps**

1.  **Geometry Setup:** Create a marching plane geometry with subdivisions. The width segments are calculated as $\lfloor camera.aspect \times 32 \rfloor$ and height segments as 32 to ensure square/evenly spaced vertices and avoid artifacts [S1, unofficial].
2.  **Vertex Shader Cone March:**
    *   Compute the cone ray origin ($cro$) and direction ($crd$) in world space.
    *   Initialize total distance $d = 0$ and step counter $i = 0$.
    *   Loop while $i < u\_maxSteps$:
        *   Calculate current position: $p = cro + d \cdot crd$.
        *   Calculate scene distance: $cd = scene(p)$.
        *   Calculate current cone radius: $ccr = \frac{d \cdot u\_camTanFov \cdot 2.0}{u\_camPlaneSubdivisions}$.
        *   Check break condition: If $cd < ccr \cdot 1.25$ OR $d \ge u\_maxDis$, break the loop. The padding factor 1.25 is used to avoid artifacts at SDF edges [S1, unofficial].
        *   Advance distance: $d += cd$.
        *   Increment steps: $i += 1$.
    *   Return `March(d, i)`.
3.  **Fragment Shader Fine March:**
    *   Receive $vDisTravelled$ and $vSteps$ from the vertex shader.
    *   If $vDisTravelled \ge u\_maxDis$, return clear color.
    *   Compute ray origin $ro$ and direction $rd$ for the specific pixel.
    *   Call `rayMarch(vDisTravelled, vSteps, ro, rd)`.
    *   The `rayMarch` function initializes its loop counter at $vSteps$ and starting distance at $vDisTravelled$, continuing until hit or max distance.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic
struct March:
    dis: float
    steps: int

def cone_march(cro: vec3, crd: vec3) -> March:
    d = 0.0
    i = 0
    while i < u_maxSteps:
        p = cro + d * crd
        cd = scene(p)
        ccr = (d * u_camTanFov * 2.0) / u_camPlaneSubdivisions
        
        # Break if intersection detected (with padding) or max distance reached
        if cd < ccr * 1.25 or d >= u_maxDis:
            break
            
        d += cd
        i += 1
        
    return March(dis=d, steps=i)

def vertex_shader():
    # Compute cro and crd in world space
    # ... (transform UV to clip, unproject, transform to world, normalize) ...
    result = cone_march(cro, crd)
    vDisTravelled = result.dis
    vSteps = result.steps

# Fragment Shader Logic
def ray_march(start_dis: float, steps_taken: int, ro: vec3, rd: vec3) -> float:
    d = start_dis
    i = steps_taken
    while i < u_maxSteps:
        p = ro + d * rd
        cd = scene(p)
        
        # Standard ray marching break condition
        if cd < u_eps or d >= u_maxDis:
            break
            
        d += cd
        i += 1
    return d

def fragment_shader():
    if vDisTravelled >= u_maxDis:
        return u_clearColor
    
    ro = u_camPos
    rd = normalize(u_camToWorldMat * (u_camInvProjMat * vec4(uv * 2.0 - 1.0, 0, 1)).xyz)
    
    dis = ray_march(vDisTravelled, vSteps, ro, rd)
    # ... shading logic ...
```

**Parameters**
*   $u\_camTanFov$: Tangent of half the camera's field of view. Value: $\tan(\frac{camera.fov}{2})$ [S1, unofficial].
*   $u\_camPlaneSubdivisions$: Number of subdivisions for the screen/marching plane. Typical value: 32 [S1, unofficial].
*   Padding Factor: 1.25 (used in break condition) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Aspect Ratio Artifacts:** Using rectangular screen vertices without accounting for aspect ratio causes artifacts. Vertices must be evenly spaced/square [S1, unofficial].
*   **Padding Selection:** The break condition uses a padding factor (1.25) on the cone radius to avoid artifacts at the edges of the SDF, unlike standard ray marching which uses a small epsilon [S1, unofficial].

## Operating characteristics
*   **Complexity:** Reduces the number of SDF evaluations in the fragment shader by advancing the ray origin in the vertex shader.
*   **Performance:** Provides FPS gains by skipping repeated calculations for bundles of rays [S1, unofficial].
*   **Limitations:** The simple approach described is less complex than multi-pass cone marching but may not decrease the cone size as effectively as multi-pass methods until it equals pixel size [S1, unofficial].

## Background
Cone Marching in this context originates from an implementation in Three.js, described as a straightforward optimization for basic ray marching setups [S1, unofficial]. It is presented as an alternative to more complex multi-pass approaches that theoretically decrease cone size until it equals pixel size [S1, unofficial].

## Sources
[S1] Cone Marching in Three.js, https://medium.com/@nabilnymansour/cone-marching-in-three-js-6d54eac17ad4, fetched 2026-10-29 (unofficial)
