# Stream Surfaces

Also called: Streamsheet, Van Wijk Stream Surface Method, Van Wijk's method

## Overview
Stream surfaces visualize steady-state vector fields by stitching together adjacent streamlines to form a continuous geometric surface, solving the problem of ascertaining overall flow direction in areas of substantial directional change where individual streamlines curl around each other [S1, unofficial]. Unlike its parent [Stream Lines](4796-stream-lines.md), which represents flow as discrete trajectories, stream surfaces provide a coherent sheet that locally separates the three-dimensional flow, restricting streamlines on one side to that portion of the volume [S1, unofficial].

## How it works
**Inputs:** A steady-state vector field $\mathbf{v}(\mathbf{x})$ defined over a volume, and a starting curve (seed line) $\mathbf{c}(s)$ from which the surface emanates.
**Outputs:** A triangular mesh representing the stream surface.

**Data Structures:**
1.  **Vector Field:** Discretized on a regular grid.
2.  **Stream Surface Mesh:** A set of vertices and connectivity indices (triangles).
3.  **Boundary Map (Van Wijk variant):** A grid-aligned scalar field storing boundary intersection coordinates.

**Method 1: Direct Stitching (Hultquist)**
This method constructs the surface by integrating streamlines from the seed curve and connecting them.

1.  **Seed Initialization:** Define the starting curve $\mathbf{c}(s)$ parameterized by $s$. Discretize this curve into a set of seed points $\mathbf{p}_i = \mathbf{c}(s_i)$.
2.  **Streamline Integration:** For each seed point $\mathbf{p}_i$, integrate the steady-state vector field $\mathbf{v}(\mathbf{x})$ forward in time to generate a streamline $\mathbf{l}_i(t)$.
    *   Use a numerical integrator (e.g., Runge-Kutta) to solve $\frac{d\mathbf{x}}{dt} = \mathbf{v}(\mathbf{x})$.
    *   Store the resulting points for each streamline.
3.  **Surface Construction:** Connect the corresponding points from adjacent streamlines $\mathbf{l}_i(t)$ and $\mathbf{l}_{i+1}(t)$ to form quadrilateral strips.
    *   For each time step $t$, create triangles between vertices $(\mathbf{l}_i(t), \mathbf{l}_{i+1}(t), \mathbf{l}_i(t+\Delta t))$ and $(\mathbf{l}_{i+1}(t), \mathbf{l}_{i+1}(t+\Delta t), \mathbf{l}_i(t+\Delta t))$.
4.  **Adaptive Refinement:** Monitor the surface mesh for distortion.
    *   Calculate the distance between adjacent streamlines. If the distance exceeds a threshold or the surface normal changes abruptly (indicating divergence around obstacles), subdivide the seed curve segment $[s_i, s_{i+1}]$ by inserting new seed points.
    *   Re-integrate streamlines for the new seeds.
    *   Allow for "tears" in the surface if the flow diverges significantly, preventing the mesh from becoming invalid or overly distorted [S1, unofficial].

**Method 2: Van Wijk’s Iso-Contour Scheme**
This simpler scheme constructs stream surfaces for closed environments by precomputing a scalar field on a regular grid.

1.  **Grid Precomputation:** Define a regular grid covering the volume of interest.
2.  **Backward Integration:** For each interior grid point $\mathbf{x}_g$:
    *   Integrate the vector field $\mathbf{v}(\mathbf{x})$ *backward* in time from $\mathbf{x}_g$ until the trajectory intersects the volume boundary $\partial V$.
    *   Let the intersection point be $\mathbf{b}_g$.
    *   Store $\mathbf{b}_g$ for each grid point.
3.  **Scalar Field Assignment:** Assign scalar values to the boundary points.
    *   The scalar value at a boundary point $\mathbf{b}$ is determined by its position along the seed curve or a parameterization of the boundary that correlates with the flow origin.
    *   Interpolate these scalar values from the boundary points to the interior grid points to create a scalar function $\phi(\mathbf{x})$ defined over the regular grid.
4.  **Iso-Contour Extraction:** Extract the iso-contour surface of $\phi(\mathbf{x})$ corresponding to the specific value associated with the seed curve. This surface represents the stream surface [S1, unofficial].

**Parameters:**
*   **Integration Step Size:** Small enough to capture curvature but large enough for performance. Not given in the sources.
*   **Refinement Threshold:** Distance or normal change limit for adaptive splitting. Not given in the sources.
*   **Boundary Condition:** The flow must enter or exit the volume boundaries for Van Wijk's method to work; artificial boundaries may be placed if necessary [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Unsteady Flows:** The utility of stream surfaces is lost for unsteady flows, as the concept of a fixed streamline surface relies on time-invariance [S1, unofficial].
*   **Loss of Tangential Direction:** Merging many streamlines into a continuous surface obscures the local flow direction tangential to the surface. It is impossible to determine if the flow went straight down the surface or weaved back and forth [S1, unofficial].
*   **Mesh Distortion:** The algorithm suffers from severe distortion of the surface mesh as the flow diverges around obstacles, necessitating adaptive refinement [S1, unofficial].
*   **Boundary Dependency:** Van Wijk's method requires flow to cross volume boundaries. If the flow is contained entirely within the volume, artificial boundaries must be introduced [S1, unofficial].

