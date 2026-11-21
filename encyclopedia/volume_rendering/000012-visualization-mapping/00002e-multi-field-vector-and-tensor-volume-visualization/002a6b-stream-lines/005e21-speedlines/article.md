# Speedlines

Also called: Motion Streaks, Streak Lines

## Overview
Speedlines are an illustrative visualization technique used to indicate the motion and history of specific features within time-varying datasets [S1, unofficial]. Unlike standard stream lines that trace particle trajectories through a vector field, speedlines superimpose geometric primitives (cones) along the receding bounding contour of a moving feature to provide intuitive visual cues of direction and velocity history [S1, unofficial]. This method is distinct from its siblings, such as [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md) which focuses on density, and [Directional Textured Tracelines](6965-directional-textured-tracelines.md) which use texture, by explicitly modeling the "streak" effect of a moving object's silhouette [S1, unofficial].

## How it works
**Inputs:**
*   Feature geometry $f$ (typically a mesh or point set).
*   Feature motion data: average velocity vector $\mathbf{v}_{avg}$ and angular velocity vector $\boldsymbol{\omega}$ [S1, unofficial].
*   User parameters: number of speedlines $N$, cone base radius $r_{base}$, time-difference $\Delta t$, number of cone facets $n_{facets}$, and color/intensity [S1, unofficial].

**Outputs:**
*   A mesh geometry containing cones representing the speedlines, colored according to user specifications [S1, unofficial].

**Data Structures:**
*   **Bounding Contour ($bc$):** A list of 3D points $\{p_1, p_2, \dots, p_m\}$ representing the smoothed boundary of the feature.
*   **Centroid ($c$):** The 3D center of mass of the feature.
*   **Indices ($i_{min}, i_{max}$):** Integer indices into the contour list identifying the start and end of the trailing portion.

**Algorithm Steps:**

1.  **Obtain Bounding Contour:** Compute the bounding contour $bc$ for feature $f$. [S1, unofficial]
2.  **Smooth Contour:** Apply Gaussian smoothing to $bc$ to reduce noise and ensure a smooth trailing edge. [S1, unofficial]
3.  **Identify Trailing Edge:**
    *   Initialize $d_{min} = \infty$ and $d_{max} = -\infty$.
    *   Initialize indices $i_{min} = 0$ and $i_{max} = 0$.
    *   For each point $p_i$ in $bc$:
        *   Calculate the distance metric $d_i$ using Equation 9:
            $$d_i = (p_i - c) \cdot \mathbf{v}_{avg}$$
            where $c$ is the feature centroid and $\mathbf{v}_{avg}$ is the average feature velocity.
        *   If $d_i < d_{min}$, set $d_{min} = d_i$ and $i_{min} = i$.
        *   If $d_i > d_{max}$, set $d_{max} = d_i$ and $i_{max} = i$.
    *   *Note:* The sources define the trailing portion as the segment between the points that minimize and maximize this dot product, effectively capturing the "rear" of the feature relative to motion [S1, unofficial].
4.  **Extract Trailing Segment:** Reduce $bc$ to the subset of points lying between indices $i_{min}$ and $i_{max}$ along the contour path. [S1, unofficial]
5.  **Distribute Seeds:** Distribute $N$ seed points evenly along the extracted trailing segment of $bc$. Let these seed points be $s_j$ for $j=1 \dots N$.
6.  **Calculate Local Velocity:** For each seed point $s_j$, calculate the local velocity vector $\mathbf{v}_{local}$ using Equation 10:
    $$\mathbf{v}_{local} = \mathbf{v}_{avg} + \boldsymbol{\omega} \times \mathbf{r}_j$$
    where $\mathbf{r}_j = s_j - c$ is the position vector of the seed point relative to the centroid, and $\times$ denotes the cross product.
