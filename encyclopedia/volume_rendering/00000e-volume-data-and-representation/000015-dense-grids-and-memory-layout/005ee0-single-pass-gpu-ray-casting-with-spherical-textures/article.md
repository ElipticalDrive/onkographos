# Single-Pass GPU Ray Casting with Spherical Textures

Also called: GPU Ray Casting (Single Pass) with Spherical 3D Textures, Single-pass GPU ray marching

## Overview
This method renders a 3D volume dataset stored in spherical coordinates by mapping the data to a 3D texture and performing ray marching entirely within a fragment shader [S1, unofficial]. It differs from standard Cartesian ray casting by requiring coordinate transformation for ray direction computation and texture sampling, addressing the geometric distortion inherent in storing spherical data in a cubic texture lattice [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D texture containing intensity values in spherical coordinates $(r, \theta, \phi)$.
*   **Output:** A 2D image where each pixel represents the alpha-composited color of the ray passing through the volume.

**Data Structures**
The volume is stored in a 3D texture with dimensions corresponding to the spherical grid resolution.
*   **Dimensions:** $(D_r, D_\theta, D_\phi) = (384, 15, 768)$ [S1, unofficial].
*   **Format:** `GL_R16F` (16-bit float, red channel only) with `GL_RED` format and `GL_FLOAT` type [S1, unofficial].
*   **Coordinate Mapping:** The texture coordinates $(u, v, w)$ map to the normalized spherical ranges. Note that the source implementation computes normalized spherical coordinates but samples using Cartesian texture coordinates [S1, unofficial].

**Parameters**
*   `MAX_SAMPLES`: Integer constant, typically 200–300 [S1, unofficial].
*   `step_size`: `vec3` uniform controlling the distance advanced per step [S1, unofficial].
*   `camPos`: `vec3` uniform for camera position in object space [S1, unofficial].
*   `texMin`: `vec3(0.0)` [S1, unofficial].
*   `texMax`: `vec3(1.0)` [S1, unofficial].

**Algorithm Steps**
1.  **Vertex Shader:** Compute clip-space position. Pass 3D texture coordinates `vUV` to the fragment shader by offsetting unit cube vertex positions from $[-0.5, 0.5]^3$ to $[0, 1]^3$ [S1, unofficial].
    $$ vUV = vVertex + vec3(0.5) $$
2.  **Fragment Shader Initialization:** Set `dataPos = vUV`. Compute the ray direction in spherical space by converting the current vertex position and camera position to spherical coordinates, subtracting them, and normalizing [S1, unofficial].
3.  **Ray Marching Loop:** Iterate up to `MAX_SAMPLES`:
    a.  Advance `dataPos` by `dirStep` (computed as normalized spherical direction scaled by `step_size`) [S1, unofficial].
    b.  **Boundary Check:** Evaluate if `dataPos` has left the $[0,1]^3$ bounds. The source uses a dot product of sign vectors:
        $$ stop = \text{dot}(\text{sign}(dataPos - texMin), \text{sign}(texMax - dataPos)) < 3.0 $$
        If `stop` is true, break the loop [S1, unofficial].
    c.  **Sampling:** Fetch intensity $I$ from the red channel of the volume texture at `dataPos` [S1, unofficial].
    d.  **Transfer Function:** Compute opacity $\alpha$ and color $C$ from $I$:
        $$ \alpha = \frac{e^{I} - 1}{e^{1} - 1} $$
        $$ C = I \cdot vec3(100.0, 20.0, 10.0) $$
        [S1, unofficial].
    e.  **Compositing:** Update accumulated color and alpha using front-to-back blending:
        $$ C'_{new} = C'_{old} + (1 - \alpha'_{old}) \alpha C $$
        $$ \alpha'_{new} = \alpha'_{old} + (1 - \alpha'_{old}) \alpha $$
        [S1, unofficial].
    f.  **Early Ray Termination:** If $\alpha'_{new} > 0.99$, break the loop [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(vVertex, MVP):
    # Offset unit cube [-0.5, 0.5]^3 to [0, 1]^3
    vUV = vVertex + vec3(0.5)
    gl_Position = MVP * vec4(vVertex, 1.0)
    return vUV

# Fragment Shader
def fragment_shader(vUV, volume_texture, camPos, step_size):
    # Constants
    MAX_SAMPLES = 200
    texMin = vec3(0.0)
    texMax = vec3(1.0)
    
    # Initial state
    dataPos = vUV
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    # Compute ray direction in spherical coordinates
    # Convert vertex position to spherical
    vertex_cart = vUV - vec3(0.5)
    vertex_sph = cart2sph(vertex_cart)
    
    # Convert camera position to spherical
    cam_sph = cart2sph(camPos)
    
    # Direction in spherical space
    dir_sph = normalize(vertex_sph - cam_sph)
    dirStep = dir_sph * step_size
    
    for i in range(MAX_SAMPLES):
        # Advance ray
        dataPos = dataPos + dirStep
        
        # Boundary check
        # Note: This check occurs after advancement, potentially allowing out-of-bounds fetch
        if dot(sign(dataPos - texMin), sign(texMax - dataPos)) < 3.0:
            break
            
        # Sample volume
        # Note: Source samples using Cartesian dataPos despite spherical context
        intensity = texture(volume_texture, dataPos).r
        
        # Transfer function
        alpha = (exp(intensity) - 1.0) / (exp(1.0) - 1.0)
        color = intensity * vec3(100.0, 20.0, 10.0)
        
        # Front-to-back compositing
        accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * alpha * color
        accumulated_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha
        
        # Early ray termination
        if accumulated_alpha > 0.99:
            break
            
    return vec4(accumulated_color, accumulated_alpha)

def cart2sph(cart):
    # Cartesian to Spherical conversion
    # r = sqrt(x^2 + y^2 + z^2)
    # phi = atan(y, x)
    # theta = atan(sqrt(x^2 + y^2), z)
    r = length(cart)
    phi = atan(cart.y, cart.x)
    theta = atan(length(cart.xy), cart.z)
    return vec3(r, theta, phi)
```

**Edge Cases and Pitfalls**
*   **Coordinate Mismatch:** The source implementation computes spherical coordinates (`spPos`) but samples the texture using the original Cartesian `dataPos`, ignoring the transformation for actual lookups [S1, unofficial]. This leads to incorrect visualization (e.g., rendering a disk instead of the expected shape) [S1, unofficial].
*   **Boundary Check Timing:** The boundary check `dot(sign(dataPos-texMin),sign(texMax-dataPos)) < 3.0` is evaluated *after* advancing `dataPos`. This means out-of-bounds texture fetches may occur before the break condition triggers, potentially causing undefined behavior or artifacts [S1, unofficial].
*   **Spherical Distortion:** Storing spherical data in a cubic texture lattice causes significant distortion near the poles and center, which is not corrected by the standard ray marching loop [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is `MAX_SAMPLES` (typically 200–300). Early ray termination reduces the average number of samples per ray.
*   **Memory Complexity:** $O(D_r \cdot D_\theta \cdot D_\phi)$ for the 3D texture. For the given dimensions $(384, 15, 768)$, this requires significant VRAM if high precision is used, though `GL_R16F` mitigates this.
*   **Strengths:** Single-pass execution allows for efficient rendering on modern GPUs. Hardware-accelerated texture sampling avoids manual interpolation.
*   **Limitations:** The method is sensitive to the coordinate system mismatch between the spherical data layout and the cubic texture sampling. Performance degrades with high `MAX_SAMPLES` or low early termination rates.
*   **When to Use:** Suitable for real-time visualization of spherical volume data (e.g., atmospheric models, spherical objects) where the distortion is acceptable or corrected. Not suitable for general Cartesian volumes without modification.

## Background
This method originates from a specific implementation discussed in an online forum, where the author attempted to render a 3D dataset stored in spherical coordinates using a single-pass GPU ray casting technique [S1, unofficial]. The approach adapts standard direct volume ray casting principles to handle spherical coordinate systems by transforming ray directions and positions into spherical space, although the implementation noted contains a critical flaw in texture sampling coordinates [S1, unofficial].

## Sources
[S1] GPU ray casting (single pass) with 3d textures in spherical coordinates, https://stackoverflow.com/questions/55528378/gpu-ray-casting-single-pass-with-3d-textures-in-spherical-coordinates, fetched 2026-10-26 (unofficial)
