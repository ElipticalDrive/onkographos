# CUDA Volume Ray Casting

Also called: CUDA Raycasting, Single-pass CUDA Raycasting, VRCUDA, cudaRC

## Overview
CUDA Volume Ray Casting is a direct volume rendering method that parallelizes the ray casting algorithm on the NVIDIA GPU using the CUDA programming model, avoiding intermediate geometric entities to directly generate 2D images from 3D datasets [S1][S4, unofficial]. It differs from shader-based GPU ray casting by utilizing explicit CUDA memory management (CUDA arrays and surfaces) for zero-overhead interoperability with OpenGL, allowing the GPU to write directly to output buffers without intermediate CPU round-trips [S2][S4, unofficial]. It is chosen when fine-grained control over memory access patterns and kernel execution is required compared to fixed-function or shader-based pipelines [S3, S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data (scalar field), transfer function (mapping scalar to RGBA), camera parameters (position, view matrix), and viewport dimensions.
*   **Output:** 2D image buffer (RGBA) stored in a Pixel Buffer Object (PBO) or surface [S1, S2].

**Data Structures:**
1.  **Volume Texture:** A 3D CUDA texture object (`texture<VolType, 3, cudaReadModeNormalizedFloat>`) bound to a 3D CUDA array. The volume data is typically `unsigned char` (8-bit) [S1, S2][S4, unofficial].
2.  **Transfer Function Texture:** A 2D CUDA texture object (`texture<float4, 2, cudaReadModeElementType>`) storing RGBA values. It is sampled using the scalar value as the x-coordinate and a constant (e.g., 0.5) as the y-coordinate [S2][S4, unofficial].
3.  **Output Surface:** A 2D CUDA surface object (`surface<void, cudaSurfaceType2D>`) bound to the output texture/array to allow writing from the kernel, as CUDA cannot write directly to textures [S2][S4, unofficial].
4.  **Hit Point Textures (Optional):** Two 2D textures (`texFirst`, `texLast`) storing pre-computed ray-box intersection points (entry and exit) for each pixel, used to avoid dynamic intersection calculations [S2][S4, unofficial].

**Parameters:**
*   `constantH`: Step size along the ray. Typically calculated as $1.0 / \text{diagonal of the volume bounding box}$ [S4, unofficial].
*   `opacityThreshold`: Threshold for early ray termination. Default value is $0.99$ [S2][S4, unofficial].
*   `constantNCP`: Near clip plane distance. Rays are clamped to this value to prevent sampling outside the volume [S4, unofficial].
*   `blockSize`: Thread block dimensions. Commonly `dim3(8, 16)` or `dim3(16, 16)` [S2][S4, S6, unofficial].

**Algorithm Steps:**
1.  **Thread Indexing:** Each thread handles one pixel. Calculate pixel coordinates $(x, y)$ from `blockIdx` and `threadIdx` [S1, S2].
2.  **Ray Initialization:**
    *   *Method A (Dynamic Intersection):* Compute ray origin $O$ (camera position) and direction $D$ using the inverse view matrix and normalized screen coordinates $(u, v) \in [-1, 1]$. Calculate intersection with the volume bounding box (e.g., $[-1, 1]^3$ or $[0, 1]^3$) to find entry ($t_{near}$) and exit ($t_{far}$) distances [S1][S4, S5, unofficial]. Clamp $t_{near}$ to `constantNCP` [S4, unofficial].
    *   *Method B (Pre-computed):* Sample `texFirst` and `texLast` at $(x, y)$ to get entry and exit points directly [S2][S4, unofficial].
3.  **Ray Casting Loop:** Iterate from $t = t_{near}$ to $t_{far}$ with step `constantH`.
    *   Calculate sample position $P$. If using dynamic intersection, $P = O + D \cdot t$. If using pre-computed points, interpolate between first and last points [S1, S2][S4, unofficial].
    *   **Coordinate Conversion:** Convert world/box coordinates to texture coordinates $[0, 1]$. For a box centered at origin with extent $[-0.5, 0.5]$, use $x_{tex} = x + 0.5$, $y_{tex} = y + 0.5$, $z_{tex} = 1.0 - (z + 0.5)$ [S4, unofficial]. For a box $[-1, 1]$, use $(P + 1) / 2$ [S1].
    *   **Sampling:** Sample scalar value $s$ from the volume texture at $P_{tex}$ [S2][S4, unofficial].
    *   **Transfer Function:** Sample RGBA $(c_r, c_g, c_b, \alpha_{raw})$ from the transfer function texture at $(s, 0.5)$ [S2][S4, unofficial].
    *   **Opacity Conversion:** Convert raw alpha to optical depth opacity: $\alpha = 1.0 - \exp(-0.5 \cdot \alpha_{raw})$ [S2][S4, unofficial].
    *   **Compositing (Front-to-Back):**
        *   Pre-multiply color: $c_{pre} = c_{rgb} \cdot \alpha$ [S2][S4, unofficial].
        *   Update accumulated color: $C_{out} = C_{in} + c_{pre} \cdot \alpha_{in}$ [S2][S4, unofficial].
        *   Update accumulated opacity: $\alpha_{out} = \alpha_{in} + \alpha \cdot (1 - \alpha_{in})$ [S2][S4, unofficial].
    *   **Early Termination:** If $1.0 - \alpha_{out} > \text{opacityThreshold}$ (i.e., accumulated opacity $> 0.99$), break the loop [S2][S4, unofficial].
4.  **Finalization:**
    *   Calculate final alpha: $\alpha_{final} = 1.0 - \alpha_{out}$ [S2][S4, unofficial].
    *   Convert to 8-bit integer: $Color_{8bit} = (C_{out} \cdot 255, \alpha_{final} \cdot 255)$ [S2][S4, unofficial].
    *   Write to output surface using `surf2Dwrite` with clamp boundary mode [S2][S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel launch configuration
# block_size = (8, 16, 1)
# grid_size = (ceil(width / 8), ceil(height / 16), 1)

__global__ void volumeRenderingKernel(
    texture<unsigned char, 3, cudaReadModeNormalizedFloat> volumeTex,
    texture<float4, 2, cudaReadModeElementType> transFuncTex,
    surface<void, cudaSurfaceType2D> outputSurf,
    int width, int height,
    float stepSize,
    float opacityThreshold,
    float4x4 invViewMatrix,
    float3 cameraPos,
    float3 boxMin,
    float3 boxMax
) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x >= width || y >= height) return;

    // 1. Ray Initialization
    // Normalize screen coordinates to [-1, 1]
    float u = (2.0f * x / width) - 1.0f;
    float v = (2.0f * y / height) - 1.0f;

    // Compute ray direction in world space
    // Assuming perspective projection with near plane at z=-1 in NDC before view transform
    float3 rayDirNDC = normalize(float3(u, v, -1.0f));
    float3 rayDir = normalize(mul(invViewMatrix, float4(rayDirNDC, 0.0f)));
    float3 rayOrigin = mul(invViewMatrix, float4(cameraPos, 1.0f));

    // 2. Ray-Box Intersection (G. Scott Owen method)
    float tNear, tFar;
    if (!intersectBox(rayOrigin, rayDir, boxMin, boxMax, tNear, tFar)) {
        return; // No intersection
    }

    // Clamp near plane
    float nearClip = 0.01f; // Example constantNCP
    if (tNear < nearClip) tNear = nearClip;

    // 3. Ray Casting Loop
    float3 pos = rayOrigin + rayDir * tNear;
    float4 accumColor = float4(0.0f, 0.0f, 0.0f, 0.0f); // R, G, B, Alpha (accumulated opacity)
    
    // Note: In the under operator, accumColor.w stores the *remaining* transparency (1 - alpha)
    // or we can store accumulated alpha. The source [S2] uses color.w as remaining transparency.
    float remainingTransparency = 1.0f;

    for (float t = tNear; t <= tFar; t += stepSize) {
        // Coordinate conversion to [0, 1]
        // Assuming box is [0,1]^3 for texture coords. If box is [-1,1], adjust accordingly.
        // Here we assume the box intersection returns coords in the same space as the texture.
        // If boxMin=(0,0,0), boxMax=(1,1,1), pos is already in [0,1].
        // If boxMin=(-1,-1,-1), boxMax=(1,1,1), use (pos + 1) / 2.
        
        float3 texPos = pos; 
        // [G] Guard against out-of-bounds if t exceeds tFar slightly due to float precision
        if (texPos.x < 0.0f || texPos.x > 1.0f || 
            texPos.y < 0.0f || texPos.y > 1.0f || 
            texPos.z < 0.0f || texPos.z > 1.0f) {
            break;
        }

        // Sample Volume
        float scalar = tex3D(volumeTex, texPos);

        // Sample Transfer Function
        float4 tf = tex2D(transFuncTex, scalar, 0.5f);
        float alpha_raw = tf.w;

        // Convert to optical depth opacity
        float alpha = 1.0f - expf(-0.5f * alpha_raw);

        // Pre-multiply color
        float3 color_pre = tf.rgb * alpha;

        // Composite
        // C_out = C_in + C_now * alpha_now * (1 - alpha_in)
        // Here, remainingTransparency is (1 - alpha_in)
        accumColor.rgb += color_pre * remainingTransparency;
        remainingTransparency *= (1.0f - alpha);

        // Early Termination
        if (1.0f - remainingTransparency > opacityThreshold) {
            break;
        }

        // Increment
        pos += rayDir * stepSize;
    }

    // Finalize
    float finalAlpha = 1.0f - remainingTransparency;
    float4 finalColor = float4(accumColor.rgb, finalAlpha);

    // Write to Surface
    // Convert to 8-bit
    uchar4 outColor = uchar4(
        min(255, int(finalColor.r * 255.0f)),
        min(255, int(finalColor.g * 255.0f)),
        min(255, int(finalColor.b * 255.0f)),
        min(255, int(finalColor.a * 255.0f))
    );
    surf2Dwrite(outColor, outputSurf, x * sizeof(uchar4), y, cudaBoundaryModeClamp);
}

