# Motion Blur via Splat Smearing

Also called: Line Drawing Smear, Voxel Motion Blur

## Overview
Motion Blur via Splat Smearing is a technique to approximate motion blur in voxel buffers by distributing each sample's value evenly across the voxels it touches along its motion vector, preserving the total sum of values [S1, unofficial]. It differs from correct temporal integration by folding the temporal domain into the spatial domain, which allows for efficient rasterization but results in loss of lighting detail and inability to cancel out camera motion relative to primitives [S1, unofficial].

## How it works
**Inputs:** Start position ($vsStart$), end position ($vsEnd$), primitive value ($value$), voxel buffer, and optional sampling factor ($samplingFactor$).
**Outputs:** Updated voxel buffer with smeared values.

**Data Structures:**
- $vsStart, vsEnd$: 3D vectors representing the primitive's position at the start and end of the frame.
- $value$: The scalar or vector quantity associated with the primitive (e.g., density, color).
- $buffer$: The voxel grid storing accumulated values.

**Steps:**
1. Calculate the motion vector: $vsLine = vsEnd - vsStart$.
2. Determine the number of samples: $numSamples = \max(2, \lceil \|vsLine\| \rceil)$. If a sampling factor is used: $numSamples = \max(2, \lceil samplingFactor \cdot \|vsLine\| \rceil)$.
3. Iterate $i$ from $0$ to $numSamples - 1$:
   a. Calculate the fraction: $fraction = \frac{i}{numSamples - 1}$.
   b. Calculate the sample position: $vsP = \text{lerp}(vsStart, vsEnd, fraction)$.
   c. Write the antialiased point with value $\frac{value}{numSamples}$ to the buffer at $vsP$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_line(vs_start, vs_end, value, buffer, sampling_factor=1.0):
    # Calculate motion vector
    vs_line = vs_end - vs_start
    
    # Calculate number of samples
    # Using standard ceil and max(2) as per source
    num_samples = max(2, int(math.ceil(vs_line.length())))
    
    # Apply sampling factor if provided
    if sampling_factor != 1.0:
        num_samples = max(2, int(math.ceil(sampling_factor * vs_line.length())))
        
    # Distribute value
    value_per_sample = value / num_samples
    
    for i in range(num_samples):
        # Calculate fraction to avoid accumulation errors
        fraction = i / (num_samples - 1)
        
        # Interpolate position
        vs_p = lerp(vs_start, vs_end, fraction)
        
        # Write antialiased point
        write_antialiased_point(vs_p, value_per_sample, buffer)
```

**Parameters:**
- $samplingFactor$: A scaling factor to allow undersampling for speed. Typical value is $1.0$ for full quality, $<1.0$ for performance [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Error Accumulation:** Using incremental position updates can cause accumulation of errors in splat positions. Calculating the fraction explicitly avoids this [S1, unofficial].
- **Lighting Detail:** Smearing before lighting is technically incorrect because it folds the temporal domain into the spatial domain, losing information about when a primitive occupies a position. This causes loss of lighting detail, making sharp features look artificially soft [S1, unofficial].
- **Camera Motion Cancellation:** If the camera moves at the same speed as primitives, motion should cancel out, but because motion blur is baked into the voxel buffer, this is not possible [S1, unofficial].

## Subtopics

### [Post-rasterization Smearing](1600-post-rasterization-smearing.md)
Post-rasterization smearing is a technique to approximate motion blur in voxel buffers by deferring the blurring operation until after the rasterization/splatting phase is complete. Unlike methods that smear individual samples during rasterization, this approach accumulates velocity information in a separate buffer and applies a single, global smear pass. It is chosen when performance is prioritized over handling complex overlapping primitives with divergent motion vectors.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of primitives and the length of their motion vectors. Undersampling reduces this by a factor of $samplingFactor$.
- **Memory Complexity:** Depends on the voxel buffer size. Post-rasterization variants require an additional velocity buffer.
- **Performance:** Faster than correct temporal integration for large numbers of primitives. Post-rasterization smearing is often faster than per-sample smearing [S1, unofficial].
- **Strengths:** Simple to implement, efficient, preserves sum of values.
- **Limitations:** Loss of lighting detail, incorrect lighting for smeared primitives, inability to cancel camera motion, artifacts from undersampling if not randomized.
- **When to Use:** When performance is critical and approximate motion blur is acceptable.
- **When Not to Use:** When correct lighting interaction with moving primitives is required, or when camera and primitive motion should cancel out.

## Background
Originated in the context of production volume rendering, specifically for handling motion blur in voxel buffers where correct temporal integration is computationally prohibitive for thousands or millions of volumetric primitives [S1, unofficial]. The technique leverages standard line-drawing algorithms adapted for 3D voxel grids.

## Sources
[S1] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
