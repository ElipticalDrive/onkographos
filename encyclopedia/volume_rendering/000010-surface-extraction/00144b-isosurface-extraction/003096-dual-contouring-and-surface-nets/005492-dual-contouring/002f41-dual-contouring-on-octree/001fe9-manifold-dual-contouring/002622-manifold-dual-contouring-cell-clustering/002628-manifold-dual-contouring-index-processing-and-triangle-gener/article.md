# Manifold Dual Contouring - Index Processing and Triangle Generation

Also called: ProcessIndexes

## Overview
This method processes the vertices of a single octree cell to determine the triangle indices that form the mesh geometry for that cell [S1, unofficial]. It resolves vertex conflicts by traversing parent pointers to find the highest valid vertex and generates triangles based on the sign changes of the contouring bits at the cell's corners [S1, unofficial]. It differs from standard dual contouring by enforcing manifold constraints through parent traversal and specific flip logic, building on [Manifold Dual Contouring - Cell Clustering](3677-manifold-dual-contouring-cell-clustering.md) [S1, unofficial].

## How it works
**Inputs:**
- `nodes`: An array of 4 octree node objects. Each node has fields: `size` (integer), `corners` (integer bitmask), `vertices` (array of vertex objects), `parent` (pointer to parent node or null), `error` (float), `euler` (integer), `face_prop2` (integer/boolean) [S1, unofficial].
- `direction`: The current processing direction (used to select edge masks) [S1, unofficial].
- `threshold`: A float value used to determine if a parent vertex is valid for traversal [S1, unofficial].
- `EnforceManifold`: A boolean flag [S1, unofficial].
- `indexes`: A global list/array to store generated triangle indices [S1, unofficial].
- `tri_count`: A global counter for the number of triangles generated [S1, unofficial].

**Outputs:**
- Updates `indexes` with new triangle vertex indices [S1, unofficial].
- Updates `tri_count` [S1, unofficial].

**Data Structures:**
- `TProcessEdgeMask`: A 2D array mapping direction and edge index to an edge identifier [S1, unofficial].
- `EdgePairs`: A mapping from edge identifier to a pair of corner indices `(c1, c2)` [S1, unofficial].
- `TransformedEdgesTable`: A 2D array mapping corner bitmasks to a list of 16 edge identifiers. Values can be valid edge IDs, `-1` (skip), or `-2` (stop) [S1, unofficial].

**Algorithm:**

1.  **Initialization:**
    - Set `min_size` = 10,000,000.
    - Set `min_index` = 0.
    - Initialize `indices` array of size 4 with values `{-1, -1, -1, -1}`.
    - Set `flip` = false.
    - Set `sign_changed` = false.
    - Set `v_count` = 0.

2.  **Iterate over edges:**
    Loop `i` from 0 to 3:
    - Get `edge` = `TProcessEdgeMask[direction][i]` [S1, unofficial].
    - Get `c1`, `c2` = `EdgePairs[edge]` [S1, unofficial].
    - Extract `m1` = bit `c1` of `nodes[i].corners` [S1, unofficial].
    - Extract `m2` = bit `c2` of `nodes[i].corners` [S1, unofficial].
    - If `nodes[i].size < min_size`:
        - Set `min_size` = `nodes[i].size`.
        - Set `min_index` = `i`.
        - Set `flip` = (`m1` == 1) [S1, unofficial].
        - Set `sign_changed` = (`(m1 == 0 && m2 != 0) || (m1 != 0 && m2 == 0)`) [S1, unofficial].

3.  **Find Vertex Index:**
    For each `i` from 0 to 3:
    - Initialize `index` = 0.
    - Initialize `skip` = false.
    - Loop `k` from 0 to 15:
        - Get `e` = `TransformedEdgesTable[nodes[i].corners][k]` [S1, unofficial].
        - If `e == -1`:
            - Increment `index`.
            - Continue.
        - If `e == -2`:
            - Set `skip` = true.
            - Break loop.
        - If `e == edge`:
            - Break loop.
    - If `skip` is true, continue to next `i` [S1, unofficial].
    - Increment `v_count`.
    - If `index >= nodes[i].vertices.length`, return (skip this node) [S1, unofficial].
    - Get `v` = `nodes[i].vertices[index]` [S1, unofficial].

