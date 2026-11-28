# Kd Tree Unrolling (JiTTree)
Also called: JiTTree, Just-in-Time Compilation Tree, Just-in-Time Compiled Tree, Just-in-time compiled kd tree, Kd Tree of Bricks with Partial Evaluation, Unrolled Kd Tree

## Overview
JiTTree is a data structure optimization that uses just-in-time (JIT) compilation to specialize tree traversal code for a specific dataset, eliminating type-checking overhead and pointer-chasing by unrolling the tree structure into nested conditionals [S1, unofficial]. It builds on [Hybrid Kd Tree Traversal](20234-hybrid-kd-tree-traversal.md) by grouping homogeneous brick types into leaf nodes and applying partial evaluation at runtime to inline type-specific logic, thereby shifting the performance bottleneck from memory-bound data fetches to instruction-cache-bound code execution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A loaded 3D volume dataset organized as a Hybrid Kd Tree of bricks (Dense, Octree, Voxel List, or Empty). The dataset is static during traversal.
*   **Output:** A specialized, unrolled traversal function (code) that maps 3D coordinates $(x, y, z)$ to the appropriate brick data or an empty value.

**Data Structures:**
The underlying structure is a Kd Tree where leaf nodes contain sub-volumes of a single brick type (homogeneous regions), rather than single bricks or mixed types [S1, unofficial].
*   **Internal Nodes:** Define splitting planes. In the unrolled code, these are represented by literals (splitting axis and coordinate) rather than struct fields.
*   **Leaf Nodes:** Contain pointers to the specific brick data. In the unrolled code, these are either literal pointers (for single-brick leaves) or local constant arrays (for multi-brick leaves) [S1, unofficial].

**Numbered Steps:**
1.  **Construct Homogeneous Kd Tree:** Build a Kd Tree where splitting planes are chosen to maximize the homogeneity of leaf nodes. Each leaf node contains a sub-volume consisting of bricks of a single type (e.g., all Dense, or all Octree) [S1, unofficial].
2.  **Partial Evaluation (Tree Unrolling):** At runtime, upon data load, feed the static dataset into a partial evaluator. This transforms the general tree traversal algorithm into data-specific code [S1, unofficial].
3.  **Apply Optimizations:** The partial evaluator applies five specific optimizations to generate the final unrolled code:
    *   **(i) Traversal Optimization:** Encode the splitting axis order directly in the conditional structure. Insert splitting plane positions as literal constants. This requires no memory fetches for traversal data [S1, unofficial].
    *   **(ii) Empty Discarding:** If a coordinate $(x, y, z)$ does not match any leaf node path, return `EMPTY` (e.g., 0) as the default value. This discards empty regions without memory fetches [S1, unofficial].
    *   **(iii) Implicit Specialization:** Inline the specialized access functions (e.g., `getDense`, `getOctree`, `getVList`) directly at the leaf nodes. This removes runtime type checks and switch statements [S1, unofficial].
    *   **(iv) Pointer In-baking:** For leaf nodes containing multiple bricks, store pointers as a local constant array within the generated function. For leaf nodes containing a single brick, use pointer literals directly in the function call, eliminating one memory fetch [S1, unofficial].
    *   **(v) Leaf Node Branch Removal:** Reduce Kd Tree branches that contain only a single brick type to a single-type bricking approach, simplifying the conditional logic [S1, unofficial].

**Pseudocode:**
The following pseudocode represents the unrolled traversal for a specific example dataset (512x512x512 volume, 64x64x64 bricks) as described in [S1, unofficial]. The literals (192, 64, 384, etc.) are splitting plane coordinates baked into the code.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Specialized JIT-compiled get function for a specific dataset
# Inputs: x, y, z (integers)
# Outputs: Data value or EMPTY

def get(x, y, z):
    # Optimization (i): Traversal with literal splitting planes
    # Splitting plane at x = 192
    if x < 192:
        # Splitting plane at z = 64
        if z < 64:
            # Leaf: Dense bricks
            # Optimization (iv): Pointer in-baking (local array or literal)
            # Optimization (iii): Implicit specialization
            return getDenseSubVolume0(x, y, z)
        else:
            # Splitting plane at z = 384
            if z < 384:
                # Leaf: Octree bricks
                return getOctreeSubVolume0(x, y, z)
            else:
                # Leaf: Voxel List bricks
                return getVListSubVolume0(x, y, z)
    else:
        # Splitting plane at x = 384
        if x < 384:
            # Splitting plane at y = 192
            if y < 192:
                # Leaf: Dense bricks
                return getDenseSubVolume1(x, y, z)
            else:
                # Leaf: Voxel List bricks
                return getVListSubVolume1(x, y, z)
        else:
            # Leaf: Octree bricks
            return getOctreeSubVolume1(x, y, z)

