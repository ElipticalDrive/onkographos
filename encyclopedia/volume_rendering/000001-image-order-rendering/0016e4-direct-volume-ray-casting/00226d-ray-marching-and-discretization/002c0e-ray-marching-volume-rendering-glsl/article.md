# Ray Marching Volume Rendering (GLSL)
Also called: Fragment Shader Ray Marching, GLSL Ray Marching Volume Rendering

## Overview
This method implements direct volume rendering within a fragment shader by marching a ray through a 3D texture volume, applying alpha blending and early ray termination [S1, unofficial]. It differs from [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) by integrating the integration loop directly into the fragment shader for per-pixel execution, and from [Volume Raycasting on GPU](6058-volume-raycasting-on-gpu.md) by relying on a fixed step size rather than adaptive or grid-based traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position in world space, view direction (derived from fragment position), a 3D texture (`volumeTexture`) containing scalar density data, model-view-projection inverse matrix, and parameters `MAX_STEPS` and `STEP_SIZE`.
*   **Outputs:** A `vec4` fragment color with accumulated RGBA values.

**Data Structures:**
*   `volumeTexture`: A 3D texture (e.g., `sampler3D` in GLSL) where the red channel (`r`) stores density values. The texture coordinates are expected to be in the range $[0, 1]$.
*   `invModel`: The inverse of the model matrix, used to transform the camera position into the volume's local coordinate system.

**Parameters:**
*   `MAX_STEPS`: Integer, maximum number of iterations per ray. Typical value: 256 [S1, unofficial].
*   `STEP_SIZE`: Float, distance between samples along the ray. Typical value: 0.01 [S1, unofficial].

**Algorithm Steps:**
1.  **Transform Camera Position:** Transform the camera position from world space to the volume's model space using the inverse model matrix [S1, unofficial].
    $$ \mathbf{rayOrigin} = \mathbf{M}^{-1} \cdot \begin{bmatrix} \mathbf{C}_{world} \\ 1 \end{bmatrix} $$
    Where $\mathbf{C}_{world}$ is the camera position.

2.  **Compute Ray Direction:** Calculate the ray direction vector from the transformed origin to the fragment's position in model space [S1, unofficial].
    $$ \mathbf{rayDir} = \text{normalize}(\mathbf{P}_{model} - \mathbf{rayOrigin}) $$

3.  **Define Bounding Box:** Define the volume's bounding box in model space. The example uses a unit cube centered at the origin [S1, unofficial]:
    $$ \mathbf{boxMin} = (-0.5, -0.5, -0.5) $$
    $$ \mathbf{boxMax} = (0.5, 0.5, 0.5) $$

4.  **Ray-Box Intersection:** Compute the intersection interval $[t_{min}, t_{max}]$ using the slab method [S1, unofficial].
    $$ \mathbf{invDir} = \frac{1}{\mathbf{rayDir}} $$
    $$ \mathbf{t0s} = (\mathbf{boxMin} - \mathbf{rayOrigin}) \cdot \mathbf{invDir} $$
    $$ \mathbf{t1s} = (\mathbf{boxMax} - \mathbf{rayOrigin}) \cdot \mathbf{invDir} $$
    $$ \mathbf{tSmaller} = \min(\mathbf{t0s}, \mathbf{t1s}) $$
    $$ \mathbf{tLarger} = \max(\mathbf{t0s}, \mathbf{t1s}) $$
    $$ t_{min} = \max(\max(\mathbf{tSmaller}_x, \mathbf{tSmaller}_y), \max(\mathbf{tSmaller}_z, 0.0)) $$
    $$ t_{max} = \min(\min(\mathbf{tLarger}_x, \mathbf{tLarger}_y), \mathbf{tLarger}_z) $$

5.  **Check Intersection:** If $t_{min} \ge t_{max}$, the ray misses the volume. Discard the fragment [S1, unofficial].

6.  **Initialize Accumulators:** Set the current ray parameter $t = t_{min}$ and initialize the accumulated color to transparent black [S1, unofficial]:
    $$ \mathbf{accumulatedColor} = (0.0, 0.0, 0.0, 0.0) $$

