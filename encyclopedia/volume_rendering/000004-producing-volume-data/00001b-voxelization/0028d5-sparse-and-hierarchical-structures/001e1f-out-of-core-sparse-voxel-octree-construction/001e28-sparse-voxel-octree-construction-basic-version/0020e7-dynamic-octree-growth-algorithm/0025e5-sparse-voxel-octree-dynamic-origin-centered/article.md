# Sparse Voxel Octree (Dynamic, Origin-Centered)

Also called: bit-svo

## Overview
A sparse voxel octree implementation where voxel widths are powers of 2, positions are integer multiples of their widths (ensuring alignment to widths), the tree is always centered at (0,0,0), and the tree expands dynamically on insert [S1, unofficial]. It differs from static SVOs by growing its depth and spatial extent only when necessary, and from general dynamic octrees by enforcing a strict origin-centered coordinate system and power-of-two voxel dimensions [S1, unofficial].

## How it works
**Inputs:** A voxel defined by integer coordinates $(x, y, z)$ and a logarithmic size $lg\_size$ (where width $w = 2^{lg\_size}$) [S1, unofficial]. The tree stores values of type $T$ [S1, unofficial].
**Outputs:** A reference to the `TreeBody` node corresponding to the voxel, or `None` if the voxel is outside the current bounds [S1, unofficial].

**Data Structures:**
*   `VoxelTree<T>`:
    *   `lg_size: u8`: The log_2 of the total tree size (width) [S1, unofficial].
    *   `contents: Branches<T>`: The root node, always a `Branches` struct [S1, unofficial].
*   `Branches<T>`:
    *   A struct with 8 fields of type `TreeBody<T>`, named `lll`, `llh`, `lhl`, `lhh`, `hll`, `hlh`, `hhl`, `hhh` in xyz ordering [S1, unofficial].
    *   Memory layout is `#[repr(C)]`, allowing direct indexing via coordinate bits [S1, unofficial].
*   `TreeBody<T>`:
    *   `Empty`: No data [S1, unofficial].
    *   `Leaf(T)`: Contains the value $T$ [S1, unofficial].
    *   `Branch(Box<Branches<T>>)`: Points to a child `Branches` node [S1, unofficial].
*   `VoxelBounds`:
    *   `x: i32`, `y: i32`, `z: i32`: Integer coordinates [S1, unofficial].
    *   `lg_size: i32`: Logarithmic size [S1, unofficial].

**Algorithm Steps:**

1.  **Bounds Check (`contains_bounds`)**:
    Determine if a voxel fits within the current tree bounds [S1, unofficial].
    *   If `voxel.lg_size < 0`, return `true` [S1, unofficial].
    *   Calculate `high = (1 << self.lg_size) >> voxel.lg_size` [S1, unofficial].
    *   Calculate `low = -high` [S1, unofficial].
    *   If `voxel.x < low` or `voxel.y < low` or `voxel.z < low`, return `false` [S1, unofficial].
    *   If `(voxel.x + 1) <= high` and `(voxel.y + 1) <= high` and `(voxel.z + 1) <= high`, return `true` [S1, unofficial].
    *   Otherwise, return `false` [S1, unofficial].

2.  **Mask Calculation (`find_mask`)**:
    Determine the traversal mask for the voxel's size relative to the tree [S1, unofficial].
    *   Initialize `mask = (1 << self.lg_size) >> 1` [S1, unofficial].
    *   If `voxel.lg_size >= 0`, shift `mask` right by `voxel.lg_size`: `mask = mask >> voxel.lg_size` [S1, unofficial].
    *   If `voxel.lg_size < 0`, shift `mask` left by `-voxel.lg_size`: `mask = mask << (-voxel.lg_size)` [S1, unofficial].

3.  **Traversal (`mut_find`)**:
    Traverse or create nodes to reach the target voxel [S1, unofficial].
    *   Initialize `mask` using `find_mask` and `branches` pointer to `&self.contents` [S1, unofficial].
    *   **First Iteration**: Use condition `|x| x >= 0` to select the branch, determining the most significant bit (sign) of the coordinates [S1, unofficial].
    *   If `mask == 0`, return the current branch [S1, unofficial].
    *   Otherwise, call `step(branch)` to get the next `Branches` node [S1, unofficial].
    *   **Subsequent Iterations**: Loop while `mask > 0` [S1, unofficial].
    *   Use condition `|x| (x & mask) != 0` to select the branch, determining the remaining bits of the coordinates [S1, unofficial].
    *   Update `mask = mask >> 1` [S1, unofficial].
    *   Step into the child branch [S1, unofficial].

4.  **Branch Selection (`get_branch`)**:
    Map coordinates $(x, y, z)$ to one of the 8 children using a `choose_branch` function [S1, unofficial].
    *   The bits determine the index: $x$ bit selects `l` (low) or `h` (high), $y$ bit selects `l` or `h`, $z$ bit selects `l` or `h` [S1, unofficial].
    *   Example: `lll` corresponds to `(false, false, false)`, `hhh` to `(true, true, true)` [S1, unofficial].

5.  **Retrieval (`get`)**:
    *   If `!contains_bounds(voxel)`, return `None` [S1, unofficial].
    *   Define `get_step` closure: if `branch` is `Branch(ref branches)`, return `Ok(branches.deref())`, else `Err(())` [S1, unofficial].
    *   Call `self.find(voxel, get_step)` [S1, unofficial].
    *   If result is `Ok(&TreeBody::Leaf(ref t))`, return `Some(t)`, else `None` [S1, unofficial].

