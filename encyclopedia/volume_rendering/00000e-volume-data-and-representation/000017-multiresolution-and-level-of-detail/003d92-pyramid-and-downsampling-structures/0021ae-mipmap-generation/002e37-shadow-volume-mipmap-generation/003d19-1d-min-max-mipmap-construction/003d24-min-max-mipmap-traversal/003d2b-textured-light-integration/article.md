# Textured Light Integration

Also called: Rectified Light Texture Integration

## Overview
Textured Light Integration extends Min-Max Mipmap Traversal to handle spatially varying light sources, such as stained-glass windows, by incorporating a rectified light texture into the volume integration process [S1]. It differs from standard integration by multiplying the transmittance term by a texture function $T(\alpha, \gamma)$, requiring precomputed prefix sums of the product $\Gamma_i(\gamma)T(\alpha, \gamma)$ rather than just $\Gamma_i(\gamma)$ [S1]. This method builds on [Min-Max Mipmap Traversal](9419-min-max-mipmap-traversal.md) to identify lit segments while accounting for texture modulation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray parameters $(x, y)$, depth $D[x, y]$, min-max mipmap data (min/max bounds per node), precomputed prefix sums of $\Gamma_i(\gamma)T(\alpha, \gamma)$, and the rectified light texture $T(\alpha, \gamma)$ [S1].
*   **Output:** Radiance $L(\alpha, \gamma)$ for the ray [S1].

**Data Structures:**
*   **Light Texture $T(\alpha, \gamma)$:** A 2D texture where $\alpha$ is the epipolar angle and $\gamma$ is the depth coordinate. The texture is rectified to align with the epipolar geometry [S1].
*   **Prefix Sum Array:** Stores cumulative sums of $\Gamma_i(\gamma)T(\alpha, \gamma)$ for each $\alpha$ slice [S1].

**Steps:**
1.  **Rectification:** Rectify the light texture map to obtain $T(\alpha, \gamma)$, aligning it with the epipolar coordinate system [S1].
2.  **Precomputation:** Compute prefix sums of the product $\Gamma_i(\gamma)T(\alpha, \gamma)$ for all $\alpha$. This requires $O(\log_2 t)$ ping-pong passes, where $t$ is the $\gamma$-resolution of the rectified light texture [S1].
3.  **Traversal:** Perform the iterative traversal defined in [Min-Max Mipmap Traversal](9419-min-max-mipmap-traversal.md) to identify lit segments [S1].
4.  **Integration:** For each lit segment, compute the integral using the formula below. Delay the integral update until encountering the next unlit node or the end of traversal to avoid adding/subtracting cancelling terms [S1].

**Formula:**
The radiance $L$ is approximated by:
$$L(\alpha, \gamma) \approx \sum_i^N \left( B_i(\beta) \sum_{\substack{\gamma < D(\alpha, \beta) \\ S(\alpha, \gamma) > \beta}} \Gamma_i(\gamma) T(\alpha, \gamma) \Delta \gamma \right)$$
where:
*   $N$ is the number of light sources or components [S1].
*   $B_i(\beta)$ is the basis function or attenuation factor for component $i$ at depth $\beta$ [S1].
*   $\Gamma_i(\gamma)$ is the transmittance or scattering term for component $i$ at depth $\gamma$ [S1].
*   $T(\alpha, \gamma)$ is the rectified light texture value at epipolar angle $\alpha$ and depth $\gamma$ [S1].
*   $S(\alpha, \gamma)$ is the shadow value (min-max mipmap result) at $\alpha, \gamma$ [S1].
*   $\Delta \gamma$ is the depth step size [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def to_epipolar(x, y, depth):
    # [G] Standard epipolar coordinate transform from screen space to (alpha, beta, gamma1)
    return alpha, beta, gamma1

def range(node):
    # [G] Returns (gamma_min, gamma_max) for the given tree node index
    return gamma_min, gamma_max

def integrate_textured_light(x, y, depth, mipmap, prefix_sums, light_texture):
    alpha, beta, gamma1 = to_epipolar(x, y, depth)
    node = 1
    gamma_min, gamma_max = range(node)
    out = [0.0, 0.0, 0.0]
    pending = 0.0
    while node is not None:
        min_val = mipmap.min[node, alpha]
        max_val = mipmap.max[node, alpha]
        if min_val <= beta < max_val:
            node = node * 2
            gamma_min, gamma_max = range(node)
        else:
            if beta < min_val:
                pending += prefix_sums[alpha][min(gamma1, gamma_max)] - prefix_sums[alpha][gamma_min]
            node = node + 1
            if node % 2 == 0:
                node = node // 2
            else:
                out[0] += pending
                out[1] += pending
                out[2] += pending
                pending = 0.0
    out[0] += pending
    out[1] += pending
    out[2] += pending
    return out
```
*Note: The pseudocode implements the delay optimization to avoid cancelling terms, critical for performance with textured lights [S1].*

**Parameters:**
*   $t$: $\gamma$-resolution of the rectified light texture. Typical value: 512 [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Accessing 24 floats (3 color channels * 4 singular vectors * 2 endpoints) per integration step increases memory bandwidth pressure compared to untextured lights (8 floats) [S1].
*   **Error:** Using lower-resolution light textures introduces error, but it is not noticeable in the final image [S1].

## Subtopics

### Low-Resolution Light Texture Approximation
This variant performs rectification and prefix sum computation at a lower resolution than the shadow map (e.g., 512x512 for light texture vs. higher for shadow map) [S1]. The resulting lower-resolution texture is accessed using hardware linear interpolation during integration [S1]. This reduces precomputation time and memory usage [S1]. The introduced error is not noticeable in the final image [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation requires $O(\log_2 t)$ ping-pong passes for prefix sums, where $t$ is the $\gamma$-resolution [S1]. Traversal complexity is similar to Min-Max Mipmap Traversal [S1].
*   **Memory Complexity:** Higher than untextured lights due to the need to store prefix sums for $\Gamma_i(\gamma)T(\alpha, \gamma)$ for each $\alpha$ slice. Accessing 24 floats per step vs. 8 floats for untextured lights increases memory bandwidth requirements [S1].
*   **Performance:** Tested on an Intel Core i7 960 (3.2 GHz) with an NVIDIA GeForce 480 GTX GPU at 1280 × 960 resolution [S1].
*   **Strengths:** Enables realistic rendering of textured light sources like stained-glass windows [S1].
*   **Limitations:** Increased memory bandwidth usage and precomputation time due to texture integration [S1].

## Background
Textured Light Integration was introduced in "Real-Time Volumetric Shadows using 1D Min-Max Mipmaps" by the MIT Computer Science and Artificial Intelligence Laboratory (CSAIL) [S1]. It extends the 1D Min-Max Mipmap method to handle complex lighting scenarios involving spatially varying light sources [S1].

## Sources
[S1] Real-Time Volumetric Shadows using 1D Min-Max Mipmaps, https://groups.csail.mit.edu/graphics/mmvs/mmvs.pdf, fetched 2026-10-28
