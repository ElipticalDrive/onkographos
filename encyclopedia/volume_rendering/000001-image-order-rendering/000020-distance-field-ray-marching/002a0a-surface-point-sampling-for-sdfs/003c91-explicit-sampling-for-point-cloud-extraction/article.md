# Explicit Sampling for Point Cloud Extraction

Also called: PC extraction from DDF, SI3DR evaluation sampling

## Overview
Explicit Sampling is a method to extract a point cloud (PC) from a Probabilistic Directed Distance Field (PDDF) or Directed Distance Field (DDF) by projecting uniformly sampled points from a bounding box onto the shape surface $S$ [S1, S2]. It solves the problem of converting implicit field representations into explicit geometric primitives for evaluation metrics like Chamfer distance [S1, S2]. Unlike its siblings, it relies on a simple "shoot-and-cast" strategy using weighted direction composition to find valid surface intersections, without guaranteeing uniform sampling density over the shape [S1, S2].

## How it works
**Inputs:**
*   A PDDF or DDF representation providing depth $d(p, v)$ and visibility $\xi(p, v)$ for any point $p$ and direction $v$ [S1, S2].
*   A bounding box $B$ enclosing the shape [S1, S2].
*   Parameters: $n_v$ (number of directions per point, typically 128) and $N_H$ (number of refinement iterations, typically 3) [S1, S2].

**Outputs:**
*   A set of 3D points representing the surface $S$ [S1, S2].

**Data Structures:**
*   Point $p \in \mathbb{R}^3$: Current sample position.
*   Direction $v \in \mathbb{R}^3$: Unit vector on the sphere $S^2$.
*   Set $V(p) = \{v_i(p)\}_{i=1}^{n_v}$: Collection of sampled directions.
*   Weighted direction $v^*_b(p) \in \mathbb{R}^3$: The composed best direction.

**Steps:**
1.  **Initial Sampling:** Sample a point $p$ uniformly from the bounding box $B$: $p \sim U[B]$ [S1, S2].
2.  **Direction Sampling:** For the current point $p$, sample $n_v$ directions uniformly from the unit sphere $S^2$:
    $$ V(p) = \{v_i(p) \mid v_i(p) \sim U[S^2], i=1 \dots n_v\} $$
    [S1, S2].
3.  **Direction Composition:** Estimate the best direction $v^*_b(p)$ that avoids occlusion ($\xi(p, v) = 0$) by computing a weighted average over the sampled directions $V(p)$ [S1, S2]. The specific weighting function is not detailed in the provided notes, but it utilizes the field's visibility/depth information to favor visible directions [S2]. Note: Arbitrary directions may not be visible ($\xi(p, v) \neq 1$), and at certain positions, multiple valid values of $v^*$ may exist [S2].
4.  **Surface Projection:** Compute the projected point $q$ on the shape using the estimated direction and the field's depth:
    $$ q(p, v^*_b(p)) = p + d(p, v^*_b(p)) \cdot v^*_b(p) $$
    The condition for $q$ being on $S$ is $\xi(p, v^*_b(p)) = 1$ [S1, S2].
5.  **Refinement (Optional):** If depths are less accurate far from the shape, repeat the process $N_H$ times, updating the starting point $p \leftarrow q$ for each iteration [S1, S2].
6.  **Collection:** Add $q$ to the point cloud. Repeat steps 1-5 for the desired number of points.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_point_cloud(pddf, bounding_box, n_v=128, n_h=3):
    """
    Extracts a point cloud from a PDDF/DDF using explicit sampling.
    
    Args:
        pddf: Object providing query(p, v) -> (depth, visibility)
        bounding_box: Bounding box for uniform sampling
        n_v: Number of directions to sample per point
        n_h: Number of refinement iterations
        
    Returns:
        list of points on the surface
    """
    points = []
    
    # Number of initial samples needed to generate the desired cloud size
    # This loop generates one point per iteration, potentially refined n_h times
    # In practice, one might sample N points and refine each, or sample N/n_h points
    # and refine each n_h times. The text implies repeating the process NH times
    # starting from the newly found point, suggesting a chain for each initial sample.
    # We assume we want to generate a final set of points. 
    # The text says "Repeat this NH times... to improve accuracy".
    # It does not specify if the output is the final q or all intermediate q's.
    # Standard interpretation: One initial p -> chain of NH projections -> 1 final point.
    
    num_initial_samples = 10000 # Example target size
    
    for _ in range(num_initial_samples):
        # Step 1: Sample initial point
        p = sample_uniform(bounding_box)
        
        current_p = p
        for h in range(n_h):
            # Step 2: Sample directions
            directions = [sample_uniform_sphere() for _ in range(n_v)]
            
            # Step 3: Compose directions to find best v*_b
            # The text mentions "weighted average over V(p)" [S2]
            # and "compose... to estimate vb*(p)" [S1]
            # Implementation detail: Weight by visibility or inverse depth error?
            # Sources do not give the exact weight formula. 
            # [G] Assuming a simple visibility-weighted average for valid directions.
            # If visibility is probabilistic, weights might be P(visibility=1).
            
            weighted_sum = [0.0, 0.0, 0.0]
            weight_total = 0.0
            
            for v in directions:
                depth, visibility = pddf.query(current_p, v)
                
                # Visibility is often a probability or binary flag.
                # If binary: weight = 1 if visible, 0 otherwise.
                # If probabilistic: weight = visibility.
                # Note: [S2] says "cannot choose arbitrary v... many will not be visible"
                # and "weighted average over V(p)".
                
                # Guard: if depth is invalid or visibility is 0, ignore or weight 0
                if visibility <= 0:
                    continue
                
                # [G] Simple visibility weighting. 
                # More sophisticated methods might use depth confidence.
                w = visibility 
                weighted_sum[0] += w * v[0]
                weighted_sum[1] += w * v[1]
                weighted_sum[2] += w * v[2]
                weight_total += w
            
            if weight_total == 0:
                # No visible directions found. 
                # [G] Fallback: keep current point or skip. 
                # Skipping is safer to avoid noise.
                break 
            
            # Normalize to get unit vector v*_b
            v_star = normalize(weighted_sum)
            
            # Step 4: Project to surface
            depth, _ = pddf.query(current_p, v_star)
            
            # Check if projection was valid
            if depth is None or depth < 0:
                break
                
            q = current_p + depth * v_star
            
            # Step 5: Update for next iteration
            current_p = q
            
        # Add the final refined point
        points.append(current_p)
        
    return points

