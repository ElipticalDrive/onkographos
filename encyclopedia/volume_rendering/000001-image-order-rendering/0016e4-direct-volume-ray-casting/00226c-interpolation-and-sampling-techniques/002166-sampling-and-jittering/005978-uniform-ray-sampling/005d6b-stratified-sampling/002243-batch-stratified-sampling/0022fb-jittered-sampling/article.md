# Jittered Sampling

Also called: Grid-based Jittered Sampling

## Overview
Jittered Sampling generates a regular grid of samples per pixel with independent uniform random offsets within each grid cell to reduce aliasing while maintaining stratification [S1, unofficial]. It enhances supersampling by introducing random offsets to ray origins within the pixel grid, distributing samples more evenly and reducing patterned artifacts compared to uniform grids [S2].

## How it works
**Inputs:**
- `Adepth`: The number of samples along each axis per pixel (integer, typically $\ge 1$) [S1, unofficial].
- `sample->getX()`, `sample->getY()`: The base coordinates of the pixel or voxel center in the image or texture space [S1, unofficial].
- `randf()`: A function returning a uniform random float in $[0, 1)$ [G].

**Outputs:**
- A set of 2D sample coordinates for the pixel [S1, unofficial].

**Data Structures:**
- No complex structures required; the method operates on scalar coordinates [G].

**Steps:**
1. Iterate `ax` from $0$ to $Adepth-1$ and `ay` from $0$ to $Adepth-1$ [S1, unofficial].
2. Compute the normalized grid offsets:
   $$xOffset = \frac{ax}{Adepth}$$
   $$yOffset = \frac{ay}{Adepth}$$
   [S1, unofficial].
3. Generate independent random offsets:
   $$xOffsetRandom = randf() \times xOffset$$
   $$yOffsetRandom = randf() \times yOffset$$
   [S1, unofficial].
4. Compute the final sample position:
   $$x_{final} = sample \rightarrow getX() + xOffset + xOffsetRandom$$
   $$y_{final} = sample \rightarrow getY() + yOffset + yOffsetRandom$$
   [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jittered_sample(Adepth, base_x, base_y, randf):
    samples = []
    for ax in range(Adepth):
        for ay in range(Adepth):
            x_offset = ax / Adepth
            y_offset = ay / Adepth
            x_jitter = randf() * x_offset
            y_jitter = randf() * y_offset
            final_x = base_x + x_offset + x_jitter
            final_y = base_y + y_offset + y_jitter
            samples.append((final_x, final_y))
    return samples
```

**Parameters:**
- `Adepth`: Controls the density of samples [S1, unofficial]. Typical values are Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** `randf()` must return uniform and independent values to maintain statistical benefits [G].
- **Boundary Conditions:** The source formula adds the jitter to the base offset [S1, unofficial]. If `Adepth` is 1, `xOffset` is 0, resulting in no jitter [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N^2)$ per pixel, where $N = Adepth$ [G].
- **Memory Complexity:** $O(1)$ auxiliary space per sample [G].
- **Strengths:** Reduces aliasing artifacts compared to uniform sampling by breaking up coherent patterns [S2].
- **Limitations:** Performance scales quadratically with `Adepth` [G].
- **When to use:** When supersampling is needed to reduce aliasing in volume rendering [S2].
- **When not to use:** Not given in the sources.

## Background
Jittered sampling is an enhancement to supersampling introduced to reduce patterned artifacts in ray casting [S2]. It is used in volume rendering to improve the visual quality of ray-cast images by distributing samples more evenly within pixels [S1, unofficial][S2]. The method is part of the family of sampling and jittering techniques in direct volume ray casting [G].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] ray casting, https://grokipedia.com/page/Ray_casting, fetched 2026-10-26