4.  **Traverse Parent Pointers:**
    - Set `highest` = `v`.
    - While `highest.parent != null`:
        - Check condition: `highest.parent.error <= threshold` AND (`!EnforceManifold` OR (`highest.parent.euler == 1` AND `highest.parent.face_prop2`)) [S1, unofficial].
        - If condition is true:
            - Set `v` = `highest.parent`.
        - Else:
            - Set `highest` = `highest.parent`.
    - Set `indices[i]` = `v.index` [S1, unofficial].

5.  **Generate Triangles:**
    If `sign_changed` is true:
    - Initialize `count` = 0.
    - If `!flip`:
        - If `indices[0] != -1` AND `indices[1] != -1` AND `indices[2] != -1` AND `indices[0] != indices[1]` AND `indices[1] != indices[3]`:
            - Add `indices[0]`, `indices[1]`, `indices[3]` to `indexes` [S1, unofficial].
            - Increment `count`.
        - If `indices[0] != -1` AND `indices[2] != -1` AND `indices[3] != -1` AND `indices[0] != indices[2]` AND `indices[2] != indices[3]`:
            - Add `indices[0]`, `indices[3]`, `indices[2]` to `indexes` [S1, unofficial].
            - Increment `count`.
    - If `flip` is true:
        - If `indices[0] != -1` AND `indices[3] != -1` AND `indices[1] != -1` AND `indices[0] != indices[1]` AND `indices[1] != indices[3]`:
            - Add `(0x10000000 | indices[0])`, `(0x10000000 | indices[3])`, `(0x10000000 | indices[1])` to `indexes` [S1, unofficial].
            - Increment `count`.
        - If `indices[0] != -1` AND `indices[2] != -1` AND `indices[3] != -1` AND `indices[0] != indices[2]` AND `indices[2] != indices[3]`:
            - Add `(0x10000000 | indices[0])`, `(0x10000000 | indices[2])`, `(0x10000000 | indices[3])` to `indexes` [S1, unofficial].
            - Increment `count`.
    - If `count > 0`:
        - Add `count` to `tri_count` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ProcessIndexes(nodes, direction, threshold, EnforceManifold, indexes, tri_count):
    # Lookup tables defined per source structure
    TProcessEdgeMask = [
        [0, 1, 2, 3], # Example mapping for direction 0
        [4, 5, 6, 7], # Example mapping for direction 1
        # ... other directions
    ]
    EdgePairs = {
        0: (0, 1), 1: (1, 2), 2: (2, 3), 3: (3, 0),
        4: (0, 4), 5: (1, 5), 6: (2, 6), 7: (3, 7),
        8: (4, 5), 9: (5, 6), 10: (6, 7), 11: (7, 4),
        12: (0, 2), 13: (1, 3), 14: (4, 6), 15: (5, 7)
    }
    # TransformedEdgesTable is a 256x16 array mapping corner masks to edge sequences
    # Values: valid edge ID, -1 (skip), -2 (stop)
    TransformedEdgesTable = [
        [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15],
        # ... 255 more rows
    ]

    min_size = 10000000
    min_index = 0
    indices = [-1, -1, -1, -1]
    flip = False
    sign_changed = False
    v_count = 0

    # Step 2: Iterate edges to find min size and determine flip/sign_changed
    for i in range(4):
        edge = TProcessEdgeMask[direction][i]
        c1, c2 = EdgePairs[edge]
        m1 = (nodes[i].corners >> c1) & 1
        m2 = (nodes[i].corners >> c2) & 1
        
        if nodes[i].size < min_size:
            min_size = nodes[i].size
            min_index = i
            flip = (m1 == 1)
            sign_changed = ((m1 == 0 and m2 != 0) or (m1 != 0 and m2 == 0))

    # Step 3 & 4: Find vertex and traverse parents
    for i in range(4):
        index = 0
        skip = False
        
        # Find the correct vertex index in the node's vertex list
        for k in range(16):
            e = TransformedEdgesTable[nodes[i].corners][k]
            if e == -1:
                index += 1
                continue
            if e == -2:
                skip = True
                break
            if e == edge:
                break
        
        if skip:
            continue
            
        v_count += 1
        if index >= len(nodes[i].vertices):
            continue
            
        v = nodes[i].vertices[index]
        highest = v
        
        # Traverse parent pointers
        while highest.parent is not None:
            parent = highest.parent
            # Condition: error <= threshold AND (not EnforceManifold OR (euler==1 AND face_prop2))
            if parent.error <= threshold and (not EnforceManifold or (parent.euler == 1 and parent.face_prop2)):
                v = parent
            else:
                highest = parent
        
        indices[i] = v.index

    # Step 5: Generate triangles if sign changed
    if sign_changed:
        count = 0
        if not flip:
            # Triangle 1: 0, 1, 3
            if (indices[0] != -1 and indices[1] != -1 and indices[2] != -1 and 
                indices[0] != indices[1] and indices[1] != indices[3]):
                indexes.append(indices[0])
                indexes.append(indices[1])
                indexes.append(indices[3])
                count += 1
            
            # Triangle 2: 0, 3, 2
            if (indices[0] != -1 and indices[2] != -1 and indices[3] != -1 and 
                indices[0] != indices[2] and indices[2] != indices[3]):
                indexes.append(indices[0])
                indexes.append(indices[3])
                indexes.append(indices[2])
                count += 1
        else:
            # Triangle 1: 0, 3, 1 (with high bit set)
            if (indices[0] != -1 and indices[3] != -1 and indices[1] != -1 and 
                indices[0] != indices[1] and indices[1] != indices[3]):
                indexes.append(0x10000000 | indices[0])
                indexes.append(0x10000000 | indices[3])
                indexes.append(0x10000000 | indices[1])
                count += 1
            
            # Triangle 2: 0, 2, 3 (with high bit set)
            if (indices[0] != -1 and indices[2] != -1 and indices[3] != -1 and 
                indices[0] != indices[2] and indices[2] != indices[3]):
                indexes.append(0x10000000 | indices[0])
                indexes.append(0x10000000 | indices[2])
                indexes.append(0x10000000 | indices[3])
                count += 1
        
        if count > 0:
            tri_count += count
