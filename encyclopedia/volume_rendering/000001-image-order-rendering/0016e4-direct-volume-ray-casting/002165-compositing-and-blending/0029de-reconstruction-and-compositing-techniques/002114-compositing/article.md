# Compositing
Also called: Composite ARGB

## Overview
Compositing is the step in direct volume ray casting that combines multiple semi-transparent contributions along a viewing ray into a final pixel color [S1, unofficial]. It solves the problem of projecting translucent 3D data onto a 2D image plane by blending overlying and underlying colors using an ARGB blending model [S2, unofficial]. Unlike methods that rely on binary classification or maximum intensity projection, compositing preserves texture and internal visibility by weighting samples by their optical density [S7, unofficial][S3, unofficial]. It is distinguished from its siblings by being the fundamental accumulation operator upon which specific traversal orders (front-to-back, back-to-front) and hardware accelerations are built.

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of optical properties $(C(x), \alpha(x))$ derived from the volume data at sample positions $x$ along the ray. $C(x)$ is the sample color (RGB), and $\alpha(x)$ is the sample opacity (alpha).
*   **Output:** The final pixel color $C_{out}$ and accumulated opacity $\alpha_{out}$.

**Data Structures:**
The method maintains two accumulators for each ray:
1.  $C_{in}$: The accumulated color before the current iteration.
2.  $\alpha_{in}$: The accumulated opacity before the current iteration.

**Steps:**
1.  **Initialization:** Initialize the accumulated color $C_{in} = (0, 0, 0)$ and accumulated opacity $\alpha_{in} = 0$. [S4, unofficial]
2.  **Traversal:** Iterate through the samples along the ray. The order depends on the specific variant (front-to-back or back-to-front).
3.  **Blending:** For each sample with color $C(x)$ and opacity $\alpha(x)$, update the accumulators. The standard recursive update equations are [S4, unofficial]:
    $$C_{out} = C_{in} + C(x) \cdot \alpha(x) \cdot (1 - \alpha_{in})$$
    $$\alpha_{out} = \alpha_{in} + \alpha(x) \cdot (1 - \alpha_{in})$$
    Here, $C_{out}$ and $\alpha_{out}$ are the values after the iteration, and $C_{in}$ and $\alpha_{in}$ are the values before.
