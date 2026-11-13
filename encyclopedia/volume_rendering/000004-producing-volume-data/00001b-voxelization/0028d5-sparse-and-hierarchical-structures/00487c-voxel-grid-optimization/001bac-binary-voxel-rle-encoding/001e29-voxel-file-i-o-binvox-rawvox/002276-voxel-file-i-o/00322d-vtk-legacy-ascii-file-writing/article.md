# VTK Legacy ASCII File Writing

Also called: VTK ASCII UNSTRUCTURED_GRID Export

## Overview
Exports voxel data to a legacy VTK ASCII `UNSTRUCTURED_GRID` file format, representing each voxel as a hexahedral cell with associated scalar fields [S1, unofficial]. It differs from binary voxel formats (e.g., [Binvox File Parsing](2862-binvox-file-parsing.md)) by using human-readable text and explicit vertex definitions for every cell, prioritizing interoperability with legacy VTK-based viewers over storage efficiency [S1, unofficial].

## How it works
**Inputs:** A voxel dataset containing per-cell bounding box coordinates (`box_min`, `box_max`), cell count, and multiple scalar fields (e.g., `void_ratio`, `particle_count`) [S1, unofficial].
**Outputs:** A `.vtk` file in ASCII format [S1, unofficial].

**Data Structures:**
The output is a VTK `UNSTRUCTURED_GRID` [S1, unofficial].
- **POINTS:** A list of 3D coordinates. Total count = `cell_count * 8` [S1, unofficial].
- **CELLS:** A flat list defining connectivity. Each cell entry consists of 9 integers: `8` (number of vertices) followed by 8 vertex indices [S1, unofficial].
- **CELL_TYPES:** A list of integers. Each entry is `11` (VTK_HEXAHEDRON) [S1, unofficial].
- **CELL_DATA:** Scalar arrays associated with each cell. Fields include `void_ratio` (double), `void_ratio_smoothed` (double), `particle_count` (int), `solid_volume` (double), and `measurement_volume` (double) [S1, unofficial].

**Steps:**
1. **Header:** Write `# vtk DataFile Version 2.0`, a title, `ASCII`, and `DATASET UNSTRUCTURED_GRID` [S1, unofficial].
2. **POINTS:** Write `POINTS {point_count} float` [S1, unofficial]. Compute the 8 vertices for each cell using the cell's bounding box (`box_min`, `box_max`) and a local index $i \in [0, 7]$ [S1, unofficial]. The local index maps to the 8 corners of the hexahedron: $i=0$ is $(x_{min}, y_{min}, z_{min})$, $i=1$ is $(x_{max}, y_{min}, z_{min})$, $i=2$ is $(x_{min}, y_{max}, z_{min})$, $i=3$ is $(x_{max}, y_{max}, z_{min})$, $i=4$ is $(x_{min}, y_{min}, z_{max})$, $i=5$ is $(x_{max}, y_{min}, z_{max})$, $i=6$ is $(x_{min}, y_{max}, z_{max})$, and $i=7$ is $(x_{max}, y_{max}, z_{max})$ [S1, unofficial]. Write each coordinate formatted as `'{:.17g}'` [S1, unofficial].
3. **CELLS:** Write `CELLS {cell_count} {cell_count * 9}` [S1, unofficial]. For each cell index $c$ from $0$ to `cell_count - 1`, calculate `start = c * 8` and write `8` followed by the 8 vertex indices: `start, start+1, ..., start+7` [S1, unofficial].
4. **CELL_TYPES:** Write `CELL_TYPES {cell_count}` [S1, unofficial]. Write the integer `11` for each cell [S1, unofficial].
5. **CELL_DATA:** Write `CELL_DATA {cell_count}` [S1, unofficial]. For each scalar field, write `SCALARS {name} {type} 1`, then `LOOKUP_TABLE default`, followed by the values in cell order [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_vtk_legacy_ascii(output_path, cell_bounds, scalar_fields):
    """
    cell_bounds: list of tuples (cell_min, cell_max), each a tuple of 3 floats.
    scalar_fields: dict mapping name -> list of values (length len(cell_bounds)).
    """
    cell_count = len(cell_bounds)
    point_count = cell_count * 8

    with open(output_path, 'w') as f:
        # 1. Header
        f.write("# vtk DataFile Version 2.0\n")
        f.write("Voxel Data\n")
        f.write("ASCII\n")
        f.write("DATASET UNSTRUCTURED_GRID\n")

        # 2. POINTS
        f.write(f"POINTS {point_count} float\n")
        for cell_min, cell_max in cell_bounds:
            xmin, ymin, zmin = cell_min
            xmax, ymax, zmax = cell_max
            for local_idx in range(8):
                x = xmin if (local_idx & 1) == 0 else xmax
                y = ymin if (local_idx & 2) == 0 else ymax
                z = zmin if (local_idx & 4) == 0 else zmax
                f.write(f"{x:.17g} {y:.17g} {z:.17g}\n")

        # 3. CELLS
        f.write(f"CELLS {cell_count} {cell_count * 9}\n")
        for c in range(cell_count):
            start = c * 8
            f.write(f"8 {start} {start+1} {start+2} {start+3} {start+4} {start+5} {start+6} {start+7}\n")

        # 4. CELL_TYPES
        f.write(f"CELL_TYPES {cell_count}\n")
        for _ in range(cell_count):
            f.write("11\n")

        # 5. CELL_DATA
        f.write(f"CELL_DATA {cell_count}\n")
        for name, values in scalar_fields.items():
            vtype = "double" if isinstance(values[0], float) else "int"
            f.write(f"SCALARS {name} {vtype} 1\n")
            f.write("LOOKUP_TABLE default\n")
            for val in values:
                f.write(f"{val}\n")
```

**Parameters:**
- `point_count`: Total vertices = `cell_count * 8` [S1, unofficial].
- `cell_count`: Number of voxels [S1, unofficial].

**Edge Cases:**
- Precision: Coordinates must be formatted with `{:.17g}` to preserve double precision [S1, unofficial].
- Empty Grids: Not given in the sources.

## Subtopics

### VoxelGrid Serialization (Grasshopper)
A variant used in the Grasshopper environment that serializes VoxelGrid objects into a custom chunk-based format rather than standard VTK ASCII [S2, unofficial]. It removes the 'PersistentData' chunk, writes a 'GridCount' Int32, and iterates through persistent data branches [S2, unofficial]. For each grid, it creates a 'Grid' chunk, converts the voxel grid to bytes, compresses them, converts to a hex string, and writes it to the 'Data' field along with the GH_Path [S2, unofficial]. Reading involves reversing this process: reading the count, finding the 'Grid' chunk by index, reading the hex string, decompressing, and reconstructing the VoxelGrid [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Human-readable text format and explicit vertex definitions for every cell [S1, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The VTK Legacy ASCII format is part of the Visualization Toolkit (VTK) file formats, specifically the `UNSTRUCTURED_GRID` representation [S1, unofficial]. It is used in point-cloud voxelization workflows to export voxel data for visualization [S1, unofficial]. The format relies on the VTK Hexahedron cell type (ID 11) to represent voxels [S1, unofficial].

## Sources
[S1] SammySuliman/point-cloud-voxelization, https://github.com/SammySuliman/point-cloud-voxelization, fetched 2026-10-27 (unofficial)
[S2] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
