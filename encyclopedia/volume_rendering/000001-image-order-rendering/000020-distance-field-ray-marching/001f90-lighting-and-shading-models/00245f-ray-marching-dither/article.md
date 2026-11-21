# Ray Marching Dither

Also called: UV Dithering, Distance-based UV Perturbation

## Overview
Ray Marching Dither is a technique to reduce banding artifacts in ray marching by perturbing the UV coordinates based on the ray distance [S1, unofficial]. It sets itself apart from [Ray Marching Jittering](5399-ray-marching-jittering.md) by applying perturbation to the UV coordinates rather than the step size, and from [Distance Color Blending](791-distance-color-blending.md) by focusing on geometric sampling stability rather than color interpolation.

## How it works
**Inputs and Outputs:**
*   Input: Current UV coordinates (`uv`), ray distance (`t`), dithering strength factor (`c_fDither`), and a pseudo-random number generator function (`rand`).
*   Output: Perturbed UV coordinates.

**Data Structures:**
*   `uv`: `vec2` representing the current UV coordinate.
*   `t`: `float` representing the ray distance (hit distance).
*   `c_fDither`: `float` parameter controlling dithering intensity.

**Steps:**
1.  Calculate the dither amount based on the squared ray distance and the strength factor:
    $$ \text{dither} = t^2 \cdot c\_fDither $$
2.  Generate two random values using the `rand` function with specific offsets derived from the original UV coordinates:
    $$ r_x = \text{rand}(uv + \text{vec2}(0.12, 0.32)) $$
    $$ r_y = \text{rand}(uv + \text{vec2}(0.42, 0.52)) $$
3.  Perturb the UV coordinates by adding the random value scaled by the dither amount, then subtracting half the dither amount to center the perturbation:
    $$ uv.x += r_x \cdot \text{dither} - \frac{\text{dither}}{2.0} $$
    $$ uv.y += r_y \cdot \text{dither} - \frac{\text{dither}}{2.0} $$
4.  Reconstruct the ray origin and direction from the perturbed UVs to continue the ray march [S1, unofficial].

**Parameters:**
*   `c_fDither`: Dithering strength factor. Typical value: `0.001` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_marching_dither(uv, t, c_fDither, rand_func):
    """
    Perturbs UV coordinates based on ray distance to reduce banding.
    
    Args:
        uv (vec2): Current UV coordinates.
        t (float): Ray distance (hit distance).
        c_fDither (float): Dithering strength factor (e.g., 0.001).
        rand_func (function): Pseudo-random number generator taking a vec2 input.
        
    Returns:
        vec2: Perturbed UV coordinates.
    """
    # Step 1: Calculate dither amount
    dither = t * t * c_fDither
    
    # Step 2: Generate random values with specific offsets
    # Offsets are hardcoded as per source implementation
    offset_x = vec2(0.12, 0.32)
    offset_y = vec2(0.42, 0.52)
    
    r_x = rand_func(uv + offset_x)
    r_y = rand_func(uv + offset_y)
    
    # Step 3: Perturb UVs
    # Add random component scaled by dither, then subtract half dither to center
    uv.x += r_x * dither - dither / 2.0
    uv.y += r_y * dither - dither / 2.0
    
    # Step 4: Return perturbed UVs
    # Note: Reconstructing ray origin/direction from UVs is handled by the caller
    # using the standard camera ray generation formula.
    return uv
```

**Edge Cases and Pitfalls:**
*   **Precision:** For very large `t`, `t^2` may cause floating-point precision issues or overflow. [G]
*   **Boundary Conditions:** If `uv` is near texture boundaries, perturbation might sample outside valid ranges. [G]
*   **Determinism:** The `rand` function must be deterministic for the same input `uv` to avoid flickering if not intended. [G]

## Subtopics

## Operating characteristics
*   **Time Complexity:** O(1) per pixel per step, as it involves simple arithmetic and a random function call.
*   **Memory Complexity:** O(1), no additional data structures required.
*   **Strengths:** Simple to implement, effective at reducing visible banding artifacts in ray marching.
*   **Limitations:** May introduce slight blurring or noise if the dither strength is too high. Not suitable for all types of artifacts (e.g., step-size related aliasing is better handled by [Ray Marching Jittering](5399-ray-marching-jittering.md)).
*   **When to Use:** When banding artifacts are visible due to fixed step sizes or quantization in UV space.
*   **When Not to Use:** When high precision is required without any perturbation, or when step-size aliasing is the primary issue.

## Background
Originated from practical implementations in WebGL ray marching, as seen in the [WebGL-RayMarch](https://github.com/nopjia/WebGL-RayMarch) repository [S1, unofficial]. It is a technique used to improve the visual quality of ray-marched scenes by mitigating banding.

## Sources
[S1] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
