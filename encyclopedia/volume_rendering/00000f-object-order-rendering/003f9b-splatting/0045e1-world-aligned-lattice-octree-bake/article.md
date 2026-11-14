# World-aligned lattice octree bake

Also called: HLOD S1a bake

## Overview
This method constructs a hierarchical level-of-detail (HLOD) octree for Gaussian splats aligned to world axes, grouping splats into leaves with bounded counts and ordering them by importance to enable efficient streaming and culling [S1, unofficial]. It differs from standard splatting by pre-computing a spatial hierarchy and bounding volumes to prevent false resident culls and optimize memory access, rather than relying solely on runtime sorting or frustum culling of individual primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of Gaussian splats defined by position, covariance, opacity, and spherical harmonics coefficients [G].
*   **Output:** An HLOD octree structure stored in memory, including interior nodes, leaf nodes, and a world-aligned bounding volume that encloses the root and all splats [S1, unofficial].

**Data Structures:**
The structure follows World format v2 specifications [S1, unofficial]:
1.  **Header:** An 80-byte extension containing the HLOD flag and the world AABB [S1, unofficial].
2.  **Node/Instance Records:** 128-byte records for nodes and instances [S1, unofficial].
3.  **Nodes:**
    *   **Interior Nodes:** Store breadth-first payload [S1, unofficial].
    *   **Leaf Nodes:** Store importance-ordered payload [S1, unofficial].
    *   **Coordinates:** All stored positions are node-relative [S1, unofficial].
    *   **Optimization:** Single-child bit identity optimization is applied where applicable [S1, unofficial].
    *   **Merge:** DC-encoding-aware merge cells are used during construction [S1, unofficial].
4.  **Bounds:** The header world AABB must enclose the root node's world AABB [S1, unofficial].