4.  **Termination:** The process stops when the ray exits the volume or when the accumulated opacity $\alpha_{out}$ reaches 1.0 (fully opaque), allowing for early ray termination. [S3, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def composite_front_to_back(samples):
    """
    Performs front-to-back compositing on a list of samples.
    Each sample is a tuple (color_rgb, opacity_alpha).
    Returns (final_color_rgb, final_opacity_alpha).
    """
    # Initialize accumulators
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    # Pre-computed factor to avoid repeated multiplication in loop
    # contribution represents the transmittance (1 - accumulated_opacity)
    contribution = 1.0
    
    for sample in samples:
        sample_color = sample[0] # RGB vector
        sample_alpha = sample[1] # Scalar alpha
        
        # Check for early termination if fully opaque
        if accumulated_opacity >= 1.0:
            break
            
        # Update color: add sample color weighted by its alpha and the remaining transmittance
        # Cout = Cin + C(x) * alpha(x) * (1 - alpha_in)
        # Note: contribution == (1 - accumulated_opacity)
        for i in range(3):
            accumulated_color[i] += contribution * sample_alpha * sample_color[i]
            
        # Update opacity: alpha_out = alpha_in + alpha(x) * (1 - alpha_in)
        accumulated_opacity += sample_alpha * contribution
        
        # Update transmittance for next iteration
        contribution *= (1.0 - sample_alpha)
        
    return accumulated_color, accumulated_opacity
```

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulating many small alpha values can lead to precision errors. Using the multiplicative transmittance update (`contribution *= (1.0 - sample_alpha)`) is numerically more stable than additive opacity updates for high sample counts [S3, unofficial].
*   **Order Dependency:** The compositing operator is associative but not commutative. The order of samples (front-to-back vs. back-to-front) matters for the final result if the transfer function is non-linear or if early termination is used. Front-to-back is standard for early ray termination [S1, unofficial].
*   **Boundary Conditions:** If the ray starts inside the volume, the initialization of $C_{in}$ and $\alpha_{in}$ remains the same, but the first sample must be correctly identified.

## Subtopics

### [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md)
Alpha Compositing Ray Casting is a direct volume rendering technique that evaluates the volume rendering integral by marching along viewing rays and accumulating color and opacity contributions from the front of the ray to the back. It solves the problem of projecting semi-transparent 3D data onto a 2D image plane without requiring binary classification, thereby eliminating false positives and negatives associated with threshold-based methods. Unlike Maximum Intensity Projection (MIP), which only preserves the highest intensity sample, this method preserves texture and internal visibility by blending all samples weighted by their optical density.

### [Ray Compositing and Accumulation](3449-ray-compositing-and-accumulation.md)
Ray Compositing and Accumulation is a hardware-accelerated method for aggregating color and opacity contributions into final pixel values during direct volume ray casting, specifically designed to handle sample streams that are not presented in strict ray order. Unlike software-based approaches that require sorting or random access memory for per-ray buffers, this method uses dedicated hardware structures (FIFOs and shift registers) to buffer partially accumulated values and resolve predecessor dependencies based on view direction and spatial coordinates. It is distinguished from [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md) by its explicit hardware implementation for non-ordered data streams and from [Raymarch Blending](5322-raymarch-blending.md) by its focus on fixed-function accumulation logic within the VolumePro architecture.

### [Average Compositing](5286-average-compositing.md)
Average Compositing is a volume rendering technique that produces an X-ray-like image by integrating all sampled values along a ray with equal weight, ignoring opacity. It differs from Alpha Compositing and Accumulation methods by discarding the emission-absorption model and early ray termination, treating the volume as a uniform medium where every sample contributes equally to the final pixel intensity.

### [Raymarch Blending](5322-raymarch-blending.md)
Raymarch Blending combines a ray-marched volumetric result with an existing background scene texture using standard alpha compositing. It solves the problem of integrating semi-transparent volumetric data into a pre-rendered scene containing opaque geometry, ensuring correct occlusion where the volume interacts with the solid meshes. Unlike [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md) which typically handles volume-to-volume or volume-to-black blending, this method explicitly blends against a pre-existing framebuffer color.

### [Alpha Blending](5750-alpha-blending.md)
Alpha blending is a compositing technique used in direct volume ray casting to combine multiple semi-transparent contributions along a viewing ray into a final pixel color. It solves the problem of rendering translucent volumes by discretizing the continuous light transport equation, allowing for physically plausible accumulation of emitted and attenuated light. It is distinguished from simple averaging or maximum intensity projection by its use of an associative, non-commutative operator that preserves depth ordering, enabling correct compositing of independent sub-volumes or distributed rendering nodes.

### Front-to-Back Alpha Blending
Front-to-Back Alpha Blending is a projection technique that composites samples along the ray using alpha blending until the ray exits the volume or becomes fully opaque [S3, unofficial]. It initializes a contribution factor to 1.0 and iteratively updates the output color and contribution factor: `outputColor.rgb += contribution * transfer.a * transfer.rgb` and `contribution *= (1 - transfer.a)` for each sample [S3, unofficial].

### Front-to-Back Compositing
Front-to-Back Compositing is a recursive implementation of the volume rendering equation to resolve redundant computation and enable real-time processing [S4, unofficial]. It updates the color and opacity using the formulas: $C_{out} = C_{in} + C(x) \cdot \alpha(x) \cdot (1 - \alpha_{in})$ and $\alpha_{out} = \alpha_{in} + \alpha(x) \cdot (1 - \alpha_{in})$, where $C_{in}, C_{out}$ are color values before and after iteration, and $\alpha_{in}, \alpha_{out}$ are opacity values before and after iteration [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray $N$, i.e., $O(N)$. Early ray termination can reduce the effective number of samples processed.
*   **Memory Complexity:** $O(1)$ per ray, as only the current accumulated color and opacity need to be stored.
*   **Strengths:** Physically plausible blending of semi-transparent surfaces; supports early ray termination for performance; standard and well-understood.
*   **Limitations:** Order-dependent (requires strict front-to-back or back-to-front traversal); sensitive to sample spacing and transfer function non-linearities; not suitable for opaque-only rendering where simple overwriting suffices.
*   **When to use:** When rendering semi-transparent volumes where internal structure and depth cues are important.
*   **When not to use:** For opaque volumes (use standard rasterization or MIP); when hardware does not support floating-point blending or early termination.

## Background
Compositing in volume rendering originated from the fundamental light transport equation, which describes the interaction of light with participating media. The recursive front-to-back formulation, often attributed to the work of Levoy and others in the early 1990s, allows for efficient real-time implementation by avoiding redundant calculations of the transmittance term [S4, unofficial]. The term "Composite ARGB" reflects the common implementation in graphics APIs where color and alpha are packed together for blending operations [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S3] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S4] Antara: An Interactive 3D Volume Rendering and Visualization Framework, https://arxiv.org/abs/1812.04233v1, fetched 2026-10-26 (unofficial)
