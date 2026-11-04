# Accumulation Buffer Composite

Also called: Alpha-Averaged Composite, Spot Noise Composite

## Overview
The Accumulation Buffer Composite is the final post-processing step in a stochastic volumetric rendering pipeline that averages accumulated color samples and blends the result with the background using a reveal factor [S1, unofficial]. It distinguishes itself from deterministic ray-marching compositing by converting noisy, stochastic accumulation buffers into a clean image, effectively removing spot noise artifacts inherent in the sampling process [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `accumbuf`: A 2D texture (`sampler2D`) storing the accumulated color (RGB) and accumulated alpha (A) per pixel.
    *   `countbuf`: A 2D texture (`sampler2D`) storing the reveal factor (0.0 to 1.0) per pixel, representing the fraction of the pixel covered by volume samples.
*   **Output**:
    *   `color_out`: A `vec4` containing the final average color and the background alpha.

**Data Structures**
The method operates on per-pixel data fetched from the accumulation buffers. No complex internal data structures are required beyond standard texture sampling.

**Parameters**
*   `EPSILON`: $0.00001$. Used for numerical stability to prevent division by zero and for approximate equality checks [S1, unofficial].

**Steps**
1.  Fetch the accumulated color and alpha from `accumbuf` at the current fragment coordinate:
    $$ \text{accum} = \text{texelFetch}(\text{accumbuf}, \text{ivec2}(\text{gl\_FragCoord.xy}), 0) $$
2.  Fetch the reveal factor from `countbuf` at the current fragment coordinate:
    $$ \text{reveal} = \text{texelFetch}(\text{countbuf}, \text{ivec2}(\text{gl\_FragCoord.xy}), 0).r $$
3.  Check if the volume is fully opaque. If the reveal factor is approximately 1.0, discard the fragment, as no compositing with the background is needed [S1, unofficial].
4.  Check for numerical instability in the accumulated color. If any component of `accum.rgb` is infinite, replace the infinite value with `accum.a` to prevent NaN propagation during division [S1, unofficial].
5.  Compute the average color by dividing the accumulated RGB by the accumulated alpha, clamped to `EPSILON` to avoid division by zero:
    $$ \text{avg} = \frac{\text{accum.rgb}}{\max(\text{accum.a}, \epsilon)} $$
    where $\epsilon = 0.00001$ [S1, unofficial].
6.  Output the final color with the background alpha:
    $$ \text{color\_out} = \text{vec4}(\text{avg.rgb}, 1.0 - \text{reveal}) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper functions for numerical stability
def is_approximately_equal(a, b, epsilon=0.00001):
    """Check if a and b are approximately equal."""
    return abs(a - b) <= (abs(a) if abs(a) < abs(b) else abs(b)) * epsilon

def max3(v):
    """Return the maximum of three components."""
    return max(v.x, v.y, v.z)

def accumulate_buffer_composite(accumbuf, countbuf, frag_coord):
    """
    Composite the accumulated volume data with the background.
    
    Args:
        accumbuf: Sampler2D for accumulated RGB and Alpha.
        countbuf: Sampler2D for Reveal Factor.
        frag_coord: Current fragment coordinates (x, y).
        
    Returns:
        vec4: Final color and background alpha.
    """
    # Step 1: Fetch accumulated color and alpha
    accum = accumbuf.fetch(ivec2(frag_coord.xy), 0)
    
    # Step 2: Fetch reveal factor
    reveal = countbuf.fetch(ivec2(frag_coord.xy), 0).r
    
    # Step 3: If volume is fully opaque, discard (handled by renderer or early exit)
    # Note: In a shader, this is typically done via discard or by skipping the write.
    if is_approximately_equal(reveal, 1.0):
        return None  # Fragment discarded or not written
    
    # Step 4: Handle infinity in accumulated RGB
    # Replace infinite values with accumulated alpha to prevent NaN propagation
    if isinf(max3(abs(accum.rgb))):
        accum.rgb = vec3(accum.a)
        
    # Step 5: Compute average color
    # Divide accumulated RGB by accumulated alpha, clamped to EPSILON
    epsilon = 0.00001
    avg_rgb = accum.rgb / max(accum.a, epsilon)
    
    # Step 6: Output final color with background alpha
    color_out = vec4(avg_rgb, 1.0 - reveal)
    
    return color_out
```

**Edge Cases and Pitfalls**
*   **Division by Zero**: Prevented by using `max(accum.a, EPSILON)` in the denominator [S1, unofficial].
*   **Infinity Propagation**: If the accumulation buffer contains infinite values (e.g., due to extreme density or lighting), `isinf(max3(abs(accum.rgb)))` detects this and replaces the value with `accum.a` to avoid NaNs in the final output [S1, unofficial].
*   **Reveal Factor Approximation**: The check for `reveal == 1.0` uses an approximate equality function to account for floating-point precision issues [S1, unofficial].

## Subtopics

### Empirical Volume Shading with Indirect Light Approximation
An alternative sibling method that approximates indirect lighting in volumetric scenes. Choose this when indirect lighting effects are critical and stochastic noise is less of a concern than lighting accuracy. [Empirical Volume Shading with Indirect Light Approximation](4092-empirical-volume-shading-with-indirect-light-approximation.md)

## Operating characteristics
*   **Time Complexity**: $O(1)$ per pixel. The operation involves a constant number of texture fetches and arithmetic operations.
*   **Memory Complexity**: $O(1)$ additional memory per pixel, as it operates directly on the accumulation buffers.
*   **Strengths**: Efficiently removes stochastic noise from volume rendering by averaging samples. Simple to implement as a post-processing step.
*   **Limitations**: Requires two full-resolution textures (`accumbuf` and `countbuf`). Does not handle temporal coherence, so noise may flicker across frames if not stabilized.
*   **When to Use**: When using a stochastic volume renderer (e.g., spot noise rendering) and a clean, noise-free image is required.
*   **When Not to Use**: When deterministic ray-marching is used, or when memory bandwidth is a critical bottleneck and storing two additional textures is prohibitive.

## Background
The Accumulation Buffer Composite is part of the volumetric spot noise rendering pipeline, popularized by the work on stochastic volume rendering. It serves as the final compositing stage, converting noisy accumulation buffers into a visually coherent image. The method is closely associated with the "volumetric-spot-noise" implementation, which uses stochastic sampling to approximate volume lighting and density [S1, unofficial].

## Sources
[S1] bottledspace/volumetric-spot-noise, https://github.com/bottledspace/volumetric-spot-noise, fetched 2026-10-27 (unofficial)
