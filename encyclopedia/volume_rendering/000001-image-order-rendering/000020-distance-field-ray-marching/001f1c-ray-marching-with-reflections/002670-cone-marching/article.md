# Cone Marching


## Overview
Cone Marching is an optimization for distance-field ray marching that groups bundles of rays into a cone to skip repeated Signed Distance Function (SDF) calculations [S1, unofficial]. It differs from standard ray marching by performing a coarse, vertex-shader-based march of these cones to advance the ray origin and step count before the fragment shader resumes fine-grained marching [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera position ($ro$), camera ray direction ($rd$), camera field of view tangent ($u\_camTanFov$), plane subdivision count ($u\_camPlaneSubdivisions$), maximum distance ($u\_maxDis$), and maximum steps ($u\_maxSteps$) [S1, unofficial].
*   **Outputs:** A `March` struct containing the total distance travelled ($dis$) and the number of steps taken ($steps$), passed from the vertex shader to the fragment shader [S1, unofficial].

**Data Structures**
*   **March Struct:**
    *   `float dis`: Total distance travelled by the cone [S1, unofficial].
    *   `int steps`: Number of steps taken during the cone march [S1, unofficial].

**Algorithm Steps**

1.  **Geometry Setup:** Create a marching plane geometry with subdivisions. The width segments are calculated as $\lfloor camera.aspect \times 32 \rfloor$ and height segments as 32 to ensure square/evenly spaced vertices and avoid artifacts [S1, unofficial].
2.  **Vertex Shader Cone March:**
    *   Compute the cone ray origin ($cro$) and direction ($crd$) in world space [S1, unofficial].
    *   Initialize total distance $d = 0$ and step counter $i = 0$ [S1, unofficial].
    *   Loop while $i < u\_maxSteps$:
        *   Calculate current position: $p = cro + d \cdot crd$ [S1, unofficial].
        *   Calculate scene distance: $cd = scene(p)$ [S1, unofficial].
        *   Calculate current cone radius: $ccr = \frac{d \cdot u\_camTanFov \cdot 2.0}{u\_camPlaneSubdivisions}$ [S1, unofficial].
        *   Check break condition: If $cd < ccr \cdot 1.25$ OR $d \ge u\_maxDis$, break the loop. The padding factor 1.25 is used to avoid artifacts at SDF edges [S1, unofficial].
        *   Advance distance: $d += cd$ [S1, unofficial].
        *   Increment steps: $i += 1$ [S1, unofficial].
    *   Return `March(d, i)` [S1, unofficial].
3.  **Fragment Shader Fine March:**
    *   Receive $vDisTravelled$ and $vSteps$ from the vertex shader [S1, unofficial].
    *   If $vDisTravelled \ge u\_maxDis$, return clear color [S1, unofficial].
    *   Compute ray origin $ro$ and direction $rd$ for the specific pixel [S1, unofficial].
    *   Call `rayMarch(vDisTravelled, vSteps, ro, rd)` [S1, unofficial].
    *   The `rayMarch` function initializes its loop counter at $vSteps$ and starting distance at $vDisTravelled$, continuing until hit or max distance [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class March:
    def __init__(self, dis, steps):
        self.dis = dis
        self.steps = steps

def cone_march(cro, crd):
    d = 0.0
    i = 0
    while i < u_maxSteps:
        p = tuple(c + d * r for c, r in zip(cro, crd))
        cd = scene(p)
        ccr = (d * u_camTanFov * 2.0) / u_camPlaneSubdivisions
        if cd < ccr * 1.25 or d >= u_maxDis:
            break
        d += cd
        i += 1
    if d >= u_maxDis:
        return March(dis=float('inf'), steps=i)
    return March(dis=d, steps=i)

def ray_march(start_dis, steps_taken, ro, rd):
    d = start_dis
    i = steps_taken
    while i < u_maxSteps:
        p = tuple(c + d * r for c, r in zip(ro, rd))
        cd = scene(p)
        if cd < u_eps or d >= u_maxDis:
            break
        d += cd
        i += 1
    if d >= u_maxDis:
        return float('inf')
    return d

def fragment_shader(uv):
    if vDisTravelled >= u_maxDis:
        return u_clearColor
    ro = u_camPos
    clip = (uv[0] * 2.0 - 1.0, uv[1] * 2.0 - 1.0, 0.0, 1.0)
    rd = unproject(clip, u_camInvProjMat)
    rd = transform_to_world(rd, u_camToWorldMat)
    rd = normalize(rd)
    dis = ray_march(vDisTravelled, vSteps, ro, rd)
    return shade(dis)

def unproject(clip, inv_proj):
    # Standard inverse projection: multiply clip by inverse projection matrix
    # and perform perspective divide
    vec = multiply_matrix_vector(inv_proj, clip)
    return (vec[0]/vec[3], vec[1]/vec[3], vec[2]/vec[3])

def transform_to_world(rd, world_mat):
    # Transform direction vector (w=0) by world matrix
    vec = multiply_matrix_vector(world_mat, (rd[0], rd[1], rd[2], 0.0))
    return (vec[0], vec[1], vec[2])

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/length, v[1]/length, v[2]/length)
```

**Parameters**
*   $u\_camTanFov$: Tangent of half the camera's field of view. Value: $\tan(\frac{camera.fov}{2})$ [S1, unofficial].
*   $u\_camPlaneSubdivisions$: Number of subdivisions for the screen/marching plane. Typical value: 32 [S1, unofficial].
*   Padding Factor: 1.25 (used in break condition) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Aspect Ratio Artifacts:** Using rectangular screen vertices without accounting for aspect ratio causes artifacts. Vertices must be evenly spaced/square [S1, unofficial].
*   **Padding Selection:** The break condition uses a padding factor (1.25) on the cone radius to avoid artifacts at the edges of the SDF, unlike standard ray marching which uses a small epsilon [S1, unofficial].

## Operating characteristics
*   **Complexity:** Reduces the number of SDF evaluations in the fragment shader by advancing the ray origin in the vertex shader [S1, unofficial].
*   **Performance:** Provides FPS gains by skipping repeated calculations for bundles of rays [S1, unofficial].
*   **Limitations:** The simple approach described is less complex than multi-pass cone marching but may not decrease the cone size as effectively as multi-pass methods until it equals pixel size [S1, unofficial].

## Background
Cone Marching in this context originates from an implementation in Three.js, described as a straightforward optimization for basic ray marching setups [S1, unofficial]. It is presented as an alternative to more complex multi-pass approaches that theoretically decrease cone size until it equals pixel size [S1, unofficial].

## Sources
[S1] Cone Marching in Three.js, https://medium.com/@nabilnymansour/cone-marching-in-three-js-6d54eac17ad4, fetched 2026-10-29 (unofficial)
