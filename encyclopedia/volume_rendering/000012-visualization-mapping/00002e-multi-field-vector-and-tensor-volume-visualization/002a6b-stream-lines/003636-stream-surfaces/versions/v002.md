# Stream Surfaces

Also called: Streamsheet, Van Wijk Stream Surface Method, Van Wijk's method

## Overview
Stream surfaces visualize steady-state vector fields by stitching together adjacent streamlines to form a continuous geometric surface, solving the problem of ascertaining overall flow direction in areas where streamlines curl around each other [S1, unofficial]. Unlike [Stream Lines](4796-stream-lines.md), which represent flow as discrete trajectories, stream surfaces provide a unified geometric entity that can be rendered efficiently with standard graphics hardware, though they suffer from the loss of tangential flow direction information [S1, unofficial].

## How it works
**Inputs:** A steady-state vector field $\mathbf{v}(\mathbf{x})$ defined over a volume; a starting curve $\mathcal{C}$ (line segment or closed loop); a discretized regular grid covering the volume.
**Outputs:** A triangular mesh representing the stream surface.

**Data Structures:**
1.  **Grid:** A regular 3D grid with cell size $\Delta$. Each interior grid point $\mathbf{p}_i$ stores a scalar value $f(\mathbf{p}_i)$ representing the boundary intersection index or coordinate.
2.  **Boundary:** A set of points on the volume boundary $\partial \Omega$.
3.  **Surface Mesh:** A collection of triangles generated via iso-contouring.

**Steps:**
1.  **Precompute Grid:** Discretize the volume into a regular grid.
2.  **Backward Integration:** For each interior grid point $\mathbf{p}_i$:
    *   Integrate the vector field backward in time: $\frac{d\mathbf{x}}{dt} = -\mathbf{v}(\mathbf{x})$.
    *   Continue integration until the trajectory intersects the volume boundary $\partial \Omega$ at point $\mathbf{b}_i$.
    *   Store the position or an identifier of $\mathbf{b}_i$ as the scalar value $f(\mathbf{p}_i)$ [S1, unofficial].
3.  **Boundary Scalar Assignment:** Assign scalar function values to the boundary points based on the starting curve $\mathcal{C}$. If $\mathcal{C}$ is a line segment, the scalar value corresponds to the distance along $\mathcal{C}$ or a parameterization of the seed line [S1, unofficial].
4.  **Iso-Contour Extraction:** Extract the iso-surface of the scalar field $f(\mathbf{x})$ defined over the grid. The specific iso-value corresponds to the parameterization of the starting curve $\mathcal{C}$. This surface consists of all streamlines emanating from $\mathcal{C}$ [S1, unofficial].

**Formulas:**
The stream surface $S$ is defined as the set of points $\mathbf{x}$ such that:
$$ S = \{ \mathbf{x} \mid \phi(\mathbf{x}) = c \} $$
where $\phi(\mathbf{x})$ is the scalar field derived from the boundary intersection mapping, and $c$ is the constant corresponding to the seed curve [S1, unofficial].

**Parameters:**
*   Grid resolution: Not given in the sources.
*   Integration step size: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Unsteady Flows:** The method assumes steady-state flows. It is not applicable to unsteady flows [S1, unofficial].
*   **Boundary Conditions:** The flow must enter or exit the volume boundaries for the backward integration to terminate. Artificial boundaries may be required if the flow is confined [S1, unofficial].
*   **Distortion:** The surface mesh suffers from severe distortion as flow diverges around obstacles. An adaptive refinement algorithm is required to handle tears and maintain mesh quality [S1, unofficial].
*   **Direction Loss:** The tangential flow direction is lost on the surface because multiple streamlines are merged. It is impossible to determine if the flow moved straight or weaved back and forth [S1, unofficial].

## Subtopics

### [Stream-Balls](7700-stream-balls.md)
Replaces explicit geometric stitching with an implicit scalar field using blobby functions, allowing the surface to gracefully split in high divergence areas.

### [LIC on Stream Surfaces](7721-lic-on-stream-surfaces.md)
Superimposes Line Integral Convolution textures onto stream surfaces to resolve the lack of directional information inherent in the geometry.

## Operating characteristics
*   **Time Complexity:** Dominated by the backward integration for all interior grid points. Complexity is $O(N \cdot T)$, where $N$ is the number of grid points and $T$ is the average integration steps to reach the boundary.
*   **Memory Complexity:** $O(N)$ to store the scalar field on the grid.
*   **Strengths:** Efficient rendering of a single surface using standard graphics hardware; provides a clear separation of flow regions (especially with closed seed curves) [S1, unofficial].
*   **Limitations:** High computational cost for precomputation; severe mesh distortion requires adaptive refinement; loss of tangential flow direction; inapplicable to unsteady flows [S1, unofficial].
*   **When to Use:** Steady-state flows where a unified geometric representation of flow separation is needed.
*   **When Not to Use:** Unsteady flows, or when precise tangential flow direction on the surface is critical without additional texturing.

## Background
Stream surfaces were introduced by Hultquist [1] as an extension of streamlines to solve the difficulty of ascertaining overall flow in 3D where streamlines curl around each other [S1, unofficial]. Van Wijk [2] proposed a simpler scheme for closed environments using a grid-based backward integration and iso-contouring approach [S1, unofficial]. Interrante et al. [3] later proposed using Line Integral Convolution (LIC) [4] to add directional information to these surfaces [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
