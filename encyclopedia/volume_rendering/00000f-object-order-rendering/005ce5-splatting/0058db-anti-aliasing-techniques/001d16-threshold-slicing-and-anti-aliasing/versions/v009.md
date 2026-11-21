# Threshold Slicing and Anti-Aliasing

Also called: Hybrid Threshold Splatting

## Overview
Threshold Slicing and Anti-Aliasing is a hybrid volume rendering technique for unstructured and curvilinear grids that combines polygon-based slicing for high-value data with splatting for low-value data [S1]. It solves the problem of aliasing and geometric artifacts in sparse or thresholded volumes by rendering cells above a specific volume threshold as projected polygons, while rendering cells below that threshold (including those lying between slice planes or pixel samples) using anti-aliased ellipsoidal splats [S1]. This approach distinguishes itself from pure slicing methods by preserving the continuous density representation of low-density regions, and from pure splatting by providing sharper geometric boundaries for high-density features [S1].

## How it works
**Inputs:**
- Unstructured or curvilinear grid volume data.
- A scalar field value for each cell.
- A user-defined volume threshold $T$.
- Camera parameters (view direction, projection type).

**Outputs:**
- A 2D image where high-threshold cells are rendered as sharp polygons and low-threshold cells as smooth, anti-aliased splats.

**Data Structures:**
- Cell list: Each cell $C_i$ contains its geometric vertices, scalar value $v_i$, and classification (above/below threshold).
- Render Queue: Two separate lists or stages: one for polygon primitives (cells with $v_i \ge T$) and one for splat primitives (cells with $v_i < T$).

**Steps:**
1. **Classification:** Iterate through all cells in the volume. For each cell $C_i$ with scalar value $v_i$:
   - If $v_i \ge T$, classify $C_i$ as a "Polygon Cell".
   - If $v_i < T$, classify $C_i$ as a "Splat Cell".
2. **Polygon Projection (High Threshold):** For each Polygon Cell:
   - Project the 3D vertices of the cell onto the 2D image plane using the current view matrix.
   - Determine the visible faces (front-facing vs. back-facing) based on the view direction.
   - For orthogonal projections, the length $l$ of the viewing ray segment within the cell is constant for all rays passing through the projected polygon [S1].
   - Rasterize the projected polygon. The intensity is modulated by the cell's density/value.
3. **Splat Rendering (Low Threshold):** For each Splat Cell:
   - Compute the centroid or representative point of the cell in 3D space.
   - Transform this point to screen space.
   - Generate an anti-aliased ellipsoidal splat footprint. The shape of the splat accounts for the cell's geometry and the view direction to minimize aliasing [S1].
   - Rasterize the splat, accumulating its contribution to the pixel buffer using alpha blending.
4. **Compositing:**
   - Combine the results from the polygon stage and the splat stage. Since polygons represent discrete high-density regions and splats represent continuous low-density regions, they are composited together. Standard back-to-front or front-to-back alpha blending is applied to the final pixel values [S1].

**Parameters:**
- Threshold $T$: The scalar value separating polygon rendering from splat rendering. Typical values depend on the data range and the desired level of detail.

**Edge Cases and Pitfalls:**
- **Boundary Artifacts:** Discontinuities may occur at the threshold boundary if the splat footprint does not adequately cover the area where the polygon ends. Careful tuning of the splat kernel size is required [S1].
- **Precision:** Floating-point precision issues in projecting small cells may lead to incorrect classification or rasterization errors.
- **Degenerate Input:** Cells with zero volume or collapsed geometry should be skipped or handled specially to avoid division by zero or invalid splat shapes.

## Subtopics

## Operating characteristics
- **Time Complexity:** Depends on the number of cells and the threshold. Polygon projection is $O(N_{poly})$ where $N_{poly}$ is the number of cells above threshold. Splatting is $O(N_{splat})$ where $N_{splat}$ is the number of cells below threshold. Total time is $O(N_{poly} + N_{splat})$.
- **Memory Complexity:** $O(N)$ to store the cell data and render queues.
- **Performance:** Provides interactive performance for large datasets by leveraging GPU rasterization for both polygons and splats [S1].
- **Strengths:** Sharp rendering of high-density structures; smooth, anti-aliased rendering of low-density structures; handles unstructured grids effectively.
- **Limitations:** Requires careful tuning of the threshold; potential visual discontinuities at the threshold boundary; complex implementation due to hybrid nature.

## Background
The method was proposed by Silva et al. in 2003 as part of a broader framework for volume rendering on curvilinear and unstructured grids [S1]. It builds upon earlier work by Yagel et al. on slicing cells into polygons and Zwicker et al. on anti-aliased ellipsoidal splatting [S1]. The technique addresses the limitations of pure slicing (aliasing, geometric artifacts) and pure splatting (loss of sharp features) by combining both approaches based on a data-driven threshold.

## Sources
[S1] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