def sample_uniform(box):
    # Returns random point in box
    return [random.uniform(box.min[i], box.max[i]) for i in range(3)]

def sample_uniform_sphere():
    # Returns random unit vector
    # Using spherical coordinates or normal distribution method
    u = random.random()
    v = random.random()
    theta = 2 * math.pi * u
    phi = math.acos(2 * v - 1)
    x = math.sin(phi) * math.cos(theta)
    y = math.sin(phi) * math.sin(theta)
    z = math.cos(phi)
    return [x, y, z]

def normalize(v):
    mag = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if mag == 0: return [0,0,0]
    return [v[0]/mag, v[1]/mag, v[2]/mag]
```

**Parameters:**
*   $n_v$: Number of directions sampled per point. Typical value: 128 [S1, S2].
*   $N_H$: Number of refinement iterations. Typical value: 3 [S1, S2].

**Edge Cases and Pitfalls:**
*   **Non-uniformity:** The method does not guarantee uniform sampling over the shape $S$ [S1, S2].
*   **Visibility:** Arbitrary directions $v$ may not be visible ($\xi(p, v) \neq 1$), necessitating the composition technique [S2].
*   **Ambiguity:** At certain positions, there may be multiple valid values of $v^*$ [S2].
*   **Depth Accuracy:** Depths are less accurate far from the shape, which is why refinement ($N_H > 1$) is recommended [S1, S2].

## Subtopics

### PC extraction from DDFs
A variant of the method applied specifically to Directed Distance Fields (DDF) rather than Probabilistic DDFs [S2]. It follows the same steps: uniform sampling in the bounding box, direction sampling, weighted average composition to estimate $v^*_b(p)$, and projection using $q(p, v^*_b(p)) = p + d(p, v^*_b(p))v^*_b(p)$ [S2]. It shares the same pitfalls: no guarantee of uniform sampling, visibility issues requiring weighted averages, and potential ambiguity in $v^*$ [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot n_v \cdot N_H)$, where $N$ is the number of initial samples, $n_v$ is the number of directions per sample, and $N_H$ is the number of refinement steps. Each step requires a query to the DDF/PDDF.
*   **Memory Complexity:** $O(n_v)$ per point for storing direction samples, plus $O(N)$ for the output point cloud.
*   **Performance:** The method is simple and fast to implement but computationally expensive due to the large number of DDF queries ($n_v=128$ per point) [S1, S2].
*   **Strengths:** Simple to implement; works with generic DDF/PDDF representations; refinement improves accuracy for points far from the surface [S1, S2].
*   **Limitations:** Does not guarantee uniform sampling; sensitive to visibility estimation errors; multiple valid directions can cause ambiguity [S2].
*   **When to use:** When evaluating reconstruction quality using point cloud metrics (Chamfer distance, F-score) and a simple, modality-agnostic extraction is needed [S1, S2].
*   **When not to use:** When uniform surface sampling is required; when computational efficiency is critical and $n_v$ cannot be reduced.

## Background
The method was introduced in the context of evaluating 3D reconstruction methods using Probabilistic Directed Distance Fields (PDDFs) [S1, S2]. It is commonly used in the SI3DR (Shape Inference from 3D Reconstructions) evaluation framework [S1]. The approach is analogous to prior work [5] (cited in [S2]) for projecting points onto shapes defined by distance fields. It is used to generate point clouds for comparison against ground truth shapes using metrics like Chamfer distance and F-score [S1, S2].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
[S2] Representing 3D Shapes with Probabilistic Directed Distance Fields, https://doi.org/10.1109/cvpr52688.2022.01874, fetched 2026-10-26
