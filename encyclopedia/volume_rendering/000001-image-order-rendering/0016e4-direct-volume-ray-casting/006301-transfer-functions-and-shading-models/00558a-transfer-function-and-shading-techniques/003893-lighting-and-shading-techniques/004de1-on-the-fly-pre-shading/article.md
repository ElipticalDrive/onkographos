# On-the-fly pre-shading

Also called: Pre-shading with 3D Texturing, VanGelder and Kim pre-shading

## Overview
On-the-fly pre-shading is a fast technique that pre-shades a volume on the fly and then slices and composites an RGB volume to obtain an image with shading effects [S1, unofficial][S2, book]. It is distinguished from sibling methods like [Illumination Caching](7548-illumination-caching.md) by computing shading dynamically during the slicing process rather than caching it in a separate data structure, and from [Direct Volume Rendering with Phong Shading](3686-direct-volume-rendering-with-phong-shading.md) by deferring the accumulation of shading into a pre-shaded RGB texture space before compositing.

## How it works
**Inputs and Outputs:**
*   Input: A scalar volume dataset $V$, transfer functions for color $C_{base}$ and opacity $\alpha$, and lighting parameters ($\mathbf{l}, k_a, k_d, k_s, n$).
*   Output: A shaded RGB image.

**Data Structures:**
1.  **Scalar Volume:** 3D array of scalar values.
2.  **Pre-shaded RGB Volume:** A 3D texture where each voxel stores the shaded color $C_{shaded}$. The opacity is typically stored separately or integrated during compositing.
3.  **Ray/Plane Parameters:** View direction, camera position, and slice spacing.

**Steps:**
1.  **Gradient Estimation:** For each voxel $\mathbf{p}_i$ in the volume, estimate the normal $\mathbf{n}_i$ using central finite differences [G]:
    $$ \nabla V(\mathbf{p}_i) \approx \left( \frac{V(\mathbf{p}_i + \Delta x) - V(\mathbf{p}_i - \Delta x)}{2\Delta x}, \frac{V(\mathbf{p}_i + \Delta y) - V(\mathbf{p}_i - \Delta y)}{2\Delta y}, \frac{V(\mathbf}_i + \Delta z) - V(\mathbf{p}_i - \Delta z)}{2\Delta z} \right) $$
    $$ \mathbf{n}_i = \frac{\nabla V(\mathbf{p}_i)}{\|\nabla V(\mathbf{p}_i)\|} $$
    [G] Handle zero-gradient voxels by assigning a default normal (e.g., $(0,0,1)$) to avoid division by zero.

2.  **Shading Computation:** Compute the lighting term $L(\mathbf{p}_i)$ using the Phong model [G]:
    $$ L(\mathbf{p}_i) = k_a + k_d (\mathbf{n}_i \cdot \mathbf{l}) + k_s (\mathbf{r}_i \cdot \mathbf{v})^n $$
    Where $\mathbf{r}_i = 2(\mathbf{n}_i \cdot \mathbf{l})\mathbf{n}_i - \mathbf{l}$.

3.  **Pre-shading:** Map the scalar value to base color $C_{base}$ and compute the shaded color:
    $$ C_{shaded}(\mathbf{p}_i) = C_{base}(V(\mathbf{p}_i)) \cdot L(\mathbf{p}_i) $$
    Store $C_{shaded}$ in the pre-shaded RGB volume.

4.  **Slicing and Compositing:** Render the pre-shaded RGB volume using 3D texturing. Slices are taken at arbitrary small distances to avoid popping [S1, unofficial][S2, book]. The RGB values are accumulated using front-to-back alpha compositing [G]. The opacity $\alpha$ is derived from the transfer function and accumulated separately or integrated with the color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def on_the_fly_pre_shading(volume, transfer_func, light_dir, view_dir, ambient, diffuse, specular, shininess):
    # Step 1 & 2: Pre-shade the volume
    shaded_volume = empty_3d_texture(volume.dimensions)
    for p in volume.voxels:
        # Estimate normal
        grad = finite_difference_gradient(volume, p)
        if norm(grad) == 0:
            normal = (0, 0, 1) # [G] Default normal
        else:
            normal = normalize(grad)
        
        # Compute lighting
        r = 2 * dot(normal, light_dir) * normal - light_dir
        lighting = ambient + diffuse * dot(normal, light_dir) + specular * pow(dot(r, view_dir), shininess)
        
        # Map to color and shade
        base_color = transfer_func.color(volume[p])
        shaded_color = base_color * lighting
        shaded_volume[p] = shaded_color

    # Step 3: Slice and composite
    image = empty_image()
    for ray in rays_from_camera():
        accumulated_color = (0, 0, 0)
        accumulated_alpha = 0
        for t in ray.samples:
            # Interpolate from pre-shaded volume
            c_shaded = shaded_volume.interpolate(ray.position_at(t))
            alpha = transfer_func.opacity(volume[ray.position_at(t)])
            
            # Front-to-back compositing
            weight = alpha * exp(-accumulated_alpha)
            accumulated_color += c_shaded * weight
            accumulated_alpha += alpha * weight
            
        image[ray.pixel] = accumulated_color
    return image
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ for pre-shading, where $N$ is the number of voxels. Slicing is $O(R \cdot S)$, where $R$ is rays and $S$ is samples.
*   **Memory Complexity:** $O(N)$ for the pre-shaded RGB volume.
*   **Performance:** Fast due to one-pass shading and efficient 3D texture interpolation [S1, unofficial][S2, book]. Avoids multiple passes through rasterization hardware [S1, unofficial][S2, book].
*   **Strengths:** High performance, no popping artifacts with small slice distances [S1, unofficial][S2, book].
*   **Limitations:** Requires additional memory for the pre-shaded volume. Shading is static per voxel, not dynamic per sample along the ray.

## Background
On-the-fly pre-shading was introduced by VanGelder and Kim [S1, unofficial][S2, book]. It was developed as a fast alternative to multi-pass methods like those by Meißner et al. [S1, unofficial][S2, book] which suffered from performance loss. It leverages 3D texture hardware interpolation [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
