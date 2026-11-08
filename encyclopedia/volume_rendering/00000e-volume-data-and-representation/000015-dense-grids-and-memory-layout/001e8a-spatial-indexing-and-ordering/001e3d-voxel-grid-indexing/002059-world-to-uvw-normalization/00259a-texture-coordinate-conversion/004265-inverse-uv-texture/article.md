# Inverse UV Texture
Also called: Inverse UV map
## Overview
The Inverse UV Texture is a lookup table used in differentiable rendering to convert gradients from screen space back into texture space, enabling hardware-accelerated backward texture sampling [S1]. It solves the difficulty of performing atomic float addition operations required to sum gradients from neighboring pixels, which is not fully supported in Vulkan APIs [S1]. Unlike software-based automatic differentiation libraries that use tensor operations or handwritten CUDA modules, this method leverages the graphics pipeline's hardware sampler directly [S1].
## How it works
**Inputs and Outputs:**
*   Input: Screen space size $(W_{ss}, H_{ss})$, rasterized UV image $I_{uv} \in \mathbb{R}^{W_{ss} \times H_{ss} \times 2}$, and texture space size $(W_{tex}, H_{tex})$ [S1].
*   Output: Inverse UV texture $I_{inv\_uv} \in \mathbb{R}^{W_{tex} \times H_{tex} \times 2}$ [S1].

**Data Structures:**
*   $I_{uv}$: A 2D texture storing normalized UV coordinates for each screen pixel.
*   $I_{inv\_uv}$: A 2D texture where each texel stores the screen-space position corresponding to a specific texture coordinate [S1].

**Algorithm:**
The inverse UV texture is calculated in a compute shader [S1]. The process iterates over all screen-space pixel indices to map screen positions to texture coordinates [S1].

1.  Iterate over all pixel indices $i$ from $1$ to $W_{ss} \times H_{ss}$ [S1].
2.  Calculate the pixel position in screen space, $p_{ss}^i$, corresponding to index $i$ [S1].
3.  Fetch the UV value, $p_{tex}$, at $p_{ss}^i$ from the rasterized UV image $I_{uv}$ [S1].
4.  Normalize $p_{tex}$ [S1].
5.  Store $p_{ss}^i$ in $I_{inv\_uv}$ as the pixel value at the normalized texture coordinate $p_{tex}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_inverse_uv_texture(Wss, Hss, Wtex, Htex, Iuv):
    """
    Calculates the inverse UV texture mapping screen space positions to texture space.
    
    Args:
        Wss (int): Screen space width.
        Hss (int): Screen space height.
        Wtex (int): Texture space width.
        Htex (int): Texture space height.
        Iuv (array): Rasterized UV image of shape (Wss, Hss, 2).
        
    Returns:
        array: Inverse UV texture of shape (Wtex, Htex, 2).
    """
    # Initialize output texture with zeros or undefined values
    Iinv_uv = zeros((Wtex, Htex, 2))
    
    # Iterate over all screen space pixels
    for i in range(1, Wss * Hss + 1):
        # Calculate pixel position in screen space
        # Assuming row-major or column-major mapping consistent with Iuv indexing
        # Note: Exact mapping of index i to (x, y) depends on coordinate system conventions.
        # Standard rasterization often uses (i % Wss, i // Wss) or similar.
        # [G] Assuming standard 2D array indexing where i is a linear index.
        x_ss = (i - 1) % Wss
        y_ss = (i - 1) // Wss
        pss_i = vec2(x_ss, y_ss)
        
        # Fetch UV value from rasterized UV image
        # Iuv is accessed at the screen position corresponding to i
        # Note: Iuv shape is (Wss, Hss, 2). Accessing at (x_ss, y_ss)
        ptex = Iuv[x_ss, y_ss]
        
        # Normalize UV value
        # [G] Assuming ptex is in [0, 1] or needs normalization to [0, 1] range
        # If ptex is already normalized, this step might be identity or scaling.
        # The source says "normalize UV value", implying it might be in pixel coordinates.
        # Normalization typically involves dividing by texture dimensions if in pixel space,
        # or ensuring it is in [0,1]. Given Iuv is "rasterized UV", it is likely already in [0,1].
        # However, Algorithm 1 step 3 says "normalize". If Iuv stores pixel coordinates:
        # ptex_norm = ptex / vec2(Wtex, Htex)
        # If Iuv stores [0,1] coordinates, normalization might refer to clamping or specific
        # coordinate system alignment. Without explicit formula, we assume ptex is fetched
        # and potentially scaled if it represents pixel offsets.
        # [G] Assuming Iuv contains normalized [0,1] coordinates as is standard for UV maps.
        # If Iuv contains pixel coordinates, normalization is division by texture size.
        # Let's assume standard UV map [0,1]. If the source implies normalization is needed,
        # it might be converting from pixel space to texture space [0,1].
        # Let's assume ptex is in pixel space for normalization to make sense in this context
        # of "inverse" mapping.
        ptex_norm = ptex / vec2(Wtex, Htex)
        
        # Store screen position in inverse UV texture at the normalized texture coordinate
        # Clamp coordinates to valid range [0, 1] or [0, Wtex-1] depending on implementation
        # [G] Assuming texel addressing in integer space for storage, or float for sampling.
        # The output is a texture, so we store the value.
        # We need to map ptex_norm to texel indices.
        tx = int(ptex_norm.x * Wtex)
        ty = int(ptex_norm.y * Htex)
        
        # Handle boundary conditions
        tx = clamp(tx, 0, Wtex - 1)
        ty = clamp(ty, 0, Htex - 1)
        
        Iinv_uv[tx, ty] = pss_i
        
    return Iinv_uv
