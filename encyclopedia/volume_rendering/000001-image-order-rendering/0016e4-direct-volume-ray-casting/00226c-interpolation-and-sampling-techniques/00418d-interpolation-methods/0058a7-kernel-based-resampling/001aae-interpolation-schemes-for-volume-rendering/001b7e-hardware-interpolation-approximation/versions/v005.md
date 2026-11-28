# Hardware Interpolation Approximation

Also called: Piecewise Bilinear/Trilinear Interpolation via Polygon Subdivision

## Overview
Hardware Interpolation Approximation approximates the volume rendering integral by assuming linear interpolation along ray segments, resulting in piecewise bilinear or trilinear interpolation depending on polygon subdivision [S1, unofficial]. It sets itself apart from its siblings by relying on view-dependent, hardware-enforced piecewise interpolation determined by polygon subdivision, rather than explicit, view-independent kernel-based reconstruction [S1, unofficial].

## How it works
**Inputs and Outputs:** Not given in the sources [S1, unofficial].

**Data Structures:** Not given in the sources [S1, unofficial].

**Steps:**
1. The polygon faces are subdivided into triangles or trapezoids [S1, unofficial].
2. The polygons are rasterized, and the graphics hardware interpolates vertex attributes across the polygon surface [S1, unofficial].
3. Interpolation mode is determined by the subdivision strategy [S1, unofficial]:
   - If divided into triangles, the hardware performs linear interpolation across each triangle [S1, unofficial]. Combined with linear interpolation along ray segments, this yields piecewise bilinear interpolation [S1, unofficial].
   - If divided into trapezoids via horizontal lines through vertices, the hardware performs bilinear interpolation within trapezoids, or linear interpolation if a trapezoid degenerates into a triangle [S1, unofficial]. Combined with linear interpolation along ray segments, this yields piecewise trilinear interpolation (or bilinear in degenerate cases) [S1, unofficial].
4. Integration approximation assumes linear interpolation along the ray segments passing through the volume [S1, unofficial].

**Formulas:** Not given in the sources [S1, unofficial]. The interpolation function $I(\mathbf{p})$ is constrained by the subdivision: linear within triangles, bilinear within trapezoids [S1, unofficial].

**Parameters:** Subdivision strategy (triangles vs. trapezoids) dictates the interpolation order [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Specification Ambiguity:** The exact interpolation produced is hard to determine because interpolation across polygons is not completely determined in the OpenGL specifications [S1, unofficial].
- **View Dependence:** The interpolation is view-dependent because the subdivision into pieces depends on the view plane polygon subdivision and any further hardware subdivision [S1, unofficial]. It only corresponds to finite element interpolation functions in very special viewing situations [S1, unofficial].
- **Non-planar Faces:** The sources mention considering the effect of non-planar faces and subdividing quadrilateral faces by diagonals, but the text cuts off before stating the conclusion [S1, unofficial].

## Subtopics

### [First Order Integration (Trapezoid Rule)](2077-first-order-integration-trapezoid-rule.md)
First Order Integration approximates the volume rendering integral along a ray segment by assuming the scalar field varies linearly between two discrete samples, corresponding to the trapezoid rule. It solves the artifact problem of zero-order (Riemann sum) integration, where piecewise-constant assumptions cause visible discontinuities and poor capture of high-frequency transfer function details. Unlike its parent [Hardware Interpolation Approximation](1027-hardware-interpolation-approximation.md), which relies on hardware interpolation of the scalar field, this method explicitly models the linear variation to enable pre-computation of the integral into a lookup table.

## Operating characteristics
Not given in the sources [S1, unofficial].

## Background
The method was discussed in the context of cell projection of meshes with non-planar faces [S1, unofficial].

## Sources
[S1] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
