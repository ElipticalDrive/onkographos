# Voxtree

Also called: sparse voxel 64-tree

## Overview
Voxtree is a sparse voxel tree data structure implemented in Rust, designed for maximal CPU performance with a simple surface API [S1, unofficial]. It uses a 64-way branching factor (4×4×4 sub-volumes) and packs node type information into the address field of branch nodes [S1, unofficial]. It builds on the Tree64 concept but implements a specific memory layout and address-shifting mechanism for dynamic updates [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D coordinates $(x, y, z)$, traversal depth $w$, and optional data $T$.
- Output: Reference to leaf data or branch node, or `None` if the path is empty [S1, unofficial].

**Data Structures:**
- `Voxtree<T>`:
  - `depth`: `u8`, maximum depth of the tree.
  - `branches`: `Vec<Branch>`, list of branch nodes.
  - `leaves`: `Vec<T>`, list of leaf data.
  - `features`: `Features`, bitflag enum (`None`=0b0, `DirectedAcyclicGraph`=0b1) [S1, unofficial].
- `Branch` (`repr(C, packed)`):
  - `bitmask`: `u64`, 64 bits indicating presence of children (4×4×4) [S1, unofficial].
  - `address`: `u32`, packed with flags and index [S1, unofficial].
    - Bit 31: Leaf flag (1=leaf, 0=branch) [S1, unofficial].
    - Bits 30–29: Solid flag (0b11=solid, others=not solid) [S1, unofficial].
    - Bits 28–0: Index into `branches` or `leaves` vector [S1, unofficial].

**Numbered Steps:**

1. **Initialization (`Voxtree::empty`)**:
   - Create root branch with `bitmask = 0` and `address = 1` [S1, unofficial].
   - Set `depth` to the provided scale [S1, unofficial].
   - Initialize `branches` with the root, `leaves` empty [S1, unofficial].

2. **Lookup (`get`)**:
   - Iterate `depth` from $w-1$ down to 0 [S1, unofficial].
   - Calculate `index` using `get_index(x, y, z, depth)` [S1, unofficial].
   - Check if bit at `index` in `bitmask` is set; if not, return `None` [S1, unofficial].
   - Calculate `transformed_index` using `get_transformed_index(bitmask, index)` [S1, unofficial].
   - If node is leaf: return `leaves[address]` if solid, else `leaves[address + transformed_index]` [S1, unofficial].
   - If node is branch, update `branch_address = address + transformed_index` and continue [S1, unofficial].
   - Return `branches[branch_address]` [S1, unofficial].

3. **Index Calculation (`get_index`)**:
   - Extract bits from $x, y, z$ shifted by `double_depth` ($depth \ll 1$) [S1, unofficial].
   - `index = (z_bit << 4) | (y_bit << 2) | x_bit` [S1, unofficial].

4. **Transformed Index Calculation (`get_transformed_index`)**:
   - Count set bits (ones) in `bitmask` up to `index` [S1, unofficial].
   - Maps logical index to compact physical index, conserving sparsity [S1, unofficial].

5. **Insertion (`set`)**:
   - Traverse from root to leaf level $w-1$ [S1, unofficial].
   - Calculate `index` and `transformed_index` at each level [S1, unofficial].
   - Update `previous_address`, `address`, and `index_store` [S1, unofficial].
   - Handle insertion based on node type (leaf/branch) and solid status [S1, unofficial].
   - Update `bitmask` to include new child [S1, unofficial].
   - Handle leaf expansion if parent was leaf [S1, unofficial].

6. **Leaf to Branch Conversion**:
   - If inserting branch into leaf parent, expand leaf into full branch [S1, unofficial].
   - Create new branch nodes for all existing leaves, setting `bitmask = u64::MAX` and `address` to original leaves [S1, unofficial].
   - Update parent's `address` and `bitmask` [S1, unofficial].

7. **Deletion (`set` with `None`)**:
   - Clear bit in parent's `bitmask` [S1, unofficial].
   - If parent is non-solid leaf, remove leaf and shift addresses [S1, unofficial].
   - Else, recursively delete subtree, shift addresses, and remove branch node [S1, unofficial].

8. **Recursive Deletion**:
   - For branch, recursively delete all children (counted by `bitmask` ones or 1 if solid) [S1, unofficial].
   - Drain children from `branches` vector [S1, unofficial].
   - Mark branch as empty (`bitmask = 0`) and shift addresses [S1, unofficial].

9. **Address Shifting**:
   - Update branch addresses in `branches` vector [S1, unofficial].
   - If range removed, addresses $\ge$ `range.end` decreased by `range.len()` [S1, unofficial].
   - If range inserted, addresses $\ge$ `range.end` increased by `range.len()` [S1, unofficial].
   - Only branches with different leaf status than the operation are updated [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Branch:
    def __init__(self, bitmask: int, address: int):
        self.bitmask = bitmask
        self.address = address
    def is_leaf(self) -> bool:
        return (self.address >> 31) & 1
    def is_solid(self) -> bool:
        return (self.address >> 29) & 3 == 3
    def get_address(self) -> int:
        return self.address & 0x1FFFFFFF

class Voxtree:
    def __init__(self, depth: int, branches: list, leaves: list, features: int):
        self.depth = depth
        self.branches = branches
        self.leaves = leaves
        self.features = features

def get_index(x: int, y: int, z: int, depth: int) -> int:
    shift = depth << 1
    return ((z >> shift) & 1) << 4 | ((y >> shift) & 1) << 2 | (x >> shift) & 1

def get_transformed_index(bitmask: int, index: int) -> int:
    count = 0
    for i in range(index):
        if (bitmask >> i) & 1:
            count += 1
    return count

def shift_addresses(branches: list, start: int, length: int, is_insert: bool, is_leaf_op: bool):
    for b in branches:
        if b.is_leaf() != is_leaf_op:
            addr = b.get_address()
            if addr >= start:
                delta = length if is_insert else -length
                b.address = (b.address & ~0x1FFFFFFF) | (addr + delta)

def recursive_delete(branches: list, address: int):
    b = branches[address]
    next_addr = b.get_address()
    count = 1 if b.is_solid() else bin(b.bitmask).count('1')
    for _ in range(count):
        recursive_delete(branches, next_addr)
        next_addr += 1
    b.bitmask = 0
    b.address = b.get_address()

def voxtree_set(voxtree: Voxtree, x: int, y: int, z: int, w: int, node):
    branch_address = 0
    previous_address = 0
    for depth in range(w - 1, -1, -1):
        branch = voxtree.branches[branch_address]
        index = get_index(x, y, z, depth)
        transformed_index = get_transformed_index(branch.bitmask, index)
        index_store = index
        previous_address = branch_address
        branch_address = branch.get_address() + transformed_index

        if branch.is_leaf():
            if node is None:
                voxtree.branches[previous_address].bitmask &= ~(1 << index_store)
                leaf_addr = branch.get_address()
                if not branch.is_solid():
                    voxtree.leaves.pop(leaf_addr)
                    shift_addresses(voxtree.branches, leaf_addr, 1, False, True)
                else:
                    recursive_delete(voxtree.branches, leaf_addr)
                    shift_addresses(voxtree.branches, leaf_addr, 1, False, True)
                    voxtree.branches.pop(leaf_addr)
                return
            # Insertion into leaf parent: expand leaf, update bitmask, shift addresses
            # (Implementation follows notes: create new branches for existing leaves,
            # set bitmask to u64::MAX, update parent address/bitmask)
            pass
        else:
            if node is None:
                voxtree.branches[previous_address].bitmask &= ~(1 << index_store)
                count = branch.bitmask.bit_count()
                recursive_delete(voxtree.branches, branch.get_address())
                shift_addresses(voxtree.branches, branch.get_address(), count, False, False)
                voxtree.branches.pop(branch.get_address())
                return
            # Insertion into branch parent: update node, handle solid leaf expansion
            # (Implementation follows notes: splice leaves if solid, shift addresses)
            pass
```

**Parameters:**
- `depth`: `u8`, maximum depth of the tree. Determines resolution [S1, unofficial].
- `features`: `Features` bitflag. `None` (0b0) or `DirectedAcyclicGraph` (0b1) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Address field bit packing: Bit 31 is leaf flag, bits 30–29 are solid flag. Incorrect parsing leads to wrong node type or index [S1, unofficial].
- Address shifting: Must be careful to only update branches with different leaf status than the operation to maintain consistency [S1, unofficial].
- Leaf expansion: When converting leaf to branch, all existing leaves must be properly mapped to new branch nodes [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Lookup and insertion are $O(w)$ where $w$ is traversal depth. Address shifting is $O(B)$ where $B$ is number of branches [S1, unofficial].
- **Memory Complexity**: Sparse, storing only non-empty nodes. Branches and leaves stored in vectors [S1, unofficial].
- **Performance**: Designed for maximal CPU performance. Simple surface API [S1, unofficial].
- **Strengths**: Simple API, high CPU performance, sparse storage [S1, unofficial].
- **Limitations**: Address shifting can be costly for frequent updates. No built-in LOD or culling (disabled by default) [S1, unofficial].

## Background
Voxtree is an unofficial Rust implementation of a sparse voxel 64-tree [S1, unofficial]. The project aims to provide a simple, high-performance CPU-based sparse voxel tree [S1, unofficial].

## Sources
[S1] HexExecute/voxtree, https://github.com/HexExecute/voxtree, fetched 2026-10-26 (unofficial)
