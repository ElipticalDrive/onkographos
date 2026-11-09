# GPU Based Single-Pass Raycasting

Also called: Single-pass raycaster, Volume Rendering Component

## Overview
GPU Based Single-Pass Raycasting is a volume rendering technique where the volume is traversed along a single ray per pixel, implemented within the Volume Rendering Component framework. It solves the limitations of polygonal-based methods (which are limited in expressive power) and slicing techniques (which can lead to artifacts) by providing high-quality visual results [S1, unofficial]. It is distinguished from its siblings by its specific implementation as a single-pass fragment shader execution on modern graphics hardware, relying on hardware-accelerated 3D texture sampling for trilinear interpolation [S1, unofficial].

## How it works
The method operates within the Volume Rendering Component, separating scene graph responsibilities into data representation (`VolumeData`) and rendering styles (`VolumeRenderStyle`). The core algorithm executes a ray marching loop in the fragment shader for each pixel.

### Inputs and Outputs
*   **Input**: A 3D texture volume (`voxels`), camera parameters, ray generation settings (`stepSize`, `type`), acceleration settings (`boundingVolumeType`, `traversalFunction`), normal calculation settings (`algorithm`), and transfer functions (`OpacityMap`).
*   **Output**: A 2D image with accumulated color and opacity per pixel.

### Data Structures
The primary data structure is the `VolumeData` node, which holds the volume data in its `voxels` field (an `MFNode` holding a 3D texture). Additional fields include:
*   `sliceThickness` (`SFVec3f`): Default `(1,1,1)`. Used to handle non-equidistant grids.
*   `dimensions` (`SFVec3f`): Currently unused.
*   `swapped` (`SFBool`): Helper for 16-bit volume byte swapping.

Rendering settings are encapsulated in:
*   `AccelerationVolumeSettings`: Contains `boundingVolumeType` (`"boxGeometry"` or `"boundaryGeometry"`) for outer space skipping and `traversalFunction` (`"simple"` or `"blockSkipping"`) for inner space skipping.
*   `NormalVolumeSettings`: Contains `algorithm` (`"precomputedCentralDifferences"`, `"onlineCentralDifferences"`, or `"onlineSobel"`).
*   `RayGenerationVolumeSettings`: Contains `type` (`"simple"` or `"interleaved"`) and `stepSize` (e.g., `0.0039`).
*   `OpacityMapVolumeStyle`: Contains `transferFunction` (an `ImageTexture`) and `type` (`"simple"` or `"preintegrated"`).

### Steps
1.  **Volume Loading**: Load the volume data into a 3D texture. Handle byte swapping if necessary using the `swapped` field. Adjust sampling coordinates based on `sliceThickness` if the grid is non-equidistant.
2.  **Ray Generation**: For each pixel, generate a ray. The `type` parameter determines the generation strategy:
    *   `"simple"`: Standard per-pixel ray generation.
    *   `"interleaved"`: Interleaved ray generation to mitigate wood artifacts [S1, unofficial].
3.  **Acceleration (Outer Space Skipping)**: Determine the entry and exit points of the ray through the volume bounding box. The `boundingVolumeType` in `AccelerationVolumeSettings` selects between `"boxGeometry"` (axis-aligned bounding box) or `"boundaryGeometry"` (more precise boundary) [S1, unofficial].
4.  **Acceleration (Inner Space Skipping)**: Traverse the volume from entry to exit. The `traversalFunction` in `AccelerationVolumeSettings` selects between `"simple"` (uniform stepping) or `"blockSkipping"` (coarser skipping in empty regions) [S1, unofficial].
5.  **Sampling and Classification**: At each step along the ray, sample the 3D texture using hardware-accelerated trilinear interpolation. Apply the transfer function (defined in `OpacityMapVolumeStyle`) to determine color $C_i$ and opacity $\alpha_i$. The `type` in `OpacityMapVolumeStyle` selects between `"simple"` (single-sample) or `"preintegrated"` (pre-integrated transfer function for better visual quality) [S1, unofficial].
6.  **Normal Calculation**: If lighting is enabled (`ShadedVolumeStyle.lighting = TRUE`), calculate surface normals. The `algorithm` in `NormalVolumeSettings` determines the method:
    *   `"precomputedCentralDifferences"`: Fastest, requires precomputed normals.
    *   `"onlineCentralDifferences"`: Most appropriate, calculates normals on the fly using central differences.
    *   `"onlineSobel"`: Nicest visual result, uses Sobel operator [S1, unofficial].
7.  **Compositing**: Perform front-to-back compositing using the standard emission-absorption model:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
8.  **Early Ray Termination (ERT)**: Terminate the ray if $\alpha'$ exceeds a threshold (typically 0.95 or 1.0) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Implementation for GPU Based Single-Pass Raycasting

# Uniforms
uniform sampler3D volume;
uniform vec3 rayOrigin;
uniform vec3 rayDirection;
uniform float stepSize;
uniform float entryPoint;
uniform float exitPoint;
uniform vec3 sliceThickness;
uniform bool usePreintegratedOpacity;
uniform sampler1D transferFunction;
uniform vec3 normalOffset; # For online normal calculation

# Output
out vec4 fragColor;