6.  **Ray Casting (`cast_ray`)**:
    *   Calculate `coords` array where each element is 1 if `origin[i] >= 0.0` else 0 [S1, unofficial].
    *   Set `child_lg_size = self.lg_size as i16` [S1, unofficial].
    *   Define `make_bounds` closure: returns `VoxelBounds` with `x = coords[0] - 1`, `y = coords[1] - 1`, `z = coords[2] - 1`, `lg_size = child_lg_size` [S1, unofficial].
    *   Call `raycast::cast_ray_branches` with `&self.contents`, `origin`, `direction`, `None`, `coords`, `&mut make_bounds`, and the action function `act` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Branches:
    def __init__(self):
        self.lll = TreeBody('Empty')
        self.llh = TreeBody('Empty')
        self.lhl = TreeBody('Empty')
        self.lhh = TreeBody('Empty')
        self.hll = TreeBody('Empty')
        self.hlh = TreeBody('Empty')
        self.hhl = TreeBody('Empty')
        self.hhh = TreeBody('Empty')
    @staticmethod
    def empty():
        return Branches()

class TreeBody:
    def __init__(self, kind, value=None, children=None):
        self.kind = kind  # 'Empty', 'Leaf', 'Branch'
        self.value = value
        self.children = children  # Branches instance

class VoxelTree:
    def __init__(self):
        self.lg_size = 0
        self.contents = Branches.empty()

    def contains_bounds(self, voxel):
        if voxel.lg_size < 0:
            return True
        high = (1 << self.lg_size) >> voxel.lg_size
        low = -high
        if voxel.x < low or voxel.y < low or voxel.z < low:
            return False
        if (voxel.x + 1) <= high and (voxel.y + 1) <= high and (voxel.z + 1) <= high:
            return True
        return False

    def find_mask(self, voxel):
        mask = (1 << self.lg_size) >> 1
        if voxel.lg_size >= 0:
            mask = mask >> voxel.lg_size
        else:
            mask = mask << (-voxel.lg_size)
        return mask

    def get_branch(self, branches, choose_func, x, y, z):
        x_bit = choose_func(x)
        y_bit = choose_func(y)
        z_bit = choose_func(z)
        if not x_bit and not y_bit and not z_bit: return branches.lll
        if not x_bit and not y_bit and z_bit: return branches.llh
        if not x_bit and y_bit and not z_bit: return branches.lhl
        if not x_bit and y_bit and z_bit: return branches.lhh
        if x_bit and not y_bit and not z_bit: return branches.hll
        if x_bit and not y_bit and z_bit: return branches.hlh
        if x_bit and y_bit and not z_bit: return branches.hhl
        return branches.hhh

    def mut_find(self, voxel, step):
        mask = self.find_mask(voxel)
        branches = self.contents
        # First iteration: sign bit
        branch = self.get_branch(branches, lambda x: x >= 0, voxel.x, voxel.y, voxel.z)
        if mask == 0:
            return branch
        branches = step(branch)
        while mask > 0:
            branch = self.get_branch(branches, lambda x: (x & mask) != 0, voxel.x, voxel.y, voxel.z)
            if mask == 0:
                return branch
            branches = step(branch)
            mask >>= 1
        return branch

    def get_mut_or_create_step(self, branch):
        if branch.kind == 'Empty':
            branch.kind = 'Branch'
            branch.children = Branches.empty()
        elif branch.kind == 'Leaf':
            branch.kind = 'Branch'
            branch.children = Branches.empty()
        return branch.children

    def get(self, voxel):
        if not self.contains_bounds(voxel):
            return None
        def get_step(branch):
            if branch.kind == 'Branch':
                return branch.children
            raise StopIteration
        try:
            node = self.mut_find(voxel, get_step)
            if node.kind == 'Leaf':
                return node.value
        except StopIteration:
            pass
        return None
```

**Pitfalls:**
*   Negative voxel `lg_size`: `contains_bounds` returns `true` immediately if `voxel.lg_size < 0`, treating it as always within bounds [S1, unofficial].
*   Leaf replacement: when inserting into a `Leaf` node during `get_mut_or_create_step`, the leaf is erased and replaced with 8 empty sub-branches, which is noted as debatable but done for easier debugging of accidental replacements [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Insertion is $O(D)$ where $D$ is the depth of the tree [S1, unofficial]. Benchmarks show approximately 1 microsecond per insert through 30 levels of depth (world width of $2^{30}$ voxels) [S1, unofficial].
*   **Memory Complexity**: Proportional to the number of non-empty nodes, with each branch node consuming memory for 8 child pointers [S1, unofficial].
*   **Strengths**: Dynamic growth avoids pre-allocating memory for empty space, and the origin-centered design simplifies coordinate arithmetic for symmetric scenes [S1, unofficial].
*   **Limitations**: The strict power-of-two width and alignment requirements limit flexibility, and leaf replacement on insertion may cause unnecessary memory churn if leaves are frequently replaced by branches [S1, unofficial].

## Background
This implementation is part of the `bit-svo` project by `bfops` [S1, unofficial]. It is designed for efficient sparse voxel storage with a focus on performance and simplicity, leveraging bit manipulation for branch selection [S1, unofficial].

## Sources
[S1] bfops/bit-svo, https://github.com/bfops/bit-svo, fetched 2026-10-26 (unofficial)
