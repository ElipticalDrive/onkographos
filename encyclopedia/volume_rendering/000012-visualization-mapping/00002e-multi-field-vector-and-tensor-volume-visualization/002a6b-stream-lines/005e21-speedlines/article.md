# Speedlines

Also called: Motion Streaks, Speedlines Effect, Streak Lines

## Overview
Speedlines are an illustrative visualization technique used to indicate the motion and history of specific features within time-varying datasets [S1, unofficial]. Unlike standard stream lines that trace particle trajectories through a vector field, speedlines superimpose geometric primitives (cones) along the receding bounding contour of a moving feature to provide intuitive visual cues of direction and velocity history [S1, unofficial].

## How it works
**Inputs:**
*   Feature geometry $f$ [S1, unofficial].
*   Feature motion data: average velocity vector $\mathbf{v}_{avg}$ and angular velocity vector $\boldsymbol{\omega}$ [S1, unofficial].
*   User parameters: number of speedlines $N$, cone base radius $r_{base}$, time-difference $\Delta t$, number of cone facets $n_{facets}$, and color/intensity [S1, unofficial].

**Outputs:**
*   A mesh geometry containing cones representing the speedlines, colored according to user specifications [S1, unofficial].

**Data Structures:**
*   **Bounding Contour ($bc$):** A list of 3D points representing the smoothed boundary of the feature [S1, unofficial].
*   **Centroid ($c$):** The 3D center of the feature [S1, unofficial].
*   **Indices ($i_{min}, i_{max}$):** Integer indices into the contour list identifying the start and end of the trailing portion [S1, unofficial].

**Algorithm Steps:**

