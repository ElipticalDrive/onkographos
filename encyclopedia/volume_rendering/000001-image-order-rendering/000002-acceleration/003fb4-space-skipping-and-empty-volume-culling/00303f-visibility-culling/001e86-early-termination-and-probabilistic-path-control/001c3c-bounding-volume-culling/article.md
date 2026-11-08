# Bounding Volume Culling

Also called: Distance-based Culling, Ray Termination by Distance

## Overview
Bounding Volume Culling is an optimization that limits raymarching to a specific bounding volume, dropping rays that exceed a certain distance from the eye to speed up rendering [S1, unofficial]. It differs from its siblings, such as [Empty-space skipping](1002-empty-space-skipping.md) which skips empty regions, and [Early Stop](1790-early-stop.md) which stops based on opacity thresholds, by terminating rays based on geometric extent rather than material properties or empty space. It builds on [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) by applying a deterministic geometric bound to the ray traversal loop.

## How it works
**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, maximum travel distance $T_{max}$.
- Output: Boolean indicating if the ray should be processed, or the remaining distance budget.

**Data Structures:**
- $t$: Accumulated distance traveled by the ray along $\mathbf{d}$. Initialized to 0.
- $T_{max}$: Predefined limit representing the radius or extent of the bounding volume.

**Steps:**
1. Initialize $t = 0$.
2. During the raymarching loop, after each step of size $\Delta t$, update $t \leftarrow t + \Delta t$.
3. Check if $t > T_{max}$.
4. If true, terminate the raymarch process immediately [S1, unofficial].

**Formula:**
The condition for termination is:
$$ t > T_{max} $$
Where $t$ is the sum of all step sizes taken along the ray direction from the eye [S1, unofficial].

**Parameters:**
- $T_{max}$: The maximum distance from the eye. Typical values depend on the scene scale; for example, a value of 64 steps was used in a demonstration where step size was normalized [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $T_{max}$ is set too low, valid geometry near the boundary may be clipped.
- If $T_{max}$ is set too high, the culling benefit is reduced.
- The method assumes the bounding volume is centered or positioned such that the distance from the eye accurately reflects the relevance of the geometry.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_volume_culling(ray_origin, ray_direction, max_distance):
    """
    Determines if a ray should be processed based on a maximum travel distance.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, normalized direction of the ray.
        max_distance: float, the maximum allowed distance from the eye.
        
    Returns:
        bool: True if the ray is within the bounding volume, False otherwise.
    """
    # Note: In a full raymarcher, this check is performed inside the loop.
    # This function represents the termination condition check.
    accumulated_distance = 0.0
    
    # Simulate a single step for demonstration of the check logic
    # In practice, this is checked after every step
    step_size = 0.1 
    accumulated_distance += step_size
    
    if accumulated_distance > max_distance:
        return False  # Ray exceeds bounding volume
    return True     # Ray is within bounding volume
```

## Subtopics

### Heap-based Bounding Volume Culling for Sphere Tracing
An algorithm to find the closest object in a collection by maintaining a heap of bounding volume distances. It computes distances to bounding volumes first, then contents, culling based on the smallest bounding volume distance [S2]. It differs from standard Bounding Volume Culling by optimizing the search for the closest object in a union of objects rather than just terminating a ray based on global distance [S2].

**Steps:**
1. Compute distances to each bounding volume in the union/collection.
2. Create a heap of these distances.
3. Initialize $d_{min}$ to infinity.
4. Loop:
    a. Let $d_{content}$ be $\min(d_{min}, \text{distance to contents of top BV})$.
    b. Remove top of heap.
    c. Let $d_{bv}$ be distance to new top BV.
    d. Stop if $d_{content} < d_{bv}$ or heap empty.
5. Return $d_{content}$ [S2].

**Parameters:**
- $d_{bv}$: The distance to the bounding volume currently at the top of the heap [S2].

## Operating characteristics
- **Time Complexity:** The basic culling check is $O(1)$ per step. The heap-based variant adds $O(\log N)$ overhead per step for $N$ bounding volumes, but reduces the number of expensive distance computations for contents.
- **Memory Complexity:** $O(N)$ for storing bounding volumes and the heap.
- **Strengths:** Significantly reduces computation for rays that miss the scene or travel through empty space far from the camera.
- **Limitations:** Requires pre-computation of bounding volumes. May degrade performance in perspective cases for typical implicit surfaces if the overhead of distance checks outweighs the savings [S2].
- **When to use:** When the scene has a well-defined spatial extent and many rays miss the geometry entirely.

## Background
The concept of limiting raymarching to a bounding volume is discussed in the context of raymarching distance fields, where it is used to speed up rendering by dropping rays that exceed a certain distance [S1, unofficial]. The more sophisticated heap-based approach for sphere tracing is described in the context of geometric methods for antialiased ray tracing of implicit surfaces, where bounding volumes help avoid distance computations for objects farther away than others [S2].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