void main() {
    // 1. Initialize compositing variables
    vec3 accumulatedColor = vec3(0.0);
    float accumulatedOpacity = 0.0;
    
    // 2. Determine ray parameters
    vec3 rayDir = normalize(rayDirection);
    float t = entryPoint;
    float maxT = exitPoint;
    
    // 3. Ray Marching Loop
    while (t < maxT && accumulatedOpacity < 0.95) {
        // Calculate current sampling position
        vec3 currentPos = rayOrigin + t * rayDir;
        
        // Adjust for non-equidistant grids if necessary
        // Note: In practice, sliceThickness might require coordinate scaling
        vec3 texCoord = currentPos; 
        
        // 4. Sample Volume
        vec4 sample = texture3D(volume, texCoord);
        float value = sample.r; // Assuming scalar value in red channel
        
        // 5. Apply Transfer Function
        float alpha = 0.0;
        vec3 color = vec3(0.0);
        
        if (usePreintegratedOpacity) {
            // Preintegrated opacity mapping
            // This typically involves a 2D texture lookup or complex 1D lookup
            // based on step size and value. Simplified here as a direct lookup.
            alpha = texture1D(transferFunction, value).a;
            color = texture1D(transferFunction, value).rgb;
        } else {
            // Simple opacity mapping
            alpha = texture1D(transferFunction, value).a;
            color = texture1D(transferFunction, value).rgb;
        }
        
        // 6. Lighting (if enabled)
        if (lightingEnabled) {
            vec3 normal = calculateNormal(texCoord);
            // Apply lighting model (e.g., Phong)
            color = applyLighting(color, normal, lightPosition, viewPosition);
        }
        
        // 7. Compositing
        if (accumulatedOpacity >= 1.0) {
            break;
        }
        
        accumulatedColor += (1.0 - accumulatedOpacity) * alpha * color;
        accumulatedOpacity += (1.0 - accumulatedOpacity) * alpha;
        
        // 8. Advance Ray
        t += stepSize;
    }
    
    // 9. Output
    fragColor = vec4(accumulatedColor, accumulatedOpacity);
}

vec3 calculateNormal(vec3 texCoord) {
    // Implementation depends on NormalVolumeSettings.algorithm
    // Example: onlineCentralDifferences
    float eps = 0.001;
    float v1 = texture3D(volume, texCoord + vec3(eps, 0, 0)).r;
    float v2 = texture3D(volume, texCoord - vec3(eps, 0, 0)).r;
    float v3 = texture3D(volume, texCoord + vec3(0, eps, 0)).r;
    float v4 = texture3D(volume, texCoord - vec3(0, eps, 0)).r;
    float v5 = texture3D(volume, texCoord + vec3(0, 0, eps)).r;
    float v6 = texture3D(volume, texCoord - vec3(0, 0, eps)).r;
    
    vec3 grad = vec3(v1 - v2, v3 - v4, v5 - v6);
    return normalize(grad);
}
```

### Parameters and Typical Values
*   `stepSize`: Controls the distance between samples. A smaller value increases quality but reduces performance. Example value: `0.0039` [S1, unofficial].
*   `type` (RayGeneration): `"simple"` or `"interleaved"`. `"interleaved"` reduces wood artifacts [S1, unofficial].
*   `boundingVolumeType`: `"boxGeometry"` or `"boundaryGeometry"`. `"boundaryGeometry"` is more precise [S1, unofficial].
*   `traversalFunction`: `"simple"` or `"blockSkipping"`. `"blockSkipping"` improves performance in empty regions [S1, unofficial].
*   `type` (OpacityMap): `"simple"` or `"preintegrated"`. `"preintegrated"` often leads to a much better look [S1, unofficial].
*   `algorithm` (NormalVolumeSettings): `"precomputedCentralDifferences"` (fastest), `"onlineCentralDifferences"` (most appropriate), `"onlineSobel"` (nicest) [S1, unofficial].
*   `lighting`: Boolean. If `TRUE`, lighting effects are calculated [S1, unofficial].

### Edge Cases and Pitfalls
*   **Wood Artifacts**: Can occur in raycasting due to aliasing. Mitigated by using the `"interleaved"` ray generation type [S1, unofficial].
*   **Non-equidistant Grids**: Handled by the `sliceThickness` field in `VolumeData`, defaulting to `(1,1,1)`. If the grid is not equidistant, this field must be set correctly to avoid distortion [S1, unofficial].
*   **16-bit Volume Byte Swapping**: Handled by the `swapped` field in `VolumeData`. If not handled correctly, data will be interpreted incorrectly [S1, unofficial].
*   **Hardware Requirements**: Requires modern graphics hardware with support for 3D textures and fragment shaders [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of steps along the ray, which depends on the volume size, `stepSize`, and ray length. Acceleration techniques (`blockSkipping`) can reduce the effective number of steps.
*   **Memory Complexity**: Dominated by the storage of the 3D volume texture. The `VolumeData` node holds the volume in `voxels`.
*   **Measured Performance**: Performance is heavily dependent on GPU capabilities and the complexity of the transfer function and lighting calculations. Preintegrated opacity and block skipping can improve performance.
*   **Strengths**: High visual quality, handles complex transfer functions and lighting, avoids artifacts associated with slicing.
*   **Limitations**: Requires modern graphics hardware, can be computationally expensive for large volumes or small step sizes. Limited to one volume per scene in the current implementation [S1, unofficial].
*   **When to Use**: When high-quality volume visualization is required and modern graphics hardware is available.
*   **When Not to Use**: When hardware does not support 3D textures or fragment shaders, or when real-time performance on low-end hardware is critical.

## Background
The GPU Based Single-Pass Raycasting method is implemented within the Volume Rendering Component, an X3D extension proposal. It was developed to alleviate the drawbacks of polygonal-based methods and slicing techniques. The component separates scene graph responsibilities into data representation and rendering styles, allowing for flexible configuration of rendering parameters. The implementation relies on GPU-based single-pass raycasting, traversing the volume along a single ray per pixel [S1, unofficial].

## Sources
[S1] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
