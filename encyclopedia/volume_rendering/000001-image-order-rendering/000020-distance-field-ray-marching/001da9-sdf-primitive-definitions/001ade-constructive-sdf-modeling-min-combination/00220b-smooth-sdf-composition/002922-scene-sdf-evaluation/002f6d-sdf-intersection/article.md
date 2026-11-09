# SDF Intersection
Also called: SDF Boolean Intersection, Max Combination

## Overview
SDF Intersection is a Constructive Solid Geometry (CSG) operation that computes the signed distance field for the geometric intersection of two shapes. It solves the problem of combining SDF primitives into complex volumes by returning the region common to both input shapes. It is distinguished from its sibling [Union](4451-union.md) (which uses `min`) and [Subtraction](4453-subtraction.md) (which uses `max(a, -b)`) by its use of the maximum operator to enforce the constraint that a point must be inside both shapes.

## How it works
**Inputs and Outputs:**
- Input: Two signed distance functions, $S_A(p)$ and $S_B(p)$, and a query point $p$.
- Output: A scalar value $S_{intersect}(p)$ representing the distance to the nearest surface of the intersected volume.

**Data Structures:**
No specific data structures are required beyond the scalar values returned by the primitive SDF evaluations. The operation is point-wise and stateless.

**Steps:**
1. Evaluate the signed distance of point $p$ against the first shape $A$: $d_A = S_A(p)$.
2. Evaluate the signed distance of point $p$ against the second shape $B$: $d_B = S_B(p)$.
3. Compute the intersection distance by taking the maximum of the two distances:
   $$ S_{intersect}(p) = \max(d_A, d_B) $$

**Formulas:**
The core formula is:
$$ S_{intersect}(p) = \max(S_A(p), S_B(p)) $$

Where:
- $S_A(p)$ is the signed distance from $p$ to shape $A$.
- $S_B(p)$ is the signed distance from $p$ to shape $B$.
- $\max(a, b)$ returns the larger of $a$ and $b$.

**Parameters:**
None. The operation is exact and deterministic.

**Edge Cases and Pitfalls:**
- **Precision:** Standard floating-point precision is sufficient for most applications. However, if $S_A(p)$ and $S_B(p)$ are very close to zero (near the surface), the resulting distance may exhibit minor jitter if the normals are not computed carefully, though the distance value itself remains continuous.
- **Boundaries:** The resulting SDF is continuous and watertight. The zero-level set (surface) is the union of the portions of $S_A=0$ and $S_B=0$ that lie inside the other shape.
- **Degenerate Input:** If one shape is entirely contained within the other, the result is the SDF of the inner shape. If the shapes do not intersect, the resulting SDF will be positive everywhere in the region of interest (empty space), but the function remains valid.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_intersection(p, sdf_a, sdf_b):
    """
    Computes the SDF for the intersection of two shapes.
    
    Args:
        p: Point in 3D space (vector).
        sdf_a: Function returning distance to shape A.
        sdf_b: Function returning distance to shape B.
        
    Returns:
        Signed distance to the intersection of A and B.
    """
    dist_a = sdf_a(p)
    dist_b = sdf_b(p)
    return max(dist_a, dist_b)
```

**Subtopics:**
- [Smooth SDF Composition](4450-smooth-sdf-composition.md): Uses smooth min/max approximations for blended intersections.
- [Scene SDF Evaluation](4452-scene-sdf-evaluation.md): The parent method that aggregates these operations into a full scene.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, requiring two primitive SDF evaluations and one comparison.
- **Memory Complexity:** $O(1)$, no additional storage.
- **Strengths:** Exact, continuous, and produces watertight surfaces without the robustness problems that plague mesh-based CSG [S1, unofficial]. No acceleration structures are needed.
- **Limitations:** Like all ray marching methods, it suffers from high per-ray evaluation counts. Each ray requires tens to hundreds of scene evaluations [S1, unofficial]. No hardware acceleration exists for this specific shader code [S1, unofficial].
- **When to use:** When constructing complex boolean shapes from primitives in a ray-marched renderer.
- **When not to use:** When performance is critical and the scene is static, as mesh-based CSG or BSP trees may offer faster traversal.

## Background
Signed Distance Functions (SDFs) allow for continuous morphing of scenes with no additional overhead [S1, unofficial]. The boolean operations (Union, Intersection, Subtraction) are trivial to implement by modifying the SDF formula [S1, unofficial]. This approach was popularized in the context of ray marching for real-time rendering, notably by Alan Zucconi [S2, unofficial] and others in the demoscene and shader programming communities.

## Sources
[S1] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S2] Volumetric Rendering: Signed Distance Functions - Alan Zucconi, https://www.alanzucconi.com/2016/07/01/signed-distance-functions/, fetched 2026-10-28 (unofficial)