7.  **Construct Cones:** For each seed point $s_j$:
    *   Determine cone length $L_j$:
        $$L_j = \|\mathbf{v}_{local}\| \cdot \Delta t$$
    *   Determine cone orientation: The central axis of the cone is oriented opposite to the estimated velocity vector ($-\mathbf{v}_{local}$) to indicate the direction from which the feature came (the "streak" trailing behind) [S1, unofficial].
    *   Generate a cone geometry with base radius $r_{base}$, length $L_j$, and $n_{facets}$ sides, centered at $s_j$ and aligned with $-\mathbf{v}_{local}$.
8.  **Render:** Add the generated cones to the output mesh geometry and apply the user-specified color/intensity. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_speedlines(feature_mesh, v_avg, omega, N, r_base, delta_t, n_facets, color):
    """
    Generates speedlines for a moving feature.
    
    Args:
        feature_mesh: Mesh or point cloud representing the feature.
        v_avg: 3D vector, average velocity of the feature.
        omega: 3D vector, angular velocity of the feature.
        N: int, number of speedlines to generate.
        r_base: float, base radius of the cones.
        delta_t: float, user-defined time-difference for length calculation.
        n_facets: int, number of facets for the cone geometry.
        color: tuple, RGB or RGBA color for the speedlines.
        
    Returns:
        list of Cone objects to be added to the scene graph.
    """
    # Step 1: Obtain bounding contour
    # Note: Algorithm 3 is referenced for contour extraction. 
    # Assuming a function get_bounding_contour exists that returns a list of 3D points.
    bc = get_bounding_contour(feature_mesh)
    
    # Step 2: Apply Gaussian smoothing
    bc = gaussian_smooth(bc)
    
    # Step 3: Identify trailing edge
    centroid = compute_centroid(feature_mesh)
    
    d_min = float('inf')
    d_max = float('-inf')
    i_min = 0
    i_max = 0
    
    for i, p_i in enumerate(bc):
        # Eqn 9: Distance function
        # Vector subtraction and dot product
        diff = vector_sub(p_i, centroid)
        d_i = dot_product(diff, v_avg)
        
        if d_i < d_min:
            d_min = d_i
            i_min = i
        if d_i > d_max:
            d_max = d_i
            i_max = i
            
    # Step 4: Reduce contour to trailing portion
    # Assuming bc is ordered cyclically or linearly. 
    # The sources imply a linear segment between i_min and i_max.
    # If the contour is a closed loop, we must determine the correct path.
    # Based on "Reduce bc to the points between imin and imax", we assume linear indexing.
    trailing_bc = bc[i_min : i_max + 1]
    
    # Step 5: Distribute points evenly
    # Interpolate N points along the trailing_bc path
    seed_points = distribute_points_evenly(trailing_bc, N)
    
    speedlines = []
    
    for s_j in seed_points:
        # Step 6: Calculate local velocity
        # Eqn 10: v_local = v_avg + omega x r
        r_j = vector_sub(s_j, centroid)
        cross_term = cross_product(omega, r_j)
        v_local = vector_add(v_avg, cross_term)
        
        # Step 7: Construct cone
        # Eqn for length: L = v_local * delta_t (interpreted as magnitude * dt)
        v_mag = vector_length(v_local)
        length = v_mag * delta_t
        
        # Orientation: opposite to velocity
        axis = vector_negate(v_local)
        if vector_length(axis) < 1e-9:
            axis = [0, 0, 1] # Fallback for zero velocity
            
        # Generate cone geometry
        # Note: VTK cone generator is referenced. We implement a simple cone here.
        cone = generate_cone(
            center=s_j,
            axis=axis,
            length=length,
            radius=r_base,
            facets=n_facets
        )
        
        cone.color = color
        speedlines.append(cone)
        
    return speedlines