```

**Parameters:**
- `threshold`: Not given in the sources. Typical values depend on the application's precision requirements.
- `EnforceManifold`: Boolean. When true, stricter parent traversal conditions apply.

**Edge Cases and Pitfalls:**
- `index >= nodes[i].vertices.length`: The method returns early for this node, potentially leaving `indices` partially filled with -1. This is handled in the triangle generation step by checking for -1 [S1, unofficial].
- `skip == true`: The node is skipped entirely [S1, unofficial].
- `indices` containing -1: Triangle generation checks for -1 to avoid invalid triangles [S1, unofficial].
- High bit masking (`0x10000000`): Used in the `flip` case to distinguish vertex indices, likely for back-face culling or orientation handling [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of cells processed, as each cell's vertices are processed once. The parent traversal depth is bounded by the octree depth.
- **Memory Complexity:** Depends on the storage of `indexes` and `tri_count`. The `indices` array is constant size (4).
- **Measured Performance:** Not given in the sources.
- **Strengths:** Enforces manifold constraints by traversing parent pointers and using specific flip logic.
- **Limitations:** Relies on specific utility tables (`TProcessEdgeMask`, `EdgePairs`, `TransformedEdgesTable`) which must be correctly implemented.
- **When to use:** When generating manifold meshes from adaptive octree data using dual contouring.
- **When not to use:** When non-manifold geometry is acceptable or when a simpler contouring method suffices.

## Background
This method is part of the Manifold Dual Contouring algorithm, specifically handling the index processing and triangle generation phase. It is implemented in the `proton2/Manifold-dual-contouring-Java` repository [S1, unofficial].

## Sources
[S1] proton2/Manifold-dual-contouring-Java, https://github.com/proton2/Manifold-dual-contouring-Java, fetched 2026-10-26 (unofficial)
