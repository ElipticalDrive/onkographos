# Suggestive Contour Extraction

Also called: Suggestive contours

## Overview
Suggestive contour extraction identifies surface locations where radial curvature is zero but increasing towards the viewer, complementing standard geometric contours to enhance shape perception [S1]. Unlike standard contours which mark the silhouette of the object relative to the camera, suggestive contours reveal the "hidden" curvature of the surface, particularly on convex regions, providing a more complete line drawing of the volume's form [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric scalar field $\phi$, an isovalue $\tau$ defining the isosurface $\phi - \tau = 0$, and a camera view vector $\mathbf{v}$.
*   **Output:** A set of line segments representing the suggestive contours on the isosurface.

**Data Structures:**
*   **Voxel Grid:** The scalar field $\phi$ is stored in a 3D grid.
*   **Normal/Gradient:** Computed numerically on demand using finite differences over a local neighborhood [S1]. Precomputing is possible but increases memory usage by 4x [S1].

**Formulas:**
The radial curvature function $s$ is defined as:
$$ s(i, j, k) = \nabla \hat{\mathbf{n}}(i, j, k) \cdot \mathbf{v}(i, j, k) \cdot \mathbf{v}(i, j, k) $$
where $\hat{\mathbf{n}}$ is the unit-length normal vector and $\mathbf{v}$ is the view vector [S1]. The sign of $s$ corresponds to the sign of the radial curvature [S1].

**Parameters:**
*   $\varepsilon$: A small positive threshold used to filter spurious zero crossings caused by noise [S1].

**Steps:**
1.  **Extract Zero Set:** Identify surface locations where $s(i, j, k) = 0$. This is the zero set of the radial curvature function [S1].
2.  **Frontface Test:** For front-facing surfaces (where the normal points towards the camera), check that the directional derivative of $s$ towards the viewer is positive. In practice, compare the derivative to the threshold: $D_s > \varepsilon$ [S1].
3.  **Backface Test:** For back-facing surfaces, negate the sign of the derivative test to maintain analogous lines to those found on frontfaces. Check that $D_s < -\varepsilon$ [S1].
4.  **Filtering:** Discard segments that do not meet the threshold criteria to avoid noise-induced artifacts [S1].
5.  **Interior Contour Detection (Optional):** Test for interior contours (e.g., bottom of indentations) which are never visible from outside. These are identified by testing if $s < 0$ once a segment has been extracted [S1]. These can be omitted or stylized based on visibility and length [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_suggestive_contours(volume, isovalue, view_vector, epsilon):
    """
    Extracts suggestive contours from a volumetric dataset.
    
    Args:
        volume: 3D array of scalar values.
        isovalue: The threshold value for the isosurface.
        view_vector: 3D vector pointing from surface to camera.
        epsilon: Threshold for derivative test to filter noise.
        
    Returns:
        List of line segments.
    """
    segments = []
    
    # Iterate through voxels to find isosurface points
    for i, j, k in volume.indices():
        if is_on_isosurface(volume[i, j, k], isovalue):
            # Compute normal and gradient numerically on demand
            normal = compute_normal(volume, i, j, k)
            grad_normal = compute_gradient_of_normal(volume, i, j, k)
            
            # Calculate radial curvature function s
            # s = grad_normal . view_vector . view_vector
            s = dot(grad_normal, dot_product(view_vector, view_vector))
            
            # Check if s is approximately zero (within a small tolerance for zero crossing)
            if abs(s) < ZERO_TOLERANCE:
                # Compute derivative of s towards viewer
                ds = compute_derivative_of_s(volume, i, j, k, view_vector)
                
                # Determine if front or back facing
                is_front_facing = dot(normal, view_vector) > 0
                
                if is_front_facing:
                    # Frontface: derivative must be positive and above threshold
                    if ds > epsilon:
                        segments.append(create_segment(i, j, k))
                else:
                    # Backface: derivative must be negative and below negative threshold
                    if ds < -epsilon:
                        segments.append(create_segment(i, j, k))
                        
    return segments
```

**Edge Cases and Pitfalls:**
*   **Noise:** Spurious zero crossings can occur due to numerical noise. The threshold $\varepsilon$ is critical for filtering these [S1].
*   **Backfacing Contours:** Previous work assumed opaque surfaces and did not address backfacing suggestive contours. Correct implementation requires negating the derivative test for backfaces [S1].
*   **Interior Contours:** Contours at the bottom of indentations have negative radial curvature and are never visible from the outside. These can be detected by testing $s < 0$ and optionally omitted [S1].
*   **Performance:** Computing normals and gradients numerically on demand is less efficient than precomputing, which would cost 4x memory [S1].

## Subtopics

### Interior Contour Detection
A method to detect and omit 'interior' contours (those never visible from outside the surface, e.g., bottom of an indentation) by testing for negative radial curvature [S1]. Once a segment is extracted, it is tested to see if $s < 0$. If so, it is identified as an interior contour and can be omitted or stylized [S1]. Normals and gradients are computed numerically on demand using finite differences, which is less efficient than precomputing (which would cost 4x memory) [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels and the efficiency of gradient computation. Randomized probing with temporal coherence can accelerate the search for new contours [S1].
*   **Memory Complexity:** Numerical computation on demand is memory efficient. Precomputing normals increases memory usage by 4x [S1].
*   **Strengths:** Complements standard contours to produce more effective renderings of shape, revealing hidden curvature [S1].
*   **Limitations:** Numerical computation of gradients can be noisy and less efficient than precomputed methods [S1].

## Background
Suggestive contours were introduced by DeCarlo et al. [2003] to complement standard contours in line drawings [S1]. The method builds on the concept of radial curvature, where contours are defined as surface locations where radial curvature is zero [S1]. The specific implementation described here extends the original work to handle backfacing surfaces and interior contours [S1].

## Sources
[S1] Line drawings from volume data, https://doi.org/10.1145/1073204.1073222, fetched 2026-10-28
