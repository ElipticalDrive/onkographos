# Ray-marching boundary clipping

Also called: Clipping box coordinates within [0, 1] space, Clipping via Ray Marching Boundary Check, Ray-marching volume clipping via boundary check

## Overview
Ray-marching boundary clipping prevents ray-marching from accumulating color and alpha when the ray position is outside a defined clipping box, thereby avoiding the black artifacts caused by discarding fragments in the fragment shader [S1, S2, unofficial]. It differs from fragment-shader discarding by allowing the ray to continue marching through empty space or other volumes without contributing to the final pixel color, preserving the visibility of interior structures [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `currentPosition`: The 3D position of the ray at the current marching step.
    *   `xClippingPlaneMin`, `yClippingPlaneMin`, `zClippingPlaneMin`: Minimum coordinates of the clipping box [S2, unofficial].
    *   `xClippingPlaneMax`, `yClippingPlaneMax`, `zClippingPlaneMax`: Maximum coordinates of the clipping box [S2, unofficial].
    *   `colorSample`: The color sampled from the volume at `currentPosition`.
    *   `alphaSample`: The alpha (opacity) sampled from the volume at `currentPosition`.
    *   `accumulatedColor`: The running sum of accumulated color.
    *   `accumulatedAlpha`: The running sum of accumulated alpha.
*   **Outputs:**
    *   Updated `accumulatedColor` and `accumulatedAlpha` if the position is within boundaries; otherwise, they remain unchanged for this step [S1, S2, unofficial].

**Data Structures:**
*   Clipping box coordinates are defined as six scalar values representing the min/max extents along the X, Y, and Z axes [S2, unofficial].
*   Clipping box coordinates must be within the [0, 1] space [S1, S2, unofficial].

**Boundary Check Function:**
The method relies on a function `withinBoundaries` that returns `true` if the position is inside the clipping box and `false` otherwise [S1, unofficial].

$$
\text{withinBoundaries}(pos) = \begin{cases} 
\text{false} & \text{if } pos.x < xClippingPlaneMin \lor pos.y < yClippingPlaneMin \lor pos.z < zClippingPlaneMin \\
\text{false} & \text{if } pos.x > xClippingPlaneMax \lor pos.y > yClippingPlaneMax \lor pos.z > zClippingPlaneMax \\
\text{true} & \text{otherwise}
\end{cases}
$$

**Algorithm Steps:**
1.  Define `withinBoundaries(pos)` returning `false` if any coordinate is less than its respective minimum or greater than its respective maximum clipping plane value [S1, S2, unofficial].
2.  Inside the ray-marching loop, evaluate `withinBoundaries(currentPosition)` [S1, S2, unofficial].
3.  If `withinBoundaries(currentPosition)` is `true`, accumulate color and alpha: `accumulatedColor += colorSample * alphaSample` and `accumulatedAlpha += alphaSample` [S1, S2, unofficial].
4.  If `withinBoundaries(currentPosition)` is `false`, skip the composition step for this iteration; do not update `accumulatedColor` or `accumulatedAlpha` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_boundary_clipping(current_position, x_min, y_min, z_min, x_max, y_max, z_max, color_sample, alpha_sample, accumulated_color, accumulated_alpha, max_steps):
    # Boundary check function
    def within_boundaries(pos):
        if pos.x < x_min or pos.y < y_min or pos.z < z_min:
            return False
        if pos.x > x_max or pos.y > y_max or pos.z > z_max:
            return False
        return True

    # Ray marching loop
    for i in range(max_steps):
        # Not given in the sources. Position update depends on ray origin, direction, and step size.
        pass

        if within_boundaries(current_position):
            accumulated_color = accumulated_color + color_sample * alpha_sample
            accumulated_alpha = accumulated_alpha + alpha_sample
        # Else: skip accumulation for this iteration

    return accumulated_color, accumulated_alpha
```

**Parameters:**
*   `xClippingPlaneMin`, `yClippingPlaneMin`, `zClippingPlaneMin`: Minimum clipping plane coordinates [S2, unofficial].
*   `xClippingPlaneMax`, `yClippingPlaneMax`, `zClippingPlaneMax`: Maximum clipping plane coordinates [S2, unofficial].
*   `MAX_STEPS`: The maximum number of ray-marching iterations. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Black Artifacts:** Discarding the pixel in the fragment shader prevents it from being rendered at all, causing black areas instead of showing the inner structure or surface [S1, S2, unofficial]. This method avoids that pitfall by skipping accumulation rather than discarding the fragment [S1, S2, unofficial].
*   **Coordinate Space:** Clipping box coordinates must be within [0, 1] space [S1, S2, unofficial]. Using coordinates outside this range may lead to incorrect clipping behavior [S1, S2, unofficial].

## Operating characteristics

## Background
This technique was discussed in the context of Three.js volume rendering to address issues with black areas appearing when clipping volumes [S1, S2, unofficial]. It was proposed as an alternative to discarding fragments in the fragment shader, which was found to prevent pixels from being rendered at all [S1, S2, unofficial]. The method is attributed to discussions on Three.js Discourse and Stack Overflow [S1, S2, unofficial].

## Sources
[S1] Clipping a volume in Three.Js giving Black area instead of Inner Material, https://stackoverflow.com/questions/56065267/clipping-a-volume-in-three-js-giving-black-area-instead-of-inner-material, fetched 2026-10-26 (unofficial)
[S2] Clipping a volume in Three.Js giving Black area instead of Inner Material, https://www.stackoverflow.com/questions/56065267, fetched 2026-10-29 (unofficial)
[S3] Clipping a volume in Three.Js giving Black area instead of Inner Material - Questions - three.js forum, https://discourse.threejs.org/t/clipping-a-volume-in-three-js-giving-black-area-instead-of-inner-material/7572, fetched 2026-10-29 (unofficial)