// Helper: Ray-Box Intersection (G. Scott Owen)
__device__ bool intersectBox(float3 origin, float3 dir, float3 min, float3 max, out float tNear, out float tFar) {
    float3 t0 = (min - origin) / dir;
    float3 t1 = (max - origin) / dir;
    float3 tmin = min(t0, t1);
    float3 tmax = max(t0, t1);
    
    tNear = max(max(tmin.x, tmin.y), tmin.z);
    tFar = min(min(tmax.x, tmax.y), tmax.z);
    
    return (tNear <= tFar) && (tFar >= 0.0f);
}
```

**Edge Cases and Pitfalls:**
*   **Texture Coordinates:** Incorrect mapping between world space and texture space (e.g., failing to shift by +0.5 or invert Z) causes visual artifacts [S4, unofficial].
*   **Writing to Textures:** CUDA cannot write to textures. The output must be bound to a surface [S2].
*   **Near Plane Clipping:** Rays must be clamped to the near plane to prevent sampling artifacts or invalid memory access [S4, unofficial].
*   **Precision:** Floating point errors in ray stepping can cause gaps or overlaps. Using a step size slightly larger than the voxel size can cause aliasing, while smaller steps increase cost [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are image dimensions and $N$ is the number of samples per ray. Early termination reduces the average $N$ [S1][S4, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the number of voxels. The transfer function is negligible [S1][S4, unofficial].
*   **Performance:** On an NVIDIA GeForce 8400M GS, CUDA-based ray casting achieved 0.703s for a 64x64x64 dataset and 0.735s for a 256x256x256 dataset, outperforming a standard GPU shader-based implementation by 1.16x and 1.15x respectively [S1].
*   **Strengths:** High performance due to direct memory access and parallelization; zero-overhead OpenGL interoperability; flexible transfer function integration [S1, S2][S4, unofficial].
*   **Limitations:** High memory bandwidth usage; performance degrades with large datasets if not optimized (e.g., using out-of-core techniques) [S3, unofficial]; requires NVIDIA hardware [S1].
*   **When to Use:** When high-quality direct volume rendering is needed on NVIDIA hardware and fine-grained control over the rendering pipeline is required. Not suitable for non-NVIDIA GPUs or when hardware-accelerated texture slicing is preferred for simplicity [S6, unofficial].

## Background
CUDA Volume Ray Casting was developed as part of the broader adoption of GPGPU computing for scientific visualization. Early implementations demonstrated significant speedups over CPU-based methods and even early GPU shader-based approaches by leveraging CUDA's explicit memory management [S1][S3, unofficial]. The method has been implemented in various educational and research projects, such as VRCUDA, to demonstrate the integration of CUDA with OpenGL for real-time volume rendering [S4, unofficial]. It builds on the foundational ray casting algorithm by Levoy and the optimization techniques for early termination and empty space skipping [S3, unofficial].

## Sources
[S1] Algorithm of Ray Casting Volume Rendering Based on CUDA, https://doi.org/10.12792/iciae2014.011, fetched 2026-10-24
[S2] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S3] Volume Ray Casting Techniques and Applications using GPGPU, https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S4] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
[S5] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S6] GPU Ray Casting, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