def generate_cone(center, axis, length, radius, facets):
    """
    Generates a simple cone mesh.
    """
    # Normalize axis
    axis = normalize(axis)
    
    # Create vertices
    vertices = []
    indices = []
    
    # Apex
    apex = vector_add(center, vector_scale(axis, length))
    vertices.append(apex)
    
    # Base center
    base_center = center
    
    # Base vertices
    # Create a basis perpendicular to axis
    if abs(axis[0]) < 0.9:
        perp1 = normalize(cross_product(axis, [1, 0, 0]))
    else:
        perp1 = normalize(cross_product(axis, [0, 1, 0]))
    perp2 = cross_product(axis, perp1)
    
    for i in range(facets):
        angle = 2 * 3.14159265 * i / facets
        offset = vector_add(
            vector_scale(perp1, radius * cos(angle)),
            vector_scale(perp2, radius * sin(angle))
        )
        base_vertex = vector_add(base_center, offset)
        vertices.append(base_vertex)
        
    # Indices: Apex connects to all base vertices
    for i in range(1, facets + 1):
        indices.append((0, i, (i % facets) + 1))
        
    return Mesh(vertices=vertices, indices=indices)
```

**Parameters:**
*   $\Delta t$: User-defined time-difference. Controls the length of the streaks. [S1, unofficial]
*   $N$: Number of speedlines. [S1, unofficial]
*   $r_{base}$: Cone base radius. [S1, unofficial]
*   $n_{facets}$: Number of facets for the cone geometry. [S1, unofficial]
*   Color/Intensity: Visual styling parameters. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Zero Velocity:** If $\mathbf{v}_{avg}$ and $\boldsymbol{\omega}$ are zero, $\mathbf{v}_{local}$ is zero, resulting in zero-length cones. A fallback axis is required [G].
*   **Contour Ordering:** The algorithm assumes the bounding contour points are ordered such that the segment between $i_{min}$ and $i_{max}$ represents the trailing edge. If the contour is a closed loop, the "between" logic must handle wrap-around or select the shorter/longer arc appropriately. The sources do not specify how to handle closed loops explicitly, only "Reduce bc to the points between imin and imax" [S1, unofficial].
*   **Angular Velocity:** If $\boldsymbol{\omega}$ is not available, it is assumed to be zero. [G]

## Subtopics

### Adaptive Stream Line Seeding
Adjusts seed point density based on flow complexity. Choose when uniform distribution is insufficient. [Link](5604-adaptive-stream-line-seeding.md)

### Directional Textured Tracelines
Uses texture mapping on lines to show direction. Choose for high-density flow visualization. [Link](6965-directional-textured-tracelines.md)

### Stream Surfaces
Generates surfaces from stream lines. Choose for volumetric flow visualization. [Link](7696-stream-surfaces.md)

### Higher Order Tensor (HOT) Lines
Visualizes tensor fields. Choose for diffusion or stress data. [Link](14000-higher-order-tensor-hot-lines.md)

## Operating characteristics
*   **Time Complexity:** $O(M + N)$, where $M$ is the number of points in the bounding contour and $N$ is the number of speedlines. Contour extraction and smoothing dominate $M$, while cone generation is linear in $N$.
*   **Memory Complexity:** $O(M + N \cdot F)$, where $F$ is the number of vertices per cone (proportional to $n_{facets}$).
*   **Strengths:** Intuitively conveys motion direction and history for discrete moving features. Simple to implement.
*   **Limitations:** Only applicable to discrete features with defined centroids and motion vectors. Not suitable for general vector fields without feature segmentation. Performance degrades with very high $N$ or $n_{facets}$.
*   **When to use:** When visualizing the trajectory of specific, identifiable objects (e.g., particles, blobs) in time-varying data.
*   **When not to use:** For continuous vector fields without distinct features, or when precise trajectory integration is required.

## Background
Speedlines were introduced by Joshi and Rheingans as part of a set of illustrative visualization techniques for time-varying datasets [S1, unofficial]. The technique is motivated by the human perception of motion streaks, where moving objects leave a visual trace. It is often used in conjunction with other techniques like opacity modulation and strobe silhouettes to provide a comprehensive view of feature history [S1, unofficial].

## Sources
[S1] Generalized Framework and Algorithms for Illustrative Visualization of Time-Varying Data on Unstructured Meshes, https://digital.library.unt.edu/ark:/67531/metadc838861/?q=illustrative volume rendering, fetched 2026-10-28 (unofficial)