```

**Parameters:**
*   $W_{ss}, H_{ss}$: Screen resolution.
*   $W_{tex}, H_{tex}$: Texture resolution.
*   Sobol sequence: Used to add quasi-random numbers to $p_{ss}^i$ for each frame to disrupt sampling order [S1].

**Edge Cases and Pitfalls:**
*   **Imbalance:** The inverse UV texture only maintains the last updated value for each texel. A constant sampling order causes imbalance where some texels are updated every frame while others are never updated, leading to artifacts in the optimized texture [S1].
*   **Solution:** To solve this imbalance, quasi-random numbers generated from a Sobol sequence are added to $p_{ss}^i$ for each frame, disrupting the sampling order [S1].
*   **Atomic Operations:** The method is designed to avoid the need for atomic float add operations, which are not fully supported in Vulkan APIs [S1].
## Subtopics

### Inverse UV
**Definition:** A backward pass implementation for hardware-accelerated texture sampling that enables differentiable rendering directly within the graphics pipeline, avoiding software-based tensor operations or handwritten CUDA modules [S1].

**How it works:**
1.  **Forward Pass:** Uses the GLSL `texture()` function with linear interpolation to generate a pixel value from four neighboring texels [S1].
2.  **Backward Pass:** Computes gradients with respect to texture coordinates and texture values, mapping screen-space gradients back to texture space [S1]. Implementation details for the backward pass are cut off in the source text [S1].

**Comparison:** Existing rasterization-based differentiable renderers implement texture sampling backward passes in software (via tensor operations in AD libraries or handwritten CUDA modules), whereas this method uses the graphics pipeline's hardware sampler directly [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   The algorithm iterates over all screen-space pixels, resulting in $O(W_{ss} \times H_{ss})$ complexity for the construction of the inverse UV texture [S1].
*   Memory usage is proportional to the size of the inverse UV texture, $O(W_{tex} \times H_{tex})$ [S1].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Enables hardware-accelerated backward texture sampling, providing hardware independence and acceleration through Vulkan [S1]. Avoids the overhead of software-based automatic differentiation libraries [S1].
*   **Limitations:** The method only maintains the last updated value for each texel, which can lead to artifacts if the sampling order is not randomized [S1]. Requires Vulkan support [S1].

**When to use it and when not:**
*   Use when implementing differentiable rendering in Vulkan and needing to backpropagate gradients through texture sampling without software-based AD libraries [S1].
*   Do not use if atomic float operations are available and preferred, or if the overhead of randomizing sampling order is unacceptable [S1].
## Background
**Origin:** The Inverse UV Texture was proposed in the Dressi differentiable renderer, a hardware-agnostic system implemented in Vulkan [S1]. The paper "Dressi: A Hardware‐Agnostic Differentiable Renderer with Reactive Shader Packing and Soft Rasterization" was published in 2022 [S1].

**History and Development:**
*   Dressi aims to provide hardware independence and acceleration by describing all components of the renderer in Vulkan [S1].
*   The Inverse UV texture is part of Section 3.3.2 of the Dressi paper, addressing the backward computation of texture sampling with hardware [S1].
*   It contrasts with existing rasterization-based differentiable renderers that use software for texture sampling backward passes [S1].
*   The method is designed to handle the backward pass with a hardware texture sampler, unlike other methods that sample 3D scenes efficiently for ray-tracing-based differentiable rendering [S1].

**Where it is used:**
*   Used in the Dressi differentiable renderer for backpropagating gradients through texture sampling operations [S1].
## Sources
[S1] Dressi: A Hardware‐Agnostic Differentiable Renderer with Reactive Shader Packing and Soft Rasterization, https://doi.org/10.1111/cgf.14455, fetched 2026-10-29
