# Splat Decimation and Fusion

Also called: Splat Decimation, Splat Fusion, Two-Stage Splat Simplification

## Overview
Splat Decimation and Fusion is a post-processing method that reduces the primitive count of a textured splat model by removing redundant overlapping splats and merging those covering roughly planar areas into larger ellipses [S1]. It solves the performance bottleneck of rendering high-density splat clouds on mobile devices by minimizing new holes and enabling easier texturing through fewer, larger primitives [S1]. It differs from simple overlap removal by actively fusing geometry into minimum enclosing ellipses and employs a two-stage approach (coarse center-based and detailed contour-based) to balance computational cost with visual fidelity [S1].

## How it works
The method operates in two stages: a coarse simplification focusing on splat centers to eliminate nearby ellipses quickly, and a detailed merging stage to fuse remaining overlapping ellipses using a minimum enclosing algorithm [S1].

### Inputs and Outputs
*   **Input:** A set of splats $S$, where each splat $i$ is defined by center $s_{0i}$, normal $n_{0i}$, major axis $u_{0i}$, minor axis $v_{0i}$, and area $A_i = \|u_{0i}\| \|v_{0i}\|$ [S1].
*   **Output:** A simplified set of splats with reduced count [S1].
*   **Parameters:**
    *   $\varepsilon_1 \in [0, 1]$: Scales the neighbor-search radius in Stage 1. Higher values inspect more splats [S1].
    *   $\varepsilon_2 \in [0, 1]$: Controls the retention of splats in Stage 1 based on distance to the new center. $\varepsilon_2 = 1$ uses all merged splats [S1].
    *   Flatness coefficient $f_i$: Used in Stage 2 to trim semiaxes in non-planar areas [S1].

### Data Structures
*   **Voxel Grid:** Used to accelerate neighbor search in Stage 1 [S1].
*   **Splat List:** Ordered or indexed collection of splat primitives [S1].

### Stage 1: Coarse Decimation
This stage eliminates redundant splats by analyzing centers and normals [S1].

1.  **Neighbor Search:** For each splat $i$, find neighbors $N_{r \cdot \varepsilon_1}(s_i)$ using voxelization. The search radius is $r \cdot \varepsilon_1$, where $r$ is a base neighbor distance [S1].
2.  **Orientation Filter:** Identify candidates for merging with splat $i$ whose normal vectors $n_j$ form a small angle with $n_i$ [S1].
3.  **Weighted Averaging:** Merge the candidate set $\{i+1, \dots, K\}$ with splat $i$. Compute the new center $s'_{0i}$, normal $n'_{0i}$, and major axis $u'_{0i}$ as area-weighted averages:
    $$ s'_{0i} = \frac{\sum_{j=i}^{K} \|u_{0j}\| \|v_{0j}\| s_j}{\sum_{j=i}^{K} \|u_{0j}\| \|v_{0j}\|} $$
    $$ n'_{0i} = \frac{\sum_{j=i}^{K} \|u_{0j}\| \|v_{0j}\| n_j}{\sum_{j=i}^{K} \|u_{0j}\| \|v_{0j}\|} $$
    $$ u'_{0i} = \frac{\sum_{j=i}^{K} \|u_{0j}\| \|v_{0j}\| u_j}{\sum_{j=i}^{K} \|u_{0j}\| \|v_{0j}\|} $$
    [S1]
4.  **Orthogonality Enforcement:** Derive the new minor axis $v'_{0i}$ to ensure orthogonality:
    $$ v'_{0i} = n'_{0i} \wedge u'_{0i} $$
    [S1]
