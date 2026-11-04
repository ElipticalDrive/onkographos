# CUDA Volume Ray Casting

Also called: VRCUDA, cudaRC

## Overview
CUDA Volume Ray Casting is a direct volume rendering method that parallelizes the ray casting algorithm on the GPU using the CUDA programming model, avoiding intermediate geometric entities to directly generate 2D images from 3D scalar data sets [S1][S3, unofficial]. It distinguishes itself from shader-based GPU approaches by utilizing explicit CUDA kernels for memory management and computation, offering performance improvements over standard OpenGL-based ray casting implementations [S1]. It builds on the general principles of [Direct volume ray casting](1117-direct-volume-ray-casting.md).

## How it works
### Inputs and Outputs
**Inputs:**
*   **Volume Data:** 3D scalar field stored in device memory. Typically loaded as a 1D unsigned char array and mapped to a CUDA 3D Array bound to a 3D texture [S1][S3, unofficial].
*   **Transfer Function:** Defines the mapping from scalar values to color (RGBA). Can be provided as a 1D texture or a text file specifying control points [S3, unofficial].
*   **Camera Parameters:** Eye position, view direction, and inverse view matrix [S1][S3, S4, unofficial].
*   **Geometry Bounds:** A bounding box (Ray-Box) defining the volume extent, typically $[-1, 1]^3$ or $[-0.5, 0.5]^3$ [S1][S3, S4, unofficial].

**Outputs:**
*   **Image Buffer:** A Pixel Buffer Object (PBO) or texture containing the final composited RGBA color for each pixel [S1][S3, S4, unofficial].

### Data Structures
*   **3D Texture (`volume`):** Type `VolumeType` (e.g., `unsigned char`), accessed with `cudaReadModeNormalizedFloat`. Uses linear filtering and clamp addressing [S3, unofficial].
*   **Transfer Function Texture (`transferFunction`):** 2D texture of type `float4`. Stores RGBA values normalized to $[0.0, 1.0]$. Uses linear filtering and clamp addressing [S3, unofficial].
*   **Ray-Box:** Axis-aligned bounding box. In [S1], defined by lower corner $(-1, -1, -1)$ and upper corner $(1, 1, 1)$. In [S3, unofficial], the volume is centered at $(0,0,0)$ with extent $[-0.5, 0.5]$, requiring coordinate shifts for texture lookup [S3, unofficial].

### Algorithm Steps

1.  **Thread Indexing:** Each thread handles one pixel. Calculate a unique linear index or use block/thread coordinates to determine the pixel $(x, y)$ [S1][S3, S4, unofficial].
2.  **Ray Initialization:**
    *   Compute normalized device coordinates $u, v \in [-1, 1]$ from pixel coordinates [S1][S3, S4, unofficial].
    *   Calculate the ray direction in world space using the inverse view matrix and camera parameters [S3, S4, unofficial].
    *   Determine the ray origin $O$ (camera position) [S4, unofficial].
3.  **Ray-Box Intersection:**
    *   Calculate the entry ($t_{in}$) and exit ($t_{out}$) distances of the ray intersecting the volume bounding box [S1][S4, unofficial].
    *   *Note:* [S3, unofficial] supports two modes: dynamic ray-box intersection or using pre-computed hit point textures (`texFirst`, `texLast`) if `NOT_RAY_BOX` is defined. Dynamic intersection is noted as potentially problematic in some implementations [S3, unofficial].
    *   Clamp $t_{in}$ to the near clip plane distance ($constantNCP$) if $t_{in} < constantNCP$ [S3, unofficial].
    *   If the ray does not intersect or $t_{out} < s$ (scale factor), skip the ray [S4, unofficial].
4.  **Ray Sampling Loop:**
    *   Iterate from $t_{out}$ down to $t_{in}$ (or vice versa, depending on compositing order) with a step size $h$ [S1][S3, S4, unofficial].
    *   Calculate sample position: $P = O + D \cdot t$ [S1][S4, unofficial].
    *   Convert $P$ to texture coordinates. For a box $[-1, 1]^3$, use $T = (P + 1) / 2$ [S1]. For a box $[-0.5, 0.5]^3$, shift by $+0.5$ for $x,y$ and use $1.0 - (z + 0.5)$ for $z$ to map to $[0, 1]$ [S3, unofficial].
    *   Sample the scalar value $s$ from the volume texture at $T$ [S1][S3, S4, unofficial].
    *   Map $s$ to color $C_{now}$ and opacity $\alpha_{now}$ using the transfer function [S1][S3, unofficial].
    *   *Opacity Conversion:* In [S3, unofficial], the transfer function alpha is converted to optical depth opacity: $\alpha_{now} = 1.0 - \exp(-0.5 \cdot \alpha_{tf})$ [S3, unofficial].
