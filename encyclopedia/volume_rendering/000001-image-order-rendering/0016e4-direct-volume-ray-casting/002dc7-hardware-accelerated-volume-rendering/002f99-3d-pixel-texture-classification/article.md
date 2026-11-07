# 3D Pixel Texture Classification
Also called: VGH Lookup
## Overview
3D Pixel Texture Classification is a hardware-accelerated volume rendering technique that maps multi-dimensional voxel attributes to color and opacity using a 3D texture lookup. It solves the problem of efficient, real-time classification by offloading the transfer function evaluation to the GPU's texture hardware, avoiding expensive shader computations for every sample [S1]. It differs from standard 1D transfer functions by incorporating gradient magnitude and second derivative information into the lookup coordinates, allowing for boundary-aware rendering [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume $V$. For each sample point $\mathbf{p}$ along a ray, the method requires the scalar value $V(\mathbf{p})$, the gradient magnitude $|\nabla V(\mathbf{p})|$, and the second derivative (Laplacian or directional) $D_2(\mathbf{p})$.
*   **Output:** A 4D color/opacity vector $(R, G, B, \alpha)$ retrieved from the 3D pixel texture.

**Data Structures:**
*   **3D Pixel Texture:** A 3D texture resource with dimensions $D \times G \times S$.
    *   Axis 1 (Data Value): Resolution $D = 256$ (8-bit).
    *   Axis 2 (Gradient Magnitude): Resolution $G = 256$ (8-bit).
    *   Axis 3 (Second Derivative): Resolution $S$ is limited (e.g., small integer). The texture stores "sheets" of opacity values.
    *   Internal Format: Typically RGBA8, storing the final color and opacity.

**Steps:**
1.  **Ray Marching:** Generate rays per pixel. Sample the volume at step size $\Delta s$.
2.  **Attribute Computation:** For each sample point $\mathbf{p}$:
    *   Compute scalar value $v = V(\mathbf{p})$.
    *   Compute gradient magnitude $g = |\nabla V(\mathbf{p})|$.
    *   Compute second derivative $d_2 = D_2(\mathbf{p})$.
3.  **Coordinate Transformation:**
    *   Map $v$ to texture coordinate $u \in [0, 1]$ using 8-bit quantization: $u = v / 255$.
    *   Map $g$ to texture coordinate $v_{tex} \in [0, 1]$ using 8-bit quantization: $v_{tex} = g / 255$.
    *   Map $d_2$ to texture coordinate $w_{tex} \in [0, 1]$. Since $d_2$ is signed, apply a scale and bias to preserve the sign for interpolation:
        $$ w_{tex} = \frac{d_2 - d_{min}}{d_{max} - d_{min}} $$
        where $d_{min}$ and $d_{max}$ are the bounds of the second derivative range. The limited resolution $S$ implies that the texture is structured as discrete sheets.
4.  **Texture Lookup:** Perform a trilinear texture fetch:
    $$ \mathbf{c} = \text{tex3D}(T, u, v_{tex}, w_{tex}) $$
    where $T$ is the 3D pixel texture. The result $\mathbf{c}$ contains the color $(R, G, B)$ and opacity $\alpha$.
5.  **Compositing:** Use the retrieved $\alpha$ and color in the front-to-back compositing equation (see Parent).

**Parameters:**
*   **Boundary Emphasis:** A scalar parameter $b \in [0, 1]$ that controls the opacity of the outer sheets (positive/negative second derivatives) relative to the center sheet (zero second derivative).
*   **Second Derivative Resolution:** $S$ is typically small (e.g., 3-5) to keep texture size manageable.