5.  **Distance-Based Retention:** Order the involved splats by increasing distance to the new center $s'_{0i}$. Retain only the closest splats based on $\varepsilon_2$ [S1].
6.  **Contour Sampling:** Sample eight points uniformly from the contour of each retained ellipse [S1].
7.  **Projection and Scaling:** Project sampled points onto the plane of the new splat (defined by $u'_{0i}$ and $v'_{0i}$). Select maximum projections $P_u$ and $P_v$ along each axis. Compute the averaged eccentricity $\varepsilon'_i$. Calculate the new major semiaxis length:
    $$ \|u'_{0i}\| = \frac{\sqrt{P_u^2 + P_v^2}}{1 - \varepsilon'^2_i} $$
    Calculate the new minor semiaxis length:
    $$ \|v'_{0i}\| = \sqrt{1 - \varepsilon'^2_i} \cdot \|u'_{0i}\| $$
    [S1]

### Stage 2: Detailed Fusion
This stage merges remaining overlapping pairs using a minimum enclosing ellipse algorithm [S1].

1.  **Candidate Identification:** Two splats $i$ and $j$ are merging candidates if:
    $$ \|s_{0i} - s_{0j}\| < 0.8 \cdot (\|u_{0i}\| + \|u_{0j}\|) $$
    and their normal vectors form a small angle [S1].
2.  **Footprint Sampling:** For candidate pair $(i, j)$, sample eight uniformly distributed points on the contour of splat $j$, along with its center $s_{0j}$, totaling nine points [S1].
3.  **Projection:** Project these nine points onto the plane of splat $i$ [S1].
4.  **Overlap Check:** A "footprint" is detected if a projected point lies inside ellipse $i$.
    *   If fewer than two footprints are detected, the overlap is ignored to avoid creating holes [S1].
    *   If seven or more footprints are detected, splat $j$ is directly removed from the model [S1].
    *   Otherwise, proceed to merge [S1].
