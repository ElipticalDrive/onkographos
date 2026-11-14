# Natural Neighbor Interpolation

Also called: Sibson interpolation, Voronoi interpolation

## Overview
Natural neighbor interpolation is a spatial interpolation method that determines the value of a query point based on the geometric relationship between its Voronoi cell and the Voronoi cells of its neighboring seed points [S1]. It solves the problem of creating smooth, continuous, and monotonic interpolation fields where each seed point exerts only local influence, ensuring that the original seed values remain unchanged [S1]. It is distinguished from linear or polynomial interpolation by its adaptive weighting scheme, which relies on the topology of the Voronoi diagram rather than fixed distance metrics or basis functions [S1].

## How it works
**Inputs:**
- A set of seed customization points $S = \{p_1, p_2, \dots, p_N\}$ with associated scalar values $V = \{v_1, v_2, \dots, v_N\}$ [S1].
- A query point $q$ for which the interpolated value $v_q$ is required [S1].
- A bounding box $B$ defining the finite domain of the volume data [S1].

**Outputs:**
- The interpolated scalar value $v_q$ [S1].

**Data Structures:**
- **Voronoi Diagram:** A partitioning of the space into cells $VC_{p_i}$, where each cell $VC_{p_i}$ contains all points closer to seed point $p_i$ than to any other seed point in $S$ [G].
- **Intersection Volumes:** The geometric volume of the intersection between the new Voronoi cell of $q$ and the original Voronoi cells of its neighbors [S1].

**Steps:**
1. **Construct Initial Voronoi Diagram:** Compute the Voronoi diagram for the set of seed points $S$ [S1].
2. **Handle Infinite Cells:** Limit the Voronoi diagram to the bounding box $B$ of the volume data set so that all Voronoi cells have a finite volume [S1].
3. **Insert Query Point:** Form the augmented set $Q = S \cup \{q\}$ and compute a new Voronoi diagram for $Q$ [S1].
4. **Identify New Cell:** Identify the new Voronoi cell $VC_q$ for point $q$ in the new diagram [S1].
5. **Calculate Intersection Volumes:** Calculate the volume of the intersection between $VC_q$ and each original Voronoi cell $VC_{p_i}$ of the neighbors [S1].
6. **Compute Weights:** Compute the weight for each neighbor as the ratio of the intersection volume to the total volume of $VC_q$ [S1].
7. **Interpolate Value:** Sum the products of weights and seed values $v_i$ to get $v_q$ [S1].

**Formula:**
$$v_q = \sum_{i=1}^N \frac{vol(VC_{p_i} \cap VC_q)}{vol(VC_q)} \cdot v_i$$
where $v_q$ is the interpolated value for query point $q$, $N$ is the number of neighbors, $vol(VC_{p_i} \cap VC_q)$ is the volume of the intersection between the original Voronoi cell of seed point $p_i$ and the new Voronoi cell of $q$, $vol(VC_q)$ is the volume of the new Voronoi cell of $q$, and $v_i$ is the value associated with seed point $p_i$ [S1].

**Parameters:**
- **Default Seed Points:** Eight default seed customization points are provided, each corresponding to a corner of the volume's bounding box, to prevent global influence from a single local seed point and allow simple linear gradients [S1].

**Pitfalls and Edge Cases:**
- **Infinite Voronoi Cells:** A Voronoi cell can have infinite volume if its associated point is not strictly inside the convex hull of the given set of points, making the weight calculation undefined for query points outside the convex hull of seed points [S1]. Clipping the diagram to the bounding box resolves this by forcing finite cell volumes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def natural_neighbor_interpolation(seed_points, seed_values, query_point, bounding_box):
    # Step 1: Compute initial Voronoi diagram for seed points S, clipped to bounding_box
    # Note: The sources do not provide the Voronoi diagram construction algorithm [G].
    initial_voronoi = compute_voronoi_diagram(seed_points, clip_to=bounding_box)
    
    # Step 2: Form augmented set Q = S U {q}
    augmented_points = list(seed_points) + [query_point]
    
    # Step 3: Compute new Voronoi diagram for Q, clipped to bounding_box
    new_voronoi = compute_voronoi_diagram(augmented_points, clip_to=bounding_box)
    
    # Step 4: Identify the new Voronoi cell VC_q for the query point
    vc_q = new_voronoi.get_cell(query_point)
    
    # Step 5: Calculate total volume of VC_q
    vol_vc_q = vc_q.volume()
    
    # Guard against zero volume (degenerate case)
    if vol_vc_q == 0:
        return 0.0 # Fallback for degenerate configuration [G]

    interpolated_value = 0.0
    N = len(seed_points)
    
    # Step 6 & 7: Iterate over neighbors and compute weighted sum
    for i in range(N):
        p_i = seed_points[i]
        vc_pi = initial_voronoi.get_cell(p_i)
        
        # Note: The sources do not provide the geometric intersection volume calculation [G].
        intersection_vol = vc_pi.intersection_volume(vc_q)
        
        # Calculate weight
        weight = intersection_vol / vol_vc_q
        
        # Accumulate weighted value
        interpolated_value += weight * seed_values[i]
        
    return interpolated_value
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** It satisfies continuity and monotonicity requirements and ensures local influence (occlusion), meaning only nearby seed points affect the interpolation result [S1]. It naturally adapts to the distribution of seed points [S1].
- **Limitations:** It is computationally expensive compared to linear interpolation due to the need for Voronoi diagram construction and geometric intersection calculations [S1]. It is sensitive to the distribution of seed points and requires careful handling of boundary conditions (infinite cells) [S1].

## Background
Natural neighbor interpolation was proposed by Sibson in 1981 [S1]. It is also known as Sibson interpolation or Voronoi interpolation [S1]. In the context of volume rendering, it is used for the position-dependent customization of transfer function classification parameters, allowing for smooth and intuitive user interaction with volume data [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
