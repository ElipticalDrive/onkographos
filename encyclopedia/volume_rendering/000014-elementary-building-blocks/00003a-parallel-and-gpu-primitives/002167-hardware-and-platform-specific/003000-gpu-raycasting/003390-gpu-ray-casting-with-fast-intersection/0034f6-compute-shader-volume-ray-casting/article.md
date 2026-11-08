# Compute Shader Volume Ray Casting

Also called: OpenGL Compute Shader Volume Rendering

## Overview
Compute Shader Volume Ray Casting is a GPU-accelerated volume rendering technique that offloads the ray-marching loop from the fragment shader to a compute shader kernel. It solves the problem of rendering volumetric data by calculating the entry and exit points of rays through the volume in a prior pass, then using a compute shader to perform the integration for each pixel in parallel. It differs from its sibling [GPU Ray Casting with Fast Intersection](7032-gpu-ray-casting-with-fast-intersection.md) by explicitly utilizing the compute shader model for the heavy lifting of ray traversal, allowing for more flexible thread organization and memory access patterns compared to fixed-function fragment pipelines.

## How it works
The method operates in two distinct stages: a rasterization pass to determine ray boundaries and a compute pass to perform volume integration.

**Inputs and Outputs:**
- **Inputs:**
  - `firstHit`: A 2D texture storing the entry point (3D coordinates) for each pixel.
  - `lastHit`: A 2D texture storing the exit point (3D coordinates) for each pixel.
  - `volume`: A 3D texture containing the scalar field data.
  - `transferFunc`: A 1D texture mapping scalar values to color and opacity.
  - `h`: Uniform float specifying the ray step size.
- **Outputs:**
  - `destTex`: A write-only 2D image (RGBA8 format) storing the final rendered color and alpha for each pixel.

**Data Structures:**
- `firstHit` and `lastHit` are sampled using `sampler2D`.
- `volume` is sampled using `sampler3D`.
- `transferFunc` is sampled using `sampler1D`.
- `destTex` is accessed via `imageStore` using `image2D`.

**Algorithm Steps:**
1.  **Initialization:** The compute shader kernel is launched for every pixel in the output resolution. Each thread identifies its global position using `gl_GlobalInvocationID.xy`.
2.  **Boundary Check:** The thread checks if its position is within the bounds of the output texture dimensions. If not, the kernel exits.
3.  **Coordinate Flip:** The Y-coordinate of the store position is flipped to match OpenGL's texture coordinate system: $storePos.y = size.y - storePos.y$.
4.  **Ray Definition:**
    - Fetch the entry point ($first$) from `firstHit` and the exit point ($last$) from `lastHit` at `storePos`.
    - Calculate the ray direction vector: $\vec{d} = last - first$.
    - Calculate the ray length $D = \|\vec{d}\|$.
    - Normalize the direction: $\vec{d} = \frac{\vec{d}}{\|\vec{d}\|}$.
5.  **State Initialization:**
    - Initialize accumulated color $C = (0.0, 0.0, 0.0, 1.0)$.
    - Initialize current ray position $tr = first$.
    - Calculate step vector: $\vec{step} = \vec{d} \times h$.
6.  **Ray Marching Loop:** Iterate with parameter $t$ from $0$ to $D$ with increment $h$:
    - Sample the scalar field value $s$ from `volume` at position $tr$.
    - Sample the transfer function using $s$ to get color $c_{samp}$ and opacity $\alpha_{samp}$.
    - Adjust opacity using the formula: $\alpha_{samp} = 1.0 - \exp(-0.5 \times \alpha_{samp})$.
    - Apply the under operator for color accumulation:
      $$C_{rgb} \leftarrow C_{rgb} + c_{samp} \cdot \alpha_{samp} \cdot C_{a}$$
      $$C_{a} \leftarrow C_{a} \cdot (1.0 - \alpha_{samp})$$
    - **Early Ray Termination:** If $1.0 - C_{a} > \text{opacityThreshold}$ (where $\text{opacityThreshold} = 0.99$), break the loop.
    - Advance ray position: $tr \leftarrow tr + \vec{step}$.
7.  **Finalization:**
    - Compute final alpha: $C_{a} = 1.0 - C_{a}$.
    - Store the resulting color $C$ into `destTex` at `storePos` using `imageStore`.