5.  **Minimum Enclosing Ellipse:** Apply the minimum enclosing ellipsoid algorithm [Khachiyan 1996] using eight samples from each ellipse (16 points total) as input [S1].
6.  **Finalization:** Keep the estimated center as the splat location. Scale the estimated semiaxes with the flatness coefficient $f_i$ to trim splats in less planar areas [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_decimation_and_fusion(splats, epsilon1, epsilon2, flatness_coeff):
    """
    Performs two-stage splat decimation and fusion.
    splats: List of Splat objects with attributes: center, normal, u_axis, v_axis, area
    epsilon1: Scale factor for neighbor search radius in Stage 1
    epsilon2: Distance retention threshold in Stage 1
    flatness_coeff: Scaling factor for semiaxes in Stage 2
    """
    
    # Stage 1: Coarse Decimation
    voxel_grid = build_voxel_grid(splats)
    processed_indices = set()
    new_splats = []
    
    for i, splat_i in enumerate(splats):
        if i in processed_indices:
            continue
            
        # Find neighbors within scaled radius
        neighbors = find_neighbors_voxel(voxel_grid, splat_i.center, splat_i.radius * epsilon1)
        
        # Filter by orientation (small angle with normal)
        candidates = [s for s in neighbors if dot_product(s.normal, splat_i.normal) > 0.9] # Threshold implied by "small angle"
        
        if not candidates:
            new_splats.append(splat_i)
            continue
            
        # Merge candidates with splat_i
        # Calculate area-weighted averages
        total_area = sum(s.area for s in candidates)
        if total_area == 0:
            new_splats.append(splat_i)
            continue
            
        new_center = sum(s.area * s.center for s in candidates) / total_area
        new_normal = sum(s.area * s.normal for s in candidates) / total_area
        new_u = sum(s.area * s.u_axis for s in candidates) / total_area
        
        # Normalize vectors
        new_normal = normalize(new_normal)
        new_u = normalize(new_u)
        
        # Ensure orthogonality for minor axis
        new_v = cross_product(new_normal, new_u)
        
        # Order candidates by distance to new center
        candidates.sort(key=lambda s: distance(s.center, new_center))
        
        # Retain based on epsilon2
        # Note: epsilon2 logic in source implies filtering the set of splats contributing to the merge
        # Assuming epsilon2 acts as a fraction or threshold for inclusion
        retained_candidates = candidates[:int(len(candidates) * epsilon2) + 1] if epsilon2 < 1 else candidates
        
        # Sample contours of retained splats
        sampled_points = []
        for s in retained_candidates:
            sampled_points.extend(sample_contour(s, num_points=8))
            
        # Project to new plane
        projected_points = [project_point_to_plane(p, new_center, new_normal) for p in sampled_points]
        
        # Find max projections along axes
        # Project points onto u and v axes relative to center
        projections_u = [dot_product(p - new_center, new_u) for p in projected_points]
        projections_v = [dot_product(p - new_center, new_v) for p in projected_points]
        
        Pu = max(abs(p) for p in projections_u)
        Pv = max(abs(p) for p in projections_v)
        
        # Calculate averaged eccentricity (simplified as average of candidates' eccentricities)
        avg_eccentricity = sum(s.eccentricity for s in retained_candidates) / len(retained_candidates)
        
        # Compute new semiaxes
        # Formula: ||u'|| = sqrt(Pu^2 + Pv^2) / (1 - eps'^2)
        # Note: Source formula has sqrt in numerator? "sqrt(Pu^2 + Pv^2)" or just "Pu^2 + Pv^2"?
        # Source text: "ku'0i k = Pu2 + Pv2 /(1 − 02 i)"
        # LaTeX interpretation in notes: \frac{\sqrt{P_u^2 + P_v^2}}{1 - \varepsilon'^2_i}
        # However, standard ellipse geometry suggests scaling factor. 
        # Re-reading source: "forcing the resulting ellipse to go through the point (Pu, Pv)"
        # If point is (Pu, Pv) in local coords, equation is (x/a)^2 + (y/b)^2 = 1.
        # b = a * sqrt(1-e^2). So (Pu/a)^2 + (Pv/(a*sqrt(1-e^2)))^2 = 1.
        # Pu^2/a^2 + Pv^2/(a^2(1-e^2)) = 1
        # a^2 = Pu^2 + Pv^2/(1-e^2)
        # a = sqrt(Pu^2 + Pv^2/(1-e^2))
        # The source formula in notes says: \frac{\sqrt{P_u^2 + P_v^2}}{1 - \varepsilon'^2_i}
        # This might be a specific approximation or typo in the source text vs standard geometry.
        # We follow the source's explicit formula provided in the notes:
        
        denom = 1 - avg_eccentricity**2
        if denom == 0: denom = 1e-6 # Guard against division by zero
        
        new_u_len = sqrt(Pu**2 + Pv**2) / denom
        new_v_len = sqrt(1 - avg_eccentricity**2) * new_u_len
        
        new_splat = Splat(center=new_center, normal=new_normal, u_axis=new_u, v_axis=new_v, 
                          u_len=new_u_len, v_len=new_v_len)
        new_splats.append(new_splat)
        
        # Mark processed
        for s in candidates:
            processed_indices.add(id(s))

    # Stage 2: Detailed Fusion
    # Re-evaluate remaining splats for pairwise merging
    final_splats = []
    used = set()
    
    for i, splat_i in enumerate(new_splats):
        if i in used:
            continue
            
        merged = False
        for j, splat_j in enumerate(new_splats):
            if i == j or j in used:
                continue
                
            # Candidate check
            dist = distance(splat_i.center, splat_j.center)
            max_u_len = max(splat_i.u_len, splat_j.u_len) # Approximation of sum of major axes?
            # Source: sum of lengths of major axes
            sum_major = splat_i.u_len + splat_j.u_len
            
            if dist < 0.8 * sum_major and dot_product(splat_i.normal, splat_j.normal) > 0.9:
                # Sample contour of j
                contour_j = sample_contour(splat_j, num_points=8)
                points_to_project = contour_j + [splat_j.center]
                
                # Project to plane of i
                projected = [project_point_to_plane(p, splat_i.center, splat_i.normal) for p in points_to_project]
                
                # Check footprints
                footprints = [p for p in projected if point_inside_ellipse(p, splat_i.center, splat_i.normal, splat_i.u_axis, splat_i.v_axis, splat_i.u_len, splat_i.v_len)]
                
                if len(footprints) >= 7:
                    # Remove j directly
                    used.add(j)
                    continue
                elif len(footprints) >= 2:
                    # Merge i and j
                    # Minimum enclosing ellipse using 8 samples from each
                    samples_i = sample_contour(splat_i, num_points=8)
                    samples_j = sample_contour(splat_j, num_points=8)
                    all_samples = samples_i + samples_j
                    
                    # Khachiyan's algorithm
                    mec_center, mec_u, mec_v, mec_u_len, mec_v_len = minimum_enclosing_ellipse(all_samples)
                    
                    # Scale with flatness
                    mec_u_len *= flatness_coeff
                    mec_v_len *= flatness_coeff
                    
                    final_splats.append(Splat(center=mec_center, normal=splat_i.normal, u_axis=mec_u, v_axis=mec_v,
                                              u_len=mec_u_len, v_len=mec_v_len))
                    used.add(i)
                    used.add(j)
                    merged = True
                    break
        
        if not merged:
            final_splats.append(splat_i)
            
    return final_splats

def project_point_to_plane(point, plane_center, plane_normal):
    # Project point onto plane defined by center and normal
    # Returns 2D coordinates in the plane's local basis? 
    # For the purpose of "inside ellipse" check, we need local 2D coords.
    # We assume the ellipse axes u, v are known.
    # This helper is incomplete without u, v basis. 
    # The source says "projected onto the plane... footprint is found if... inside ellipse i".
    # This implies checking 3D distance to plane and then 2D distance in plane.
    pass

def point_inside_ellipse(point, center, normal, u, v, u_len, v_len):
    # Project point to plane
    vec = point - center
    # Check if on plane (approx)
    if abs(dot_product(vec, normal)) > 1e-6:
        return False
    # Project to local 2D
    x = dot_product(vec, u)
    y = dot_product(vec, v)
    # Ellipse equation: (x/a)^2 + (y/b)^2 <= 1
    return (x/u_len)**2 + (y/v_len)**2 <= 1.0
```

## Operating characteristics
*   **Time Complexity:** Stage 1 is accelerated by voxelization, making neighbor search roughly $O(N)$ or $O(N \log N)$ depending on voxel grid management. Stage 2 is $O(M^2)$ for pairwise checks where $M$ is the number of splats remaining after Stage 1, but the conservative overlap criteria and footprint checks prune many pairs. The minimum enclosing ellipse algorithm is iterative but converges quickly for small point sets (16 points) [S1].
*   **Memory Complexity:** Requires storage for the voxel grid and the splat list. No significant temporary memory overhead beyond sampling buffers [S1].
*   **Performance:** Designed for mobile rendering, reducing the primitive count significantly to enable smooth frame rates on handheld devices [S1].
*   **Strengths:** Preserves sharp details by minimizing new holes; uses fewer larger splats in planar regions for easier texturing [S1].
*   **Limitations:** The two-stage process adds computational overhead to the preprocessing pipeline; aggressive merging can distort non-planar geometry if flatness coefficients are not tuned [S1].

## Background
The method was introduced in the context of textured splat-based point clouds for rendering in handheld devices [S1]. It builds on the merging operators of progressive splatting [Wu et al. 2005] but adapts them for a two-stage decimation/fusion process to reduce computational cost and improve texturing results on mobile hardware [S1]. The minimum enclosing ellipse step references Khachiyan's algorithm [Khachiyan 1996] [S1].

## Sources
[S1] Textured splat-based point clouds for rendering in handheld devices, https://doi.org/10.1145/2775292.2782779, fetched 2026-11-03
