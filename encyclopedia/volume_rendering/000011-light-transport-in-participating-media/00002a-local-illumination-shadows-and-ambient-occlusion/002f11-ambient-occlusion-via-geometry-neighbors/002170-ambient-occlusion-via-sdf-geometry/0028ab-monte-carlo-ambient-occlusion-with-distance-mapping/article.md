# Monte Carlo Ambient Occlusion with Distance Mapping

Also called: Random Ray Casting AO

## Overview
This method computes the ambient occlusion factor for a point $\mathbf{p}$ by numerically integrating visibility over the hemisphere of incoming light directions using random sampling [S1, unofficial]. It distinguishes itself from deterministic grid-based or screen-space approaches by relying on a distance mapping function $v(\mathbf{p}, \mathbf{l}_i)$ to evaluate occlusion along random ray directions, allowing for flexible integration with various geometric representations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Point position $\mathbf{p}$, surface normal $\mathbf{n}$, number of samples $N$, distance mapping function $v(\mathbf{p}, \mathbf{l})$.
*   **Output:** Ambient occlusion factor $k_A(\mathbf{p})$.

**Data Structures:**
No specific complex data structures are required beyond the ability to evaluate the distance mapping function $v$ for a given point and direction. The function $v$ typically returns a floating-point value representing the distance to the nearest geometry or an occupancy indicator [S1, unofficial].

**Steps:**
1.  Initialize the accumulator $k_A = 0$.
2.  For $i$ from $1$ to $N$:
    a. Generate a random direction vector $\mathbf{l}_i$ uniformly distributed over the hemisphere defined by $\mathbf{n}$.
    b. Evaluate the distance mapping function $v_i = v(\mathbf{p}, \mathbf{l}_i)$.
    c. Compute the dot product $d = \mathbf{n} \cdot \mathbf{l}_i$.
    d. Apply the positive part function: $w = \max(0, d)$.
    e. Accumulate the term: $k_A \leftarrow k_A + v_i \cdot w$.
3.  Normalize the result: $k_A(\mathbf{p}) = \frac{1}{N} k_A$.

**Formula:**
The ambient occlusion factor is calculated as [S1, unofficial]:

$$ k_A(\mathbf{p}) = \frac{1}{N} \sum_{i=1}^N v(\mathbf{p}, \mathbf{l}_i) (\mathbf{n} \cdot \mathbf{l}_i)^+ $$

Where:
*   $k_A(\mathbf{p})$ is the ambient occlusion factor for position $\mathbf{p}$.
*   $N$ is the integer number of random samples.
*   $v(\mathbf{p}, \mathbf{l}_i)$ is the distance mapping function returning a floating-point value.
*   $\mathbf{l}_i$ is the $i$-th random direction vector.
*   $\mathbf{n}$ is the surface normal vector.
*   $(\cdot)^+$ denotes the positive part function $\max(0, \cdot)$.

**Parameters:**
*   $N$: Integer representing the number of random samples. Typical values depend on desired quality; higher $N$ reduces noise but increases computation time [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Halo Effect:** Because this basic formulation is not cosine-weighted in the sampling distribution (rays are uniform, not biased by cosine), it may account for occlusion that should not be accounted for, leading to blackened edges around objects, often referred to as the "halo effect" [S1, unofficial].
*   **Distance Mapping Definition:** The behavior of $v(\mathbf{p}, \mathbf{l}_i)$ is critical. If $v$ returns 1 for free space and 0 for occluded, the formula works as a visibility integrator. If $v$ returns distance, the interpretation changes. The sources define it generally as a "distance mapping function" returning a float [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_ao_with_distance_mapping(p, n, N, distance_mapping_func):
    """
    Computes ambient occlusion using Monte Carlo integration with a distance mapping function.
    
    Args:
        p: 3D vector, the point position.
        n: 3D vector, the surface normal (assumed normalized).
        N: int, number of random samples.
        distance_mapping_func: function(p, l) -> float, returns the occlusion/distance value.
        
    Returns:
        float, the ambient occlusion factor k_A(p).
    """
    k_A = 0.0
    
    for i in range(N):
        # Generate a random direction l_i uniformly in the hemisphere defined by n
        # [G] Implementation detail: Sample a point on the unit hemisphere.
        # A common method is to sample theta in [0, pi/2] and phi in [0, 2pi]
        # with appropriate PDF weighting, or use a rejection sampling method.
        l_i = random_hemisphere_direction(n)
        
        # Evaluate distance mapping function
        v = distance_mapping_func(p, l_i)
        
        # Compute cosine weight (positive part)
        cos_theta = dot(n, l_i)
        weight = max(0.0, cos_theta)
        
        # Accumulate
        k_A += v * weight
        
    # Normalize
    k_A /= N
    
    return k_A

def random_hemisphere_direction(n):
    """
    [G] Helper to generate a random direction uniformly distributed on the hemisphere
    defined by normal n.
    """
    # Generate random point on unit sphere
    x = random_uniform(-1, 1)
    y = random_uniform(-1, 1)
    z = random_uniform(-1, 1)
    length = sqrt(x*x + y*y + z*z)
    if length == 0:
        return n # Fallback
    
    # Normalize to get point on sphere
    s_x = x / length
    s_y = y / length
    s_z = z / length
    
    # Ensure it is in the hemisphere (dot product with n > 0)
    # If not, flip the vector
    if dot(n, (s_x, s_y, s_z)) < 0:
        s_x = -s_x
        s_y = -s_y
        s_z = -s_z
        
    return (s_x, s_y, s_z)
```

## Subtopics

### Volumetric Obscurance (Loos and Sloan)
A Monte Carlo integration formulation of screen-space AO that integrates analytically in the z-dimension to reduce sample count [S1, unofficial]. It defines the occlusion as $v_A(\mathbf{p}) = \int_{x \in X} \rho(d(\mathbf{p}, x)) O(x) dx$, where $X$ is a 3D spherical neighborhood, $\rho$ is a distance mapping function, and $O(x)$ is the occupancy function [S1, unofficial]. The distance mapping function can be substituted with a constant function as it has little impact on visual quality [S1, unofficial]. It requires fewer samples than Crytek's method to achieve the same image quality [S1, unofficial].

### Malley's Method
An approximation that substitutes the cosine weight factor with importance sampling by imbuing the cosine weight into the ray cast distribution [S1, unofficial]. Instead of casting rays evenly along the hemisphere and applying a cosine weight, rays are cast unevenly, biased towards the surface normal direction [S1, unofficial]. This reduces the need for a separate cosine weight factor in the integration sum [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per point, where $N$ is the number of samples.
*   **Memory Complexity:** Low, primarily storing the precomputed occlusion values if done offline, or evaluating on the fly.
*   **Strengths:** Flexible integration with various geometric representations via the distance mapping function; conceptually simple [S1, unofficial].
*   **Limitations:** Can suffer from the "halo effect" due to lack of cosine-weighted sampling distribution; noisy for low $N$ [S1, unofficial].
*   **When to use:** When a flexible, geometry-agnostic AO method is needed and performance allows for random sampling [S1, unofficial].

## Background
Monte Carlo methods are the most common way to precompute ambient occlusion, using either a visibility function or a distance mapping function approach [S1, unofficial]. The specific formulation using the distance mapping function $v(\mathbf{p}, \mathbf{l}_i)$ is described in [S1, unofficial]. Kontkanen and Laine [29] and Loos and Sloan [34] are cited in the context of related AO techniques, with Loos and Sloan expanding on Crytek's approach [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
