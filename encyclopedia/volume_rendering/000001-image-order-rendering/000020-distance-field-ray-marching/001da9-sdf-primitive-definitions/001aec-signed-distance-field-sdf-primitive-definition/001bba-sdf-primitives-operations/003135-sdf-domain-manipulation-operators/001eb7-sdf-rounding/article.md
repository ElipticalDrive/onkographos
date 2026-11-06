# SDF Rounding

Also called: Distance Offset, Rounded, Rounded Corners in SDFs, Rounding, Rounding Transformation, SDF Domain Manipulation, SDF Offset, SDF Rounding/Offset

## Overview
SDF Rounding is a geometric transformation that modifies a Signed Distance Field (SDF) primitive by subtracting a constant radius value from its distance output, effectively rounding off hard edges and corners [S1, S4, unofficial]. Unlike [Elongation](2440-elongation.md) which stretches geometry, or [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) which combine shapes, rounding alters the metric of a single primitive to create smooth, filleted surfaces [S2, S3, unofficial]. It is a specific instance of [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) that operates on the distance value rather than the input position vector.

## How it works
**Inputs and Outputs:**
- Input: A base SDF function $d_{base}(\mathbf{p})$ returning the signed distance from point $\mathbf{p}$ to the surface, and a scalar rounding radius $r \ge 0$.
- Output: A new SDF function $d_{rounded}(\mathbf{p})$ representing the shape with rounded features.

**Data Structures:**
The implementation typically encapsulates the base SDF and the radius parameter. For example, a `Rounded` struct contains:
- `sdf`: The base SDF primitive instance.
- `radius` (f32): The constant offset value [S3, unofficial].

**Steps:**
1. Evaluate the base SDF at the query point $\mathbf{p}$ to obtain the raw signed distance $d = d_{base}(\mathbf{p})$ [S1, S3, unofficial].
2. Subtract the rounding radius $r$ from the raw distance: $d_{rounded} = d - r$ [S1, S2, S4, unofficial].
3. Return $d_{rounded}$ [S2, unofficial].

**Formula:**
$$ d_{rounded}(\mathbf{p}) = d_{base}(\mathbf{p}) - r $$

**Parameters:**
- $r$ (float): The rounding radius. Typical values depend on the scale of the primitive (e.g., 0.1 for a unit cube) [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- If $r$ is larger than the minimum feature size of the primitive (e.g., half the width of a thin box), the interior of the shape may become empty or inverted, depending on the desired semantic [S2, unofficial].
- Precision issues may arise if $d_{base}(\mathbf{p}) \approx r$, leading to numerical instability near the new surface boundary [G].
- The operation does not preserve the exact distance metric outside the rounded region; it is an approximation that creates a smooth transition but changes the distance values globally [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_rounding(sdf_func, radius):
    """
    Returns a new SDF function that rounds the edges of the base SDF.
    
    Args:
        sdf_func: A callable that takes a point p and returns the signed distance.
        radius: A non-negative float representing the rounding radius.
        
    Returns:
        A callable that returns the rounded signed distance.
    """
    def rounded_sdf(p):
        # Step 1: Query the base SDF at the original point
        base_distance = sdf_func(p)
        
        # Step 2: Subtract the radius from the SDF value
        # This effectively rounds off hard edges and corners [S4, unofficial]
        rounded_distance = base_distance - radius
        
        # Step 3: Return the result
        return rounded_distance
        
    return rounded_sdf
```

## Subtopics

### [SDF Onion](1846-sdf-onion.md)
The SDF Onion operation creates a hollow shell or layered structure from a solid SDF primitive by taking the absolute value of the distance and subtracting a specified radius. It is a specific application of SDF Rounding that inverts the interior of the shape to create a void. Unlike standard rounding which smooths exterior edges, Onion creates internal layers by exploiting the symmetry of the absolute distance function.

### SDF Displacement Maps
A variant where the constant offset $r$ is replaced by a spatially varying function, such as a sine wave [S4, unofficial]. The formula becomes $d_{disp}(\mathbf{p}) = d_{base}(\mathbf{p}) + \text{offset}(\mathbf{p})$. This allows for dynamic surface details like ripples or noise, as seen in the example: $\text{offset} = \sin(k \cdot x) \sin(k \cdot y) \sin(k \cdot z) (0.25 + 0.25 \sin(\text{time}))$ [S4, unofficial]. This technique adds high-frequency detail without modifying the underlying geometry's topology.

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation, identical to the base SDF, as it involves only one additional arithmetic subtraction [S1, S3, unofficial].
- **Memory Complexity:** $O(1)$, requiring storage for the radius parameter [S3, unofficial].
- **Performance:** Negligible overhead compared to the base SDF evaluation.
- **Strengths:** Simple to implement; computationally cheap; effective for creating smooth, organic, or filleted shapes from basic primitives [S4, unofficial].
- **Limitations:** Does not produce true geometric fillets (the distance is not exact in the rounded region); large radii can distort the shape significantly or create voids [S2, S4, unofficial].
- **When to use:** When visual smoothness is preferred over mathematical precision of the distance field, or for quick prototyping of rounded shapes.
- **When not to use:** When exact distance metrics are required for accurate ray marching convergence or collision detection in the rounded regions.

## Background
SDF Rounding is a fundamental technique in ray marching and implicit surface rendering, popularized by the "ray marching and signed distance fields" educational materials from the University of Cambridge [S4, unofficial]. It leverages the property that subtracting a constant from a distance function shifts the zero-level set (the surface) inward, smoothing sharp features. The technique is widely used in real-time graphics for its efficiency in creating complex-looking geometry from simple primitives [S1, S2, S3, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S4] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
