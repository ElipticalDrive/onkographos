# RGB Cube AVIP

Also called: AVIPRGB

## Overview
RGB Cube AVIP is a direct volume ray casting technique that maps the 3D spatial coordinates of the current voxel directly to the RGB color channels during the accumulation process, providing a qualitative visualization of spatial and spectral coordinates [S1]. It sets itself apart from standard transfer functions by eliminating the need for a scalar-to-color mapping; instead, the geometry of the volume dictates the color, allowing the viewer to perceive the 3D structure and position of data within the volume through color alone [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The spatial location of the current voxel along the ray ($loc = (x, y, z)$), the scalar voxel value ($val$), the accumulated fragment state ($tempFrag$), and a weighting factor ($k$) [S1].
*   **Output:** The updated accumulated fragment ($tempFrag$) containing the blended color and opacity [S1].

**Data Structures:**
*   $tempFrag$: A structure or vector containing color components ($r, g, b$) and an alpha component ($a$).
*   $loc$: A 3D coordinate vector $(x, y, z)$ representing the current position in the volume.
*   $val$: The scalar value of the voxel at $loc$.
*   $k$: An arbitrary weighting factor controlling opacity contribution [S1].
*   $minVal$: An arbitrary small value greater than 0, used to prevent division by zero or numerical instability [S1].

**Parameters:**
*   $k$: Weighting factor. Value not specified in sources [G].
*   $minVal$: Small positive constant. Value not specified in sources [G].
*   Termination threshold: 0.99. If accumulated alpha exceeds this, the ray is terminated [S1].

**Algorithm Steps:**
1.  **Color Mapping:** Map the spatial coordinates to the color channels:
    *   $color.r \leftarrow loc.z$
    *   $color.g \leftarrow loc.y$
    *   $color.b \leftarrow loc.x$
    [S1]
2.  **Alpha Calculation:**
    *   Retrieve the current accumulated alpha: $\alpha_1 \leftarrow tempFrag.a$ [S1].
    *   Calculate the new alpha contribution: $\alpha_2 \leftarrow val \times k \times (1 - \alpha_1)$ [S1].
    *   Calculate the new total alpha: $\alpha \leftarrow \max(\alpha_1 + \alpha_2, minVal)$ [S1].
3.  **Color Blending:** Update the color component of $tempFrag$ by blending the previous color with the new voxel color, weighted by their respective alpha contributions relative to the new total alpha:
    $$ tempFrag \leftarrow tempFrag \times \frac{\alpha_1}{\alpha} + color \times \frac{\alpha_2}{\alpha} $$
    [S1]
4.  **Alpha Update:** Update the alpha channel of the fragment:
    $$ tempFrag.a \leftarrow \alpha $$
    [S1]
5.  **Termination Check:** If $\alpha > 0.99$, set the ray iteration flag $i$ to $N$ (indicating the end of the ray) and return $tempFrag$ [S1].
6.  **Return:** If the ray is not terminated, return $tempFrag$ to continue accumulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transferFunction(loc, val, tempFrag, k, minVal):
    # 1. Map spatial coordinates to RGB color
    color_r = loc.z
    color_g = loc.y
    color_b = loc.x
    
    # 2. Calculate alphas
    alpha1 = tempFrag.a
    alpha2 = val * k * (1 - alpha1)
    alpha = max(alpha1 + alpha2, minVal)
    
    # 3. Blend color
    # Note: tempFrag is assumed to have r, g, b, a components
    tempFrag.r = tempFrag.r * (alpha1 / alpha) + color_r * (alpha2 / alpha)
    tempFrag.g = tempFrag.g * (alpha1 / alpha) + color_g * (alpha2 / alpha)
    tempFrag.b = tempFrag.b * (alpha1 / alpha) + color_b * (alpha2 / alpha)
    
    # 4. Update alpha
    tempFrag.a = alpha
    
    # 5. Termination check
    if alpha > 0.99:
        # Set flag to end ray (N)
        return tempFrag, True # True indicates termination
    
    return tempFrag, False
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of $minVal$ prevents division by zero in the blending step if both $\alpha_1$ and $\alpha_2$ are zero or extremely small [S1].
*   **Boundaries:** The method assumes $loc$ coordinates are normalized or scaled appropriately to fit within the RGB color space (typically [0, 1]). If coordinates are not normalized, colors will be incorrect or clipped [G].
*   **Qualitative Nature:** This method provides no quantitative color mapping (no color bar); it is purely for structural visualization [S1].

## Subtopics

### First moment-inspired AVIP
A variant of AVIP0 where the color is determined by the spatial coordinate (specifically $z$) of the voxel, while the weighting for opacity and blending uses the voxel intensity ($val$) [S1]. It is chosen when one wishes to visualize the variation along a single spatial axis (e.g., depth or redshift) while maintaining opacity based on data density or intensity [S1]. Unlike RGB Cube AVIP, it does not encode all three spatial dimensions in color, making it easier to evaluate specific coordinate values but less informative about 3D structure [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, identical to standard volume ray casting, as the color calculation is a constant-time operation per sample [S1].
*   **Memory Complexity:** Depends on the storage of the 3D texture. For a typical commodity GPU with 2 GB VRAM, it supports up to a $1024^3$ voxel spectral cube [S1].
*   **Strengths:** Provides immediate visual feedback on the 3D spatial distribution of the data without requiring a transfer function lookup [S1]. It is interactive and suitable for in-situ visualization on GPUs [S1].
*   **Limitations:** It is a purely qualitative visualization; users cannot easily determine the scalar value of a voxel from its color [S1]. Evaluating exact spatial coordinates of a pixel in the final image can be difficult because the color is a composite of the blending process, not a direct mapping of a single voxel's position [S1].
*   **When to use:** When the goal is to understand the 3D shape and location of a spectral cube or volume data set quickly.
*   **When not to use:** When quantitative analysis of scalar values is required, or when a specific color mapping (e.g., for material identification) is needed.

## Background
RGB Cube AVIP was introduced as a generalization of MIP1 and AVIP1 transfer functions, which mapped a single spatial coordinate (like velocity or redshift) to color [S1]. The RGB Cube approach exploits the 3D nature of the RGB color space to assign a unique color to every 3D coordinate of a spectral cube, thereby providing information about both spatial and spectral coordinates simultaneously [S1]. It is used in the context of real-time coloring and filtering with graphics shaders, particularly for in-situ visualization of spectral cube data loaded into GPU memory [S1].

## Sources
[S1] Real-time colouring and filtering with graphics shaders, https://doi.org/10.1093/mnras/stx1676, fetched 2026-10-26