8.  **Execution:**
    - Launch the kernel using `glDispatchCompute`.
    - Call `glMemoryBarrier` with `GL_SHADER_IMAGE_ACCESS_BARRIER_BIT` to ensure writes are visible.
    - Rasterize a full-screen quad using `destTex` for final display.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Layout definitions
# layout(binding = 0, rgba8) uniform writeonly image2D destTex;
# layout(binding = 1) uniform sampler1D transferFunc;
# layout(binding = 2) uniform sampler3D volume;
# layout(binding = 3) uniform sampler2D lastHit;
# layout(binding = 4) uniform sampler2D firstHit;
# uniform float h;
# const float opacityThreshold = 0.99;

def compute_shader_main():
    # 1. Get thread position
    store_pos = vec2(gl_GlobalInvocationID.x, gl_GlobalInvocationID.y)
    
    # 2. Get texture dimensions
    size = image_size(destTex)
    
    # 3. Boundary check
    if store_pos.x >= size.x or store_pos.y >= size.y:
        return

    # 4. Flip Y coordinate
    store_pos.y = size.y - store_pos.y
    
    # 5. Fetch entry and exit points
    first = texel_fetch(firstHit, store_pos, 0).xyz
    last = texel_fetch(lastHit, store_pos, 0).xyz
    
    # 6. Calculate ray properties
    direction = last - first
    D = length(direction)
    direction = normalize(direction)
    
    # 7. Initialize state
    color = vec4(0.0, 0.0, 0.0, 1.0)
    tr = first
    ray_step = direction * h
    
    # 8. Ray marching loop
    t = 0.0
    while t <= D:
        # Sample scalar field
        scalar = texture(volume, tr).x
        
        # Sample transfer function
        samp = texture(transferFunc, scalar).rgba
        
        # Calculate adjusted alpha
        samp.a = 1.0 - exp(-0.5 * samp.a)
        
        # Accumulate color and alpha (Under operator)
        # color.rgb += samp.rgb * samp.a * color.a
        # color.a *= (1.0 - samp.a)
        color.rgb += samp.rgb * samp.a * color.a
        color.a = color.a * (1.0 - samp.a)
        
        # Early ray termination
        if (1.0 - color.a) > opacityThreshold:
            break
            
        # Increment ray position
        tr += ray_step
        t += h
        
    # 9. Finalize alpha
    color.a = 1.0 - color.a
    
    # 10. Store result
    image_store(destTex, store_pos, color)
```

**Parameters:**
- `h`: Step size for ray marching. Not given in the sources.
- `opacityThreshold`: Constant set to 0.99.

**Edge Cases and Pitfalls:**
- **Precision:** The loop uses a floating-point counter `t` compared against `D`. Accumulation errors in `t` or `tr` may cause the loop to run slightly longer or shorter than intended.
- **Boundaries:** The explicit check `storePos.x < size.x && storePos.y < size.y` prevents out-of-bounds memory access.
- **Coordinate System:** The Y-flip is critical because OpenGL textures typically have the origin at the bottom-left, while screen coordinates often assume top-left.

## Operating characteristics
- **Time Complexity:** $O(N \cdot \frac{D}{h})$, where $N$ is the number of pixels and $D/h$ is the number of steps per ray.
- **Memory Complexity:** $O(N)$ for the output texture and auxiliary hit-point textures.
- **Performance:** The compute shader approach allows for parallel execution across all pixels. Performance is limited by memory bandwidth when sampling the 3D volume texture and the transfer function.
- **Strengths:** Decouples the ray intersection logic from the fragment shader, allowing for more complex intersection algorithms if needed (though this specific implementation assumes pre-computed hits). It leverages the compute shader's ability to write to arbitrary textures.
- **Limitations:** Requires two passes (one for hit points, one for rendering). The fixed step size `h` may lead to aliasing or slow convergence if not tuned.
- **When to use:** When high parallelism is required and the intersection data is already available.
- **When not to use:** When memory bandwidth is the primary bottleneck and simpler fragment-based approaches might benefit from hardware optimizations.

## Background
This method is described in the paper "A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA" [S1]. It represents an adaptation of the fragment shader volume ray casting technique to the OpenGL Compute Shader API, highlighting the similarities in logic but differences in execution model and memory binding.

## Sources
[S1] A Comparison between GPU-based Volume Ray Casting Implementations: Fragment Shader, Compute Shader, OpenCL and CUDA, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
