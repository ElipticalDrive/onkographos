# Manifold Voxel Repair

Also called: Manifold Voxelization Repair

## Overview
Manifold Voxel Repair is a post-processing step for voxelized meshes that resolves non-manifold contacts by filling empty cells between touching voxels, ensuring the resulting structure is watertight [S1, unofficial]. It specifically targets edge-only and corner-only contacts where voxels share an edge or vertex but no face, which are invalid for manifold mesh generation [S1, unofficial]. Unlike sibling methods that focus on generation or carving, this method operates on an existing voxel set to enforce topological validity, growing the set only to fill gaps [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of occupied voxel coordinates (or keys) representing a voxelized object.
- Output: A modified set of voxel coordinates where all non-manifold contacts are resolved.

**Data Structures:**
- `keys`: A list or array of voxel coordinates (e.g., `(x, y, z)` tuples or integer indices).
- `present(offset)`: A helper function that returns a boolean mask indicating which voxels in the current set have a neighbor at the specified relative offset.
- `_CONTACTS`: A predefined list of tuples `(delta, first, between)` defining the geometric patterns of non-manifold contacts. `delta` is the offset to check for existing voxels, `first` is the offset of the voxel to add, and `between` is a list of offsets representing the gap cells that must be empty for the contact to be considered non-manifold.

**Parameters:**
- `max_iter`: Integer, optional safety cap on repair rounds. Default is 8 [S1, unofficial]. Iteration is required because sealing one contact can create another [S1, unofficial].

**Algorithm Steps:**
1.  **Margin Expansion:** Convert input voxels to keys with a 1-voxel margin to account for neighbors. This ensures that contacts involving voxels adjacent to the input set are detected.
    $$ \text{keys}_{\text{expanded}} = \text{to\_keys}(\text{voxels}, \text{margin}=1) $$
2.  **Empty Check:** If the expanded keys are empty, return an empty result.
    ```python
    if len(keys) == 0:
        return from_keys(keys, shift, voxels.dtype)
    ```
3.  **Initialization:** Initialize `rounds = 0` and `add = _contact_fixes(keys)` if `max_iter` is non-zero, else `add` is empty [S1, unofficial].
4.  **Iterative Repair:**
    - While `add` is not empty and `rounds < max_iter`:
        - Update `keys` by adding `add`.
        - Recompute `add = _contact_fixes(keys)`.
        - Increment `rounds`.
    - If contacts remain after the loop, emit a warning [S1, unofficial].
5.  **Return:** Convert final keys back to voxels.

**Helper Function `_contact_fixes(keys)`:**
1.  Initialize `add` as an empty list.
2.  Iterate through each tuple `(delta, first, between)` in `_CONTACTS`:
    - Select voxels at `delta`: `sel = present(delta)`.
    - For each gap in `between`:
        - If `sel` is not empty, update `sel` to exclude voxels where the gap is present:
          $$ \text{sel} = \text{sel} \ \& \ \sim \text{present}(\text{gap}) $$
    - If `sel` has any elements, add `keys[sel] + first` to the `add` list.
3.  Return unique concatenated additions, or empty array if none.

**Edge Cases and Pitfalls:**
- **Topology Change:** Filling a cell between offending voxels changes topology: it can close a tunnel or merge two components that previously met at a single edge [S1, unofficial]. The set only grows, never shrinking [S1, unofficial].
- **Corner to Edge Resolution:** Corner contacts resolve to edge contacts first, requiring iteration to reach a fixed point [S1, unofficial]. If `max_iter` is hit with outstanding contacts, the result may not be manifold [S1, unofficial].
- **Precision:** The method relies on discrete grid offsets, so precision issues are minimal, but the margin expansion must be handled correctly to avoid boundary errors [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method optimizes ~46 sorted lookups into 17 by memoizing membership tests for distinct offsets [S1, unofficial]. Each iteration involves checking contacts and updating the voxel set. The number of iterations is bounded by `max_iter` (default 8) [S1, unofficial].
- **Memory Complexity:** Proportional to the number of voxels and the margin expansion. The memoization of membership tests reduces overhead [S1, unofficial].
- **Strengths:** Ensures manifold validity for voxelized meshes; simple and effective for resolving common non-manifold artifacts; iterative approach handles cascading contacts [S1, unofficial].
- **Limitations:** Can alter topology (e.g., closing tunnels); may not fully resolve all non-manifold contacts if `max_iter` is insufficient; only adds voxels, never removes them [S1, unofficial].
- **When to Use:** After voxelization when a manifold mesh is required for rendering or simulation; when edge-only or corner-only contacts are present [S1, unofficial].
- **When Not to Use:** When preserving the exact original topology is critical and non-manifold contacts are acceptable; when performance is critical and `max_iter` is large [S1, unofficial].

## Background
Manifold Voxel Repair is part of the `sparse-cubes` library by navis-org, which provides tools for voxelization and skeletonization of biological structures [S1, unofficial]. The method addresses the common issue of non-manifold contacts in voxelized meshes, which are problematic for downstream applications like mesh generation and simulation [S1, unofficial]. The iterative approach and contact definitions are designed to be efficient and robust for large-scale voxel data [S1, unofficial].

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
