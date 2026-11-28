# Nested Clip-Boxes
Also called: Clip-Box, Nested Clip-Box Algorithm

## Overview
Nested Clip-Boxes is a level-of-detail (LOD) rendering technique for massive procedural volumetric terrains that caches polygonal mesh geometry in a set of nested cubic volumes (Clip-Boxes) centered on the viewpoint [S1, unofficial]. It solves the problem of real-time visualization of arbitrary-sized 3D terrain functions by converting sampled volume data into triangle meshes via iso-surface extraction, while maintaining high frame rates through incremental updates and seamless LOD transitions [S1, unofficial]. Unlike its sibling [Geometry Clipmap for LOD Rendering](9126-geometry-clipmap-for-lod-rendering.md), which operates on 2D heightmaps with regular grids, Nested Clip-Boxes handles complex, rapidly changing 3D mesh topologies derived from volumetric data [S1, unofficial].

## How it works
**Inputs and Outputs**
The method takes a user-defined 3D terrain function $f(x, y, z)$ as input [S1, unofficial]. It outputs a hardware-accelerated triangle mesh representing the terrain iso-surface, rendered with LOD [S1, unofficial].

**Data Structures**
Each Clip-Box (CB) maintains four structures [S1, unofficial]. The volume data is an 8-bit voxel grid where each voxel is either set (opaque) or unset (transparent) [S1, unofficial]. The polygon data consists of triangle strips containing vertices with $(x, y, z)$ coordinates and normal vectors [S1, unofficial]. Adjacency information stores pointers or indices linking voxels to surfaces and surfaces to surfaces to speed up extraction and connection [S1, unofficial]. A cache stores previously computed volume data to enable differential updates [S1, unofficial].

**Parameters**
The voxel count $N_v$ remains constant for all CBs, typically $100^3$ voxels per CB [S1, unofficial]. The scale factor $S$ increases exponentially by powers of two, so if CB $i$ has size $L_i$, then $L_{i+1} = 2 \cdot L_i$ [S1, unofficial]. The voxel size $v_i$ doubles with each outer level, calculated as $v_i = L_i / N_v^{1/3}$, with examples of CB 1 at size 1 and CB 2 at size 2 [S1, unofficial]. An implicit iso-surface threshold is used during the conversion from volume data to triangles, such as in Marching Cubes [S1, unofficial].

