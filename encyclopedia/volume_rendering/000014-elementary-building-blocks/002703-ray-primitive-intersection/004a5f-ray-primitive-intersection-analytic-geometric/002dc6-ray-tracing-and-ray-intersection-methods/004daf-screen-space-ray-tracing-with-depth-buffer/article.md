# Screen-Space Ray Tracing with Depth Buffer

Also called: SSRT, Single depth-buffer tracing

## Overview
Screen-Space Ray Tracing with Depth Buffer (SSRT) traces rays in screen space against a depth buffer to find surface intersections, supporting up to four depth layers per pixel [S1, unofficial]. It solves the problem of efficiently determining ray-surface intersections for effects like reflections, refractions, and ambient occlusion without traversing complex scene graphs or voxel structures [S1, unofficial]. What sets it apart from [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) and [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) is that it operates directly on the rendered depth buffer rather than a volumetric or implicit representation, allowing it to capture any geometry already present in the frame buffer [S1, unofficial]. Unlike [Ray Tracing](8334-ray-tracing.md) which traverses acceleration structures, SSRT traverses the 2D screen grid, making it highly parallelizable on GPUs but limited to visible surfaces [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `csOrig`: Camera-space ray origin (`vec3`) [S1, unofficial].
    *   `csDir`: Camera-space ray direction (`vec3`) [S1, unofficial].
    *   `proj`: Matrix projecting from camera-space position to pixel screen position (`mat4x4`) [S1, unofficial].
    *   `nearPlaneZ`: Explicit negative near plane Z value (`float`) [S1, unofficial].
    *   `csZBuffer`: The depth buffer (`sampler2D`) [S1, unofficial].
    *   `csZBufferSize`: Dimensions of the depth buffer (`vec2`) [S1, unofficial].
    *   `csZBufferIsHyperbolic`: Boolean flag indicating if the depth buffer uses hyperbolic encoding (`bool`) [S1, unofficial].
    *   `zThickness`: Camera-space Z thickness ascribed to each point in the depth buffer (`float`) [S1, unofficial].
    *   `maxDistance`: Maximum trace distance in camera-space (`float`) [S1, unofficial].
    *   `maxSteps`: Maximum number of steps to trace (`float`/`int`) [S1, unofficial].
    *   `stride`: Step size multiplier in screen space (`float`) [S1, unofficial].
    *   `jitter`: Random value in $[0,1]$ used to conceal banding (`float`) [S1, unofficial].
    *   `clipInfo`: Information used for reconstructing camera-space Z from hyperbolic depth values (`vec3`) [S1, unofficial].
*   **Outputs:**
    *   `hitPixel`: Screen-space coordinates of the hit (`out point2`) [S1, unofficial].
    *   `hitLayer`: Index of the hit layer in the depth buffer (`out int`) [S1, unofficial].
    *   `csHitPoint`: Camera-space hit point (`out point3`) [S1, unofficial].
    *   Return value: Boolean indicating if the hit pixel is within buffer bounds [S1, unofficial].

**Data Structures**
The method tracks up to four depth-buffer layers in `sceneZMax` at each pixel [S1, unofficial]. If `csZBufferIsHyperbolic` is true, the buffer contains hyperbolically encoded depth values; otherwise, it is assumed to contain precomputed negative camera-space Z values for performance [S1, unofficial].

**Algorithm Steps**

1.  **Clip to Near Plane:** Calculate the ray length within the valid range. If the ray endpoint exceeds the near plane, clip it.
    $$ \text{rayLength} = \begin{cases} \frac{\text{nearPlaneZ} - \text{csOrig}.z}{\text{csDir}.z} & \text{if } (\text{csOrig}.z + \text{csDir}.z \times \text{maxDistance}) > \text{nearPlaneZ} \\ \text{maxDistance} & \text{otherwise} \end{cases} $$
    Compute the clipped endpoint: $\text{csEndPoint} = \text{csOrig} + \text{csDir} \times \text{rayLength}$ [S1, unofficial].

2.  **Project to Screen Space:** Convert endpoints to homogeneous coordinates and then to screen space.
    $$ H_0 = \text{proj} \times \text{vec4}(\text{csOrig}, 1.0), \quad H_1 = \text{proj} \times \text{vec4}(\text{csEndPoint}, 1.0) $$
    $$ k_0 = \frac{1.0}{H_0.w}, \quad k_1 = \frac{1.0}{H_1.w} $$
    $$ P_0 = H_0.xy \times k_0, \quad P_1 = H_1.xy \times k_1 $$
    $$ Q_0 = \text{csOrig} \times k_0, \quad Q_1 = \text{csEndPoint} \times k_1 $$
    Here, $P$ represents screen-space coordinates and $Q$ represents camera-space coordinates scaled by $1/w$ [S1, unofficial].

3.  **Handle Single-Pixel Lines:** Ensure the ray has a non-zero delta to avoid division by zero.
    $$ P_1 += \text{vec2}((\text{distanceSquared}(P_0, P_1) < 0.0001) ? 0.01 : 0.0) $$
    [S1, unofficial].

4.  **Determine Traversal Direction and Derivatives:**
    Compute delta: $\Delta = P_1 - P_0$.
    If $|\Delta.x| < |\Delta.y|$, permute coordinates (swap x/y for $P_0, P_1, \Delta$) and set `permute = true` [S1, unofficial].
    Compute step direction and inverse delta x:
    $$ \text{stepDir} = \text{sign}(\Delta.x), \quad \text{invdx} = \frac{\text{stepDir}}{\Delta.x} $$
    Compute derivatives per unit step in the dominant axis:
    $$ dP = \text{vec2}(\text{stepDir}, \Delta.y \times \text{invdx}) \times \text{stride} $$
    $$ dQ = (Q_1 - Q_0) \times \text{invdx} \times \text{stride} $$
    $$ dk = (k_1 - k_0) \times \text{invdx} \times \text{stride} $$
    [S1, unofficial].

5.  **Apply Jitter:** Bump the starting position to conceal banding artifacts caused by `stride > 1`.
    $$ P_0 += dP \times \text{jitter}, \quad Q_0 += dQ \times \text{jitter}, \quad k_0 += dk \times \text{jitter} $$
    [S1, unofficial].

6.  **Initialize Loop Variables:**
    $$ \text{prevZMaxEstimate} = \text{csOrig}.z $$
    $$ Q = Q_0, \quad k = k_0, \quad \text{stepCount} = 0.0 $$
    $$ \text{end} = P_1.x \times \text{stepDir} $$
    [S1, unofficial].

7.  **Loop Over Pixels:** Iterate while the ray is within bounds and steps remain.
    Condition: $(P.x \times \text{stepDir}) \le \text{end} \land \text{stepCount} < \text{maxSteps}$
    Update: $P += dP, \quad Q.z += dQ.z, \quad k += dk, \quad \text{stepCount} += 1.0$
    [S1, unofficial].

8.  **Inside Loop - Compute Ray Z Bounds:**
    Compute ray Z bounds for the current step. The ray covers the interval $[\text{rayZMin}, \text{rayZMax}]$.
    $$ \text{rayZMin} = \text{prevZMaxEstimate} $$
    $$ \text{rayZMax} = \frac{dQ.z \times 0.5 + Q.z}{dk \times 0.5 + k} $$
    $$ \text{prevZMaxEstimate} = \text{rayZMax} $$
    If $\text{rayZMin} > \text{rayZMax}$, swap them [S1, unofficial].

9.  **Inside Loop - Fetch Depth & Check Intersection:**
    Determine hit pixel coordinates: $\text{hitPixel} = \text{permute} ? P.yx : P$ [S1, unofficial].
    Fetch depth values: $\text{sceneZMax} = \text{texelFetch}(\text{csZBuffer}, \text{int2}(\text{hitPixel}), 0)$ [S1, unofficial].
    If `csZBufferIsHyperbolic` is true, reconstruct camera-space Z for each layer:
    $$ \text{sceneZMax}[L] = \text{reconstructCSZ}(\text{sceneZMax}[L], \text{clipInfo}) \quad \text{for } L \in [0, 3] $$
    [S1, unofficial].
    Compute scene Z bounds: $\text{sceneZMin} = \text{sceneZMax} - \text{zThickness}$ [S1, unofficial].
    Check for overlap with any layer $L$:
    If $(\text{rayZMax} \ge \text{sceneZMin}[L] \land \text{rayZMin} \le \text{sceneZMax}[L]) \lor (\text{sceneZMax}[L] == 0)$, then a hit is found. Set `hitLayer = L` and break out of both loops [S1, unofficial].

10. **Post-Loop - Compute Hit Point:**
    Advance $Q.xy$ based on steps taken:
    $$ Q.xy += dQ.xy \times \text{stepCount} $$
    Compute final hit point:
    $$ \text{hitPoint} = Q \times \frac{1.0}{k} $$
    [S1, unofficial].

11. **Return Result:**
    Check if `hitPixel` is within buffer bounds:
    $$ \text{return all}(\text{lessThanEqual}(|\text{hitPixel} - (\text{csZBufferSize} \times 0.5)|, \text{csZBufferSize} \times 0.5)) $$
    [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Minimal vector types for self-contained Python execution
class Vec2:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, o): return Vec2(self.x + o.x, self.y + o.y)
    def __sub__(self, o): return Vec2(self.x - o.x, self.y - o.y)
    def __mul__(self, s): return Vec2(self.x * s, self.y * s)
    def __truediv__(self, s): return Vec2(self.x / s, self.y / s)
    def __abs__(self): return Vec2(abs(self.x), abs(self.y))
    @property
    def yx(self): return Vec2(self.y, self.x)
    @property
    def xy(self): return Vec2(self.x, self.y)

class Vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return Vec3(self.x + o.x, self.y + o.y, self.z + o.z)
    def __sub__(self, o): return Vec3(self.x - o.x, self.y - o.y, self.z - o.z)
    def __mul__(self, s): return Vec3(self.x * s, self.y * s, self.z * s)
    def __truediv__(self, s): return Vec3(self.x / s, self.y / s, self.z / s)
    @property
    def xy(self): return Vec2(self.x, self.y)
    @xy.setter
    def xy(self, v): self.x, self.y = v.x, v.y

class Vec4:
    def __init__(self, x, y, z, w): self.x, self.y, self.z, self.w = x, y, z, w
    def __add__(self, o): return Vec4(self.x + o.x, self.y + o.y, self.z + o.z, self.w + o.w)
    def __sub__(self, o): return Vec4(self.x - o.x, self.y - o.y, self.z - o.z, self.w - o.w)
    def __mul__(self, s): return Vec4(self.x * s, self.y * s, self.z * s, self.w * s)

def trace_screen_space_ray(csOrig, csDir, proj, csZBuffer, csZBufferSize, zThickness, csZBufferIsHyperbolic, clipInfo, nearPlaneZ, stride, jitter, maxSteps, maxDistance):
    def distance_squared(a, b): return (a.x - b.x)**2 + (a.y - b.y)**2
    def int2(v): return (int(v.x), int(v.y))
    def swap(a, b): return b, a
    def texel_fetch(tex, coord, level): return [0.0, 0.0, 0.0, 0.0] # Placeholder for texture fetch
    def reconstruct_cs_z(depth, clipInfo):
        # Formula for reconstructing camera-space Z from hyperbolic depth not provided in sources [G]
        return depth

    # 1. Clip to near plane
    if (csOrig.z + csDir.z * maxDistance) > nearPlaneZ:
        rayLength = (nearPlaneZ - csOrig.z) / csDir.z
    else:
        rayLength = maxDistance
    csEndPoint = csOrig + csDir * rayLength

    # 2. Project to screen space
    H0 = proj * Vec4(csOrig.x, csOrig.y, csOrig.z, 1.0)
    H1 = proj * Vec4(csEndPoint.x, csEndPoint.y, csEndPoint.z, 1.0)
    k0 = 1.0 / H0.w
    k1 = 1.0 / H1.w
    Q0 = Vec3(csOrig.x * k0, csOrig.y * k0, csOrig.z * k0)
    Q1 = Vec3(csEndPoint.x * k1, csEndPoint.y * k1, csEndPoint.z * k1)
    P0 = Vec2(H0.x * k0, H0.y * k0)
    P1 = Vec2(H1.x * k1, H1.y * k1)

    # 3. Handle single-pixel lines
    if distance_squared(P0, P1) < 0.0001:
        P1 = P1 + Vec2(0.01)

    delta = P1 - P0
    permute = False
    if abs(delta.x) < abs(delta.y):
        permute = True
        delta = delta.yx
        P0 = P0.yx
        P1 = P1.yx

    stepDir = 1.0 if delta.x > 0 else (-1.0 if delta.x < 0 else 0.0)
    invdx = stepDir / delta.x
    dP = Vec2(stepDir, delta.y * invdx) * stride
    dQ = (Q1 - Q0) * invdx * stride
    dk = (k1 - k0) * invdx * stride

    # 4. Apply jitter
    P0 = P0 + dP * jitter
    Q0 = Q0 + dQ * jitter
    k0 = k0 + dk * jitter

    # 5. Initialize loop variables
    prevZMaxEstimate = csOrig.z
    Q = Q0
    k = k0
    stepCount = 0.0
    end = P1.x * stepDir

    hitPixel = Vec2(-1, -1)
    hitLayer = -1
    csHitPoint = Vec3(0, 0, 0)

    # 6. Loop over pixels
    P = P0
    while (P.x * stepDir) <= end and stepCount < maxSteps:
        # Compute ray Z bounds
        rayZMin = prevZMaxEstimate
        rayZMax = (dQ.z * 0.5 + Q.z) / (dk * 0.5 + k)
        prevZMaxEstimate = rayZMax
        if rayZMin > rayZMax:
            rayZMin, rayZMax = swap(rayZMin, rayZMax)

        # Determine hit pixel
        hitPixel = P.yx if permute else P

        # Fetch depth buffer
        sceneZMax = texel_fetch(csZBuffer, int2(hitPixel), 0)
        if csZBufferIsHyperbolic:
            for layer in range(4):
                sceneZMax[layer] = reconstruct_cs_z(sceneZMax[layer], clipInfo)

        # Check intersection
        sceneZMin = [sceneZMax[i] - zThickness for i in range(4)]
        hit_found = False
        for L in range(4):
            if (rayZMax >= sceneZMin[L] and rayZMin <= sceneZMax[L]) or (sceneZMax[L] == 0):
                hitLayer = L
                hit_found = True
                break
        if hit_found:
            break

        # Update loop variables
        P = P + dP
        Q.z = Q.z + dQ.z
        k = k + dk
        stepCount = stepCount + 1.0

    # 7. Post-Loop - Compute Hit Point
    Q.xy = Q.xy + dQ.xy * stepCount
    csHitPoint = Vec3(Q.x * (1.0 / k), Q.y * (1.0 / k), Q.z * (1.0 / k))

    # 8. Return Result
    bounds_check = all(abs(hitPixel.x - (csZBufferSize.x * 0.5)) <= (csZBufferSize.x * 0.5)) and all(abs(hitPixel.y - (csZBufferSize.y * 0.5)) <= (csZBufferSize.y * 0.5))
    return bounds_check, hitPixel, hitLayer, csHitPoint
```

**Edge Cases and Pitfalls**
*   **Division by Zero:** Avoided because the ray is clipped to the near plane (no $w=0$ singularity) and `delta.x` is guaranteed non-zero for single-pixel lines [S1, unofficial].
*   **Coordinate System:** The method assumes pixel $(0,0)$ is the upper-left corner. To switch to lower-left, modify the final hit point by `hitPoint.y = csZBufferSize.y - hitPoint.y` [S1, unofficial].
*   **Performance:** Converting hyperbolic depth values per iteration is expensive. It is preferred to use a floating-point buffer of precomputed negative camera-space values [S1, unofficial].
*   **Miss Handling:** Rays that miss the depth buffer (travel farther than `maxDistance` or towards the camera) require a fallback strategy, such as sparse voxels or environment cube maps [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity**
The time complexity is linear with respect to the number of iterations before a hit is encountered or the `maxDistance` is reached [S1, unofficial]. The method tracks up to four depth-buffer layers in `sceneZMax` at each pixel [S1, unofficial].

**Measured Performance**
At 1920x1080 resolution, casting a diagonal ray for 25 iterations per pixel costs 1.2 ms on an NVIDIA GeForce Titan and 6.4 ms on a MacBook NVIDIA GeForce 650M [S1, unofficial]. Ambient occlusion and radiosity examples, which cast 25 rays per pixel, spend about 30 ms on ray casts, making them not viable for real-time pipelines with other tasks [S1, unofficial].

**Scaling**
Performance scales linearly with resolution and the number of rays per pixel [S1, unofficial]. Using `stride > 1` reduces the number of samples, improving performance at the cost of quality [S1, unofficial].

**Strengths and Limitations**
*   **Strengths:** Efficient for single-ray-per-pixel effects like reflections and refractions. Captures any geometry present in the depth buffer without additional scene traversal [S1, unofficial].
*   **Limitations:** Limited to visible surfaces. High ray counts (e.g., for global illumination) are too slow for real-time. Stride > 1 introduces banding artifacts [S1, unofficial].

**When to Use**
Use for reflection and refraction (one ray cast per pixel) where real-time performance is critical. Not recommended for ambient occlusion or radiosity if other graphics tasks are present [S1, unofficial].

## Background
**Origin**
The method was introduced in the paper "Efficient GPU screen-space ray tracing" [S1, unofficial]. It is a feature of the G3D Innovation Engine [S1, unofficial].

**History and Development**
The technique has been used in commercial games such as *Killzone: Shadow Fall* for reflections and volumetrics [S1, unofficial]. It was presented at SIGGRAPH Advances in Real-Time Rendering in Games course [S1, unofficial].

**Usage**
It is used for reflection and refraction (one ray cast per pixel) and ambient occlusion/radiosity (25 rays per pixel) [S1, unofficial].

## Sources

[S1] Efficient GPU screen-space ray tracing, https://jcgt.org/published/0003/04/04/paper.pdf, fetched 2026-10-25 (unofficial)
