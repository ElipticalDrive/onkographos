# Hough Space Silhouette Detection

Also called: Li's silhouette detection

## Overview
Hough Space Silhouette Detection is a method for identifying object silhouettes in volume rendering by transforming surface geometry into a dual parameter space (Hough space) [S1]. It solves the problem of efficiently sampling silhouette edges for illustrative rendering by detecting intersections in this transformed domain [S1]. Unlike volumetric gradient-based methods that rely on local intensity derivatives, this approach uses global geometric properties defined by the view direction and surface normals [S1]. It sets itself apart from siblings like [Line Illustration Detection](772-line-illustration-detection.md) by explicitly leveraging the duality between points and lines to handle silhouette visibility, a technique that [Dual Surface Silhouette Detection](22175-dual-surface-silhouette-detection.md) later refines for both orthographic and perspective projections [S1].

## How it works
**Inputs:**
- A set of surface points or primitives representing the object geometry.
- A view direction vector $\mathbf{v}$.

**Outputs:**
- A set of silhouette edge samples.

**Data Structures:**
- **Hough Space:** A parameter space where each dimension corresponds to a geometric property of the silhouette. For a 2D projection, a line is defined by $(\rho, \theta)$. In 3D volume rendering contexts, the space typically maps surface normals and view directions to detect occlusion boundaries [S1].
- **Accumulator Array:** A discretized grid in Hough space used to count votes for potential silhouette parameters.

**Steps:**
1. **Parameterization:** For each surface element (e.g., a point or small patch) with normal $\mathbf{n}$, compute its contribution to the Hough space based on the current view direction $\mathbf{v}$ [S1]. The condition for a silhouette is typically $\mathbf{n} \cdot \mathbf{v} = 0$ (for orthographic) or a specific angular relationship for perspective [S1].
2. **Voting:** Increment the accumulator cells in Hough space that correspond to the geometric parameters of the detected silhouette candidates [S1].
3. **Peak Detection:** Identify peaks in the accumulator array. High counts indicate consistent silhouette parameters across multiple surface elements [S1].
4. **Sampling:** Extract edge samples from the surface elements associated with the detected peaks [S1].
5. **Filtering:** Reject nodes or samples that do not correspond to visible silhouettes from the current viewpoint [S1].

**Formulas:**
- Silhouette condition (orthographic): $\mathbf{n} \cdot \mathbf{v} = 0$ [S1].
- The specific mapping to Hough space coordinates is detailed in Li's thesis [S1].

**Parameters:**
- Resolution of the Hough space accumulator: Not given in the sources.
- Threshold for peak detection: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Secondary Visibility:** The method is not extremely efficient for secondary visibility (shadowing/occlusion from other objects) when scene complexity is high [S1].
- **Perspective Projection:** While applicable, the duality map requires careful handling for perspective projections compared to orthographic ones [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hough_space_silhouette_detection(surface_elements, view_direction):
    # Initialize Hough space accumulator
    # Dimensions depend on the specific parameterization (e.g., normal angles)
    hough_space = initialize_accumulator()
    
    for element in surface_elements:
        normal = element.normal
        # Check silhouette condition
        # For orthographic: dot(normal, view_direction) == 0
        # Specific Hough mapping applied here [S1]
        if is_silhouette_candidate(normal, view_direction):
            # Map to Hough space coordinates
            coords = map_to_hough_space(normal, view_direction)
            # Increment accumulator
            hough_space.increment(coords)
            
    # Find peaks in Hough space
    peaks = find_peaks(hough_space)
    
    # Retrieve silhouette samples
    silhouette_samples = []
    for peak in peaks:
        # Get surface elements contributing to this peak
        contributing_elements = get_contributors(peak)
        for elem in contributing_elements:
            silhouette_samples.append(elem.sample())
            
    return silhouette_samples
```

## Subtopics

### [Dual Surface Silhouette Detection](22175-dual-surface-silhouette-detection.md)
A geometric method that transforms surface geometry into a dual space, reducing silhouette detection to the intersection of the viewplane with the dual surface. It builds on Hough Space Silhouette Detection by utilizing duality maps to simplify visibility computation and handles both orthographic and perspective projections more generally.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Reasonably efficient for directly visible surfaces [S1]. Not extremely efficient for secondary visibility in high-complexity scenes [S1].
- **Strengths:** Effective for edge sampling of directly visible surfaces [S1].
- **Limitations:** Efficiency drops with scene complexity due to secondary visibility checks [S1].

## Background
- **Origin:** Proposed by Li et al. [S1].
- **History:** Used in Li's edge sampling approach [22, 23] for illustrative volume rendering [S1].
- **Usage:** Applied in differentiable rendering contexts for edge enhancement [S1].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
