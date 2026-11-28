# Truncated Ellipsoid Primitive

Also called: Cube-Ellipsoid Intersection Primitive

## Overview
The Truncated Ellipsoid Primitive is a voxel-based volume representation that models non-uniform intra-voxel geometry by intersecting a cubic voxel with a fitted bounding ellipsoid [S1, unofficial]. It solves the double-counting artifacts (checkerboard patterns) inherent in simple cube voxels when viewing diagonally, by providing a tighter fit to the actual geometry distribution within the voxel [S1, unofficial]. Unlike 3D Gaussians, which can overlap ambiguously, the truncation to the cube avoids ambiguity in defining inter-occlusion between primitives [S1, unofficial].

## How it works
**Inputs:**
- A set of voxels, each defined by its cubic bounds.
- Precomputed ellipsoid parameters for each voxel, fitted to the geometry contained within that voxel.

**Outputs:**
- A dense volume representation where each voxel contributes to rendering based on its truncated ellipsoid shape.

**Data Structures:**
- **Voxel Cube:** Implicitly defined by the voxel grid index and voxel size.
- **Ellipsoid:** Defined by its center $\mathbf{c}$, orientation (rotation matrix $R$), and scale (diagonal matrix $S$). The ellipsoid equation is $(\mathbf{x} - \mathbf{c})^T (R S^{-2} R^T) (\mathbf{x} - \mathbf{c}) \le 1$.
- **Primitive Coverage:** A scalar value $c(\omega)$ per voxel per view direction $\omega$.

**Steps:**
1. **Precomputation (Ellipsoid Fitting):** For each voxel, fit a bounding ellipsoid to the geometry contained within it. The ellipsoid adapts to the geometry: it becomes tight for flat surfaces or thin structures, and falls back to approximating the cube for unstructured geometry [S1, unofficial].
2. **Storage:** Store the cube bounds and the ellipsoid parameters separately. Do not explicitly store the intersection shape [S1, unofficial].
3. **Rendering (On-the-fly Calculation):** For each ray or view direction $\omega$:
    a. Calculate $|B|_\omega$, the projected area of the primitive (the intersection of the cube and the ellipsoid).
    b. Calculate $|A|_\omega$, the projected area of the surfaces bounded by the primitive (the actual geometry).
    c. Compute the primitive coverage:
       $$ c(\omega) = \frac{|A|_\omega}{|B|_\omega} $$
    d. Use $c(\omega)$ to accumulate contributions, ensuring semi-transparency reflects the quantity of underlying geometry [S1, unofficial].

**Formulas:**
- **Primitive Coverage:**
  $$ c(\omega) = \frac{|A|_\omega}{|B|_\omega} $$
  Where:
  - $|A|_\omega$: Projected area of the surfaces bounded by the primitive.
  - $|B|_\omega$: Projected area of the primitive itself (intersection of cube and ellipsoid).
  - Note: In accumulation, $|A|_\omega$ often cancels out, so only $|B|_\omega$ is explicitly needed for weighting [S1, unofficial].

- **Projected Area of Cube-Ellipsoid Intersection ($|B|_\omega$):**
  Calculated using an efficient Monte Carlo estimator or by integrating over the projected contour using Green's theorem [S1, unofficial].

**Parameters:**
- **Voxel Size:** Chosen to be half of the pixel footprint to match the image-space Nyquist-Shannon sampling rate [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Double-Counting Artifacts:** Simple cube voxels cause checkerboard-like artifacts when viewing diagonally because voxels have thickness and rays may intersect multiple voxels for a single surface point [S1, unofficial]. The truncated ellipsoid primitive reduces this by providing a tighter fit [S1, unofficial].
- **Ambiguity:** Unlike 3D Gaussians, the truncation avoids ambiguity in defining inter-occlusion between primitives [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_primitive_coverage(voxel_cube, ellipsoid, view_direction):
    """
    Calculates the primitive coverage c(omega) for a truncated ellipsoid primitive.
    
    Args:
        voxel_cube: Object representing the cubic voxel bounds.
        ellipsoid: Object representing the fitted ellipsoid (center, rotation, scale).
        view_direction: Normalized 3D vector omega.
        
    Returns:
        float: Primitive coverage c(omega).
    """
    # Step 1: Calculate |B|_omega, the projected area of the intersection of cube and ellipsoid
    # This can be done via Monte Carlo or Green's theorem integration.
    # Here we assume a function exists for this calculation.
    projected_area_primitive = calculate_projected_area_intersection(voxel_cube, ellipsoid, view_direction)
    
    # Step 2: Calculate |A|_omega, the projected area of the surfaces bounded by the primitive
    # This represents the actual geometry inside the voxel.
    # In many accumulation schemes, this cancels out, but it is part of the definition.
    projected_area_geometry = calculate_projected_area_geometry(voxel_cube, ellipsoid, view_direction)
    
    # Step 3: Compute coverage
    if projected_area_primitive == 0:
        return 0.0
        
    coverage = projected_area_geometry / projected_area_primitive
    
    return coverage

def calculate_projected_area_intersection(cube, ellipsoid, omega):
    """
    Calculates the projected area of the intersection of a cube and an ellipsoid.
    Uses Monte Carlo estimation as per [S1, unofficial].
    """
    # Implementation detail: Efficient Monte Carlo estimator for projected area of cube-ellipsoid intersection.
    # This is detailed in the supplemental document of [S1, unofficial].
    # Alternatively, integrate over the projected contour using Green's theorem.
    # Not given in the sources.
    pass

def calculate_projected_area_geometry(cube, ellipsoid, omega):
    """
    Calculates the projected area of the surfaces bounded by the primitive.
    """
    # Implementation detail: Projected area of the actual geometry within the voxel.
    # Not given in the sources.
    pass
```

## Operating characteristics
- **Time Complexity:** Depends on the method used to calculate $|B|_\omega$. Monte Carlo estimation is efficient but stochastic; Green's theorem integration is deterministic but potentially more complex to implement.
- **Memory Complexity:** Compact storage, as only the cube bounds and ellipsoid parameters are stored, not the explicit intersection shape [S1, unofficial].
- **Strengths:** Reduces double-counting artifacts; provides a tighter fit to geometry; avoids ambiguity in inter-occlusion compared to 3D Gaussians; easy to fit and compact to store [S1, unofficial].
- **Limitations:** Requires precomputation to fit ellipsoids; calculation of projected area is non-trivial [S1, unofficial].
- **When to Use:** When rendering scenes with non-uniform geometry distribution within voxels and diagonal viewing angles, to avoid checkerboard artifacts [S1, unofficial].
- **When Not to Use:** When simple uniform distribution is sufficient, or when precomputation cost is prohibitive.

## Background
The Truncated Ellipsoid Primitive was introduced in "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" by Yang Zhou, Tao Huang, Ravi Ramamoorthi, Pradeep Sen, and Ling-Qi Yan [S1, unofficial]. It addresses the limitations of simple cube voxels in level-of-detail rendering, specifically the systematic error manifesting as checkerboard-like artifacts when the film plane is not axis-aligned [S1, unofficial]. The method is designed to be a cost-effective alternative to brute-force supersampling or coverage masks [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
