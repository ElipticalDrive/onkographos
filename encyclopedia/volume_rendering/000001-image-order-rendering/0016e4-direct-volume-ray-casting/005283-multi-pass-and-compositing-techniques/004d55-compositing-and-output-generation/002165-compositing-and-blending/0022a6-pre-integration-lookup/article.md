# Pre-Integration Lookup

Also called: 2D Pre-Integration, 2D Pre-integration Lookup, 3D Pre-integration, 3D Pre-integration Table

## Overview
Pre-integration lookup is a technique in direct volume ray casting that replaces iterative per-sample accumulation within a volume slab with a single texture fetch from a pre-computed table [S1, unofficial][S2]. It provides optical properties for volume slices assuming linear data evolution within the slab [S1, unofficial]. It sets itself apart from standard ray casting by moving the integration step to a pre-computation phase, and from [Simplified Pre-integration](6186-simplified-pre-integration.md) by accounting for color attenuation within the slab rather than neglecting it [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset, a transfer function mapping scalar values to color and opacity, and ray samples $(s_f, s_m, s_b)$ representing the scalar values at the front, middle, and back of a slab (or entry/exit densities and slab thickness) [S1, unofficial][S2].
*   **Output:** The pre-integrated color $C_{slab}$ and opacity $\alpha_{slab}$ contribution of the slab, which is then composited with other slabs [S2].

**Data Structures:**
*   **Pre-integration Lookup Table:** A texture (2D or 3D) storing pre-calculated optical properties [S1, unofficial][S3].
    *   **3D Table Layout:** Dimensions $256 \times 256 \times 256$ for 8-bit scalar domains. The axes correspond to front sample value ($s_f$), middle sample value ($s_m$), and back sample value ($s_b$) [S2].
    *   **2D Table Layout:** Dimensions dependent on entry density, exit density, and slab thickness. The axes typically correspond to these three parameters [S1, unofficial][S3].
    *   **Stored Values:** Color ($C$) and Opacity ($\alpha$) resulting from the pre-calculated optical properties for the slab [S1, unofficial].

**Steps:**
1.  **Pre-computation (Offline or On-the-fly):**
    *   For a 3D table, render a quadrilateral parallel to the projection plane 256 times, filling the viewport [S2].
    *   In the fragment shader, map screen coordinates to front ($s_f$) and middle ($s_m$) sample values, while the back sample ($s_b$) is provided as a uniform variable [S2].
    *   Compute the optical properties for the slab defined by $s_f, s_m, s_b$ using the pre-calculated functions [S1, unofficial][S2].
    *   Store the result in the 3D texture at coordinate $(s_f, s_m, s_b)$ [S2].
    *   *Note:* 2D tables are preferred on hardware with faster 2D texturing capabilities, as 3D texturing can be slower [S3].

2.  **Rendering (Per Ray):**
    *   For each ray, divide the volume into slabs of thickness $2d$ [S2].
    *   For each slab, retrieve the three scalar samples $(s_f, s_m, s_b)$ from the volume [S2].
    *   Perform a 3D (or 2D) texture lookup using these samples as coordinates to fetch the pre-integrated color $C_{slab}$ and opacity $\alpha_{slab}$ [S2].
    *   Composite the fetched slab contribution using the standard front-to-back compositing equation [S2]:
        $$
        C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_{slab} C_{slab}
        $$
        $$
        \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_{slab}
        $$
    *   Repeat for all slabs along the ray.

**Parameters:**
*   **Slab Thickness:** Defined as $2d$, where $d$ is the distance between samples [S2].
*   **Table Resolution:** $256^3$ for 8-bit data [S2].

**Edge Cases and Pitfalls:**
*   **Hardware Performance:** 3D texture lookups may be slower than 2D lookups on some GPUs, making 2D pre-integration a practical choice despite potential artifacts [S3].
*   **Linear Assumption:** The method assumes linear evolution of data within the slice, which may introduce errors if the data varies non-linearly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_integration_lookup(ray_samples, background_color, lookup_table):
    """
    Renders a ray using pre-integration lookup tables.
    ray_samples: list of (s_f, s_m, s_b) tuples for each slab along the ray.
    background_color: vec3 for the scene background.
    lookup_table: 3D texture or array supporting coordinate-based fetch.
    """
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    
    for s_f, s_m, s_b in ray_samples:
        # Standard hardware texture fetch operation
        C_slab, alpha_slab = texture_fetch(lookup_table, (s_f, s_m, s_b))
        
        transmittance = 1.0 - alpha_acc
        C_acc = C_acc + transmittance * alpha_slab * C_slab
        alpha_acc = alpha_acc + transmittance * alpha_slab
        
        # Early ray termination
        if alpha_acc >= 1.0:
            break
            
    # Composite over background as per standard optical model [G]
    final_color = C_acc + (1.0 - alpha_acc) * background_color
    return final_color, alpha_acc
```

## Subtopics

### [Pre-Integrated Diffuse Shading with Non-Linear Gradient Interpolation](4932-pre-integrated-diffuse-shading-with-non-linear-gradient-inte.md)
This method computes the diffuse component of Blinn-Phong shading in pre-integrated volume rendering by decomposing the integral into four 2D look-up tables weighted by coefficients dependent on light direction and gradient angles. It avoids the prohibitive memory cost of full 4D normal-based pre-integration while reducing shading errors caused by large angles between consecutive sample gradients compared to linear interpolation.

### [Non-Linear Gradient Interpolation for Pre-Integrated Volume Rendering](4933-non-linear-gradient-interpolation-for-pre-integrated-volume-.md)
Non-linear gradient interpolation is a technique for pre-integrated direct volume rendering that reduces shading errors by interpolating gradients between front and back samples along a quadratic curve instead of a linear segment, allowing accurate computation of 2D pre-integration look-up tables. It sets itself apart from linear gradient interpolation by approximating the non-linear path of a normalized gradient vector as a quadratic polynomial, thereby avoiding the energy loss and shading errors inherent in linear interpolation which assumes the normalization term $\eta(t)=1$.

### [Simplified Pre-integration](6186-simplified-pre-integration.md)
Simplified Pre-integration is a faster approximation of full pre-integration used in direct volume ray casting. It solves the performance bottleneck of computing pre-integrated transfer function (TF) lookups by neglecting color attenuation inside volume slabs. It sets itself apart from full pre-integration [Pre-Integration Lookup](2854-pre-integration-lookup.md) by trading accuracy for speed, making it suitable for interactive interfaces where multiple TFs are modified simultaneously.

### [Pre-integrated Volume Rendering at Clip Boundaries](7452-pre-integrated-volume-rendering-at-clip-boundaries.md)
This method corrects rendering artifacts that occur at clip boundaries when using pre-integrated volume rendering with low sampling rates. It computes the optical contribution of the ray segment between a clipping surface and the first volume slice behind it, avoiding the need for interleaved rendering or complex boundary layer models. It differs from sibling methods by specifically addressing the discontinuity in sampling distance at clip planes rather than general gradient interpolation errors.

### Pre-integrated Volume Rendering with Shading
A method for efficiently reconstructing 3D medical images that combines pre-integrated volume rendering with the Shear-Warp algorithm to eliminate aliasing artifacts from under-sampling and improve reconstruction quality via Phong lighting shading [S4, unofficial]. It implements classification coding for volume data by utilizing the correlation of the opacity transfer function (OTF) and a min-max Octree data structure, establishing a new run-length encoded (RLE) data structure for fast reconstruction when OTF and view angle change [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The per-slab cost is reduced to a single texture lookup and compositing step, significantly faster than iterative integration per sample [S2].
*   **Memory Complexity:** Requires storing a large lookup table. For 8-bit data, a 3D table of size $256^3$ is needed [S2].
*   **Measured Performance:**
    *   On ATI 9700 Pro with 5x5 tiles:
        *   spx1 dataset: 609-750 ms (2D), 688-907 ms (3D) [S3].
        *   blunt dataset: 141-391 ms (2D), 344-500 ms (3D) [S3].
        *   f117 dataset: 359-640 ms (2D), 453-719 ms (3D) [S3].
    *   On GeForce 6800 GT with 5x5 tiles:
        *   spx1 dataset: 312-531 ms (2D), 313-547 ms (3D) [S3].
        *   blunt dataset: 109-1094 ms (2D), 109-1032 ms (3D) [S3].
        *   f117 dataset: 266-563 ms (2D), 172-438 ms (3D) [S3].
    *   Ray casting passes (2D pre-integration):
        *   spx1: Min 571, Max 601 [S3].
        *   blunt: Min 341, Max 439 [S3].
        *   f117: Min 571, Max 601 [S3].
*   **Strengths:** Reduces aliasing artifacts from under-sampling; improves shading accuracy; suitable for interactive rendering when combined with tiling [S3][S4, unofficial].
*   **Limitations:** High memory usage for 3D tables; 2D tables may introduce artifacts if not carefully designed; assumes linear data evolution within slabs [S1, unofficial][S3].
*   **When to Use:** When fast texture lookups are available and interactive medical image reconstruction is required [S1, S4, unofficial].
*   **When Not to Use:** When memory is constrained and 3D tables are too large; when non-linear data variation within slabs is significant [S1, unofficial].

## Background
Pre-integration lookup was introduced to address the inaccuracies of linear interpolation in volume rendering [S1, unofficial]. It is based on the principle that optical properties can be pre-calculated for volume slices assuming linear data evolution [S1, unofficial]. The method has been implemented on GPU hardware [S1, unofficial]. Variants include 2D and 3D lookup tables, with 2D often preferred for performance on certain hardware [S3]. It has been applied to medical image reconstruction [S4, unofficial] and implemented on GPU hardware [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
[S3] PDF GPU-based Tiled Ray Casting using Depth Peeling, https://sci.utah.edu/~csilva/papers/gpu_volume_ray_casting.pdf, fetched 2026-10-29
[S4] The Perspective Shear-Warp Algorithm in a Virtual Environment., https://www.researchgate.net/publication/220943822_The_Perspective_Shear-Warp_Algorithm_in_a_Virtual_Environment, fetched 2026-10-29 (unofficial)
