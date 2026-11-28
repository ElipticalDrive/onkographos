# Temporal Depth Upsampling

Also called: Depth Upsampling, Jittered Depth Upsampling

## Overview
Temporal Depth Upsampling extends the temporal reprojection techniques used for color in volumetric rendering to generate a high-resolution depth buffer from lower-resolution samples [S1, unofficial]. It solves the problem of maintaining accurate depth information for object-interaction (e.g., clouds occluding terrain) without the performance cost of full-resolution depth rasterization [S1, unofficial]. It is distinguished from spatial upsampling methods by its reliance on temporal reprojection and jittered sampling convergence rather than static interpolation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Previous frame's color buffer (`prevColor`), previous frame's depth buffer (`prevDepth`), current frame's jittered low-resolution color samples (`jitteredColor`), and current frame's jittered low-resolution depth samples (`jitteredDepth`).
*   **Outputs:** A high-resolution fragment output structure containing upsampled color and depth.

**Data Structures:**
The fragment output structure must contain both color and depth fields to be written in a single pass:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct ForwardFragmentOutput {
    float4 Color : SV_Target;
    float Depth : SV_Depth;
};
```

**Parameters:**
*   `convergenceSpeed`: A scalar value $c \in [0, 1]$ determining the weight given to the new jittered sample versus the previous frame's reprojection.
    *   Typical values vary by position. In volumetric cloud rendering, it is often interpolated based on the view direction: `lerp(convergenceSpeedZenith, convergenceSpeedHorizon, distanceFromZenith01)` [S1, unofficial].
    *   Alternatively, it can be influenced by depth values themselves [S1, unofficial].

**Steps:**
1.  **Reproject Previous Depth:** Transform the previous frame's depth buffer into the current frame's coordinate space using motion vectors or camera transform differences. Let this be `prevDepth`.
2.  **Sample Jittered Depth:** Render the volume at a lower resolution with temporal jittering to obtain `jitteredDepth` for the current pixel location.
3.  **Compute Convergence Speed:** Calculate the scalar `convergenceSpeed` based on the chosen heuristic (e.g., zenith distance or depth magnitude) [S1, unofficial].
4.  **Linear Interpolation:** Blend the reprojected depth and the new jittered depth using the convergence speed:
    $$ \text{output.Depth} = \text{lerp}(\text{prevDepth}, \text{jitteredDepth}, \text{convergenceSpeed}) $$
5.  **Output:** Write the result to the depth buffer alongside the upsampled color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TemporalDepthUpsample(prevDepth, jitteredDepth, convergenceSpeed):
    """
    Upsamples depth using temporal reprojection.
    
    Args:
        prevDepth (float): Depth value from the previous frame, reprojected to current coordinates.
        jitteredDepth (float): Depth value from the current frame's low-resolution jittered sample.
        convergenceSpeed (float): Blending factor between 0 and 1.
        
    Returns:
        float: The upsampled depth value.
    """
    # Linear interpolation between previous reprojection and new sample
    # lerp(a, b, t) = a * (1 - t) + b * t
    output_depth = prevDepth * (1.0 - convergenceSpeed) + jitteredDepth * convergenceSpeed
    return output_depth
```

**Edge Cases and Pitfalls:**
*   **Skydome Rendering:** If the volume is rendered onto a skydome and does not interact with other objects, the depth buffer is unnecessary, and this upsampling step can be skipped entirely [S1, unofficial].
*   **Object Interaction:** If objects interact with the volume (e.g., occlusion), depth must be upsampled alongside color to ensure correct depth testing [S1, unofficial].
*   **Precision:** Standard floating-point depth precision is assumed. No specific bit-packing details are provided in the sources [S1, unofficial].

## Subtopics

### Temporal Self-Supervision (Full-Dt)
An extension of the full algorithm using a temporal discriminator $D_t$ proposed for fluid super-resolution, intended for settings without physical priors [S2]. It was evaluated in the context of global transport for fluid reconstruction [S2]. However, it does not yield substantial improvements for optimized motion, highlighting the constraining capabilities of global transport methods [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels processed, as it involves a single pass of reprojection and interpolation [S1, unofficial].
*   **Memory Complexity:** Requires storage for the previous frame's depth buffer and motion vectors for reprojection [S1, unofficial].
*   **Strengths:** Significantly reduces rendering cost by allowing low-resolution depth sampling while maintaining temporal stability [S1, unofficial].
*   **Limitations:** Relies on accurate reprojection; motion estimation errors can cause artifacts. The sources do not provide specific performance metrics or hardware scaling data [S1, unofficial].

## Background
The method is described in the context of volumetric cloud rendering optimization, specifically referencing techniques used in *Horizon Zero Dawn* [S1, unofficial]. The approach leverages temporal reprojection to update a fraction of pixels per frame, extending the color upsampling logic to depth [S1, unofficial].

## Sources
[S1] Upsampling to Improve Volumetric Cloud Render Performance · Vertex Fragment, https://www.vertexfragment.com/ramblings/volumetric-cloud-upsampling/, fetched 2026-10-30 (unofficial)
[S2] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