7.  **Marching Loop:** Iterate up to `MAX_STEPS` times while $t < t_{max}$ [S1, unofficial]:
    a.  **Sample Position:** Compute the 3D position in model space:
        $$ \mathbf{pos} = \mathbf{rayOrigin} + t \cdot \mathbf{rayDir} $$
    b.  **Sample Density:** Sample the density from the volume texture. An offset of $+0.5$ is applied to map the model space coordinates (centered at 0) to texture coordinates (centered at 0.5) [S1, unofficial]:
        $$ \text{density} = \text{texture}(\text{volumeTexture}, \mathbf{pos} + (0.5, 0.5, 0.5)).r $$
    c.  **Map to Color:** Convert density to a color sample. The example uses a simple grayscale mapping where color equals density [S1, unofficial]:
        $$ \mathbf{colorSample} = (\text{density}, \text{density}, \text{density}, \text{density}) $$
    d.  **Front-to-Back Blending:** Accumulate the color using standard front-to-back alpha compositing [S1, unofficial]:
        $$ \mathbf{accumulatedColor}.rgb += (1.0 - \mathbf{accumulatedColor}.a) \cdot \mathbf{colorSample}.rgb \cdot \mathbf{colorSample}.a $$
        $$ \mathbf{accumulatedColor}.a += (1.0 - \mathbf{accumulatedColor}.a) \cdot \mathbf{colorSample}.a $$
    e.  **Early Termination:** If $\mathbf{accumulatedColor}.a \ge 0.95$, break the loop [S1, unofficial].
    f.  **Advance Ray:** Increment $t$ by `STEP_SIZE` [S1, unofficial]:
        $$ t = t + \text{STEP\_SIZE} $$

8.  **Final Output:** After the loop, if the final fragment alpha is negligible ($\le 0.001$), discard the fragment [S1, unofficial]. Otherwise, output `accumulatedColor`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: cameraPos (vec3), vPosition (vec3), invModel (mat4), volumeTexture (sampler3D)
# Parameters: MAX_STEPS = 256, STEP_SIZE = 0.01

# 1. Transform camera position to model space
rayOrigin = invModel * vec4(cameraPos, 1.0)
rayOrigin = rayOrigin.xyz

# 2. Compute ray direction
rayDir = normalize(vPosition - rayOrigin)

# 3. Define volume bounding box
boxMin = vec3(-0.5)
boxMax = vec3(0.5)

# 4. Compute ray-box intersection
invDir = 1.0 / rayDir
t0s = (boxMin - rayOrigin) * invDir
t1s = (boxMax - rayOrigin) * invDir
tSmaller = min(t0s, t1s)
tLarger = max(t0s, t1s)
tMin = max(max(tSmaller.x, tSmaller.y), max(tSmaller.z, 0.0))
tMax = min(min(tLarger.x, tLarger.y), tLarger.z)

# 5. Check intersection
if tMin >= tMax:
    discard

# 6. Initialize accumulators
t = tMin
accumulatedColor = vec4(0.0)

# 7. Marching loop
for i in range(MAX_STEPS):
    if t >= tMax:
        break
    
    # a. Sample position
    pos = rayOrigin + t * rayDir
    
    # b. Sample density (offset +0.5 for texture coordinates)
    density = texture(volumeTexture, pos + vec3(0.5)).r
    
    # c. Map density to color
    colorSample = vec4(density)
    
    # d. Front-to-back blending
    accumulatedColor.rgb += (1.0 - accumulatedColor.a) * colorSample.rgb * colorSample.a
    accumulatedColor.a += (1.0 - accumulatedColor.a) * colorSample.a
    
    # e. Early termination
    if accumulatedColor.a >= 0.95:
        break
    
    # f. Advance ray
    t += STEP_SIZE

# 8. Final output check
if accumulatedColor.a <= 0.001:
    discard

return accumulatedColor
```

## Subtopics

### [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md)
The compositing strategy used in the accumulation step. Choose when opacity is the primary visual cue.

### [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md)
The underlying traversal technique. Choose when uniform sampling is sufficient and adaptive steps are too complex.

### [Volume Raycasting on GPU](6058-volume-raycasting-on-gpu.md)
A broader category of GPU-based volume rendering. Choose this when comparing shader-based approaches to compute-shader or voxel-based approaches.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels (fragments) and $P$ is the number of steps per ray (bounded by `MAX_STEPS`).
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the 3D texture volume. The shader state is constant per fragment.
*   **Performance:** Performance is heavily dependent on `MAX_STEPS` and `STEP_SIZE`. A fixed step size of 0.01 with 256 steps results in a maximum path length of 2.56 units. If the volume is smaller than this, many steps are wasted. Early ray termination at 0.95 alpha can significantly reduce average step counts for opaque regions.
*   **Strengths:** Simple to implement in a fragment shader; leverages hardware texture sampling; early termination provides optimization.
*   **Limitations:** Fixed step size leads to aliasing if steps are too large or wasted computation if steps are too small; no adaptive traversal based on density gradients; limited to the resolution of the 3D texture.
*   **When to use:** Real-time rendering of simple volumes where high fidelity is not critical, or as a baseline implementation.
*   **When not to use:** High-quality offline rendering, volumes with high dynamic range density, or scenes requiring adaptive sampling.

## Background
This method is a standard implementation of direct volume rendering in real-time graphics, often found in educational resources and hobbyist projects. The specific implementation details (unit cube, offset +0.5, fixed parameters) are derived from the HNanoSolver project [S1, unofficial]. It builds on the general principles of ray casting and front-to-back compositing described in [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md).

## Sources
[S1] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