**Steps:**
1.  **State Capture:** Capture the full resident count, payload, spherical harmonics (SH), 2D metadata, and revision under the existing data lock [S1, unofficial].
2.  **Leaf Creation:** Create leaves with a strict maximum of 16,384 splats per node [S1, unofficial].
3.  **Importance Ordering:** Order leaf payloads by importance [S1, unofficial].
4.  **Interior Storage:** Store interior node payloads in breadth-first order [S1, unofficial].
5.  **Coordinate Transformation:** Convert all stored coordinates to node-relative positions [S1, unofficial].
6.  **Bit Identity Optimization:** Apply single-child bit identity optimization to the tree [S1, unofficial].
7.  **Merge Cell Application:** Use DC-encoding-aware merge cells during tree construction [S1, unofficial].
8.  **Bounding Volume Calculation:** Compute the world's bounding volume as the union of the world's bounds and the root node's bounds [S1, unofficial].
9.  **Precision Rounding:** Round the computed bounds outward to `real_t` precision to prevent precision errors [S1, unofficial].
10. **Publication:** Publish the world's bounds, chunks, tree, and revision before emitting change notifications [S1, unofficial].
11. **Exit Handling:** Implement an explicit all-pass-through exit that preserves payload and spherical harmonics unchanged [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_hlod(splats, world_lock):
    # Step 1: Capture state atomically
    with world_lock.read():
        payload = splats.get_payload()
        sh_coeffs = splats.get_sh()
        metadata = splats.get_2d_metadata()
        revision = splats.get_revision()
        if not splats.validate_revision(revision):
            return False

    # Step 2: Construct Tree
    root = Node(type="INTERIOR", children=[])
    
    # Group splats into leaves (max 16,384 per node)
    leaves = []
    for chunk in spatial_partition(splats, max_splats=16384):
        leaf = Node(type="LEAF", splats=[])
        # Step 3: Order by importance
        leaf.splats = sort_by_importance(chunk)
        # Step 5: Convert to node-relative
        for splat in leaf.splats:
            splat.position = splat.position - leaf.center
        leaves.append(leaf)
    
    # Step 4: Build hierarchy (breadth-first for interior)
    tree = build_octree(leaves, root)
    
    # Step 6 & 7: Apply optimizations
    tree = apply_single_child_bit_identity(tree)
    tree = apply_dc_encoding_aware_merge(tree)

    # Step 8 & 9: Compute Bounds
    root_aabb = tree.get_world_aabb()
    world_aabb = world.get_bounds().union(root_aabb)
    rounded_aabb = round_outward(world_aabb, real_t_precision)
    
    # Step 10: Publish
    if tree.is_non_empty():
        world.clear_static_chunks()
        
    world.set_bounds(rounded_aabb)
    world.set_tree(tree)
    world.set_payload(payload)
    world.set_sh(sh_coeffs)
    world.set_revision(revision + 1)
    
    world.emit_changed()
    
    # Step 11: Explicit all-pass-through exit preserves payload and SH unchanged
    return True

# Helper definitions
class Node:
    def __init__(self, type, children=None, splats=None, center=None):
        self.type = type
        self.children = children or []
        self.splats = splats or []
        self.center = center or [0, 0, 0]
    def get_world_aabb(self):
        return compute_aabb_from_nodes(self)

def spatial_partition(splats, max_splats):
    # Groups splats into spatial chunks respecting the maximum count
    chunks = []
    current_chunk = []
    for s in splats:
        current_chunk.append(s)
        if len(current_chunk) >= max_splats:
            chunks.append(current_chunk)
            current_chunk = []
    if current_chunk:
        chunks.append(current_chunk)
    return chunks

def sort_by_importance(splat_list):
    # Sorts splats by contribution metric (e.g., opacity or projected area)
    return sorted(splat_list, key=lambda s: s.importance, reverse=True)

def build_octree(leaves, root):
    # Constructs the octree hierarchy in breadth-first order
    root.children = leaves
    return root

def apply_single_child_bit_identity(tree):
    # Optimizes nodes with exactly one child by encoding identity in bits
    return tree

def apply_dc_encoding_aware_merge(tree):
    # Merges cells aware of DC-encoding format constraints
    return tree

def round_outward(aabb, precision):
    # Rounds AABB min/max outward to the specified floating-point precision
    return {
        "min": [floor(v, precision) for v in aabb["min"]],
        "max": [ceil(v, precision) for v in aabb["max"]]
    }

real_t_precision = 1e-6
```

**Edge Cases and Pitfalls:**
*   **False Resident Culls:** A rotated anisotropic splat's root may reach $x = \pm 6$ m while its AABB spans only $x = \pm 0.03$ m [S1, unofficial]. Using only the splat's AABB causes false resident culls; bounds must be united with the root and padded [S1, unofficial].
*   **State Corruption:** If `set_gaussian_data()` is called on a world with an HLOD tree, it previously dropped the tree but kept leaf chunks indexing the old payload [S1, unofficial]. This is fixed by clearing `static_chunks` when a non-empty tree exists [S1, unofficial].
*   **Pre-bake Bounds:** Publishing pre-bake bounds fails validation checks; the bounds must be the united and rounded bounds [S1, unofficial].

## Subtopics

### HLOD baking with bounded enclosure
Computes and publishes the world's bounding volume during HLOD baking, ensuring it encloses the root node and all splats to prevent false resident culls at large offsets [S1, unofficial]. It captures payload, SH, 2D metadata, and revision under the existing data lock, computes bounds as the world's bounds united with the root node's bounds, and rounds them outward to `real_t` precision [S1, unofficial]. A key pitfall is that a rotated anisotropic splat's root may reach $x = \pm 6$ m while its AABB spans only $x = \pm 0.03$ m; using only the splat's AABB causes false resident culls, so bounds must be united with the root and padded [S1, unofficial]. Dropping the chunk clear fails 3 checks in 2 cases; publishing pre-bake bounds fails 5 checks in 3 cases [S1, unofficial].

### Snapshot saving with read lock
Captures world state atomically under a single read lock to prevent stale edits during save and ensures revision consistency [S1, unofficial]. It captures payload, SH, 2D metadata, and revision under one read lock, rejects stale pre-capture edits before writing to disk, and invalidates old tree and leaf chunk addresses when an active file-backed source is replaced [S1, unofficial]. Intervening GaussianData edits are retained and reject the bake rather than being overwritten [S1, unofficial]. A pitfall is that `set_gaussian_data()` on a world with an HLOD tree previously dropped the tree but kept leaf chunks indexing the old payload, fixed by clearing `static_chunks` when a non-empty tree exists [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Not given in the sources.
*   **Memory:** Not given in the sources.

**Measured Performance:**
*   The method is designed for efficient streaming and culling [S1, unofficial].
*   Specific performance metrics (FPS, memory usage) are not provided in the sources [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Prevents false resident culls, enables efficient streaming, and optimizes memory access through hierarchical grouping [S1, unofficial].
*   **Limitations:** Requires a baking step, which adds latency before rendering can begin with the HLOD structure [S1, unofficial]. The method assumes single-threaded main-thread ownership for world setters [S1, unofficial].

**When to Use:**
*   Use when rendering large scenes with many Gaussian splats where streaming and culling efficiency are critical [S1, unofficial].
*   Do not use if real-time updates to the splat set are frequent and baking latency is unacceptable [S1, unofficial].

## Background
The method originates from the GaussianSplatWorld implementation in the Godot engine's Gaussian Splatting module [S1, unofficial]. It was developed to address issues with false resident culls and inefficient memory access in large-scale Gaussian splatting scenes [S1, unofficial]. The HLOD S1a bake is part of the World format v2 specification [S1, unofficial].

## Sources
[S1] gaussian_splatting: HLOD S1a: .gsplatworld v2 node table and the world-aligned octree bake (Refs #1131), https://github.com/klausi3D/godotGS/pull/1149, fetched 2026-10-27 (unofficial)
