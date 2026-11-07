# Natural Neighbor Interpolation

Also called: Sibson interpolation, Voronoi interpolation

## Overview
Natural neighbor interpolation is a spatial interpolation method that determines the value of a query point based on the geometric relationship between its Voronoi cell and the Voronoi cells of its neighboring seed points [S1]. It solves the problem of creating smooth, continuous, and monotonic interpolation fields where each seed point exerts only local influence, ensuring that the original seed values remain unchanged [S1]. It is distinguished from linear or polynomial interpolation by its adaptive weighting scheme, which relies on the topology of the Voronoi diagram rather than fixed distance metrics or basis functions [S1].

## How it works
**Inputs:**
- A set of seed customization points $S = \{p_1, p_2, \dots, p_N\}$ with associated scalar values $V = \{v_1, v_2, \dots, v_N\}$.
- A query point $q$ for which the interpolated value $v_q$ is required.
- A bounding box $B$ defining the finite domain of the volume data [S1].

**Outputs:**
- The interpolated scalar value $v_q$ [S1].

**Data Structures:**
- **Voronoi Diagram:** A partitioning of the space into cells $VC_{p_i}$, where each cell $VC_{p_i}$ contains all points closer to seed point $p_i$ than to any other seed point in $S$.
- **Intersection Volumes:** The geometric volume of the intersection between the new Voronoi cell of $q$ and the original Voronoi cells of its neighbors [S1].

**Steps:**
1.  **Construct Initial Voronoi Diagram:** Compute the Voronoi diagram for the set of seed points $S$. This defines the original Voronoi cells $VC_{p_i}$ for all $p_i \in S$ [S1].
2.  **Handle Infinite Cells:** To ensure all Voronoi cells have finite volume, clip the Voronoi diagram to the bounding box $B$ of the volume data set. This prevents undefined behavior for points outside the convex hull of $S$ [S1].
3.  **Insert Query Point:** Form the augmented set $Q = S \cup \{q\}$. Compute a new Voronoi diagram for $Q$ [S1].
4.  **Identify New Cell:** Identify the Voronoi cell $VC_q$ corresponding to the query point $q$ in the new diagram [S1].
5.  **Calculate Intersection Volumes:** For each neighbor $p_i$ (where $VC_{p_i}$ shares a boundary with $VC_q$), calculate the volume of the intersection between the original cell $VC_{p_i}$ and the new cell $VC_q$, denoted as $vol(VC_{p_i} \cap VC_q)$ [S1].
6.  **Compute Weights:** Calculate the weight $w_i$ for each neighbor $p_i$ as the ratio of the intersection volume to the total volume of the new cell $VC_q$:
    $$ w_i = \frac{vol(VC_{p_i} \cap VC_q)}{vol(VC_q)} $$
    [S1]
7.  **Interpolate Value:** Compute the final value $v_q$ by summing the products of the weights and the seed values:
    $$ v_q = \sum_{i=1}^N w_i \cdot v_i = \sum_{i=1}^N \frac{vol(VC_{p_i} \cap VC_q)}{vol(VC_q)} \cdot v_i $$
    [S1]

**Parameters:**
- **Default Seed Points:** Eight default seed customization points are provided, located at the corners of the volume's bounding box. This configuration prevents a single local seed point from having global influence and allows for simple linear gradients across the volume [S1].

**Pitfalls and Edge Cases:**
- **Infinite Voronoi Cells:** If the query point $q$ lies outside the convex hull of the seed points $S$, the Voronoi cell $VC_q$ may have infinite volume, making the denominator in the weight calculation undefined [S1]. Clipping the diagram to the bounding box $B$ resolves this by forcing finite cell volumes [S1].
- **Precision:** The calculation requires robust geometric predicates to determine Voronoi cell boundaries and intersection volumes accurately, especially in degenerate configurations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def natural_neighbor_interpolation(seed_points, seed_values, query_point, bounding_box):
    """
    Computes the interpolated value for a query point using natural neighbor interpolation.
    
    Args:
        seed_points: List of points P = [p1, p2, ..., pN]
        seed_values: List of values V = [v1, v2, ..., vN] corresponding to seed_points
        query_point: Point q
        bounding_box: Bounding box B to clip Voronoi cells
        
    Returns:
        Interpolated value v_q
    """
    # Step 1: Compute initial Voronoi diagram for seed points S
    # Note: Implementation of Voronoi diagram generation is assumed to be available.
    # The diagram must be clipped to bounding_box to ensure finite volumes.
    initial_voronoi = compute_voronoi_diagram(seed_points, clip_to=bounding_box)
    
    # Step 2: Form augmented set Q = S U {q}
    augmented_points = seed_points + [query_point]
    
    # Step 3: Compute new Voronoi diagram for Q
    # This diagram is also clipped to bounding_box
    new_voronoi = compute_voronoi_diagram(augmented_points, clip_to=bounding_box)
    
    # Step 4: Identify the new Voronoi cell VC_q for the query point
    vc_q = new_voronoi.get_cell(query_point)
    
    # Step 5: Calculate total volume of VC_q
    vol_vc_q = vc_q.volume()
    
    # Guard against zero volume (degenerate case)
    if vol_vc_q == 0:
        return 0.0 # Or handle as error/nearest neighbor fallback [G]

    interpolated_value = 0.0
    
    # Step 6 & 7: Iterate over neighbors and compute weighted sum
    # Neighbors are points whose original Voronoi cells intersect with VC_q
    for i, p_i in enumerate(seed_points):
        vc_pi = initial_voronoi.get_cell(p_i)
        
        # Calculate intersection volume between original cell VC_pi and new cell VC_q
        intersection_vol = vc_pi.intersection_volume(vc_q)
        
        # Calculate weight
        weight = intersection_vol / vol_vc_q
        
        # Accumulate weighted value
        interpolated_value += weight * seed_values[i]
        
    return interpolated_value
```

## Operating characteristics
- **Time Complexity:** The method requires constructing two Voronoi diagrams (one for $S$ and one for $S \cup \{q\}$). The complexity depends on the dimensionality and the number of seed points $N$. In 3D, Voronoi diagram construction is generally $O(N \log N)$ on average, but can be worse in degenerate cases. The intersection volume calculations add further geometric overhead [S1].
- **Memory Complexity:** Requires storing the Voronoi diagrams for both the initial and augmented sets. The memory usage scales with the complexity of the Voronoi tessellation [S1].
- **Strengths:** It satisfies continuity and monotonicity requirements and ensures local influence (occlusion), meaning only nearby seed points affect the interpolation result [S1]. It naturally adapts to the distribution of seed points [S1].
- **Limitations:** It is computationally expensive compared to linear interpolation due to the need for Voronoi diagram construction and geometric intersection calculations. It is sensitive to the distribution of seed points and requires careful handling of boundary conditions (infinite cells) [S1].

## Background
Natural neighbor interpolation was proposed by Sibson in 1981 [S1]. It is also known as Sibson interpolation or Voronoi interpolation [S1]. In the context of volume rendering, it is used for the position-dependent customization of transfer function classification parameters, allowing for smooth and intuitive user interaction with volume data [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