**Edge Cases and Pitfalls:**
*   **Precision:** Full 8-bit resolution is used for Data Value and Gradient Magnitude to ensure smooth transitions. Limited resolution for the Second Derivative axis reduces texture memory but may cause quantization artifacts if $S$ is too low [S1].
*   **Sign Preservation:** The second derivative must be scaled and biased to maintain its sign. Failure to do so results in incorrect interpolation between positive and negative values, as the texture coordinates would not reflect the signed nature of the derivative [S1].
*   **Texture Size:** A full 8-bit resolution for all three axes would require $256^3$ entries, which is prohibitively large for early hardware. The method explicitly limits the second derivative axis to avoid this [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume T is a 3D texture of size D x G x S
# D=256, G=256, S is small (e.g., 3)
# boundary_emphasis is a uniform parameter

def classify(v, grad_mag, second_deriv, boundary_emphasis):
    # 1. Map Data Value to [0, 1]
    u = clamp(v / 255.0, 0.0, 1.0)
    
    # 2. Map Gradient Magnitude to [0, 1]
    v_tex = clamp(grad_mag / 255.0, 0.0, 1.0)
    
    # 3. Map Second Derivative to [0, 1]
    # Note: second_deriv is signed. 
    # We assume a known range [d_min, d_max] for the dataset.
    # The texture is structured such that w=0.5 corresponds to zero second derivative.
    # The outer sheets are at w=0.0 and w=1.0 (or similar mapping depending on S).
    # For S=3, sheets might be at 0.0, 0.5, 1.0 representing neg, zero, pos.
    
    # Normalize to [0, 1]
    w_raw = (second_deriv - d_min) / (d_max - d_min)
    w_tex = clamp(w_raw, 0.0, 1.0)
    
    # 4. Lookup
    color_alpha = texture_3d(T, u, v_tex, w_tex)
    
    # 5. Apply boundary emphasis if necessary
    # If the lookup returns base values, scale opacity for non-zero second derivatives
    # This step might be handled by the texture content itself if pre-computed
    # or by a post-lookup multiplication.
    # The source suggests the texture stores "sheets" with linear control.
    # If the texture stores the base opacity for the center sheet, and the outer sheets
    # are scaled by boundary_emphasis, this scaling might be baked into the texture
    # or applied here. Given the description "opacity on outer sheets ... is scaled",
    # it implies the texture lookup returns the appropriate value based on w_tex.
    
    return color_alpha
```
[G] The exact mapping of the second derivative axis to texture coordinates depends on the specific implementation of the "sheets". A common approach for $S=3$ is to map negative values to the first sheet, zero to the middle, and positive to the last. The `boundary_emphasis` parameter likely scales the opacity contribution of the outer sheets relative to the center sheet. If the texture is static, this scaling must be pre-computed for different emphasis values or handled via a secondary lookup/multiplication.

## Subtopics

### Adaptive Pre-integrated Voxel Classification
A technique included in a real-time raycaster built by Roettger et al. to exploit the intrinsically parallel nature of GPUs, along with volume clipping. It adapts the pre-integration concept to voxel classification, optimizing the rendering pipeline by reducing the number of texture lookups and computations per sample. It is chosen when real-time performance is critical and the volume data allows for adaptive sampling strategies. It differs from standard 3D Pixel Texture Classification by incorporating adaptive step sizes and pre-integrated classification results, reducing aliasing and improving performance. [S2]

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. The classification step is $O(1)$ due to hardware texture lookup.
*   **Memory Complexity:** $O(D \times G \times S)$ for the 3D pixel texture. With $D=256, G=256$, and small $S$, this is significantly smaller than a full 3D transfer function texture.
*   **Performance:** Enables real-time rendering on hardware that supports 3D textures. Performance is limited by memory bandwidth and texture fetch latency.
*   **Strengths:** Efficient classification using hardware acceleration; supports multi-dimensional transfer functions; reduces shader complexity.
*   **Limitations:** Texture size constraints limit the resolution of the second derivative axis; requires pre-computation of the texture for different boundary emphasis values if dynamic; less flexible than shader-based classification for complex transfer functions.
*   **When to use:** Real-time volume rendering with boundary-aware classification; hardware with 3D texture support.
*   **When not to use:** When high-resolution multi-dimensional transfer functions are required; when dynamic, complex transfer functions are needed that cannot be pre-computed into a texture.
## Background
The method was introduced in the context of interactive volume rendering using multi-dimensional transfer functions. It addresses the limitation of 1D transfer functions by incorporating gradient and second derivative information, which are crucial for boundary detection. The approach leverages the hardware capabilities of GPUs to perform the classification lookup efficiently. [S1]
## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
