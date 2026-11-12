# Cone-Traced Supersampling

Also called: Cone-Traced Supersampling With Subpixel Edge Reconstruction, Subpixel Edge Reconstruction (SER)

## Overview
Cone-Traced Supersampling (CTSS) is an efficient spatial antialiasing technique for Signed Distance Function (SDF) rendering that performs supersampling along the traced ray near surfaces with partial visibility [S1, S1, unofficial]. It identifies object contours by evaluating cone intersections within a pixel's view frustum, avoiding the need for casting additional rays per pixel or offline prefiltering [S1, unofficial]. Unlike conventional supersampling, it minimizes shading computations, and unlike post-processing filters, it avoids blurriness and ghosting [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The standard sphere tracing ray $r(t) = \mathbf{o} + t\mathbf{d}$, the SDF $f(\mathbf{p})$, and the pixel's view frustum geometry.
- Output: A final pixel color with reduced aliasing artifacts.

**Data Structures:**
- Ray state: Origin $\mathbf{o}$, direction $\mathbf{d}$, current distance $t$.
- Cone parameters: Defined by the pixel's angular extent and the current ray position.

**Steps:**
1. **Cone Intersection Evaluation:** As the sphere tracing loop advances, evaluate whether the current ray position intersects with the cone defined by the pixel's view frustum [S1, unofficial]. This identifies surfaces with partial visibility (object contours) [S1, unofficial].
2. **Supersampling:** If a contour is detected, perform supersampling along the traced ray near the identified location [S1, unofficial].
3. **Specialized Sampling Strategy:** Use a strategy that minimizes the number of shading computations while correlating sample visibility [S1, unofficial].
4. **Aggregation:** Aggregate the correlated sample visibility to compute the final pixel color [S1, unofficial].
5. **Subpixel Edge Reconstruction (SER):** For complex pixels with geometric edges in relatively flat regions that were not detected by the cone intersection step, apply SER to locate and resolve these edges [S1, unofficial].

**Pitfalls:**
- Complex pixels with geometric edges in relatively flat regions are undetected by standard cone intersections; SER is required to handle these cases [S1, unofficial].

## Operating characteristics
CTSS provides antialiasing quality comparable to conventional supersampling but at a significantly lower computational cost [S1, unofficial]. It is suitable for real-time SDF renderers where performance is critical [S1, unofficial].

## Background
CTSS was introduced as an efficient and robust spatial antialiasing solution that naturally complements the sphere tracing algorithm [S1, unofficial]. It was designed to be easily implemented in existing real-time SDF renderers [S1, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