## Subtopics

### [Stream-Balls](7700-stream-balls.md)
Stream-Balls is a technique for visualizing steady-state vector fields that constructs a scalar potential field from advected particles or streamlines using blobby functions, then extracts iso-contours to mimic stream surfaces. It sets itself apart from its parent [Stream Surfaces](7696-stream-surfaces.md) by replacing explicit geometric stitching with an implicit scalar field, which allows the resulting surface to gracefully split apart in areas of high divergence and break apart in areas of high velocity to indicate magnitude.

### [LIC on Stream Surfaces](7721-lic-on-stream-surfaces.md)
LIC on Stream Surfaces is a visualization technique that superimposes a texture generated by the Line Integral Convolution (LIC) algorithm onto stream surfaces to resolve the lack of directional information inherent in standard surface rendering. It differs from standard stream surfaces by providing dense, local flow direction cues across the entire surface area, whereas siblings like [Stream-Balls](7700-stream-balls.md) or [Hairs](12773-hairs.md) rely on discrete geometric primitives or point-based markers that may fail to convey continuous flow coherence in regions of substantial directional change.

### [Hairs](12773-hairs.md)
Hairs is a visualization technique for representing steady-state flow fields near or around contour surfaces, specifically addressing the failure of fiber bundles (or line bundles) when the flow vector points slightly into the surface. It solves this by growing tiny, segmented hairs from the surface that bend according to the local velocity field. Unlike its sibling [Stream Surfaces](7696-stream-surfaces.md), which stitches adjacent streamlines into a continuous surface, Hairs remains a discrete set of line segments, offering a lighter-weight alternative for sparse or near-surface flow visualization where surface continuity is difficult to maintain or unnecessary.

### [Streamtape Rendering](14472-streamtape-rendering.md)
Streamtape rendering is a technique for visualizing steady-state vector fields by representing selected representative streamlines as twisted surfaces (streamtapes) rather than simple lines or ribbons. It solves the problem of visual clutter and poor depth perception inherent in traditional streamline visualization, particularly in complex flows where streamlines curl and twist. Unlike [Stream Surfaces](7696-stream-surfaces.md), which stitch adjacent streamlines into a continuous sheet, streamtapes operate on individual representative streamlines, using local torsion to determine width and orientation to automatically manage visual complexity and convey local flow structure without the orientation-dependent flaws of streamribbons.

### Streak Surface
Streak surfaces model flow regions passing through a domain by tracking particles emerging from a seeding curve at different ages in a time-dependent flow [S2, unofficial]. Unlike steady stream surfaces, streak surfaces account for temporal changes in the velocity field. The surface $S$ is defined by the integral:
$$ S(r, s, t) = c(s) + \int_{t-r}^{t} f(S(r - (t - \tau), s, \tau), \tau) d\tau $$
where $r \in [0, t]$ is the particle age parameter, $s$ indexes the seeding curve, and $f$ is the velocity field [S2, unofficial]. The parameter $r$ identifies individual particle instances within the surface [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of seed points and the integration length. Adaptive refinement increases complexity dynamically based on flow divergence. Van Wijk's method requires a backward integration for every grid point, making it $O(N)$ where $N$ is the number of grid cells, plus the cost of iso-contour extraction.
*   **Memory Complexity:** Stores the vector field, the surface mesh, and potentially the boundary map for Van Wijk's method.
*   **Strengths:** Provides a clear, continuous visualization of flow topology; efficient rendering using standard graphics hardware for a single surface; resolves ambiguity in curling flow fields better than individual streamlines.
*   **Limitations:** Computationally expensive due to integration and adaptive refinement; loses local tangential flow direction information; ineffective for unsteady flows; prone to mesh distortion in divergent flows.
*   **When to Use:** Steady-state flows where understanding the global separation of flow regions is important, and individual streamlines are too cluttered or ambiguous.
*   **When Not to Use:** Unsteady flows, or when precise local tangential flow direction on the surface is required without additional texturing.

## Background
Stream surfaces were introduced as an extension of streamlines to address the difficulty of ascertaining overall flow in areas of substantial directional change where streamlines curl around each other [S1, unofficial]. Hultquist [1] proposed the original method of stitching adjacent streamlines to build a stream surface, noting that for steady-state flows, these surfaces locally separate the three-dimensional flow [S1, unofficial]. Van Wijk [2] later proposed a simpler scheme for closed environments using backward integration and iso-contour extraction [S1, unofficial]. Interrante and others [3] later proposed using Line Integral Convolution (LIC) [4] to superimpose texture on stream surfaces to recover the lost tangential flow direction [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
