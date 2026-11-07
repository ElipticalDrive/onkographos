# Hardware-assisted Volume Deformation

Also called: Deformed Volume Rendering via Inverse Deformation in Fragment Shaders, Shader-based volume warping

## Overview
Hardware-assisted volume deformation renders non-linearly deformed volumes by integrating the deformation process directly into the fragment shader, avoiding the computational expense of pre-computing a resampled deformed volume [S1]. Unlike methods that pre-compute the deformed volume (e.g., [LGL95]), this approach allows for dynamic deformations at interactive rates by evaluating the inverse deformation function per-pixel [S1]. It builds on [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md) by modifying the texture coordinate generation and sampling logic to account for spatial transformations.

## How it works
**Inputs and Outputs:**
*   **Input:** Undeformed volume data $v$ stored in a 3D texture; inverse deformation function $u = d^{-1}$, where $d$ maps undeformed object space to deformed object space; camera parameters for ray generation.
*   **Output:** Rendered image with accumulated color and opacity.

**Data Structures:**
*   **3D Texture:** Stores the scalar field of the undeformed volume $v$. Coordinates in this texture correspond to undeformed object space (texture space).
*   **Proxy Geometry:** A series of polygons (view-aligned planes) aligned with the image plane. These polygons define the sampling planes in view space.

**Numbered Steps:**
1.  **Setup:** Store the undeformed volume data $v$ in a 3D texture.
2.  **Geometry:** Draw a series of polygons aligned with the image plane (view-aligned planes) in back-to-front order.
3.  **Texture Coordinates:** Assign texture coordinates to each polygon fragment representing the **deformed** object-space coordinates. These coordinates are interpolated across the polygon during rasterization.
4.  **Fragment Shader Execution:** For each fragment:
    a.  Retrieve the deformed object-space coordinates ($\mathbf{x}_{deformed}$) from the interpolated texture coordinates.
    b.  Apply the inverse deformation function $u$ to these coordinates to obtain undeformed object-space coordinates ($\mathbf{x}_{undeformed}$):
        $$ \mathbf{x}_{undeformed} = u(\mathbf{x}_{deformed}) $$
    c.  Sample the 3D texture using $\mathbf{x}_{undeformed}$ to retrieve the scalar value $d$:
        $$ d = \text{texture3D}(texD, \mathbf{x}_{undeformed}) $$
    d.  Apply the transfer function to $d$ to compute the color $C$ and opacity $\alpha$.
    e.  Accumulate the color and opacity into the framebuffer using front-to-back compositing (as per the parent method's compositing equation).

**Formulas:**
The deformed volume $v'$ is defined as the composition of the undeformed volume $v$ and the inverse deformation function $u$:
$$ v'(x, y, z) = (v \circ u)(x, y, z) $$
where $u = d^{-1}$ and $d$ is the deformation mapping from undeformed to deformed object space [S1].

**Parameters:**
*   **Inverse Deformation Function ($u$):** The mapping from deformed object space to undeformed object space. If only the forward deformation $d$ is provided, $u$ must be computed using scattered data interpolation [S1].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Point sampling in the volume texture can lead to aliasing. To correctly filter the warp, the kernel radius must be determined for the spacing of the sampling in texture space (undeformed space), not object space (deformed space) [S1].
*   **Resource Limits:** Fragment programs have limited resources, making some complex deformation functions impractical to implement directly in the shader [S1].
*   **Non-Uniform Sampling:** If the spacing of samples in view space is non-uniform, errors may occur. These can be addressed through non-planar proxy geometry or applying a per-pixel correction factor [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic for Hardware-assisted Volume Deformation

# Inputs:
# textCoord: vec3, interpolated deformed object-space coordinates
# texD: sampler3D, 3D texture containing undeformed volume v
# u: function, inverse deformation mapping deformed -> undeformed space
# transfer: function, maps scalar value to (color, opacity)

# Output:
# outColor: vec4, accumulated color and opacity

void main() {
    // 1. Get deformed object-space coordinates from texture coordinates
    vec3 deformed_coords = textCoord;

    // 2. Apply inverse deformation to get undeformed (texture space) coordinates
    vec3 undeformed_coords = u(deformed_coords);

    // 3. Sample the undeformed volume texture
    float scalar_value = texture3D(texD, undeformed_coords).r;

    // 4. Apply transfer function to get color and opacity
    vec4 color_opacity = transfer(scalar_value);
    vec3 color = color_opacity.rgb;
    float opacity = color_opacity.a;

    // 5. Compositing (assuming this is called within a loop over sampling planes)
    // Note: The parent method [5616-hardware-accelerated-volume-rendering.md] defines
    // the compositing equation. Here we show the per-step update.
    // C' = C' + (1 - alpha') * alpha * C
    // alpha' = alpha' + (1 - alpha') * alpha
    // This logic is handled by the fragment shader's accumulation state.
}
```

## Operating characteristics
*   **Time Complexity:** The deformation function $u$ is evaluated per fragment. If $u$ is complex, this can be computationally expensive. The number of fragments rendered depends on the resolution and the number of view-aligned planes [S1].
*   **Memory Complexity:** Requires storing the undeformed volume in a 3D texture. No additional memory for pre-computed deformed volumes is needed [S1].
*   **Strengths:** Allows for dynamic, interactive deformations without pre-computation. Leverages GPU hardware for parallel texture sampling and compositing [S1].
*   **Limitations:** Limited by fragment program resources for complex deformation functions. Aliasing issues if not properly filtered. Performance may degrade with complex $u$ or high resolution [S1].
*   **When to use:** Interactive visualization of deforming volumes, such as in non-linear registration or real-time animation [S1].
*   **When not to use:** When the deformation is static and can be pre-computed, or when the deformation function is too complex for the fragment shader [S1].

## Background
This method was introduced in "Integrating Dynamic Deformations into Interactive Volume Visualization" [S1]. It addresses the limitation of traditional volume rendering where deformations require expensive 3D resampling, precluding dynamic deformations. By integrating the deformation into the rendering pipeline, it enables interactive rates for applications like non-linear registration and real-time volume animation [S1].

## Sources
[S1] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