1.  **Obtain Bounding Contour:** Compute the bounding contour $bc$ for feature $f$ [S1, unofficial].
2.  **Smooth Contour:** Apply Gaussian smoothing to $bc$ [S1, unofficial].
3.  **Identify Trailing Edge:** Initialize $d_{min} = \infty$ and $d_{max} = -\infty$. Initialize $i_{min} = 0$ and $i_{max} = 0$. For each point $p_i$ in $bc$, calculate the distance metric $d_i = (p_i - c) \cdot \mathbf{v}_{avg}$. If $d_i < d_{min}$, set $d_{min} = d_i$ and $i_{min} = i$. If $d_i > d_{max}$, set $d_{max} = d_i$ and $i_{max} = i$ [S1, unofficial].
4.  **Extract Trailing Segment:** Reduce $bc$ to the subset of points lying between indices $i_{min}$ and $i_{max}$ along the contour path [S1, unofficial].
5.  **Distribute Seeds:** Distribute $N$ seed points evenly along the extracted trailing segment of $bc$ [S1, unofficial].
6.  **Calculate Local Velocity:** For each seed point $s_j$, calculate the local velocity vector $\mathbf{v}_{local} = \mathbf{v}_{avg} + \boldsymbol{\omega} \times (s_j - c)$ [S1, unofficial].
7.  **Construct Cones:** Determine cone length $L_j = \|\mathbf{v}_{local}\| \cdot \Delta t$. Orient the central axis of the cone opposite to the estimated velocity vector ($-\mathbf{v}_{local}$) [S1, unofficial]. Generate a cone geometry with base radius $r_{base}$, length $L_j$, and $n_{facets}$ sides, centered at $s_j$ and aligned with $-\mathbf{v}_{local}$ [S1, unofficial].
8.  **Render:** Add the generated cones to the output mesh geometry and apply the user-specified color/intensity [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vector_add(a, b): return [a[0]+b[0], a[1]+b[1], a[2]+b[2]]
def vector_sub(a, b): return [a[0]-b[0], a[1]-b[1], a[2]-b[2]]
def dot_product(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def cross_product(a, b): return [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]
def vector_length(v): return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
def normalize(v):
    l = vector_length(v)
    if l < 1e-9: return [0, 0, 1]
    return [v[0]/l, v[1]/l, v[2]/l]

def generate_speedlines(feature_mesh, v_avg, omega, N, r_base, delta_t, n_facets, color):
    # Step 1 & 2: Obtain and smooth contour (Algorithm 3 and Gaussian smoothing referenced)
    # Placeholder for external contour extraction/smoothing as sources do not provide implementation details.
    bc = get_bounding_contour_and_smooth(feature_mesh)
    
    # Step 3: Identify trailing edge
    centroid = compute_centroid(feature_mesh)
    d_min = float('inf')
    d_max = float('-inf')
    i_min = 0
    i_max = 0
    
    for i, p_i in enumerate(bc):
        diff = vector_sub(p_i, centroid)
        d_i = dot_product(diff, v_avg)
        if d_i < d_min:
            d_min = d_i
            i_min = i
        if d_i > d_max:
            d_max = d_i
            i_max = i
            
    # Step 4: Extract trailing segment
    # Sources state "Reduce bc to the points between imin and imax".
    # Assumes linear ordering. If closed loop, wrap-around logic is required [S1, unofficial].
    trailing_bc = bc[i_min : i_max + 1]
    
    # Step 5: Distribute seeds evenly
    seed_points = distribute_points_evenly(trailing_bc, N)
    
    speedlines = []
    for s_j in seed_points:
        # Step 6: Calculate local velocity
        r_j = vector_sub(s_j, centroid)
        cross_term = cross_product(omega, r_j)
        v_local = vector_add(v_avg, cross_term)
        
        # Step 7: Construct cone
        v_mag = vector_length(v_local)
        length = v_mag * delta_t
        axis = normalize(vector_sub([0,0,0], v_local))
        
        cone = generate_cone(center=s_j, axis=axis, length=length, radius=r_base, facets=n_facets)
        cone.color = color
        speedlines.append(cone)
        
    return speedlines

def distribute_points_evenly(polyline, N):
    # Linear interpolation along polyline segments
    if len(polyline) < 2: return polyline
    total_len = sum(vector_length(vector_sub(polyline[i+1], polyline[i])) for i in range(len(polyline)-1))
    step = total_len / N
    points = []
    current_dist = 0
    seg_idx = 0
    for i in range(N):
        target_dist = step * (i + 1)
        while seg_idx < len(polyline) - 1 and current_dist + vector_length(vector_sub(polyline[seg_idx+1], polyline[seg_idx])) < target_dist:
            current_dist += vector_length(vector_sub(polyline[seg_idx+1], polyline[seg_idx]))
            seg_idx += 1
        seg_len = vector_length(vector_sub(polyline[seg_idx+1], polyline[seg_idx]))
        if seg_len > 1e-9:
            t = (target_dist - current_dist) / seg_len
            points.append(vector_add(polyline[seg_idx], vector_sub(polyline[seg_idx+1], polyline[seg_idx]) * t))
        else:
            points.append(polyline[seg_idx])
    return points

def generate_cone(center, axis, length, radius, facets):
    axis = normalize(axis)
    vertices = [center] # Apex
    indices = []
    base_center = vector_sub(center, vector_scale(axis, length))
    
    if abs(axis[0]) < 0.9:
        perp1 = normalize(cross_product(axis, [1, 0, 0]))
    else:
        perp1 = normalize(cross_product(axis, [0, 1, 0]))
    perp2 = cross_product(axis, perp1)
    
    for i in range(facets):
        angle = 2 * math.pi * i / facets
        offset = vector_add(vector_scale(perp1, radius * math.cos(angle)), vector_scale(perp2, radius * math.sin(angle)))
        vertices.append(vector_add(base_center, offset))
        
    for i in range(1, facets + 1):
        indices.append((0, i, (i % facets) + 1))
        
    return {"vertices": vertices, "indices": indices, "color": None}

def vector_scale(v, s): return [v[0]*s, v[1]*s, v[2]*s]
def get_bounding_contour_and_smooth(mesh): return [] # Not given in the sources.
def compute_centroid(mesh): return [0, 0, 0] # Not given in the sources.
```

**Parameters:**
*   $\Delta t$: User-defined time-difference controlling streak length [S1, unofficial].
*   $N$: Number of speedlines [S1, unofficial].
*   $r_{base}$: Cone base radius [S1, unofficial].
*   $n_{facets}$: Number of cone facets [S1, unofficial].
*   Color: User-specified color/intensity for the speedlines [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Velocity:** If $\mathbf{v}_{avg}$ and $\boldsymbol{\omega}$ are zero, $\mathbf{v}_{local}$ is zero, resulting in zero-length cones [G].
*   **Contour Ordering:** The algorithm assumes the bounding contour points are ordered such that the segment between $i_{min}$ and $i_{max}$ represents the trailing edge. If the contour is a closed loop, the "between" logic must handle wrap-around or select the appropriate arc [S1, unofficial].
*   **Angular Velocity:** If $\boldsymbol{\omega}$ is not available, it is assumed to be zero [G].

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
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Intuitively conveys motion direction and history for discrete moving features [S1, unofficial].
*   **Limitations:** Only applicable to discrete features with defined centroids and motion vectors [S1, unofficial].
*   **When to use:** Visualizing the trajectory of specific, identifiable objects in time-varying data [S1, unofficial].
*   **When not to use:** Continuous vector fields without distinct features [G].

## Background
Speedlines were introduced by Joshi and Rheingans as part of a set of illustrative visualization techniques for time-varying datasets [S1, unofficial]. The technique is motivated by the human perception of motion streaks, where moving objects leave a visual trace [S1, unofficial].

## Sources
[S1] Generalized Framework and Algorithms for Illustrative Visualization of Time-Varying Data on Unstructured Meshes, https://digital.library.unt.edu/ark:/67531/metadc838861/?q=illustrative volume rendering, fetched 2026-10-28 (unofficial)
