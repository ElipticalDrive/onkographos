# Isotropic Gaussian Point-Sprite Rendering

Also called: Gaussian-opacity point sprites, Soft Gaussian primitives

## Overview
A rendering technique using isotropic 3D point primitives with a Gaussian alpha falloff in screen space, rendered via custom GLSL shaders [S1, unofficial]. It demonstrates soft Gaussian opacity and transparency but does not implement full 3D Gaussian Splatting (no anisotropic covariance, no spherical harmonics, no GPU tile-based rasterization) [S1, unofficial]. It sets itself apart from its sibling [WebGL Gaussian Splatting](2084-webgl-gaussian-splatting.md) by explicitly implementing a CPU-side back-to-front sort to correct the arbitrary alpha blending order inherent in `THREE.Points`, whereas the parent article describes the general bounding quad approach without detailing the specific sorting mechanism for isotropic sprites.

## How it works
**Inputs and Outputs:**
*   **Input:** A `THREE.Points` object containing attributes: `position` (vec3), `aColor` (vec3), and `aRadius` (float).
*   **Uniforms:** `uScale` (float), `uPixelRatio` (float), `uPointMode` (float, 0.0 for Gaussian, 1.0 for hard-edged), `uOpacity` (float).
*   **Output:** Screen-space pixels with color and alpha, composited using standard source-over blending.

**Data Structures:**
*   `THREE.BufferGeometry` with `Float32Array` attributes for positions, colors, and radii.
*   CPU-side index array for sorting.

**Numbered Steps:**

1.  **Vertex Shader Execution:**
    For each point primitive:
    a.  Compute the model-view position: `mv = modelViewMatrix * vec4(position, 1.0)`.
    b.  Calculate screen-space radius in pixels:
        $$ \text{radiusPixels} = \text{aRadius} \times \text{uScale} \times 220.0 \times \text{uPixelRatio} / \max(1.0, -\text{mv.z}) $$
    c.  Set `gl_PointSize` by clamping the radius scaled by the point mode factor:
        $$ \text{gl\_PointSize} = \text{clamp}(\text{radiusPixels} \times \text{mix}(1.0, 0.22, \text{uPointMode}), 1.0, 150.0 \times \text{uPixelRatio}) $$
    d.  Pass color to fragment shader: `vColor = aColor`.

2.  **Fragment Shader Execution:**
    For each fragment covered by the point sprite:
    a.  Compute normalized sprite coordinates: `p = gl_PointCoord * 2.0 - 1.0`.
    b.  Calculate squared distance from center: `r2 = dot(p, p)`.
    c.  Discard fragment if `r2 > 1.0`.
    d.  Compute alpha:
        $$ \alpha = \text{mix}(\exp(-4.5 \times r^2), 1.0, \text{uPointMode}) \times \text{uOpacity} $$
        Where $r = \sqrt{r2}$.
    e.  Discard fragment if $\alpha < 0.008$.
    f.  Output `gl_FragColor = vec4(vColor, alpha)`.

3.  **CPU-Side Back-to-Front Sorting:**
    To ensure correct transparency compositing:
    a.  Trigger sort when camera changes (`dirtySort = true`).
    b.  Compute camera depth for each point $i$ using the inverse world matrix $M$:
        $$ z_i = M_{2}x_i + M_{6}y_i + M_{10}z_i + M_{14} $$
        Where $M$ is the 16-element array of `camera.matrixWorldInverse.elements`, and $(x_i, y_i, z_i)$ are the world-space coordinates of point $i$.
    c.  Sort indices by $z_i$ ascending (more negative is farther).
    d.  Reorder `position`, `aColor`, and `aRadius` arrays in the `THREE.BufferGeometry` attributes according to the sorted indices.
    e.  Set `needsUpdate = true` on the attributes.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (GLSL)
def vertex_shader(position, aColor, aRadius, modelViewMatrix, uScale, uPixelRatio, uPointMode):
    mv = modelViewMatrix * vec4(position, 1.0)
    vColor = aColor
    gl_Position = projectionMatrix * mv
    
    # Calculate radius in pixels
    # Note: -mv.z is depth in view space. max(1.0, ...) prevents division by zero/negative depth issues.
    radius_pixels = aRadius * uScale * 220.0 * uPixelRatio / max(1.0, -mv.z)
    
    # Scale radius based on mode (0.22 for Gaussian, 1.0 for hard-edged)
    scale_factor = mix(1.0, 0.22, uPointMode)
    gl_PointSize = clamp(radius_pixels * scale_factor, 1.0, 150.0 * uPixelRatio)
    
    return vColor

