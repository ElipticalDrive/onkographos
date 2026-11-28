# World-aligned lattice octree bake
Also called: HLOD S1a bake

## Overview
This method constructs a hierarchical level-of-detail (HLOD) octree for Gaussian splats aligned to world axes, grouping splats into leaves with bounded counts and ordering them by importance to enable efficient streaming and culling [S1, unofficial]. It differs from sibling methods by explicitly baking a world-aligned lattice structure with strict node size limits and importance-ordered payloads, rather than relying on runtime generation or isotropic seed placement [S1, unofficial]. It builds on [Data Structure and Storage](10326-data-structure-and-storage.md).

## How it works
**Inputs:** A set of Gaussian splats (position, covariance, color, spherical harmonics) and a world bounding volume.
**Outputs:** An HLOD octree structure, updated world bounds, and associated chunk data.

**Data Structures:**
*   **Node Record:** 128 bytes. Contains node type, child pointers (or leaf chunk index), and bounds.
*   **Leaf Payload:** Stores importance-ordered splat data. Coordinates are stored as node-relative positions.
*   **Interior Payload:** Stores breadth-first traversal order information.
*   **World Header (v2):** 80-byte extension containing HLOD flag, world AABB, and version info.

**Steps:**
1.  **Initialization:** Start with an origin-centred root node. The root node's local coordinate system aligns with world axes [S1, unofficial].
2.  **Leaf Creation:** Group splats into leaves. Each leaf node must contain a strict maximum of 16,384 splats [S1, unofficial].
3.  **Ordering:**
    *   Leaf nodes store splats ordered by importance [S1, unofficial].
    *   Interior nodes store payload in breadth-first order [S1, unofficial].
4.  **Position Storage:** All stored coordinates within nodes are converted to node-relative positions [S1, unofficial].
5.  **Optimization:** Apply single-child bit identity optimization where applicable [S1, unofficial].
6.  **Merge:** Use DC-encoding-aware merge cells during tree construction to optimize structure [S1, unofficial].
7.  **Exit Condition:** Implement an explicit all-pass-through exit that preserves payload and spherical harmonics (SH) unchanged if no further optimization is possible or required [S1, unofficial].
8.  **Bounds Calculation:** Compute the world's bounding volume as the union of the world's current bounds and the root node's bounds [S1, unofficial].
9.  **Rounding:** Round the computed bounds outward to `real_t` precision to prevent false culls [S1, unofficial].
10. **Publication:** Publish the world's bounds, chunks, tree, and revision. This must occur under the existing data lock [S1, unofficial].
11. **Notification:** Emit change notifications after successful publication [S1, unofficial].

**Parameters:**
*   **Max Splats per Node:** 16,384 [S1, unofficial].
*   **Header Size:** 80 bytes (v2 extension) [S1, unofficial].
*   **Node/Instance Record Size:** 128 bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **False Resident Culls:** A rotated anisotropic splat's root may reach significant world coordinates (e.g., $x = \pm 6$ m) while its local AABB is small (e.g., $x = \pm 0.03$ m). Using only the splat's local AABB causes false resident culls. The world bounds must be united with the root node's bounds and padded [S1, unofficial].
*   **State Consistency:** If baking fails, the existing state must be preserved without modification [S1, unofficial].
*   **Chunk Clearing:** Dropping the chunk clear when a non-empty tree exists fails checks. Static chunks must be cleared if a tree is present [S1, unofficial].
*   **Concurrency:** Snapshot saving captures payload, SH, 2D metadata, and revision under a single read lock. Stale pre-capture edits are rejected [S1, unofficial].
*   **Source Replacement:** When an active file-backed source is replaced, its old tree and leaf chunk addresses must be invalidated [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_hlod(splats, world_bounds):
    # Initialize root node at origin
    root = Node(origin=(0,0,0), type='root')
    
    # Group splats into leaves with max 16,384 splats
    leaves = []
    current_leaf = Leaf()
    for splat in splats:
        if current_leaf.count >= 16384:
            leaves.append(current_leaf)
            current_leaf = Leaf()
        current_leaf.add_splat(splat)
    if current_leaf.count > 0:
        leaves.append(current_leaf)
        
    # Order leaves by importance
    leaves.sort(key=lambda l: l.importance, reverse=True)
    
    # Build tree structure
    # Apply single-child bit identity optimization
    # Use DC-encoding-aware merge cells
    tree = build_octree(root, leaves)
    
    # Calculate bounds
    # Union world bounds with root node bounds
    new_bounds = union(world_bounds, root.get_world_aabb())
    
    # Round outward to real_t precision
    new_bounds = round_outward(new_bounds)
    
    # Validate bounds
    if not is_finite(new_bounds) or has_negative_size(new_bounds):
        raise ValueError("Invalid bounds")
        
    # Check if new bounds enclose root
    if not encloses(new_bounds, root.get_world_aabb()):
        raise ValueError("Bounds do not enclose root")
        
    # Publish under lock
    with data_lock:
        # Capture current state for consistency
        current_revision = get_revision()
        
        # Clear static chunks if tree is non-empty
        if tree.is_non_empty():
            clear_static_chunks()
            
        # Install new state
        world.tree = tree
        world.chunks = tree.get_chunks()
        world.bounds = new_bounds
        world.revision = current_revision + 1
        
    # Emit notifications
    emit_changed()
    
    return True
```

## Subtopics

### HLOD baking with bounded enclosure
Computes and publishes the world's bounding volume during HLOD baking, ensuring it encloses the root node and all splats to prevent false resident culls at large offsets [S1, unofficial]. It differs from standard baking by explicitly uniting and rounding bounds outward to `real_t` precision [S1, unofficial]. Choose this when precise culling at large offsets is required. It links to [World-aligned lattice octree bake (method)](10326-world-aligned-lattice-octree-bake.md).

### Snapshot saving with read lock
Captures world state atomically under a single read lock to prevent stale edits during save and ensures revision consistency [S1, unofficial]. It differs from standard saving by rejecting stale pre-capture edits and invalidating old tree/chunk addresses upon source replacement [S1, unofficial]. Choose this when atomic consistency is critical for file I/O. It links to [World-aligned lattice octree bake (method)](10326-world-aligned-lattice-octree-bake.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Enables efficient streaming and culling via HLOD structure; prevents false culls with bounded enclosure; maintains consistency with atomic snapshots.
*   **Limitations:** Strict node size limit (16,384 splats); requires careful bounds calculation to avoid culling errors; complex concurrency handling.
*   **When to use:** When streaming large Gaussian splat scenes is required and culling efficiency is critical.
*   **When not to use:** For small scenes where streaming overhead outweighs benefits; when strict real-time consistency is not required and simpler structures suffice.

## Background
Originated in the GaussianSplatWorld implementation for Godot, specifically in the HLOD S1a bake feature [S1, unofficial]. It was developed to address the need for efficient level-of-detail management and streaming of Gaussian splat data in real-time rendering contexts [S1, unofficial]. The method is part of the .gsplatworld v2 format specification [S1, unofficial].

## Sources
[S1] gaussian_splatting: HLOD S1a: .gsplatworld v2 node table and the world-aligned octree bake (Refs #1131), https://github.com/klausi3D/godotGS/pull/1149, fetched 2026-10-27 (unofficial)
