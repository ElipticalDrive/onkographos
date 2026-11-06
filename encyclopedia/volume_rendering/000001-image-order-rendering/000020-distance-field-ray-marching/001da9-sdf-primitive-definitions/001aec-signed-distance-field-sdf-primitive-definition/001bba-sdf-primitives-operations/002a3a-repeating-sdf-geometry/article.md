# Repeating SDF Geometry
Also called: Modulated SDF, Repeated, Repeated SDF, SDF Repetition (Folding), Tiled SDF, Tiling SDF

## Overview
Repeating SDF Geometry creates a periodic tiling of a base signed distance field by folding 3D coordinates into a fundamental domain centered at the origin [S1, unofficial]. It segments space into infinite parallel regions of repeated distance by taking the modulus of a point’s position along one or more axes before computing the signed distance [S2, unofficial][S3, unofficial][S4, unofficial]. This technique allows for the infinite repetition of geometry for no extra computational cost, as the ray marching algorithm only evaluates the base primitive within the folded coordinate space [S2, unofficial][S3, unofficial][S4, unofficial]. It is distinct from [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) in that it specifically exploits periodicity for tiling, and from [Quadric SDF](2682-quadric-sdf-first-order-approximation.md) as it relies on coordinate wrapping rather than algebraic approximation.

## How it works
**Inputs and Outputs**
*   **Input:** A query point $P = (x, y, z)$ in world space, a base SDF function $S_{base}(P)$, and a repetition period $T$ (typically a scalar or vector).
*   **Output:** The signed distance value $d$ from the query point to the nearest surface of the tiled geometry.

**Data Structures**
The method requires no persistent data structures. It operates statelessly on the input coordinates. The period $T$ is typically stored as a uniform variable or constant. In some implementations, it is part of a struct field, e.g., `Repeated struct { base_sdf, period }` [S1, unofficial].

**Parameters**
*   $T$: The spacing between repetitions along each axis. In the canonical example, $T=4$ [S3, unofficial][S4, unofficial].
*   Offset: An offset applied before the modulus operation to center the repetition domain. In the canonical example, the offset is $T/2 = 2$ [S3, unofficial][S4, unofficial].

**Algorithm Steps**
1.  For each coordinate $c \in \{x, y, z\}$ that is to be repeated:
    a.  Apply an offset: $c_{off} = c + \frac{T}{2}$.
    b.  Apply the modulo operation: $c_{mod} = c_{off} \pmod T$.
    c.  Center the result: $c' = c_{mod} - \frac{T}{2}$.
    *   Note: Coordinates not intended for repetition are passed through unchanged ($c' = c$) [S2, unofficial][S3, unofficial][S4, unofficial].
2.  Construct the transformed point $P' = (x', y', z')$ [S1, unofficial].
3.  Evaluate the base SDF at the transformed point: $d = S_{base}(P')$ [S1, unofficial][S4, unofficial].

**Formula**
For a coordinate $c$ and period $T$:
$$ c' = \left( (c + \frac{T}{2}) \pmod T \right) - \frac{T}{2} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def repeating_sdf(pt: Vector3, base_sdf: Callable[[Vector3], float], period: float) -> float:
    """
    Computes the signed distance for repeating geometry.
    
    Args:
        pt: The query point (x, y, z).
        base_sdf: The base signed distance function (e.g., sphere, cube).
        period: The spacing between repetitions (scalar applied to x and z in the canonical example).
        
    Returns:
        The signed distance value.
    """
    # Canonical example from sources uses period 4 and offset 2 for x and z, leaving y unchanged [S2, unofficial][S3, unofficial][S4, unofficial]
    # Generalizing for any axis:
    
    # Step 1: Fold coordinates into the fundamental domain centered at origin
    # Formula: c' = ((c + period/2) % period) - period/2
    
    x_folded = ((pt.x + period / 2.0) % period) - period / 2.0
    y_folded = pt.y  # Not repeated in the canonical example [S2, unofficial][S3, unofficial][S4, unofficial]
    z_folded = ((pt.z + period / 2.0) % period) - period / 2.0
    
    # Construct the transformed point
    pt_folded = Vector3(x_folded, y_folded, z_folded)
    
    # Step 2: Evaluate base SDF
    return base_sdf(pt_folded)
```

**Edge Cases and Pitfalls**
*   **Precision:** The modulo operation can suffer from floating-point precision issues near the boundaries of the fundamental domain ($\pm T/2$). This may cause visible seams or incorrect distance calculations if the ray step size is not sufficiently small relative to the period [S4, unofficial].
*   **Boundary Conditions:** The technique relies on the periodicity of the modulus function. If the base SDF is not symmetric or periodic-friendly, artifacts may appear at the tile boundaries [S4, unofficial].
*   **Non-repeated axes:** Failing to leave non-repeated axes (like $y$ in the canonical example) unchanged will result in unintended tiling in those directions [S2, unofficial].

## Subtopics

### Modulated SDF
A variation where the repetition is combined with displacement maps or other modulations to create complex surface details. It sets itself apart by adding variation to the repeated instances. Choose this when you need varied, non-identical tiles. See [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md).

### Tiled SDF
Often used interchangeably with Repeating SDF, but may imply a finite number of tiles rather than infinite repetition. Choose this when memory constraints require culling distant tiles. See [Basic Primitive Definitions](5346-basic-primitive-definitions.md).

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray step, identical to the base SDF evaluation. The folding operation adds negligible overhead.
*   **Memory Complexity:** $O(1)$. No additional memory is required for the tiling logic.
*   **Measured Performance:** Infinite repetition is achieved for "no extra cost" compared to a single instance [S2, unofficial][S3, unofficial][S4, unofficial].
*   **Strengths:** Extremely efficient for creating large-scale environments, grids, or repetitive patterns without increasing draw calls or geometry complexity.
*   **Limitations:** Only supports periodic, regular tiling. Cannot easily create irregular or sparse repetitions. Precision issues may arise at tile boundaries.
*   **When to use:** When you need to render large numbers of identical objects (e.g., floor tiles, pillars, stars) efficiently.
*   **When not to use:** When objects need to be unique, irregularly spaced, or when the repetition period is very small relative to the ray step size (risk of aliasing/seams).

## Background
The technique is attributed to the work of Inigo Quilez, who popularized the use of SDFs for procedural geometry in computer graphics [S2, unofficial][S3, unofficial][S4, unofficial]. It builds on the foundational work of John C. Hart on Sphere Tracing and ray tracing deterministic fractals [S2, unofficial][S3, unofficial][S4, unofficial]. The specific method of folding coordinates via modulo arithmetic to create infinite repetition is a standard technique in the SDF community, often demonstrated in live demos and educational materials [S2, unofficial][S3, unofficial][S4, unofficial].

## Sources
[S1] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S2] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S3] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S4] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