# Fragment Shader Logic (GLSL)
def fragment_shader(vColor, uOpacity, uPointMode):
    # gl_PointCoord is in [0, 1]
    p = gl_PointCoord * 2.0 - 1.0
    r2 = dot(p, p)
    
    if r2 > 1.0:
        discard()
        
    # Gaussian falloff: exp(-4.5 * r^2)
    # If uPointMode is 1.0, alpha becomes 1.0 (hard edge)
    gaussian_alpha = exp(-4.5 * r2)
    alpha = mix(gaussian_alpha, 1.0, uPointMode) * uOpacity
    
    if alpha < 0.008:
        discard()
        
    gl_FragColor = vec4(vColor, alpha)

# CPU Sorting Logic (JavaScript/Python-like)
def sort_back_front(camera, geometry_attributes):
    # Get inverse world matrix elements
    m = camera.matrixWorldInverse.elements
    positions = geometry_attributes['position'].array
    count = geometry_attributes['position'].count
    
    # Create index array
    indices = list(range(count))
    
    # Sort indices by depth
    # Depth formula: z = m[2]*x + m[6]*y + m[10]*z + m[14]
    def get_depth(idx):
        i3 = idx * 3
        return m[2] * positions[i3] + m[6] * positions[i3+1] + m[10] * positions[i3+2] + m[14]
    
    indices.sort(key=get_depth)
    
    # Reorder attributes
    for attr_name in ['position', 'aColor', 'aRadius']:
        attr = geometry_attributes[attr_name]
        old_data = attr.array
        new_data = [old_data[i*3:(i+1)*3] if attr_name == 'position' else 
                    old_data[i*3:(i+1)*3] if attr_name == 'aColor' else 
                    [old_data[i]] for i in indices]
        
        # Flatten if necessary (simplified for pseudocode)
        # In JS: new Float32Array([...])
        attr.array = new_data
        attr.needsUpdate = True
```

**Parameters:**
*   `uOpacity`: Peak opacity multiplier. Typical value: 1.0.
*   `uPointMode`: Switch between Gaussian (`0.0`) and hard-edged (`1.0`).
*   `uScale`: Global scale factor for point sizes.
*   `uPixelRatio`: Device pixel ratio (e.g., 1.0 or 2.0).
*   Sorting Cadence: Performed approximately every 150 ms while the camera is moving [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Alpha Blending Order:** Three.js does not automatically sort individual points inside a `THREE.Points` object. Without sorting, alpha blending order is arbitrary, leading to incorrect transparency [S1, unofficial].
*   **Depth Calculation:** The depth $z_i$ is computed in view space using the inverse world matrix. More negative values indicate points farther from the camera [S1, unofficial].
*   **Precision:** The alpha discard threshold is `0.008`. Values below this are discarded to optimize rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ where $N$ is the number of points. This is performed on the CPU, causing overhead.
*   **Memory Complexity:** $O(N)$ for storing positions, colors, radii, and the index array.
*   **Measured Performance:** The CPU sort introduces latency, especially with large $N$. The approximate update cadence (150 ms) mitigates this but results in visual jitter if the camera moves rapidly [S1, unofficial].
*   **Strengths:** Simple implementation using standard WebGL point sprites; visually demonstrates Gaussian opacity; works in browsers via Three.js.
*   **Limitations:** No anisotropic covariance (sprites are always circular); no spherical harmonics (view-independent color); no GPU-based tile rasterization; CPU sorting bottleneck [S1, unofficial].
*   **When to use:** Educational demos, simple browser-based visualizations where high performance is not critical.
*   **When not to use:** Production 3DGS rendering, scenes with millions of points, or when anisotropic splats are required.

## Background
Originating from an interactive browser-based experiment exploring Gaussian point-sprite rendering with Three.js and custom GLSL shaders [S1, unofficial]. Built as an educational graphics demo to understand the basic visual effect of soft Gaussian primitives and transparency, not as a full 3D Gaussian Splatting (3DGS) implementation [S1, unofficial]. It references Kerbl et al. for the underlying 3DGS concept but explicitly states it does not reconstruct scenes from photos or implement production 3DGS [S1, unofficial].

## Sources
[S1] JoshuaParfitt/gaussian-splatting-lab, https://github.com/JoshuaParfitt/gaussian-splatting-lab, fetched 2026-10-28 (unofficial)
