# Compositing

Also called: Composite ARGB

## Overview
Compositing is the process of combining multiple color and opacity contributions into a final pixel value, a step required exclusively for semi-transparent volume rendering [S1, unofficial]. It solves the problem of integrating optical properties along a viewing ray to produce a physically plausible image [S1, unofficial]. Unlike [Alpha Blending](5750-alpha-blending.md), which typically blends two distinct layers, or [Pre-Integration Lookup](2854-pre-integration-lookup.md), which approximates integration via texture lookups, standard compositing explicitly accumulates samples along the ray using the emission-absorption model [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of volume samples $(c_i, \alpha_i)$ encountered along a ray, where $c_i$ is the sample color (RGB) and $\alpha_i$ is the sample opacity (A). The ray originates from the camera and traverses the volume.
*   **Output:** A final pixel color $C_{final}$ and accumulated opacity $\alpha_{final}$.

**Data Structures:**
*   **Accumulators:** Two variables are maintained per ray:
    *   `accumColor`: A 3-component vector (RGB), initialized to $(0, 0, 0)$.
    *   `accumAlpha`: A scalar (A), initialized to $1.0$ (representing 100% transparency/background).
    *   *Note:* Some implementations use a single 4-component `vec4` or `struct` containing both [S2, unofficial].

**Steps:**
1.  **Traverse:** Iterate through volume samples $i = 0 \dots N$ along the ray, from front (nearest to camera) to back (farthest), or vice versa depending on the chosen order.
2.  **Transfer Function:** For each sample position, retrieve the scalar value from the volume, apply a transfer function to determine the sample's color $c_i$ and opacity $\alpha_i$.
3.  **Composite:** Update accumulators based on the traversal order.

**Formulas:**

*Front-to-Back Compositing (Standard):*
This is the most common approach for semi-transparent rendering [S1, unofficial]. It simulates light emission and absorption.

$$
\begin{aligned}
C_{final} &= \sum_{i=0}^{N-1} c_i \cdot \alpha_i \cdot \prod_{j=0}^{i-1} (1 - \alpha_j) \\
\alpha_{final} &= 1 - \prod_{i=0}^{N-1} (1 - \alpha_i)
\end{aligned}
$$

In iterative form (per sample $i$):
$$
\begin{aligned}
C_{final} &\leftarrow C_{final} + c_i \cdot \alpha_i \cdot \alpha_{transmitted} \\
\alpha_{transmitted} &\leftarrow \alpha_{transmitted} \cdot (1 - \alpha_i)
\end{aligned}
$$
Where $\alpha_{transmitted}$ is initialized to $1.0$ and represents the accumulated transparency $\prod (1-\alpha_j)$ [S5, unofficial].

*Back-to-Front Compositing:*
Less common for standard volume rendering but used in specific contexts. The order of accumulation is reversed [S1, unofficial].

**Parameters:**
*   **Step Size:** Determines sample density. Smaller steps increase accuracy but cost more [S4, unofficial].
*   **Thresholds:** Opacity thresholds may be used for early ray termination (if $\alpha_{final} > 1 - \epsilon$, stop) or to ignore negligible samples [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulating many small opacity values can lead to precision errors in $\alpha_{transmitted}$ if it approaches zero. Using `1.0 - alpha` directly is standard, but care must be taken with very small $\alpha$ values [S5, unofficial].
*   **Order Dependency:** The order of compositing matters. Front-to-back is required for the standard emission-absorption model. Back-to-front requires different weighting [S1, unofficial].
*   **Empty Space:** If the ray does not intersect the volume, the output should be the background color (often black or the clear color) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_to_back_compositing(samples):
    """
    Composites volume samples using front-to-back alpha blending.
    
    Args:
        samples: List of tuples (color_rgb, opacity_a) in front-to-back order.
                 color_rgb is a 3-tuple (r, g, b) in [0, 1].
                 opacity_a is a float in [0, 1].
    
    Returns:
        final_color: 3-tuple (r, g, b)
        final_opacity: float
    """
    # Initialize accumulators
    accum_color = (0.0, 0.0, 0.0)
    alpha_transmitted = 1.0
    
    for color_rgb, opacity_a in samples:
        # Skip samples with zero opacity to save computation
        if opacity_a == 0.0:
            continue
            
        # Contribution of current sample
        # c_i * alpha_i * (1 - alpha_0) * ... * (1 - alpha_{i-1})
        contribution = (
            color_rgb[0] * opacity_a * alpha_transmitted,
            color_rgb[1] * opacity_a * alpha_transmitted,
            color_rgb[2] * opacity_a * alpha_transmitted
        )
        
        # Add to accumulated color
        accum_color = (
            accum_color[0] + contribution[0],
            accum_color[1] + contribution[1],
            accum_color[2] + contribution[2]
        )
        
        # Update transmitted alpha
        alpha_transmitted *= (1.0 - opacity_a)
        
        # Early ray termination: if fully opaque, stop
        if alpha_transmitted < 1e-6:
            break
            
    final_color = accum_color
    final_opacity = 1.0 - alpha_transmitted
    
    return final_color, final_opacity
```

## Subtopics

### [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md)
Projects a semi-transparent 3D scalar field onto a 2D image plane by integrating optical properties along viewing rays, eliminating the need for binary classification of the data. It differs from Maximum Intensity Projection (MIP) by preserving internal texture and visibility through opacity-weighted accumulation rather than selecting only the highest intensity value.

### [Ray Compositing and Accumulation](3449-ray-compositing-and-accumulation.md)
The process of combining color and opacity contributions from volume samples into final pixel values for semi-transparent rendering. It solves the problem that volume samples are often generated in slice-by-slice or beam-by-beam order rather than strict ray order, requiring buffering to maintain correct compositing sequence. Unlike Alpha Compositing Ray Casting, which assumes a linear ray traversal, this method handles arbitrary view directions and angled rays by using hardware buffers to align predecessor values for each sample.

### Average Compositing
A compositing scheme that produces an X-ray picture by taking no opacity into account. It sends a ray through the volume and composites all found values with the same weight [S3, unofficial].
1. Send a ray through the volume.
2. Composite all found values with the same weight [S3, unofficial].

### Raymarch Blending
Combines the raymarched result with the existing scene texture using standard alpha blending [S4, unofficial].
1. Sample the existing scene color from `_MainTex` at the current UV.
2. Call `raymarch(ro, rd)` to get the raymarched color `add`.
3. Blend the colors: `col * (1.0 - add.w) + add.xyz * add.w` [S4, unofficial].

### Front-to-Back Alpha Blending
A projection technique that composites samples along the ray using alpha blending until the ray exits the volume or becomes fully opaque [S5, unofficial].
1. Initialize `contribution = 1.0f`.
2. Composite each sample: `outputColor.rgb += contribution * transfer.a * transfer.rgb`.
3. Update contribution: `contribution *= (1 - transfer.a)` [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray.
*   **Memory Complexity:** $O(1)$ per ray for accumulators, plus storage for the volume dataset.
*   **Performance:** Highly parallelizable across pixels and rays. Performance is often limited by memory bandwidth when fetching volume data [S1, unofficial].
*   **Strengths:** Physically plausible results for semi-transparent volumes; supports early ray termination for optimization [S3, unofficial].
*   **Limitations:** Computationally expensive for high-resolution volumes; requires careful handling of sample order and precision [S1, unofficial].
*   **When to Use:** Standard direct volume rendering of semi-transparent data (e.g., medical CT/MRI, clouds, smoke).
*   **When Not to Use:** When only the highest intensity is needed (use MIP) or when opacity is irrelevant (use Average Compositing) [S3, unofficial].

## Background
Compositing in volume rendering originated with the development of direct volume rendering techniques in the late 1980s and early 1990s. The front-to-back and back-to-front compositing approaches were established as fundamental methods for integrating optical properties along rays [S1, unofficial]. The emission-absorption model, which underpins standard alpha compositing, has been widely adopted in scientific visualization and real-time rendering [S3, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S3] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S4] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S5] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
