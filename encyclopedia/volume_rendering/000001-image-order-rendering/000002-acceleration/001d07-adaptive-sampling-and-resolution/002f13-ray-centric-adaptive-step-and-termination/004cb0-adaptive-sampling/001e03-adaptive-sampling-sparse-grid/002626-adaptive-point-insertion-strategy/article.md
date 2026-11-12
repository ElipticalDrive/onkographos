# Adaptive Point Insertion Strategy
Also called: Adaptive point sampling strategy, Quality-driven insertion approach
## Overview
The Adaptive Point Insertion Strategy is a method for optimizing the distribution of sampling points in signed distance field (SDF) reconstruction by prioritizing topologically critical regions. It solves the problem of insufficient geometric fidelity in complex areas (e.g., high curvature, thin structures) where uniform sampling fails, ensuring superior topological preservation with a constrained sampling budget. It sets itself apart from uniform upsampling by using quality-driven criteria to insert points, thereby preserving local connectivity and aligning new sites with the SDF gradient [S1].

## How it works
**Inputs and Outputs:**
- Input: An initial uniform grid of sampling points (typically $8^3$) and a target total point count (e.g., $50^3$).
- Output: A refined set of sampling points and their corresponding surface projections, optimized for topological fidelity.

**Data Structures:**
- The strategy maintains a set of sampling points $P$ and a set of surface projections $Q$.
- Points are managed in a structure that allows for efficient querying of local density and curvature, though specific bit-packing is not detailed in the sources [S1].

**Steps:**
1. **Initialization:** Start with a coarse uniform grid of resolution $8^3$. Let $N_{init} = 8^3$ and $N_{target} = 50^3$ [S1].
2. **Quality Assessment:** Evaluate the current point set to identify regions of high curvature variation or low site density. These regions are identified as topologically critical [S1].
3. **Insertion:** Iteratively insert new points into the identified critical regions. The insertion is guided by a scoring function based on local spacing and curvature [S1].
4. **Projection:** For each new sampling point, compute its corresponding surface projection. The method utilizes both sampling points and their projections to enhance reconstruction quality [S1].
5. **Termination:** Repeat steps 2-4 until the total number of points (sampling points plus projections) reaches $N_{target}$ [S1].

**Formulas:**
- The sources do not provide explicit mathematical formulas for the scoring function or the insertion criterion. They state that reconstruction quality correlates directly with the proportion of adaptively inserted samples [S1].

**Parameters:**
- Initial adaptive grid resolution: $8^3$ [S1].
- Total points for adaptive expansion: $50^3$ (including sample points and projections) [S1].

**Edge Cases and Pitfalls:**
- Uniform sampling produces reconstructions with notable topological artifacts and missing features in geometrically complex regions [S1].
- The strategy is particularly important for high-genus models where intricate details (e.g., chair backs) must be preserved [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_point_insertion(initial_resolution=8, target_points=50**3):
    # Initialize with uniform grid
    points = generate_uniform_grid(initial_resolution)
    
    # Calculate initial count
    current_count = len(points)
    
    # Iteratively insert points until target is reached
    while current_count < target_points:
        # Identify regions of high curvature or low density
        # Note: Specific scoring function not provided in sources [S1]
        critical_regions = identify_critical_regions(points)
        
        # Insert new points in critical regions
        new_points = insert_points_in_regions(points, critical_regions)
        
        # Compute surface projections for new points
        projections = compute_projections(new_points)
        
        # Update point set
        points.extend(new_points)
        # Note: Projections are included in the total count [S1]
        current_count = len(points) + len(projections)
        
    return points, projections
```

## Subtopics

### [DCCVT Upsampling Strategy](9892-dccvt-upsampling-strategy.md)
The DCCVT Upsampling Strategy is an adaptive refinement method for Differentiable Clipped Centroidal Voronoi Tessellation (DCCVT) that improves mesh quality by inserting new Voronoi sites in regions of high curvature or low site density. It solves the problem of insufficient site resolution in complex topological regions by using a scoring function based on local spacing and curvature to guide importance sampling. It sets itself apart from uniform upsampling by preserving local connectivity through tetrahedral insertion and aligning new sites with the SDF gradient.

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Measured Performance:** At $50^3$ resolution, the adaptive approach (Our2) demonstrates significantly superior topological fidelity compared to uniform sampling and other methods like RFTA, which exhibits spherical artifacts [S1].
- **Scaling:** Reconstruction quality correlates directly with the proportion of adaptively inserted samples; a higher percentage of points added via quality-driven criteria yields superior topological fidelity [S1].
- **Strengths:** Preserves intricate details (e.g., chair backs) and surface cavities that conventional methods (Marching Cubes, Dual Contouring, RFTS) fail to capture [S1].
- **Limitations:** Not given in the sources.
- **When to use:** When high-genus models or complex topological features need to be reconstructed with a limited sampling budget.
- **When not:** When uniform sampling is sufficient for simple geometries, as the adaptive strategy adds computational overhead for identification and insertion.

## Background
The Adaptive Point Insertion Strategy is described in the context of Power Diagram Enhanced Adaptive Isosurface Extraction from Signed Distance Fields. It is used to improve the reconstruction of surface cavities and high-genus models, addressing failures of methods like RFTA to capture intricate details [S1].

## Sources
[S1] Power Diagram Enhanced Adaptive Isosurface Extraction From Signed Distance Fields., https://doi.org/10.1109/tvcg.2026.3685989, fetched 2026-10-26