# Helper functions (inlined in final code)
def getDenseSubVolume0(x, y, z):
    # Access dense brick data using baked-in pointers
    pass

def getOctreeSubVolume0(x, y, z):
    # Access octree brick data using baked-in pointers
    pass

def getVListSubVolume0(x, y, z):
    # Access voxel list brick data using baked-in pointers
    pass

# ... (other specialized sub-volume functions)

# Optimization (ii): Empty discarding
# If the path is not covered by the unrolled conditionals, return EMPTY
# Note: In the specific example from [S1, unofficial], the structure covers
# the relevant space, but the general principle is:
# return EMPTY
```

**Parameters:**
*   **Brick Size:** 64x64x64 voxels (typical example value) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Standard Kd Tree Inefficiency:** Standard Kd Trees split homogeneous regions unnecessarily, increasing traversal depth. JiTTree avoids this by grouping same-type bricks into leaves [S1, unofficial].
*   **Instruction Cache Pressure:** Unrolling increases program length (instruction count), which may impact instruction cache efficiency or register pressure [S1, unofficial].
*   **Code Size Growth:** The access time becomes independent of the number of elemental brick types, but the generated code size grows with the number of type-specific branches [S1, unofficial].

## Subtopics

### Unrolled Binary Search Tree
A technique where a binary search tree's data is explicitly compiled into the code (instructions) to eliminate memory fetches during traversal, shifting the caching burden from data cache to instruction cache [S1, unofficial].
*   **Mechanism:** Replace recursive function calls and pointer dereferences with explicit nested `if-else` statements comparing the search key against hardcoded node keys, returning hardcoded values [S1, unofficial].
*   **Pitfall:** Increases program length, potentially impacting instruction cache efficiency [S1, unofficial].
*   **Comparison:** Unlike standard BSTs requiring memory fetches for `node.key` and `node.value`, the unrolled version has no memory fetches for node data, eliminating pointer overhead and cache misses [S1, unofficial].

### Type Routing via Tree Unrolling
A technique where type-specific functions are inlined directly into the unrolled tree traversal code, eliminating switch statements and runtime type resolution overhead [S1, unofficial].
*   **Mechanism:** Traverse the unrolled tree using hardcoded key comparisons. When a leaf node is reached, directly call the specialized function associated with that node's type, passing the value as an argument [S1, unofficial].
*   **Pitfall:** Access time is independent of the number of elemental brick types, but code size grows with the number of type-specific branches [S1, unofficial].
*   **Comparison:** Differs from standard type routing (which uses a switch statement on an element's type field after a lookup) by eliminating the lookup of the type field and the switch dispatch [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Traversal time is $O(\log N)$ in terms of tree depth, but with a significantly reduced constant factor due to the elimination of pointer chasing and type checks. The overhead depends on the homogeneity of the dataset rather than the number of elemental types [S1, unofficial].
*   **Memory Complexity:** The data structure itself (the Kd Tree) is not stored in memory during traversal; instead, the "structure" is stored in the instruction cache. This reduces memory bandwidth requirements for traversal [S1, unofficial].
*   **Measured Performance:** In a 512x512x512 volume with 64x64x64 bricks, JiTTree shifts the algorithm from being memory-bound to being instruction-throughput-bound or compute-bound, improving performance on GPUs where memory fetches are costly [S1, unofficial].
*   **Strengths:** Eliminates type-checking overhead; removes pointer chasing; efficient discarding of empty regions; specialized access for homogeneous regions.
*   **Limitations:** Increases program length (instruction count); code size grows with the number of brick types; requires JIT compilation capability at runtime.
*   **When to Use:** When the dataset is static and loaded at runtime, and the hardware has a large instruction cache or limited memory bandwidth for traversal data.
*   **When Not to Use:** When the dataset changes frequently (requiring frequent recompilation) or when instruction cache size is severely constrained.

## Background
JiTTree was introduced in a thesis by [S1, unofficial] as a just-in-time compiled tree data structure. The idea exploits information not known during earlier compilation stages (i.e., the specific dataset loaded at runtime) to optimize traversal code using partial evaluation. It is an extension of the hybrid volume bricking approach, aiming to improve access performance by incorporating the dataset into the traversal code [S1, unofficial].

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
