# Scanline and Geometric Primitives

Also called: Analytic Voxelization, Scanline Voxelization, Geometric Scan-Conversion

## Overview
This family generates volumetric data directly from analytic geometric primitives (planes, lines, polygons) using mathematical intersection tests or scanline traversal, rather than relying on rasterization or ray-marching against a mesh. It sets itself apart from rasterization-based voxelization [11836-rasterization-and-projection.md] by offering topological guarantees (such as N-separability) and avoiding aliasing through analytic bandlimiting or optimal scanline definitions, which are critical for high-fidelity volume rendering [S1, unofficial][S2, unofficial].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [3D Scan-conversion](1895-3d-scan-conversion.md)
[3D Scan-conversion](1895-3d-scan-conversion.md) converts geometric primitives (polygons) into a volumetric dataset, combines them with existing volumetric data using volume matting operators, and renders the resulting composite ensemble using standard volume rendering techniques. It sets itself apart from hybrid ray tracing by pre-filtering and sampling geometry into the voxel grid, allowing the use of standard ray-casting for the final image while avoiding aliasing artifacts through analytic anti-aliasing.

### [Voxelization of Lines and Planes](2472-voxelization-of-lines-and-planes.md)
[Voxelization of Lines and Planes](2472-voxelization-of-lines-and-planes.md) generates a discrete volumetric representation of 2D lines or 3D planes that is both **N-separating** (topologically robust against N-connected ray penetration) and **N-minimal** (containing no redundant voxels). It solves the problem of topological holes in voxelized surfaces by defining a "slab" of thickness $t$ around the geometric primitive, including only voxels whose centers fall within this slab. Unlike general rasterization-based voxelization [11836-rasterization-and-projection.md], this analytic approach guarantees specific connectivity properties (4/8 for 2D, 6/26 for 3D) based on the surface normal and grid resolution, ensuring that 26-connected paths cannot pass through the discretized surface without intersecting a voxel.

### [Distance-based Voxelization](3500-distance-based-voxelization.md)
[Distance-based Voxelization](3500-distance-based-voxelization.md) is a method to voxelize objects directly without polygon conversion by including voxels where the distance to the surface S is smaller than a threshold t.

### [Scan-line Clipping Algorithm](10945-scan-line-clipping-algorithm.md)
[Scan-line Clipping Algorithm](10945-scan-line-clipping-algorithm.md) is a 3D generalization of a scan-line algorithm adjusted for low-level processor implementation to determine which portions of a room are not obscured by higher priority rooms or figurines.

### 3D Scan-Conversion for Geometric/Volumetric Mixing
This variant focuses on the integration of bandlimited geometry into volumetric rendering pipelines. It requires bandlimiting the object to the Nyquist frequency in all three dimensions [S1, unofficial]. The object is then sampled in a manner that limits losses due to quantization [S1, unofficial]. The method leverages volume rendering's partial opacity capability to handle the resulting voxel representation, thereby avoiding aliasing artifacts common in binary voxel representations [S1, unofficial].

### Floating-Point Optimal Scanline
This algorithm, proposed by Zhang et al., uses line voxelization as its core concept to voxelize triangles [S2, unofficial]. It calculates voxel occupancy for each triangle by sorting vertices based on the most dominant axis, defined as the axis with the greatest absolute value of the normal's components: $\max(|n_x|, |n_y|, |n_z|)$ [S2, unofficial]. The algorithm performs line voxelizations between each vertex using RLV to voxelize all edges [S2, unofficial]. It then splits edges into slices in the dominant axis (e.g., z-axis) where each edge has the same integer value, and for each slice, performs 2D line voxelization between the edges of the triangle [S2, unofficial]. The "optimal" aspect relies on a distance $l = |dsl_x| + |dsl_y|$, where $dsl$ is the voxel's diagonal projected onto the scanline direction, ensuring no voxel exists between two parallel lines within this distance [S2, unofficial].

## Background
The concept of scan-conversion for volume rendering was introduced to address aliasing in binary voxel representations. Kaufman et al. demonstrated that polyhedra, parametric curves, and surfaces could be rendered without aliasing if bandlimited to the Nyquist frequency and sampled appropriately [S1, unofficial]. Later, Zhang et al. introduced the "Floating-Point Optimal Scanline" algorithm, which refined triangle voxelization by using optimal scanline distances to minimize voxel count while maintaining topological integrity [S2, unofficial]. The analytic slab method for lines and planes was developed to provide topological guarantees (N-separability) that rasterization methods lack, ensuring robust connectivity for volume rendering and simulation [S2, unofficial].

## Sources
[S1] DTIC ADA208091: Interactive Visualization of 3D Medical Data, https://archive.org/details/DTIC_ADA208091, fetched 2026-10-26 (unofficial)
[S2] PDF A Comparison of Optimal Scanline Voxelization Algorithms, https://liu.diva-portal.org/smash/get/diva2:1459330/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