5.  **Compositing:**
    *   **Standard Volume Rendering (Front-to-Back):**
        *   Initialize accumulated color $C_{out} = (0,0,0,0)$ and opacity $\alpha_{out} = 0$ [S1][S4, unofficial].
        *   Update color: $C_{out} \cdot \alpha_{out} = C_{in} \cdot \alpha_{in} + C_{now} \cdot \alpha_{now} \cdot (1 - \alpha_{in})$ [S1].
        *   Update opacity: $\alpha_{out} = \alpha_{in} + \alpha_{now} \cdot (1 - \alpha_{in})$ [S1].
        *   Alternatively, in [S4, unofficial]: $sum = \text{lerp}(sum, col, col.w \cdot density)$ [S4, unofficial].
    *   **X-Ray Rendering:**
        *   Initialize $maxVal = 0$ [S4, unofficial].
        *   If $s \ge isoValue$, update $maxVal = \max(maxVal, s)$ [S4, unofficial].
        *   Final output is grayscale: $maxVal \cdot xrayBrightness$ [S4, unofficial].
6.  **Early Termination:**
    *   If $\alpha_{out}$ exceeds a threshold (e.g., $0.99$ in [S3, unofficial] or $1.0$ in [S1]), break the loop [S1][S3, unofficial].
7.  **Output:**
    *   Convert final accumulated color to 8-bit integer (RGBA) and write to the output buffer (PBO or surface) [S1][S3, S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CUDA Kernel for Volume Ray Casting
# Assumes global variables: volumeTex (3D texture), tfTex (2D texture), 
# invViewMatrix (float4x4), constantH (float step size), opacityThreshold (float)

__global__ void volumeRayCastKernel(unsigned char* output, int width, int height) {
    // 1. Thread Indexing
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x >= width || y >= height) return;

    // 2. Ray Initialization
    // Normalize pixel coordinates to [-1, 1]
    float u = (2.0f * x / (float)width) - 1.0f;
    float v = (2.0f * y / (float)height) - 1.0f;

    // Ray direction in view space (assuming camera looks down -Z)
    // Note: [S4, unofficial] uses normalize(make_float3(u, v, -2.0f)) then transforms by invViewMatrix
    float3 rayDirView = make_float3(u, v, -2.0f);
    rayDirView = normalize(rayDirView);
    
    // Transform direction to world space
    float3 rayDirWorld = mul(invViewMatrix, make_float4(rayDirView, 0.0f)).xyz;
    
    // Ray origin is camera position (transformed from view space origin)
    float3 rayOriginWorld = mul(invViewMatrix, make_float4(0.0f, 0.0f, 0.0f, 1.0f)).xyz;

    // 3. Ray-Box Intersection
    // Assuming unit box [-1, 1]^3 as per [S1]
    float t_near = -1.0f;
    float t_far = 1.0f;
    
    // Simplified Ray-Box Intersection (G. Scott Owen method)
    // This is a placeholder for the actual slab method implementation
    float t_in, t_out;
    bool hit = rayBoxIntersect(rayOriginWorld, rayDirWorld, t_in, t_out);

    if (!hit) return;

    // Clamp to near plane if necessary (constantNCP)
    // t_in = max(t_in, constantNCP); 

    // 4. Sampling Loop
    float4 accumColor = make_float4(0.0f, 0.0f, 0.0f, 0.0f);
    float accumAlpha = 0.0f;

    // Iterate from t_in to t_out
    // Note: [S3, unofficial] iterates t from 0 to D with step constantH
    // [S4, unofficial] iterates from outAbs down to inAbs
    for (float t = t_in; t <= t_out; t += constantH) {
        
        // Sample Point Calculation
        float3 pos = rayOriginWorld + rayDirWorld * t;

        // Texture Coordinate Conversion
        // Map [-1, 1] to [0, 1]
        float3 texCoord = (pos + 1.0f) / 2.0f;

        // Boundary check
        if (texCoord.x < 0.0f || texCoord.x > 1.0f ||
            texCoord.y < 0.0f || texCoord.y > 1.0f ||
            texCoord.z < 0.0f || texCoord.z > 1.0f) {
            continue;
        }

        // Data Sampling
        float scalar = tex3D(volumeTex, texCoord);

        // Transfer Function Lookup
        // [S3, unofficial] uses 2D texture, [S1] uses 1D lookup
        float4 tfColor = tex1D(transferFunction, scalar); // Assuming 1D for simplicity based on [S1]
        
        // Opacity Conversion (if needed, e.g., [S3, unofficial])
        // float alpha_now = 1.0f - expf(-0.5f * tfColor.w);
        float alpha_now = tfColor.w; 

        // 5. Compositing (Front-to-Back)
        // C_out * alpha_out = C_in * alpha_in + C_now * alpha_now * (1 - alpha_in)
        // alpha_out = alpha_in + alpha_now * (1 - alpha_in)
        
        float weight = alpha_now * (1.0f - accumAlpha);
        accumColor.rgb += tfColor.rgb * weight;
        accumAlpha += weight;

        // 6. Early Termination
        if (accumAlpha >= opacityThreshold) {
            break;
        }
    }

    // 7. Result Storage
    // Convert to 8-bit
    int idx = y * width + x;
    output[idx * 4 + 0] = (unsigned char)(accumColor.r * 255.0f);
    output[idx * 4 + 1] = (unsigned char)(accumColor.g * 255.0f);
    output[idx * 4 + 2] = (unsigned char)(accumColor.b * 255.0f);
    output[idx * 4 + 3] = (unsigned char)(accumColor.a * 255.0f);
}
```

### Parameters
*   **constantH:** Step size along the ray. Typically $1.0 / \text{diagonal of volume bounding box}$ [S3, unofficial].
*   **opacityThreshold:** Threshold for early ray termination. Common values: $0.99$ [S3, unofficial] or $1.0$ [S1].
*   **Eye Position:** Fixed at $(0, 0, 6)$ in some implementations [S1].
*   **View Plane:** Located at $z=3$, size $3 \times 3$, centered at $(0, 0, 3)$ [S1].
*   **Ray-Box:** Lower left $(-1, -1, -1)$, upper right $(1, 1, 1)$ [S1].

### Edge Cases and Pitfalls
*   **Coordinate Transformation:** The volume center and extent must match the texture coordinate mapping. If the volume is $[-0.5, 0.5]$, texture coordinates require shifting by $+0.5$ for $x,y$ and $1.0-z$ for $z$ [S3, unofficial].
*   **Ray-Box Intersection:** Dynamic ray-box intersection can be problematic or incorrect if not implemented carefully [S3, unofficial]. Pre-computed hit points are an alternative but require additional OpenGL textures [S3, unofficial].
*   **Near Plane Clipping:** Rays must be clamped to the near plane distance to prevent sampling outside the volume or causing artifacts [S3, unofficial].
*   **Precision:** Floating-point precision in ray stepping can lead to gaps or overlaps; adaptive stepping or higher precision types may be required for high-quality rendering [G].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels (threads) and $M$ is the number of samples per ray. Early termination reduces $M$ on average.
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the volume size. Additional memory for transfer function and PBO.
*   **Performance:** 
    *   On NVIDIA GeForce 8400M GS, CUDA-based ray casting was 1.16x faster than a standard GPU-based (OpenGL shader) algorithm for a $64^3$ dataset (0.703s vs 0.812s) and 1.15x faster for a $256^3$ dataset (0.735s vs 0.842s) [S1].
    *   Achieved a 1.5x speedup over CELL processor and 15x over Intel Xeon on NVIDIA 8800GTX [S2, unofficial].
*   **Strengths:** High parallelism, direct memory access, optimization of early termination and empty space skipping [S2, unofficial].
*   **Limitations:** High memory bandwidth usage, potential for precision issues in ray-box intersection, dependency on CUDA hardware [S1][S3, unofficial].

## Background
*   **Origin:** The method was developed to leverage the multi-core parallel computing ability of GPUs for direct volume rendering, which is computationally intensive and difficult to render with conventional graphics hardware [S1].
*   **Development:** Early implementations focused on optimizing ray casting on CUDA, including out-of-core data management and stream-based parallel processing [S2, unofficial]. Subsequent projects like VRCUDA provided open-source implementations using CUDA toolkit and OpenGL [S3, unofficial].
*   **Usage:** Implemented on NVIDIA GeForce 8400M GS with CUDA SDK 3.1 and OpenGL 3.1 [S1]. Later versions used CUDA Toolkit 7.5 with OpenGL 4.5 [S3, unofficial].

## Sources
[S1] Algorithm of Ray Casting Volume Rendering Based on CUDA, https://doi.org/10.12792/iciae2014.011, fetched 2026-10-24
[S2] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S3] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
[S4] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
