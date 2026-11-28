# Raycasting with Transfer Function and Shading

Also called: Front-to-back volume raycasting with gradient-based diffuse shading

## Overview
This method performs direct volume ray casting by sampling a 3D texture storing pre-computed integer isovalue and gradient magnitude data, allowing the fragment shader to sample surface properties directly without numerical differentiation [S1, unofficial]. It distinguishes itself from standard raycasting by weighting the transfer function opacity with the gradient magnitude to enhance edges and surfaces, and by using a lookup-based approach for lighting and visibility selection [S1, unofficial]. It builds on [Volumetric Raycasting](3462-volumetric-raycasting.md) by replacing on-the-fly gradient computation with pre-sampled gradient magnitudes and adding isovalue-based selection logic [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `position`: `vec4` representing the screen-space position of the fragment [S1, unofficial].
    *   `cubeFront`: `sampler2D` (binding 0) storing the entry point of the ray into the volume [S1, unofficial].
    *   `cubeBack`: `sampler2D` (binding 1) storing the exit point of the ray from the volume [S1, unofficial].
    *   `volume`: `sampler3D` (binding 2) storing the scalar field data [S1, unofficial].
    *   `gradients`: `sampler3D` (binding 3) storing encoded normal vectors [S1, unofficial].
    *   `transferFunction`: `sampler1D` (binding 4) mapping scalar values to RGBA color and opacity [S1, unofficial].
    *   `bakedNoise`: `sampler2D` (binding 5) storing pre-computed noise for jittering [S1, unofficial].
    *   `ciModelMatrixInverseTranspose`: `mat3` for transforming normals from object to view space [S1, unofficial].
    *   `light`: Struct containing `direction`, `ambient`, and `diffuse` color components [S1, unofficial].
    *   `threshold`: `ivec2` defining the inclusive range of iso values (0-255) to render [S1, unofficial].
    *   `stepSize`: `vec3` defining the step size in each dimension [S1, unofficial].
    *   `iterations`: `int` defining the maximum number of ray marching steps [S1, unofficial].
    *   `diffuseShading`: `bool` flag to enable/disable gradient-based shading [S1, unofficial].
    *   `stepScale`: `float` parameter used in opacity correction [S1, unofficial].
*   **Outputs**:
    *   `oColor`: `vec4` (location 0) containing the final composited color with alpha 1.0 [S1, unofficial].
    *   `oNormal`: `vec3` (location 1) containing the normal of the last sampled point [S1, unofficial].

**Data Structures**
The method relies on standard GLSL sampler types and uniform structs. The `light` struct is defined as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
struct Light {
    vec3 direction;
    vec3 ambient;
    vec3 diffuse;
};
```
The gradient texture stores normals in a compressed 2D format (spheremap) to save memory, requiring decoding [S1, unofficial].

**Steps**
1.  **Ray Setup**:
    *   Convert screen position to texture coordinates: `texC = position.xy / position.w`.
    *   Adjust coordinates for the cube map sampling: `texC.x = 0.5 * texC.x + 0.5`, `texC.y = 0.5 * texC.y - 0.5`.
    *   Sample the front and back faces of the volume bounding box: `front = texture(cubeFront, texC).xyz`, `back = texture(cubeBack, texC).xyz`.
    *   Compute the ray direction: `dir = normalize(back - front)`.
    *   Initialize the ray position: `pos = vec4(front, 0)`.
    *   Initialize the destination color buffer: `dst = vec4(0)`.
    *   Initialize the source color buffer: `src = vec4(0)`.
    *   Initialize the sampled value: `value = vec4(0)`.
    *   Compute the step vector: `step = dir * stepSize`.
    *   Jitter the start position to reduce banding: `pos.xyz += step * texture(bakedNoise, gl_FragCoord.xy / 256).x` [S1, unofficial].

2.  **Ray Marching Loop**:
    *   Iterate `i` from 0 to `iterations - 1`.
    *   Set the homogeneous coordinate: `pos.w = 0`.
    *   Sample the volume scalar field: `value.a = texture(volume, pos.xyz).x`.
    *   Compute the integer iso value: `isoValue = int(255.0f * value.a)`.
    *   Check if the iso value is within the rendering threshold: `if (isoValue >= threshold.x && isoValue <= threshold.y)`.
        *   Sample the transfer function: `src = texture(transferFunction, value.a)`.
        *   Apply opacity correction to account for step size: `src.a = 1 - pow((1 - src.a), stepScale / 0.5)`.
        *   If `diffuseShading` is true:
            *   Sample the encoded gradient: `encodedGrad = texture(gradients, pos.xyz).xy`.
            *   Decode the normal vector:
                *   `n.z = dot(encodedGrad, encodedGrad) * 2.0 - 1.0`.
                *   `n.xy = normalize(encodedGrad.xy) * sqrt(1.0 - n.z * n.z)`.
                *   `value.xyz = n`.
            *   Transform the normal to view space: `normal = normalize(ciModelMatrixInverseTranspose * value.xyz)`.
            *   Compute the light direction: `lightDir = normalize(-light.direction)`.
            *   Compute the Lambertian diffuse factor: `lambert = max(dot(normal, lightDir), 0.0)`.
            *   Apply shading to the source color: `src.rgb = light.diffuse * lambert * src.rgb + light.ambient * src.rgb`.
        *   Perform front-to-back blending:
            *   Modulate source color by alpha: `src.rgb *= src.a`.
            *   Update destination: `dst = (1.0 - dst.a) * src + dst`.
            *   Break the loop if accumulated opacity is sufficient: `if (dst.a >= 0.95) break`.
    *   Advance the ray position: `pos.xyz += step`.
    *   Check for out-of-bounds conditions: `if (pos.x > 1.0 || pos.y > 1.0 || pos.z > 1.0) break`.

3.  **Output**:
    *   Set the final color: `oColor = vec4(dst.rgb, 1.0)`.
    *   Set the output normal: `oNormal = value.xyz`.
    *   Set the depth: `gl_FragDepth = pos.z` [S1, unofficial].

**Formulas**
*   **Normal Decoding** (Spheremap):
    $$n_z = \mathbf{e} \cdot \mathbf{e} \times 2.0 - 1.0$$
    $$\mathbf{n}_{xy} = \frac{\mathbf{e}_{xy}}{\|\mathbf{e}_{xy}\|} \sqrt{1.0 - n_z^2}$$
    Where $\mathbf{e}$ is the encoded 2D vector from the gradient texture [S1, unofficial].
*   **Opacity Correction**:
    $$src.a = 1 - (1 - src.a)^{\frac{stepScale}{0.5}}$$
    This corrects the opacity sampled at a discrete step to approximate continuous integration [S1, unofficial].
*   **Front-to-Back Blending**:
    $$dst = (1 - dst.a) \times src + dst$$
    Standard alpha compositing equation [S1, unofficial].

**Parameters**
*   `stepScale`: Float parameter used in opacity correction. Typical value not specified, but used as divisor in exponent [S1, unofficial].
*   `threshold`: `ivec2` defining the range of iso values (0-255) to render. Default from TransferFunction is (0, 255) [S1, unofficial].
*   `stepSize`: `vec3` defining the step size in each dimension [S1, unofficial].
*   `iterations`: `int` defining the maximum number of ray marching steps [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Out of Bounds**: The ray position is checked against 1.0 in x, y, z. If any exceed 1.0, the loop breaks [S1, unofficial].
*   **Early Termination**: Ray marching breaks if accumulated alpha `dst.a >= 0.95` [S1, unofficial].
*   **Normal Decoding**: The spheremap transform is attributed to Cry Engine 3 and Martin Mittring [S1, unofficial].

## Subtopics

### [Gradient-Based Volume Rendering with Isovalue Lookup](10998-gradient-based-volume-rendering-with-isovalue-lookup.md)
This method performs direct volume ray casting by sampling a 3D texture storing pre-computed integer isovalue and gradient magnitude data, allowing the fragment shader to sample surface properties directly without numerical differentiation. It distinguishes itself from standard raycasting by weighting the transfer function opacity with the gradient magnitude to enhance edges and surfaces, and by using a lookup-based approach for lighting and visibility selection. It builds on [Raycasting with Transfer Function and Shading](3773-raycasting-with-transfer-function-and-shading.md) by replacing on-the-fly gradient computation with pre-sampled gradient magnitudes and adding isovalue-based selection logic.

### [Gradient-Density Raycasting with Transfer Function Blending](13551-gradient-density-raycasting-with-transfer-function-blending.md)
A fragment-based volume rendering technique that performs raycasting in the fragment shader, sampling density and gradient simultaneously, blending between two transfer functions based on an index, and applying silhouette and gradient magnitude modulation for shading. It differs from standard raycasting by explicitly computing the gradient vector per sample to drive both shading (via `Illuminate`) and opacity modulation (via `SilhouetteModulation` and `GradientMagnitudeModulation`), while using a fixed step count derived from ray length rather than adaptive traversal.

### [Front-to-Back Ray Casting](22171-front-to-back-ray-casting.md)
Front-to-Back Ray Casting is a direct volume rendering technique that accumulates color and opacity along a ray traversing a 3D texture space, starting from the viewer and moving toward the back of the volume. It solves the problem of compositing semi-transparent volumetric data into a 2D image by applying the standard front-to-back alpha blending equation at each sample step. It is distinguished from its siblings by its explicit use of a fixed iteration count combined with boundary checks, and its specific normalization of raw volume values to a fixed [0, 1] range before blending, rather than relying solely on the transfer function's alpha channel for opacity weighting.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of iterations (ray steps) per pixel.
*   **Memory Complexity**: Depends on the size of the 3D volume texture and gradient texture. The method requires additional 2D textures for front/back face lookup and baked noise.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Performance scales linearly with the number of iterations and the resolution of the output image.
*   **Strengths**:
    *   Uses pre-computed gradients for efficient lighting calculations [S1, unofficial].
    *   Isovalue thresholding allows for selective rendering of specific density ranges [S1, unofficial].
    *   Jittered stepping reduces banding artifacts [S1, unofficial].
*   **Limitations**:
    *   Fixed step size may lead to aliasing or excessive computation if not tuned [S1, unofficial].
    *   Early termination threshold (0.95) may cause transparency artifacts in thin regions [S1, unofficial].
*   **When to use**: Suitable for real-time rendering of volumetric data where gradient-based shading is desired and pre-computed gradients are available [S1, unofficial].
*   **When not**: Not suitable for adaptive step sizes or when high-quality anti-aliasing is required without additional techniques [S1, unofficial].

## Background
The method is implemented in the `VolumeRendering` project by Jose Villegas [S1, unofficial]. The normal decoding technique uses a spheremap transform, attributed to Cry Engine 3 and Martin Mittring [S1, unofficial]. The implementation includes a UI for editing transfer functions, allowing users to add alpha and color control points based on iso values [S1, unofficial].

## Sources
[S1] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
