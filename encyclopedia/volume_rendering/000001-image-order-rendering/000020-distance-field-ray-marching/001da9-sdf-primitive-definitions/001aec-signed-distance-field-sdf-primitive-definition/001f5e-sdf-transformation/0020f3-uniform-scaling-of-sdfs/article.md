# Uniform Scaling of SDFs

Also called: Uniform Scaling of Signed Distance Functions, SDF Scaling, Scaling SDFs, SDF Scale

## Overview
Uniform scaling of Signed Distance Fields (SDFs) is a technique to transform an implicit surface by a uniform factor while preserving the correctness of the distance value for ray marching [S1, unofficial][S3, unofficial]. It solves the problem that scaling is not a rigid body transformation and distorts distances; without correction, the ray marching algorithm may overestimate the distance to the surface, causing the object to disappear [S1, unofficial][S3, unofficial]. Unlike [Non-Uniform Scaling of SDFs](2423-non-uniform-scaling-of-sdfs.md), which requires conservative bounds to avoid overestimation, uniform scaling allows for an exact compensation of distance distortion [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A base SDF function $f(p)$, a query point $p$ in world space, and a uniform scaling factor $s$ (where $s > 0$) [S1, unofficial][S4].
- Output: The signed distance from $p$ to the scaled surface [S1, unofficial].

**Data Structures:**
The scaling factor is stored as a scalar floating-point value (e.g., `f32`) as the second field of the `Scaled` struct [S2, unofficial].

**Steps:**
1. **Inverse Transform Point:** Divide the query point $p$ by the scaling factor $s$ to compress the evaluation space [S1, unofficial][S3, unofficial][S4]. If the primitive has a center offset $o$, the operation is $(p - o) / s$ [S4].
   $$ p' = \frac{p}{s} $$
2. **Evaluate Base SDF:** Query the base SDF function with the transformed point $p'$ to get the raw distance $d'$ [S1, unofficial][S3, unofficial].
   $$ d' = f(p') $$
3. **Rescale Distance:** Multiply the returned distance $d'$ by the scaling factor $s$ to compensate for the distortion caused by the non-rigid scaling [S1, unofficial][S3, unofficial][S4].
   $$ d = d' \times s $$

**Formula:**
The general formula for a uniformly scaled SDF is:
$$ d_{scaled}(p) = f\left(\frac{p}{s}\right) \times s $$
For a specific primitive like a sphere with center $o$ and base radius $r$, the scaled SDF is:
$$ d_{sphere}(p) = \left( \left\| \frac{p - o}{s} \right\| - r \right) \times s $$
where $\|\cdot\|$ denotes the Euclidean norm [S4].

**Parameters:**
- $s$: The uniform scaling factor. Values $s > 1$ enlarge the surface footprint; values $s < 1$ shrink it [S4].

**Pitfalls and Edge Cases:**
- **Overestimation:** If the distance is overestimated (e.g., scaling down without correction: `sphereSDF(samplePoint / 0.5)`), the ray marching algorithm may step past the surface, causing the object to disappear completely [S1, unofficial][S3, unofficial].
- **Underestimation:** If the distance is underestimated (e.g., using no correction or a smaller correction factor), the ray marching algorithm still works but requires more iterations, resulting in slower performance [S1, unofficial].
- **Sign Preservation:** The sign of the distance is preserved regardless of the scaling correction factor, so the object remains visible even with incorrect distance values, but the geometric accuracy is lost [S1, unofficial].
- **Zero Scaling:** $s=0$ is undefined and should be guarded against [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scale_sdf_uniform(sdf_func, point, scale_factor):
    """
    Computes the signed distance to a uniformly scaled SDF.
    
    Args:
        sdf_func: A function that takes a point and returns the signed distance.
        point: The query point in world space (vector3).
        scale_factor: The uniform scaling factor (scalar > 0).
        
    Returns:
        The signed distance from point to the scaled surface.
    """
    if scale_factor == 0:
        raise ValueError("Scaling factor must be non-zero.")
        
    # Step 1: Inverse transform point
    scaled_point = point / scale_factor
    
    # Step 2: Evaluate base SDF
    raw_distance = sdf_func(scaled_point)
    
    # Step 3: Rescale distance
    final_distance = raw_distance * scale_factor
    
    return final_distance
```

## Subtopics

### [Non-Uniform Scaling of SDFs](2423-non-uniform-scaling-of-sdfs.md)
Non-uniform scaling of Signed Distance Fields (SDFs) is a technique to transform an SDF primitive by scaling its axes by different factors, ensuring the ray marching algorithm does not overestimate the distance to the surface. Unlike uniform scaling, which allows for exact distance compensation, non-uniform scaling distorts distances anisotropically; this method uses the minimum scaling factor as a conservative correction multiplier to guarantee distance underestimation, which is safe for ray marching convergence.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Exact distance preservation prevents overestimation, which would cause the ray marching algorithm to step past the surface and miss the object [S1, unofficial][S3, unofficial].
- **Limitations:** Only applicable to uniform scaling. Non-uniform scaling requires different techniques [S1, unofficial].
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The technique applies the inverse transformation to the query point before evaluating the base SDF, then scales the returned distance to compensate for non-rigid distortion [S1, unofficial][S3, unofficial].

## Sources
[S1] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S3] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S4] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
