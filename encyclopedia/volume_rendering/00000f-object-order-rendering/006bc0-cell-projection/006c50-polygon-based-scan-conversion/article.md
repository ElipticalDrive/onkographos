# Polygon-based scan conversion
Also called: Polygon-based volume rendering, Polygon scan conversion

## Overview
This family renders volume data by decomposing 3D cells into 2D screen-space polygons and rasterizing them using scan-line algorithms, rather than casting rays per pixel or projecting entire cells as splats [S1, unofficial][S2, unofficial]. It differs from [Cell projection](26-cell-projection.md) by operating on individual bounding polygons of cells (allowing arbitrary cell types and overlapping grids) and from [Parallel Cell Projection](875-parallel-cell-projection.md) by relying on software-based polygon scan conversion instead of hardware-accelerated or parallel cell projection techniques.

## Shared principles
The polygon-based scan conversion family shares the following mechanism, terms, and math:

1.  **Polygon Decomposition:** The volume is decomposed into a set of bounding polygons. Any cell type (hexahedra, tetrahedra, or hybrid/multi-sided cells resulting from decimation) is treated as a collection of planar faces [S1, unofficial].
2.  **Culling:** Polygons are culled if they do not lie within the viewing frustum or if their projected size is smaller than one pixel [S1, unofficial].
3.  **Scan Conversion:** Visible polygons are rasterized using a generalized scan-line algorithm. This algorithm renders both opaque surfaces and semi-transparent regions of space between polygons [S1, unofficial].
4.  **Handling Overlaps:** The method correctly handles overlapping cells in multi-grid datasets by rendering the semi-transparent regions between the projected polygons [S1, unofficial].
5.  **Performance Metric:** Rendering time is proportional to the number of polygons remaining after culling [S1, unofficial].

## Subtopics

### [3D Cube Projection and Rasterization](6678-3d-cube-projection-and-rasterization.md)
A software rendering technique that transforms the eight corners of a 3D voxel cube using sequential translation and rotation matrices, applies perspective projection to the vertices of each face, and rasterizes visible faces as 2D polygons. It differs from hardware rasterization by performing all depth testing and pixel writing in software, and from standard raycasting by explicitly drawing polygonal faces rather than casting rays per pixel. Choose this when explicit software control over cube face transformation and rasterization is required without hardware dependencies.

### [Incoherent Projection](7541-incoherent-projection.md)
Incoherent projection is a cell projection method for general convex hexahedra that renders the 2D projection of each cell as an arrangement of polygons by analyzing each cell individually in screen space, thereby avoiding case analysis for the high number of projection topologies. It builds on [Cell projection](26-cell-projection.md) by extending the coherent projection technique for rectilinear grids to irregular hexahedra, where different cells may fall into different nondegenerate or degenerate projection topologies. Unlike coherent projection, which assumes all cells share the same topology, incoherent projection handles the complexity of general grids by processing each cell's projection geometry independently.

### Polygon-Based Volume Rendering
#### definition
- [S1, unofficial] A direct volume rendering system based on generalized software scan conversion of polygons rather than ray-casting, projection, or splatting of cells. It decomposes any cell type into a set of bounding polygons to render semi-transparent regions of space between polygons as well as opaque surfaces.
#### step
- [S1, unofficial] Decompose any cell type into a set of bounding polygons.
- [S1, unofficial] Render semi-transparent regions of space between polygons, as well as opaque surfaces, using a scan conversion algorithm that generalizes traditional polygon scan-line methods.
- [S1, unofficial] Cull out polygons that do not lie in the viewing frustum.
- [S1, unofficial] Cull out polygons smaller than one pixel.
#### parameter
- [S1, unofficial] Rendering time is based on the number of polygons in the volume after culling.
#### pitfall
- [S1, unofficial] Overlapping cells in multi-grid data sets are handled correctly because the method renders semi-transparent regions between polygons.
- [S1, unofficial] Grids not made up of simple cell types are handled because the method is completely independent of grid type.
#### comparison
- [S1, unofficial] Unlike ray-casting which is complicated and slow for irregular grids, this method uses scan conversion of polygons. Unlike splatting/cell projection which may require re-gridding, this method works on arbitrary polygons without re-gridding.

### Polygon Backface Culling & Projection
#### step
- [S2, unofficial] Recompute polygon normal: `this.normal = new Plane(this).normal.normalise()`.
- [S2, unofficial] Backface culling: compute dot product of normal and vector from first vertex to player position. If `>= 0`, polygon faces away or is parallel; return false.
- [S2, unofficial] Project center point: `project = this.getCentre().project(player)`. If `project.z > 0.5`, discard polygon.
- [S2, unofficial] Map center to screen coordinates: `projectC.x = (Window.width/2 - player.viewFocus.x) + project.x * player.zoom`, same for Y, Z unchanged.
- [S2, unofficial] Loop over vertices: project each to camera space. If `project.z < 0`, discard. Map to screen: `projectX[i] = (int)((Window.width/2 - player.viewFocus.x) + project.x * player.zoom)`, same for Y.
#### layout
- [S2, unofficial] Polygon stores `vertexes` (Vector array), `normal` (Vector), `projectX` (int array), `projectY` (int array), `projectC` (Vector for center), `projectN` (Vector for normal line), `lighting` (double), `distance` (double).

## Background
The concept of polygon-based volume rendering was introduced to address the limitations of ray-casting and cell projection/splatting for irregular and decimated grids. Traditional ray-casting is slow for irregular grids, while cell projection and splatting often require re-gridding to maintain simple cell types, which is problematic during volume decimation [S1, unofficial]. The method was developed as part of the "Tenderer" system, which prioritized rendering decimated volumes where cells could be hybrid or multi-sided [S1, unofficial]. It relies on the observation that any cell type can be decomposed into bounding polygons, allowing the use of generalized software scan-conversion algorithms [S1, unofficial]. The approach was cited in works such as [WVGTG96] and discussed in the context of rapid decimation for direct volume rendering [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19990088074: Rapid Decimation for Direct Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990088074, fetched 2026-10-26 (unofficial)
[S2] edgorman/Voxel-Engine, https://github.com/edgorman/Voxel-Engine, fetched 2026-10-27 (unofficial)
