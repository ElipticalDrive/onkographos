# Compositing
Also called: Composite ARGB

## Overview
Compositing is the image-order rendering step that combines multiple semi-transparent sample contributions along a viewing ray into a final pixel color [S1, unofficial][S2, unofficial]. It solves the problem of projecting a 3D density field into a 2D image while preserving internal structure and texture, which binary classification methods fail to do due to false positives and negatives [S4, unofficial]. It is distinguished from its siblings by its iterative accumulation of color and opacity using the OVER operator, which enables early ray termination when accumulated opacity reaches unity, a feature not available in back-to-front compositing [S1, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of samples $(c_i, \alpha_i)$ encountered along a ray, where $c_i$ is the sample color (RGB) and $\alpha_i$ is the sample opacity (A) [S1, unofficial][S2, unofficial].
*   **Output:** A final pixel color $C_{final}$ and accumulated opacity $\alpha_{acc}$.

**Data Structures:**
The method maintains an accumulator state $(C_{acc}, \alpha_{acc})$ initialized to $(0, 0)$ (or background color/opacity) and updated iteratively for each sample [S1, unofficial][S2, unofficial].

**Steps:**
1.  Initialize the accumulated color $C_{acc} = 0$ and accumulated opacity $\alpha_{acc} = 0$. Alternatively, initialize transmittance $T = 1.0$, representing the accumulated transparency $\prod (1-\alpha_j)$ [G].
2.  Iterate through samples $i$ from front to back:
    a. Update accumulated color:
    $$ C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i C_i $$
    b. Update accumulated opacity:
    $$ \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i $$
    where $C_{acc}^{(0)} = 0$ and $\alpha_{acc}^{(0)} = 0$ [S12, S43, S50, unofficial].
    c. The term $(1 - \alpha_{acc}^{(i-1)})$ represents the transmittance, i.e., the fraction of light reaching sample $i$ from the camera side [S1, S51, unofficial].
3.  (Optional Optimization) Perform early ray termination: if $\alpha_{acc}$ exceeds a threshold (e.g., 1.0), break the loop as further samples contribute negligibly to the color [S10, S12, unofficial].
4.  Composite the result over a background color $C_{bg}$:
    $$ C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg} $$ [S1, S14, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compositing(samples, background_color):
    """
    samples: list of tuples (color, opacity) from front to back.
    background_color: RGB tuple for the background.
    Returns: final RGB color.
    """
    acc_color = [0.0, 0.0, 0.0]
    acc_alpha = 0.0
    
    for color, alpha in samples:
        # Update color and opacity using front-to-back compositing
        # C_acc = C_acc + (1 - alpha_acc) * alpha * color
        # alpha_acc = alpha_acc + (1 - alpha_acc) * alpha
        
        weight = (1.0 - acc_alpha) * alpha
        acc_color[0] += weight * color[0]
        acc_color[1] += weight * color[1]
        acc_color[2] += weight * color[2]
        acc_alpha += weight
        
        # Early ray termination
        if acc_alpha >= 1.0:
            break
            
    # Composite over background
    final_color = [
        acc_color[0] + (1.0 - acc_alpha) * background_color[0],
        acc_color[1] + (1.0 - acc_alpha) * background_color[1],
        acc_color[2] + (1.0 - acc_alpha) * background_color[2]
    ]
    return final_color
```

**Parameters:**
*   **Threshold for early termination:** Typically 1.0, but can be slightly less (e.g., 0.99) to save computation if the background is dark or uniform [S10, S12, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulating small opacity values can lead to precision loss if not handled carefully, though standard floating-point arithmetic is usually sufficient for typical volume rendering resolutions.
*   **Order:** The order of samples is critical. Front-to-back compositing is non-commutative. Back-to-front compositing is also possible but does not support early ray termination [S1, unofficial].
*   **Background:** The choice of background color affects the final result, especially for semi-transparent regions. A black background is common, but a white or scene-specific background may be required [S1, S14, unofficial].

## Subtopics

### [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md)
Alpha compositing ray casting evaluates the volume rendering integral by marching along viewing rays and iteratively blending semi-transparent samples from the front of the ray to the back. It solves the problem of projecting a 3D density field into a 2D image while preserving internal structure and texture, unlike binary classification methods which suffer from false positives/negatives. It is distinguished from its siblings by its recursive accumulation of color and opacity using the OVER operator, which enables early ray termination when the accumulated opacity reaches unity, a feature not available in back-to-front compositing.

### [Ray Compositing and Accumulation](3449-ray-compositing-and-accumulation.md)
Ray Compositing and Accumulation is a hardware-accelerated method for aggregating color and opacity contributions into final pixel values during direct volume ray casting, specifically designed to handle sample streams that are not presented in strict ray order. Unlike software-based approaches that require sorting or random access memory for per-ray buffers, this method uses dedicated hardware structures (FIFOs and shift registers) to buffer partially accumulated values and resolve predecessor dependencies based on view direction and spatial coordinates. It is distinguished from [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md) by its explicit hardware implementation for non-ordered data streams, and from [Raymarch Blending](5322-raymarch-blending.md) by its focus on fixed-function accumulation logic within the VolumePro architecture.

### [Average Compositing](5286-average-compositing.md)
Average Compositing is a volume rendering technique that produces an X-ray-like image by integrating all sampled values along a ray with equal weight, ignoring opacity. It differs from Alpha Compositing and Accumulation methods by discarding the emission-absorption model and early ray termination, treating the volume as a uniform medium where every sample contributes equally to the final pixel intensity.

### [Raymarch Blending](5322-raymarch-blending.md)
Raymarch Blending combines a ray-marched volumetric result with an existing background scene texture using standard alpha compositing. It solves the problem of integrating semi-transparent volumetric data into a pre-rendered scene containing opaque geometry, ensuring correct occlusion where the volume interacts with the solid meshes. Unlike [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md) which typically handles volume-to-volume or volume-to-black blending, this method explicitly blends against a pre-existing framebuffer color.

### [Alpha Blending](5750-alpha-blending.md)
Alpha blending is a compositing technique used in direct volume ray casting to combine multiple semi-transparent contributions along a viewing ray into a final pixel color. It solves the problem of rendering translucent volumes by discretizing the continuous light transport equation, allowing for physically plausible accumulation of emitted and attenuated light. It is distinguished from simple averaging or maximum intensity projection by its use of an associative, non-commutative operator that preserves depth ordering, enabling correct compositing of independent sub-volumes or distributed rendering nodes.

### [Front-to-Back Compositing](7859-front-to-back-compositing.md)
Front-to-Back Compositing is a recursive implementation of the volume rendering equation that resolves redundant computation, enabling real-time processing of semi-transparent volume data. It differs from back-to-front methods by accumulating contributions from the viewer outward, allowing for early ray termination when the accumulated opacity approaches unity.

### Front-to-Back Alpha Blending
A projection technique that composites samples along the ray using alpha blending until the ray exits the volume or becomes fully opaque [S3, unofficial]. Initialize contribution = 1.0f. Composite each sample: outputColor.rgb += contribution * transfer.a * transfer.rgb. contribution *= (1 - transfer.a) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray $N$, i.e., $O(N)$. Early ray termination can reduce the average number of samples processed, especially in dense volumes.
*   **Memory Complexity:** $O(1)$ per ray, as only the accumulator state is needed.
*   **Measured Performance:** Performance depends heavily on the volume density and the effectiveness of early ray termination. In sparse volumes, performance is similar to back-to-front methods. In dense volumes, front-to-back compositing with early termination is significantly faster.
*   **Strengths:** Physically plausible results for participating media; supports early ray termination; simple to implement.
*   **Limitations:** Order-dependent (non-commutative); requires samples to be ordered along the ray; early ray termination is not possible with back-to-front compositing.
*   **When to use:** Standard direct volume ray casting where semi-transparency and internal structure visualization are required.
*   **When not to use:** When maximum intensity projection (MIP) or average projection is desired; when samples are not available in ray order and hardware acceleration for unordered accumulation is not available.

## Background
Compositing in volume rendering originated with the development of direct volume ray casting techniques in the early 1990s. The front-to-back compositing equation is derived from the volume rendering integral, which models the interaction of light with participating media. The method has been widely adopted in scientific visualization and medical imaging due to its ability to render semi-transparent structures effectively.

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S3] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
