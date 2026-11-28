# SDF Rounding

Also called: Distance Offset, Rounded, Rounded Corners in SDFs, Rounding Transformation, SDF Domain Manipulation, SDF Offset, SDF Rounding/Offset

## Overview
SDF Rounding is a geometric transformation that modifies a Signed Distance Field (SDF) primitive by subtracting a constant scalar value from its distance result, effectively rounding off hard edges and corners with a specified radius [S1, S5, unofficial]. It solves the problem of creating smooth, organic, or chamfered geometry from sharp primitives without altering the underlying mesh or topology. Unlike CSG operations which combine shapes, or Elongation which stretches them, Rounding uniformly expands or contracts the implicit surface by a fixed distance [S3, unofficial]. It is a specific instance of domain manipulation where the distance value itself is offset rather than the query position [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$ and a base SDF function $d_{base}(\mathbf{p})$.
*   **Parameter:** A scalar radius $r$ (or offset value) representing the rounding amount. If $r > 0$, the shape shrinks (or edges round inward); if $r < 0$, the shape expands (or edges round outward) [S1, S3, unofficial].
*   **Output:** The rounded signed distance $d_{rounded}$.

**Data Structures:**
No complex data structures are required. The operation is a pure function evaluation. In object-oriented implementations, the radius is often stored as a field within the rounding operator struct [S2, S3, unofficial].

**Steps:**
1.  Evaluate the base SDF at the query point $\mathbf{p}$ to obtain the raw distance $d = d_{base}(\mathbf{p})$.
2.  Subtract the rounding radius $r$ from $d$.
3.  Return the result.

**Formula:**
$$ d_{rounded}(\mathbf{p}) = d_{base}(\mathbf{p}) - r $$

Where:
*   $d_{base}(\mathbf{p})$ is the signed distance from $\mathbf{p}$ to the unrounded primitive.
*   $r$ is the rounding radius (scalar).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_rounding(p: Vec3, base_sdf: Callable[[Vec3], float], radius: float) -> float:
    """
    Rounds the edges of a shape defined by base_sdf by a specified radius.
    
    Args:
        p: The query point in 3D space.
        base_sdf: A function that returns the signed distance of the base primitive at p.
        radius: The radius of rounding. Positive values round inward (shrink), 
                negative values round outward (expand).
                
    Returns:
        The rounded signed distance.
    """
    # Step 1: Query the base SDF at the original point
    d = base_sdf(p)
    
    # Step 2: Subtract the radius from the SDF value
    # Note: Subtracting a positive radius reduces the distance, 
    # effectively moving the isosurface (d=0) inward.
    d_rounded = d - radius
    
    # Step 3: Return the result
    return d_rounded
```

**Edge Cases and Pitfalls:**
*   **Precision:** When $r$ is large relative to the primitive size, the shape may disappear entirely if the entire volume is rounded away. The SDF becomes positive everywhere inside the original volume and negative outside, but the "surface" may vanish if $r$ exceeds the minimum feature size.
*   **Sign Convention:** Ensure consistency with the base SDF's sign convention. If $d_{base}$ is positive outside and negative inside, subtracting $r$ moves the zero-crossing inward. If the base SDF is inverted, the effect is reversed.
*   **Degenerate Input:** If the base SDF is not a valid distance field (e.g., discontinuous), the rounded result will also be invalid.

## Subtopics

### [SDF Onion](1846-sdf-onion.md)
The SDF Onion operation creates a hollow shell or layered structure from a solid SDF primitive by taking the absolute value of the distance and subtracting a specified radius. It is a specific application of SDF Rounding that inverts the interior of the shape to create a void. It is chosen when a hollow object or a series of concentric shells is required. Link: [SDF Onion](1846-sdf-onion.md).

### SDF Displacement Maps
This variant applies a spatially varying offset to the SDF, effectively creating a displacement map on the implicit surface. The offset is calculated as a function of position and time, allowing for dynamic, organic deformation. It is chosen when procedural noise or wave-like distortions are needed on a surface. The formula adds a variable offset to the base SDF: $d_{result} = d_{base}(\mathbf{p}) + \text{offset}(\mathbf{p}, t)$ [S5, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(1)$ per query point. The operation involves one function call to the base SDF and one subtraction. It does not increase the computational cost of ray marching, only the cost of each step.
*   **Memory:** $O(1)$. No additional memory is required beyond storing the radius parameter.

**Measured Performance:**
Performance is identical to the base SDF evaluation. On modern GPUs, the subtraction is negligible compared to the cost of evaluating complex primitives (e.g., torus, box with rounding).

**Scaling:**
Scales linearly with the complexity of the base SDF. Rounding a simple sphere is faster than rounding a complex CSG combination, but the rounding operation itself adds no overhead.

**Strengths and Limitations:**
*   **Strengths:** Extremely fast; creates perfectly smooth, mathematically accurate rounded edges; no tessellation artifacts; works with any base SDF.
*   **Limitations:** Cannot create sharp features; large rounding radii can distort the shape significantly or cause it to vanish; does not support varying radius across the surface (unless combined with displacement maps).

**When to Use:**
Use when you need smooth, organic, or chamfered geometry from sharp primitives. Ideal for UI elements, soft-body physics approximations, or stylized graphics.

**When Not to Use:**
Do not use when sharp edges are required, or when the rounding radius is too large relative to the object size, causing topological changes or disappearance of the object.

## Background
SDF Rounding is a fundamental technique in ray marching and implicit surface rendering, popularized by Inigo Quilez in his seminal articles on ray marching and SDFs [S5, unofficial]. It leverages the mathematical property that the signed distance function represents the distance to the surface, so offsetting this distance uniformly shifts the isosurface. It is widely used in real-time graphics, particularly in demoscene productions and procedural generation, due to its efficiency and visual appeal.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S3] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S4] nschaievitch/sdf-editor, https://github.com/nschaievitch/sdf-editor, fetched 2026-10-26 (unofficial)
[S5] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
