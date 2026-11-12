# Average Compositing

Also called: Linear Compositing

## Overview
Average Compositing is a volume rendering technique that produces an X-ray-like image by sending a ray through the volume and compositing all found values with the same weight, taking no opacity into account [S1, unofficial][S2, unofficial]. It differs from Alpha Compositing and Accumulation methods by treating every sample equally regardless of depth or transparency [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Ray defined by origin and direction; scalar volume dataset. Not given in the sources.
**Outputs:** Single scalar intensity value for the pixel. Not given in the sources.
**Data Structures:** Not given in the sources.
**Numbered Steps:**
1. Send a ray through the volume [S1, unofficial][S2, unofficial].
2. Composite all found values with the same weight [S1, unofficial][S2, unofficial].
**Formulas:** Not given in the sources.
**Parameters:** Step size and sampling density are not given in the sources.
**Edge Cases and Pitfalls:** If the ray does not intersect the volume, the sample count is zero; a guard must return a background value to avoid division by zero [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_compositing(ray_origin, ray_direction, volume_sample_fn, step_size, volume_min, volume_max):
    """
    Computes average intensity along a ray through a volume.
    Args:
        ray_origin: (x, y, z) tuple
        ray_direction: (dx, dy, dz) tuple, assumed normalized
        volume_sample_fn: callable returning scalar at (x, y, z)
        step_size: float, distance between samples
        volume_min: (x, y, z) tuple, lower bounds
        volume_max: (x, y, z) tuple, upper bounds
    Returns:
        float: Average intensity, or 0.0 if no intersection
    """
    sum_values = 0.0
    count = 0
    t = 0.0
    step_vector = (ray_direction[0] * step_size,
                   ray_direction[1] * step_size,
                   ray_direction[2] * step_size)

    while True:
        x = ray_origin[0] + ray_direction[0] * t
        y = ray_origin[1] + ray_direction[1] * t
        z = ray_origin[2] + ray_direction[2] * t

        if not (volume_min[0] <= x <= volume_max[0] and
                volume_min[1] <= y <= volume_max[1] and
                volume_min[2] <= z <= volume_max[2]):
            break

        value = volume_sample_fn(x, y, z)
        sum_values += value
        count += 1
        t += step_size

    if count == 0:
        return 0.0

    return sum_values / count
```

## Subtopics

### [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md)
Uses the over operator with opacity; choose when transparency and depth cues are needed [G].

### [Ray Compositing and Accumulation](3449-ray-compositing-and-accumulation.md)
Accumulates intensity until opacity threshold is met; choose for emission-absorption models [G].

### [Raymarch Blending](5322-raymarch-blending.md)
General blending of samples; choose for flexible compositing rules [G].

### [Alpha Blending](5750-alpha-blending.md)
Standard alpha blending; choose for 2D/3D overlay compositing [G].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples along the ray [G].
- **Memory Complexity:** $O(1)$ per ray, requiring only a running sum and count [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Simple to implement; produces an X-ray-like overview of the entire volume [S1, unofficial][S2, unofficial].
- **Limitations:** Ignores opacity and depth cues; does not highlight high-intensity structures [S1, unofficial][S2, unofficial].
- **When to use:** When an X-ray-like overview of the entire volume is desired, and opacity or depth cues are not critical [S1, unofficial][S2, unofficial].
- **When not to use:** When transparency, occlusion, or highlighting of specific high-intensity features is required [S1, unofficial][S2, unofficial].

## Background
Average Compositing is a compositing scheme used in Direct Volume Rendering [S1, unofficial][S2, unofficial]. It is described alongside Maximum Intensity Projection (MIP) and Accumulation compositing in introductory literature [S1, unofficial][S2, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume_Rendering, fetched 2026-10-25 (unofficial)
[S2] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
