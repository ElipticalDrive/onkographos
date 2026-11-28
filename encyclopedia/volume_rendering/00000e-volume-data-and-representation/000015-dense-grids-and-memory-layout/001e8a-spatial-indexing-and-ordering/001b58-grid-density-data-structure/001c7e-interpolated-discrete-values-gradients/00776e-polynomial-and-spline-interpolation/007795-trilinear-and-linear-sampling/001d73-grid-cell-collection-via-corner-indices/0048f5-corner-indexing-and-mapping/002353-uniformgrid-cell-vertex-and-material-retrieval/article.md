# UniformGrid Cell Vertex and Material Retrieval


## Overview
This method retrieves the linear indices and associated material data for the 8 vertices of a grid cell within a uniform 3D grid, given the linear index of the cell itself [S1, unofficial]. It provides the explicit arithmetic formula for linear index calculation in a uniform grid, mapping a single cell identifier to its constituent geometric corners [S1, unofficial].

## How it works
**Inputs:**
- `cellIdx`: The linear index of the target grid cell (integer) [S1, unofficial].
- `m_size`: The number of cells along one axis of the uniform grid (integer) [S1, unofficial].

**Outputs:**
- `v`: An array of 8 integers representing the linear indices of the 8 vertices of the cell [S1, unofficial].
- `materials`: An array of 8 material identifiers corresponding to the vertices [S1, unofficial].

**Data Structures:**
- The grid is stored in a flat, contiguous memory array (linearized) [S1, unofficial].

**Formulas:**
The linear index calculation relies on three stride constants derived from `m_size`:
$$
\begin{aligned}
dx &= m\_size + 1 \\
dy &= (m\_size + 1) \times (m\_size + 1) \\
dz &= 1
\end{aligned}
$$
The 8 vertex linear indices `v[0]` through `v[7]` are computed as:
$$
\begin{aligned}
v[0] &= cellIdx \\
v[1] &= cellIdx + dz \\
v[2] &= cellIdx + dx \\
v[3] &= cellIdx + dx + dz \\
v[4] &= cellIdx + dy \\
v[5] &= cellIdx + dy + dz \\
v[6] &= cellIdx + dx + dy \\
v[7] &= cellIdx + dx + dy + dz
\end{aligned}
$$
Material retrieval follows the exact same indexing logic [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def retrieve_cell_vertices_and_materials(cellIdx, m_size, material_array):
    dx = m_size + 1
    dy = (m_size + 1) * (m_size + 1)
    dz = 1
    
    v = [0] * 8
    v[0] = cellIdx
    v[1] = cellIdx + dz
    v[2] = cellIdx + dx
    v[3] = cellIdx + dx + dz
    v[4] = cellIdx + dy
    v[5] = cellIdx + dy + dz
    v[6] = cellIdx + dx + dy
    v[7] = cellIdx + dx + dy + dz
    
    materials = [0] * 8
    for i in range(8):
        materials[i] = material_array[v[i]]
        
    return v, materials
```

**Edge Cases and Pitfalls:**
- Boundary conditions: Not given in the sources.
- Grid dimensions: The formula assumes a cubic grid where the number of cells is the same in all dimensions (`m_size`) [S1, unofficial].
- Precision: Linear indices are integers [S1, unofficial].

## Subtopics

### [Cube Edge Vertex Retrieval](3217-cube-edge-vertex-retrieval.md)
Retrieves all 12 edge vertex pairs for a specific cube in the voxel grid. It differs from its parent [UniformGrid Cell Vertex and Material Retrieval](2999-uniformgrid-cell-vertex-and-material-retrieval.md) by focusing specifically on the 12 edges rather than the 8 corner vertices.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
