# Shading Replacement for Relighting

Also called: Normal-Based Shading Transfer, Shading Replacement

## Overview
Shading Replacement for Relighting is a technique for object insertion that avoids inferring scene illumination by replacing the inserted object's shading with shading from existing scene points [S1]. It solves the problem of lighting inconsistency when compositing objects into neural radiance field (NeRF) scenes, where naive composition leads to unrealistic results due to varying illumination [S1]. It sets itself apart from direct relighting methods by not requiring explicit light source estimation, instead relying on the correlation between surface normals and shading in Lambertian scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Object point cloud or grid points $O$, each with a position $\mathbf{p}_o$, normal vector $\mathbf{n}_o$, and initial shading value $S_o$ (from the object's original lighting).
    *   Scene point cloud or grid points $V$, each with a position $\mathbf{p}_v$, normal vector $\mathbf{n}_v$, and shading value $S_v$ (from the scene NeRF).
    *   Average brightness of scene reflectance $\beta_{avg,r}$ and object reflectance $\beta_{o,avg,r}$.
    *   Confidence level $p$ (number of top-ranked pairs to select).
*   **Outputs:**
    *   Updated shading values $S_{updated}$ for each object point.

**Data Structures:**
*   Scene points $V$: List of tuples $(\mathbf{p}_v, \mathbf{n}_v, S_v)$.
*   Object points $O$: List of tuples $(\mathbf{p}_o, \mathbf{n}_o, S_o)$.
*   Pair Score: A scalar value combining normal similarity and distance.

**Steps:**
1.  **Pair Formation:** For each object point $o \in O$, find all scene points $v \in V$ that share similar normal vectors. This forms a set of candidate pairs $(o, v)$.
2.  **Scoring:** Rank the candidate pairs for each object point based on a score that considers normal similarity and distance. Solely relying on normal similarity results in global shading (too broad), while solely relying on distance fails to align with physical interpretations. Combining both allows querying shading within a localized range [S1].
3.  **Selection:** Select the top $p$ pairs for each object point based on the ranking. The parameter $p$ is a confidence level specifying how many neighbors to consider [S1].
4.  **Mean Shading Calculation:** Compute the mean shading value $S_{mean}$ of the selected $p$ scene points. This is preferred over maximum shading (which is not plausible) or normal-similarity-only shading (which is noisy) [S1].
5.  **Reflectance Compensation:** Adjust the mean shading to account for differences in the average brightness of the object and scene reflectance.
6.  **Update:** Assign the final updated shading $S_{updated}$ to the object point.

**Formulas:**
The updated shading $S_{updated}$ is calculated as:
$$ S_{updated} = S_{mean} \frac{\beta_{avg,r}}{\beta_{o,avg,r}} $$
where:
*   $S_{mean}$ is the average shading of the top $p$ selected scene points.
*   $\beta_{avg,r}$ is the average brightness of the scene reflectance.
*   $\beta_{o,avg,r}$ is the average brightness of the object reflectance [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shading_replacement(object_points, scene_points, p, beta_avg_r, beta_o_avg_r):
    """
    object_points: List of dicts with keys 'pos', 'normal', 'shading'
    scene_points: List of dicts with keys 'pos', 'normal', 'shading'
    p: int, number of top-ranked pairs to select
    beta_avg_r: float, average brightness of scene reflectance
    beta_o_avg_r: float, average brightness of object reflectance
    Returns: List of updated shading values for object points
    """
    updated_shadings = []
    
    for obj in object_points:
        candidates = []
        for scene in scene_points:
            # Calculate normal similarity (e.g., dot product)
            # Note: Exact similarity metric not specified, assume dot product of normalized normals
            normal_sim = dot(normalize(obj['normal']), normalize(scene['normal']))
            
            # Calculate distance
            dist = distance(obj['pos'], scene['pos'])
            
            # Score calculation: product of normal similarity and distance score
            # Note: Distance score usually implies inverse distance or similar to favor closer points
            # The source mentions "distance score" and "product". 
            # A common interpretation for "distance score" in this context is 1/(dist + epsilon)
            # However, the source does not specify the exact formula for the score.
            # We will use a placeholder comment for the exact scoring function as it is not explicitly defined.
            # score = normal_sim * (1 / (dist + 1e-8)) 
            # [G] Assuming a standard inverse distance weighting for the distance component.
            score = normal_sim * (1.0 / (dist + 1e-8))
            
            candidates.append({
                'scene': scene,
                'score': score
            })
        
        # Rank pairs according to scores (descending)
        candidates.sort(key=lambda x: x['score'], reverse=True)
        
        # Select top p pairs
        top_p_candidates = candidates[:p]
        
        # Compute mean shading of selected points
        if len(top_p_candidates) > 0:
            s_mean = sum(c['scene']['shading'] for c in top_p_candidates) / len(top_p_candidates)
        else:
            s_mean = obj['shading'] # Fallback if no candidates
            
        # Apply reflectance compensation
        s_updated = s_mean * (beta_avg_r / beta_o_avg_r)
        
        updated_shadings.append(s_updated)
        
    return updated_shadings
```

**Parameters:**
*   $p$: Confidence level, number of top-ranked pairs. Typical value not given in sources.
*   $\beta_{avg,r}$: Average brightness of scene reflectance.
*   $\beta_{o,avg,r}$: Average brightness of object reflectance.

**Edge Cases and Pitfalls:**
*   **Normal Similarity Only:** Results in global shading, which is too broad and noisy [S1].
*   **Distance Only:** Fails to align with physical interpretations of lighting [S1].
*   **Maximum Shading:** Generates implausible shading results [S1].
*   **Naive Composition:** Leads to unrealistic results due to varying illumination if lighting consistency is not considered [S1].
*   **Missing Areas:** Direct composition can lead to missing areas of the object in rendered images [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given. Depends on the number of object points, scene points, and the efficiency of finding similar normals and distances.
*   **Memory Complexity:** Not explicitly given. Requires storing scene points and their shading values.
*   **Strengths:** Avoids the need to infer scene illumination; handles lighting variations between object and scene; produces plausible shading by averaging; works well for Lambertian scenes.
*   **Limitations:** Relies on the assumption that points with similar normals have similar shading (Lambertian assumption); requires scene points to be available; performance depends on the density of scene points.
*   **When to use:** When inserting objects into NeRF scenes and lighting consistency is crucial, but explicit light estimation is difficult or unavailable.
*   **When not to use:** For non-Lambertian scenes where normal-shading correlation is weak; when high-frequency lighting details are critical and scene point density is low.

## Background
Originated in the paper "Relighting Scenes With Object Insertions in Neural Radiance Fields" [S1]. The method addresses the challenge of integrating objects into neural radiance field scenes with consistent lighting. It builds on the observation that for Lambertian scenes, surface normals are correlated with shading. The method is part of a larger pipeline that includes hybrid parametric lighting models (Spherical Harmonics and Spherical Gaussians) for scene representation [S1].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
