# Simplified Dual Contouring

Also called: Dual Contouring (Simplified/Naive), naive dual contouring

## Overview
Simplified Dual Contouring is an isosurface extraction method that generates a quad mesh by placing a single vertex at the center of every voxel cell where the scalar field values at the eight corners exhibit a sign change along any axis-aligned edge [S1, unofficial]. It differs from standard Dual Contouring by bypassing the Quadratic Error Function (QEF) minimization and normal-based vertex optimization, instead using a fixed geometric placement and direct quad construction based on grid topology [S1, unofficial]. This approach trades topological accuracy and geometric fidelity for implementation simplicity and computational speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes an input implementing the `DCInput` trait, which provides scalar values and gradients at integer grid coordinates, and returns a `DCOutput` struct containing vertex positions, normals, and triangle indices.

**Data Structures:**
*   `DCBounds`: A struct defining the processing volume with fields `x`, `y`, and `z`, each being a tuple `(i32, i32)` representing inclusive start and exclusive end bounds.
*   `DCInput`: A trait requiring methods `get_bounds() -> DCBounds`, `get_value(x, y, z) -> f32`, and `get_gradient(x, y, z) -> [f32; 3]`.
*   `DCOutput`: A struct with fields `vertices: Vec<[f32; 3]>`, `normals: Vec<[f32; 3]>`, and `indices: Vec<u32>`.
*   `vertex_map`: A `HashMap<IVec3, u32>` mapping grid integer positions to indices in the `vertex_buffer`.
*   `vertex_buffer`: A `Vec<[f32; 3]>` storing vertex positions.
*   `normal_buffer`: A `Vec<[f32; 3]>` storing vertex normals.
*   `index_list`: A `Vec<u32>` storing triangle indices.

**Algorithm Steps:**
1.  Initialize `vertex_buffer`, `normal_buffer`, `index_list` as empty vectors, and `vertex_map` as an empty hash map.
2.  Define the fixed plane basis vectors: `planes = [(IVec3::X, IVec3::Y), (IVec3::Z, IVec3::X), (IVec3::Y, IVec3::Z)]`. The order is critical; changing it causes mesh faces to invert [S1, unofficial].
3.  Iterate over the grid cells defined by `bounds`. For each cell `(x, y, z)` where `x` ranges from `bounds.x.0` to `bounds.x.1 - 2`, `y` from `bounds.y.0` to `bounds.y.1 - 2`, and `z` from `bounds.z.0` to `bounds.z.1 - 2`:
    a.  Call `dc_place_vertex(input, x, y, z)`.
    b.  If the result is `Some(vertex)`:
        i.  Let `position = IVec3::new(x, y, z)`.
        ii. Insert `position` into `vertex_map` with value `vertex_buffer.len() as u32`.
        iii. Append `vertex` to `vertex_buffer`.
        iv. Append `input.get_gradient(x, y, z)` to `normal_buffer`.
        v.  For each `(basis_1, basis_2)` in `planes`:
            1.  Call `check_quad(input, vertex_map, position, basis_1, basis_2, &mut index_list)`.

**Subroutines:**

*   `dc_place_vertex(input, x, y, z)`:
    1.  Retrieve the 8 corner scalar values of the voxel:
        *   `points[0] = input.get_value(x, y, z)`
        *   `points[1] = input.get_value(x + 1, y, z)`
        *   `points[2] = input.get_value(x + 1, y + 1, z)`
        *   `points[3] = input.get_value(x, y + 1, z)`
        *   `points[4] = input.get_value(x, y, z + 1)`
        *   `points[5] = input.get_value(x + 1, y, z + 1)`
        *   `points[6] = input.get_value(x + 1, y + 1, z + 1)`
        *   `points[7] = input.get_value(x, y + 1, z + 1)`
    2.  Iterate `index` from 0 to 6 (inclusive).
    3.  If `sign_change(points[index], points[index + 1])` is true, return `Some([x as f32 + 0.5, y as f32 + 0.5, z as f32 + 0.5])`.
    4.  If no sign change is found along any edge, return `None`.