**Algorithm Steps**
1. Initialization defines the initial set of nested Clip-Boxes centered on the viewpoint, with the innermost CB being the smallest and outer CBs larger by powers of two [S1, unofficial].
2. Volume sampling evaluates the terrain function $f(x, y, z)$ within the cubic region of each CB and stores the result as 8-bit volume data [S1, unofficial].
3. Iso-surface extraction converts the volume data of each CB into a triangle mesh using a conventional algorithm such as Marching Cubes [S1, unofficial]. This step generates immense triangle counts, necessitating the LOD strategy [S1, unofficial].
4. Nesting and LOD culling omits geometry for each CB that would interfere with the next inner CB, reducing the polygon count for distant layers [S1, unofficial].
5. Seamless connection links bordering vertices of the current CB to the next outer CB to prevent gaps, achieved using adjacency information and pointers [S1, unofficial].
6. The viewpoint update strategy verifies CB positions to preserve the concentric LOD structure when the viewpoint moves [S1, unofficial]. Inner CBs are updated frequently, while outer CBs are updated rarely [S1, unofficial]. If the viewpoint moves slightly, only the inner CB is updated while outer CBs remain stationary, covering any gaps arising from the inner CB's movement [S1, unofficial].
7. Differential updates cache previously computed volume data and only compute differential updates for the portions of the CB that have changed relative to the cache [S1, unofficial].
8. Post-processing applies Laplacian smoothing to the generated mesh to reduce block-like artifacts from the initial volume-to-polygon conversion [S1, unofficial].
9. Dynamic triangle adjustment optionally skips the innermost CBs to dynamically reduce the number of triangles on screen [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ClipBox:
    def __init__(self, level, voxel_count, terrain_func, cache):
        self.level = level
        self.voxel_count = voxel_count
        self.terrain_func = terrain_func
        self.cache = cache
        self.voxel_size = 2 ** level
        self.center = (0.0, 0.0, 0.0)
        self.volume_data = None
        self.mesh = None
        self.adjacency = None

    def update(self, viewpoint, outer_cb):
        half_size = self.voxel_size * self.voxel_count / 2.0
        region_min = (self.center[0] - half_size, self.center[1] - half_size, self.center[2] - half_size)
        region_max = (self.center[0] + half_size, self.center[1] + half_size, self.center[2] + half_size)
        new_volume = self._sample_terrain(region_min, region_max)
        raw_mesh = self._extract_iso_surface(new_volume)
        if outer_cb is not None:
            raw_mesh = self._cull_inner_geometry(raw_mesh, outer_cb)
        if outer_cb is not None:
            raw_mesh = self._connect_to_outer(raw_mesh, outer_cb)
        self.mesh = self._smooth_mesh(raw_mesh)
        self.cache.update(new_volume)

    def _sample_terrain(self, min_coords, max_coords):
        volume = []
        for x in range(self.voxel_count):
            row = []
            for y in range(self.voxel_count):
                col = []
                for z in range(self.voxel_count):
                    world_x = min_coords[0] + x * self.voxel_size
                    world_y = min_coords[1] + y * self.voxel_size
                    world_z = min_coords[2] + z * self.voxel_size
                    val = self.terrain_func(world_x, world_y, world_z)
                    col.append(1 if val > 0.5 else 0)
                row.append(col)
            volume.append(row)
        return volume

    def _extract_iso_surface(self, volume):
        triangles = []
        for x in range(self.voxel_count - 1):
            for y in range(self.voxel_count - 1):
                for z in range(self.voxel_count - 1):
                    cell = [volume[x][y][z], volume[x+1][y][z], volume[x+1][y+1][z], volume[x][y+1][z],
                            volume[x][y][z+1], volume[x+1][y][z+1], volume[x+1][y+1][z+1], volume[x][y+1][z+1]]
                    triangles.extend(self._marching_cubes_cell(cell, x, y, z))
        return triangles

    def _marching_cubes_cell(self, cell, x, y, z):
        indices = [i for i, v in enumerate(cell) if v == 1]
        if len(indices) == 0 or len(indices) == 8:
            return []
        return [(x, y, z, 0.0, 0.0, 1.0)]

    def _cull_inner_geometry(self, mesh, inner_cb):
        return [tri for tri in mesh if not self._intersects_volume(tri, inner_cb.center, inner_cb.voxel_size * inner_cb.voxel_count / 2.0)]

    def _intersects_volume(self, tri, center, radius):
        return True

    def _connect_to_outer(self, mesh, outer_cb):
        return mesh

    def _smooth_mesh(self, mesh):
        return mesh

class NestedClipBoxes:
    def __init__(self, terrain_func, num_levels):
        self.terrain_func = terrain_func
        self.cbs = []
        self.cache = {}
        for i in range(num_levels):
            cb = ClipBox(level=i, voxel_count=100**3, terrain_func=terrain_func, cache=self.cache)
            self.cbs.append(cb)
        self.cbs.reverse()

    def render(self, viewpoint):
        for i, cb in enumerate(self.cbs):
            outer_cb = self.cbs[i+1] if i+1 < len(self.cbs) else None
            cb.center = viewpoint
            cb.update(viewpoint, outer_cb)
        for cb in self.cbs:
            if cb.mesh:
                draw_triangles(cb.mesh)
```

**Edge Cases and Pitfalls**
LOD gaps occur if boundaries of nested CBs are not well connected, which is handled by explicitly connecting bordering vertices to the next outer CB [S1, unofficial]. Complexity varies strongly as CBs shift through volume data, requiring robust adjacency management [S1, unofficial]. Average memory consumption is higher than conventional heightmap-based methods due to volumetric data storage [S1, unofficial].

## Operating characteristics
Time complexity is not explicitly given, but the method achieves 25 frames per second on average for highly detailed landscapes [S1, unofficial]. Memory complexity is higher than conventional heightmap-based methods because it stores 3D volume data rather than 2D heightmaps [S1, unofficial]. The method achieves real-time visualization for arbitrary-sized volumetric terrains without pre-computation [S1, unofficial]. Strengths include immediate visualization of arbitrary-sized terrains, high quality rendering via iso-surface extraction, and efficient LOD via nesting and differential updates [S1, unofficial]. Limitations include high memory consumption and the requirement for sufficient resolution to hide geometry updates [S1, unofficial]. It is best used for real-time visualization of procedural volumetric terrains where pre-computation is not feasible or desired [S1, unofficial]. It should not be used in memory-constrained environments where heightmap-based methods are sufficient [S1, unofficial].

## Background
Nested Clip-Boxes was proposed by Sven Forstmann and Jun Ohya in the paper "Visualizing Large Procedural Volumetric Terrains Using Nested Clip-Boxes" [S1, unofficial]. It extends the geometry clip-map approach of Losasso and Hoppe from 2D heightmaps to 3D volume data [S1, unofficial]. The method was developed to address the limitations of existing clip-map techniques, which could not handle the complex, rapidly changing mesh topologies of volumetric terrain data [S1, unofficial].

## Sources
[S1] Visualizing Large Procedural Volumetric Terrains Using Nested Clip-Boxes, https://waseda.repo.nii.ac.jp/records/19543, fetched 2026-10-26 (unofficial)
