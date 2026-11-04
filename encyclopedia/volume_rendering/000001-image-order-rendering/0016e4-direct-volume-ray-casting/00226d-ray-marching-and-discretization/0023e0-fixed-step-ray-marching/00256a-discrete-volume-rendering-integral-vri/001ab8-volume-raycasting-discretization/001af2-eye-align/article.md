# Eye Align

Also called: Eye-aligned sampling, eye-origin alignment

## Overview
Eye Align is a sample placement strategy for volume raycasting where the first sample point inside the volume is calculated relative to the eye position, creating a sampling pattern across pixels that forms concentric lines [S1, unofficial]. It sets itself apart from siblings like [Entry Align](855-entry-align.md) by aligning samples to the viewer's perspective rather than the volume entry boundary, which can introduce specific aliasing artifacts at the ray endpoints [S1, unofficial].

## How it works
**Inputs:**
*   `entryPoint`: 3D vector, entry point of the ray into the volume (in texture coordinates).
*   `exitPoint`: 3D vector, exit point of the ray from the volume (in texture coordinates).
*   `eyePos`: 3D vector, position of the camera/eye (in texture coordinates).
*   `dim`: 3D vector, dimensions of the volume grid.
*   `samplingRate`: float, number of samples per unit length in texture space.

**Outputs:**
*   `tStart`: float, the parameter value for the first sample point inside the volume.
*   `tIncr`: float, the step increment between consecutive samples.
*   `tEnd`: float, the total length of the ray within the volume.

**Data Structures:**
Standard floating-point vectors and scalars. No complex packing required.

**Steps:**
1.  Calculate the ray vector in texture coordinates:
    $$ \vec{ray} = \text{exitPoint} - \text{entryPoint} $$
2.  Calculate the total number of samples based on ray length, volume dimensions, and sampling rate:
    $$ \text{samples} = \|\vec{ray} \cdot \text{dim}\| \times \text{samplingRate} $$
    *Note: The source code uses `length(ray * dim)`, implying element-wise multiplication followed by Euclidean norm.*
3.  Calculate the total ray length:
    $$ t_{\text{End}} = \|\vec{ray}\| $$
4.  Calculate the step increment:
    $$ t_{\text{Incr}} = \frac{t_{\text{End}}}{\text{samples}} $$
5.  Calculate the distance from the eye to the entry point to determine the offset for the first sample:
    $$ \text{numSteps} = \frac{\|\text{entryPoint} - \text{eyePos}\|}{t_{\text{Incr}}} $$
6.  Calculate the start parameter for the first sample inside the volume by removing the fractional part of the steps from the eye:
    $$ t_{\text{Start}} = (1 - \text{fract}(\text{numSteps})) \times t_{\text{Incr}} $$
    Where $\text{fract}(x) = x - \lfloor x \rfloor$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def eye_align(entryPoint, exitPoint, eyePos, dim, samplingRate):
    # 1. Ray vector in texture coordinates
    ray = exitPoint - entryPoint
    
    # 2. Total number of samples
    # Element-wise multiplication then length
    ray_scaled = vec3(ray.x * dim.x, ray.y * dim.y, ray.z * dim.z)
    samples = length(ray_scaled) * samplingRate
    
    # 3. Total ray length
    tEnd = length(ray)
    
    # 4. Step increment
    tIncr = tEnd / samples
    
    # 5. Distance from eye to entry in terms of steps
    dist_eye_entry = length(entryPoint - eyePos)
    numSteps = dist_eye_entry / tIncr
    
    # 6. Start parameter
    # fract(x) = x - floor(x)
    frac = numSteps - floor(numSteps)
    tStart = (1.0 - frac) * tIncr
    
    return tStart, tIncr, tEnd
```

**Parameters:**
*   `samplingRate`: Not given in the sources. Typical values depend on the desired visual fidelity and performance constraints.

**Edge Cases and Pitfalls:**
*   **Endpoint Estimation Error:** The Riemann sum mid-point estimate used in conjunction with this sampling strategy tends to over- or underestimate contributions at both the first and last sample points. This is visually indicated by overshooting artifacts (e.g., red sections in comparative visualizations) [S1, unofficial].
*   **Zero Ray Length:** If `entryPoint` equals `exitPoint`, `tEnd` and `tIncr` become zero, leading to division by zero in step 4 and 5. This should be handled by skipping rendering or returning default values.
*   **Precision:** Floating-point precision issues may arise when `numSteps` is very large or very small, affecting the accuracy of `tStart`.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. The calculation of `tStart` and `tIncr` is $O(1)$ per ray.
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray.
*   **Strengths:** Simple to implement; aligns sampling with the viewer's perspective, which can be beneficial for certain types of aliasing patterns.
*   **Limitations:** Prone to endpoint estimation errors (over/underestimation) due to the mid-point rule interaction [S1, unofficial]. The concentric line pattern may be visually noticeable or undesirable in some contexts.
*   **When to use:** When viewer-centric sampling alignment is desired and endpoint artifacts are acceptable or can be mitigated by other means.
*   **When not to use:** When high accuracy at volume boundaries is critical, or when the concentric aliasing pattern is undesirable. [Entry Align](855-entry-align.md) may be preferred to mitigate endpoint errors.

## Background
The method was described in the context of volume raycasting sampling strategies, specifically as a "first attempt" following work by Kruger et al. [6] (cited in [S1, unofficial]). It is presented as an alternative to aligning samples with the volume entry point (Entry Align) [S1, unofficial]. The source is an unofficial document titled "Volume Raycasting Sampling Revisited" [S1, unofficial].

## Sources
[S1] Volume Raycasting Sampling Revisited, https://osf.io/download/u9qnz/, fetched 2026-10-29 (unofficial)