*   `sign_change(a, b)`:
    Returns true if `(a.max(b) > 0. && a.min(b) < 0.) || (a == 0. && b != 0.) || (a != 0. && b == 0.)` [S1, unofficial].

*   `check_quad(input, vertex_map, position, basis_1, basis_2, index_list)`:
    1.  Compute neighbor positions:
        *   `top_left = position - basis_1`
        *   `top_right = position`
        *   `bottom_left = position - basis_1 - basis_2`
        *   `bottom_right = position - basis_2`
    2.  Check boundary conditions: If `bottom_left.x > bounds.x.0 && bottom_left.y > bounds.y.0 && bottom_left.z > bounds.z.0`:
    3.  Check existence: If `vertex_map` contains keys `top_left`, `bottom_left`, and `bottom_right`:
    4.  Determine orientation: Let `cw = input.get_value(position.x, position.y, position.z) > 0`.
    5.  Call `create_quad` with the indices retrieved from `vertex_map` for the four positions and the `cw` flag.

*   `create_quad(indices, cw)`:
    (Implementation detail not fully specified in source, but implied to generate two triangles from the four quad vertices. The `cw` flag likely determines the winding order or diagonal split, though the source notes do not explicitly define the triangle indexing logic beyond the quad creation call [S1, unofficial].)

**Parameters:**
*   Grid bounds: Defined by `DCBounds`.
*   Plane basis order: Fixed as `[(X, Y), (Z, X), (Y, Z)]` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Face Inversion:** The ordering of `planes` is fixed to ensure outward-facing quads. Changing this order causes mesh faces to invert [S1, unofficial].
*   **Zero Values:** The `sign_change` function explicitly handles cases where one corner is exactly zero, treating it as a sign change if the other corner is non-zero [S1, unofficial].
*   **Boundary Checks:** `check_quad` requires `bottom_left` to be strictly greater than the lower bounds to ensure valid neighbor lookups [S1, unofficial].

## Subtopics

### [Dual Contouring with Adaptive Split Selection](3667-dual-contouring-with-adaptive-split-selection.md)
A variant that constructs quads from connected voxels and splits them into triangles based on geometric alignment or weight products. Choose when adaptive splitting is needed for better mesh quality.

### [Gradient-Consistent Quadrilateral Splitting](11144-gradient-consistent-quadrilateral-splitting.md)
A post-processing method for converting the quadrilateral meshes produced by Simplified Dual Contouring into triangle meshes. It resolves quad triangulation ambiguity by selecting diagonal splits based on gradient consistency. Choose when high-quality triangle meshes are required from the initial quad output.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the bounds, as each voxel is processed once for vertex placement and a constant number of quad checks.
*   **Memory Complexity:** Linear with respect to the number of extracted vertices and quads. The `vertex_map` requires memory proportional to the number of active vertices.
*   **Performance:** Fast due to the absence of iterative QEF solving. However, the resulting mesh may have topological errors or geometric inaccuracies compared to full Dual Contouring [S1, unofficial].
*   **Strengths:** Simple implementation, no need for complex solvers, fast execution.
*   **Limitations:** Potential for topological inaccuracies, fixed vertex placement may not capture fine geometric details as well as QEF-based methods, quad-based output requires post-processing for triangle meshes [S1, unofficial].
*   **When to use:** When speed is critical and approximate geometry is acceptable, or as a baseline for comparison.
*   **When not to use:** When high geometric fidelity or strict topological correctness is required.

## Background
Simplified Dual Contouring is an informal implementation attempt described in an unofficial GitHub repository [S1, unofficial]. It draws inspiration from general dual-contouring tutorials and voxel-based surface extraction techniques [S1, unofficial]. The method is presented as an "unfinished" general-purpose approach, highlighting its experimental nature [S1, unofficial].

## Sources
[S1] dual_contouring, https://github.com/soundeffects/dual_contouring, fetched 2026-10-26 (unofficial)
