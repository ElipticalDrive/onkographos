# In-Scattering Integral Approximation via Riemann Sum

Also called: Discrete In-Scattering Integral, Riemann Sum In-Scattering

## Overview
This method approximates the continuous influx of light energy scattered toward the eye (in-scattering) along a camera ray segment by discretizing the integral into a sum of small rectangular areas [S1, unofficial]. It solves the problem that the in-scattering integral has no analytical solution for general volume shapes and lighting scenarios, necessitating numerical approximation [S1, unofficial].

## How it works
**Inputs:**
- Camera ray segment defined by parametric distances $t_0$ (entry) and $t_1$ (exit) [S1, unofficial].
- Step size $dx$ [S1, unofficial].
- Volume density field to determine light travel distance [S1, unofficial].
- Light source direction [S1, unofficial].

**Outputs:**
- Approximated in-scattered radiance accumulated over the segment [S1, unofficial].

**Data Structures:**
- Accumulator variable for total in-scattered light (scalar) [S1, unofficial].
- Current position $x$ along the ray (scalar) [S1, unofficial].
- Light travel distance (scalar) [S1, unofficial].

**Steps:**
1. Initialize the total in-scattered light accumulator to zero [S1, unofficial].
2. Divide the camera ray segment contained within the volume boundaries (from $t_0$ to $t_1$) into smaller sub-segments of length $dx$ [S1, unofficial].
3. For each sub-segment, identify the sample point $x$ at the middle of the interval [S1, unofficial].
4. Evaluate the incident light radiance $L_i(x, \omega)$ at point $x$: shoot a ray from $x$ in the direction of the light source to determine the distance light traveled through the volume [S1, unofficial]. Apply Beer's law to the light travel distance to determine the remaining light energy at point $x$ [G]. Note that the basic Riemann sum description does not account for the scattering phase function, which is required for the full calculation of $L_i(x, \omega)$ [S1, unofficial].
5. Compute the area of the rectangle for the current sub-segment as $L_i(x, \omega) \times dx$ [S1, unofficial].
6. Accumulate the contribution of each small segment to the total in-scattered light [S1, unofficial].
7. Return the accumulated sum as the approximation of the integral [S1, unofficial].

**Formula:**
The integral to approximate is:
$$ \int_{x=t_0}^{t_1} L_i(x, \omega) dx $$
where $L_i(x, \omega)$ is the radiance scattered towards the eye direction $\omega$ at point $x$ on the segment between parametric distances $t_0$ and $t_1$ [S1, unofficial].

The Riemann sum approximation is:
$$ \sum_{i} L_i(x_i, \omega) \cdot dx $$

**Parameters:**
- $dx$: The width of the rectangles in the Riemann sum approximation. Smaller $dx$ yields higher accuracy but increases processing time [S1, unofficial].

**Pitfalls:**
- The integral has no analytical solution for general volume shapes and lighting scenarios [S1, unofficial].
- The basic Riemann sum description provided does not yet account for the scattering phase function, which is required for the full calculation of $L_i(x, \omega)$ [S1, unofficial].
- Precision issues may arise if $dx$ is too large, leading to significant deviation from the true area under the curve [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def approximate_in_scattering(t0, t1, dx, density_func, light_direction, light_radiance):
    """
    Approximates the in-scattering integral using a Riemann sum.
    
    Args:
        t0: Parametric distance where ray enters volume.
        t1: Parametric distance where ray exits volume.
        dx: Step size (width of rectangles).
        density_func: Function returning volume density at a point.
        light_direction: Direction vector to the light source.
        light_radiance: Intensity of the light source.
        
    Returns:
        Approximated in-scattered radiance.
    """
    total_in_scattered = 0.0
    num_steps = int((t1 - t0) / dx)
    for i in range(num_steps):
        x = t0 + (i + 0.5) * dx
        # Shoot ray from x in light direction to find distance through volume [S1, unofficial]
        # Integrate density along path to compute optical depth [G]
        light_travel_distance = 0.0
        t = 0.0
        step = 0.01
        while t < 100.0:
            light_travel_distance += density_func(x + light_direction * t) * step
            t += step
        # Apply Beer's law to determine remaining light energy [G]
        li_x = light_radiance * exp(-light_travel_distance)
        total_in_scattered += li_x * dx
    return total_in_scattered
```

## Subtopics

### [Discrete Ray-Marching with Forward Scattering Approximation](13373-discrete-ray-marching-with-forward-scattering-approximation.md)
This method approximates the in-scattering integral for volumetric clouds by discretizing the continuous light propagation equation into a Riemann sum using fixed-step ray marching. It distinguishes itself from general in-scattering approximations by explicitly assuming that 50% of scattered light propagates in the forward direction within a small solid angle (~0.0001 steradians), ignoring the remaining 50% to reduce computational cost. It builds on [In-Scattering Integral Approximation via Riemann Sum](1585-in-scattering-integral-approximation-via-riemann-sum.md) by providing the specific physical assumptions and numerical simplifications (Maclaurin expansion) required for real-time GPU implementation.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of sub-segments ($N = (t_1 - t_0) / dx$) [S1, unofficial]. Each sub-segment requires a light ray traversal, making the complexity $O(N \cdot M)$ where $M$ is the number of steps for the light ray traversal [G].
- **Memory Complexity:** Constant, $O(1)$, as it only requires storing the accumulator and current position [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Accuracy scales inversely with $dx$; smaller $dx$ yields higher accuracy but increases processing time linearly [S1, unofficial].
- **Strengths:** Simple to implement; provides a foundational approximation for in-scattering [S1, unofficial].
- **Limitations:** Does not account for the scattering phase function in its basic form [S1, unofficial]; computationally expensive due to the nested ray traversal for light [S1, unofficial].
- **When to use:** When a basic approximation of in-scattering is sufficient [S1, unofficial].
- **When not to use:** When the scattering phase function is negligible or can be approximated differently (e.g., using [Discrete Ray-Marching with Forward Scattering Approximation](13373-discrete-ray-marching-with-forward-scattering-approximation.md)) [G].

## Background
The concept of using a Riemann sum to approximate the in-scattering integral is discussed in "An Introduction to Volume Rendering" [S1, unofficial]. The method is presented as a practical solution to the problem that the in-scattering integral has no analytical solution for general volume shapes and lighting scenarios [S1, unofficial]. The source emphasizes that while the result is an approximation, it becomes closer to the true area under the curve as the width of the rectangles ($dx$) decreases [S1, unofficial].

## Sources
[S1] An Introduction to Volume Rendering, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/intro-volume-rendering.html, fetched 2026-10-25 (unofficial)
