# Manifold Dual Contouring - Index Processing and Triangle Generation

Also called: ProcessIndexes

## Overview
This method processes the vertices associated with the octree cells clustered in the previous step to generate the final triangle indices for the dual contouring mesh. It resolves vertex sharing between adjacent cells by traversing parent pointers to find a common "highest valid" vertex, ensuring manifold connectivity. It then determines the correct triangle winding order based on edge sign changes and flip states, outputting a list of triangle indices. It builds on [Manifold Dual Contouring - Cell Clustering](3677-manifold-dual-contouring-cell-clustering.md).

## How it works
**Inputs:**
- `nodes`: An array of cell objects. Each cell object contains:
  - `corners`: An integer array of size 8, representing corner indices or flags.
  - `size`: An integer representing the cell size (used for minimization).
  - `vertices`: An array of vertex objects.
  - `parent`: A pointer to the parent cell object, or `null` if root.
  - `error`: A float representing reconstruction error.
  - `euler`: An integer representing Euler characteristic contribution.
  - `face_prop2`: A boolean or integer flag related to face properties.
  - `index`: An integer unique identifier for the vertex.
- `threshold`: A float threshold for error comparison.
- `EnforceManifold`: A boolean flag.
- `indexes`: An output array/list to store triangle indices.
- `tri_count`: An output integer counter for total triangles.

**Outputs:**
- `indexes`: The generated triangle indices.
- `tri_count`: The total number of triangles generated.

**Data Structures:**
- `Utilities.TProcessEdgeMask`: A 2D array mapping direction and edge index to edge identifiers.
- `Utilities.EdgePairs`: A mapping from edge identifier to a pair of corner indices `(c1, c2)`.
- `Utilities.TransformedEdgesTable`: A 2D array mapping corner configurations to edge sequences. Values are edge identifiers, `-1` (skip/invalid), or `-2` (end of sequence).
- Vertex object: Contains `index` (int), `parent` (pointer), `error` (float), `euler` (int), `face_prop2` (bool/int).

**Algorithm Steps:**

1.  **Initialization:**
    - Set `min_size` to a large value (e.g., 10,000,000).
    - Set `min_index` to 0.
    - Initialize `indices` array of size 4 with values `{-1, -1, -1, -1}`.
    - Set `flip` to `false`.
    - Set `sign_changed` to `false`.
    - Set `v_count` to 0.

2.  **Iterate over Cell Edges:**
    - Loop `i` from 0 to 3:
        - Get `edge` from `Utilities.TProcessEdgeMask[direction][i]`.
        - Get corner indices `c1` and `c2` from `Utilities.EdgePairs[edge]`.
        - Extract bit `m1` from `nodes[i].corners` at index `c1`.
        - Extract bit `m2` from `nodes[i].corners` at index `c2`.
        - If `nodes[i].size < min_size`:
            - Update `min_size = nodes[i].size`.
            - Update `min_index = i`.
            - Set `flip = (m1 == 1)`.
            - Set `sign_changed = ((m1 == 0 && m2 != 0) || (m1 != 0 && m2 == 0))`.

3.  **Find Vertex Index:**
    - Initialize `index = 0`.
    - Initialize `skip = false`.
    - Loop `k` from 0 to 15:
        - Get `e` from `Utilities.TransformedEdgesTable[nodes[i].corners][k]`.
        - If `e == -1`:
            - Increment `index`.
            - Continue to next `k`.
        - If `e == -2`:
            - Set `skip = true`.
            - Break loop.
        - If `e == edge`:
            - Break loop.

4.  **Skip if Invalid:**
    - If `skip` is `true`, continue to next `i`.
    - Increment `v_count`.
    - If `index >= nodes[i].vertices.length`, return (skip this cell).
    - Get vertex `v = nodes[i].vertices[index]`.

5.  **Traverse Parent Pointers for Highest Valid Vertex:**
    - Set `highest = v`.
    - While `highest.parent != null`:
        - Check condition: `highest.parent.error <= threshold` AND (`!EnforceManifold` OR (`highest.parent.euler == 1` AND `highest.parent.face_prop2`)).
        - If condition is true:
            - Set `v = highest.parent`.
        - Else:
            - Set `highest = highest.parent`.
    - Set `indices[i] = v.index`.

6.  **Generate Triangles if Sign Changed:**
    - If `sign_changed` is `true`:
        - Initialize `count = 0`.
        - If `!flip`:
            - If `indices[0] != -1` AND `indices[1] != -1` AND `indices[2] != -1` AND `indices[0] != indices[1]` AND `indices[1] != indices[3]`:
                - Add `indices[0]`, `indices[1]`, `indices[3]` to `indexes`.
                - Increment `count`.
            - If `indices[0] != -1` AND `indices[2] != -1` AND `indices[3] != -1` AND `indices[0] != indices[2]` AND `indices[2] != indices[3]`:
                - Add `indices[0]`, `indices[3]`, `indices[2]` to `indexes`.
                - Increment `count`.
        - If `flip` is `true`:
            - If `indices[0] != -1` AND `indices[3] != -1` AND `indices[1] != -1` AND `indices[0] != indices[1]` AND `indices[1] != indices[3]`:
                - Add `(0x10000000 | indices[0])`, `(0x10000000 | indices[3])`, `(0x10000000 | indices[1])` to `indexes`.
                - Increment `count`.
            - If `indices[0] != -1` AND `indices[2] != -1` AND `indices[3] != -1` AND `indices[0] != indices[2]` AND `indices[2] != indices[3]`:
                - Add `(0x10000000 | indices[0])`, `(0x10000000 | indices[2])`, `(0x10000000 | indices[3])` to `indexes`.
                - Increment `count`.
        - If `count > 0`:
            - Add `count` to `tri_count`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_indexes(nodes, threshold, enforce_manifold, indexes, tri_count):
    min_size = 10000000
    min_index = 0
    indices = [-1, -1, -1, -1]
    flip = False
    sign_changed = False
    v_count = 0

    for i in range(4):
        edge = Utilities.TProcessEdgeMask[direction][i]
        c1, c2 = Utilities.EdgePairs[edge]
        
        # Extract bits from corners array
        m1 = (nodes[i].corners[c1] & 1)
        m2 = (nodes[i].corners[c2] & 1)
        
        if nodes[i].size < min_size:
            min_size = nodes[i].size
            min_index = i
            flip = (m1 == 1)
            sign_changed = ((m1 == 0 and m2 != 0) or (m1 != 0 and m2 == 0))
            
        index = 0
        skip = False
        
        for k in range(16):
            e = Utilities.TransformedEdgesTable[nodes[i].corners][k]
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
        
        while highest.parent is not None:
            parent = highest.parent
            condition = (parent.error <= threshold) and (not enforce_manifold or (parent.euler == 1 and parent.face_prop2))
            if condition:
                v = parent
            else:
                highest = parent
                
        indices[i] = v.index
        
    if sign_changed:
        count = 0
        if not flip:
            if indices[0] != -1 and indices[1] != -1 and indices[2] != -1 and indices[0] != indices[1] and indices[1] != indices[3]:
                indexes.append(indices[0])
                indexes.append(indices[1])
                indexes.append(indices[3])
                count += 1
            if indices[0] != -1 and indices[2] != -1 and indices[3] != -1 and indices[0] != indices[2] and indices[2] != indices[3]:
                indexes.append(indices[0])
                indexes.append(indices[3])
                indexes.append(indices[2])
                count += 1
        else:
            if indices[0] != -1 and indices[3] != -1 and indices[1] != -1 and indices[0] != indices[1] and indices[1] != indices[3]:
                indexes.append(0x10000000 | indices[0])
                indexes.append(0x10000000 | indices[3])
                indexes.append(0x10000000 | indices[1])
                count += 1
            if indices[0] != -1 and indices[2] != -1 and indices[3] != -1 and indices[0] != indices[2] and indices[2] != indices[3]:
                indexes.append(0x10000000 | indices[0])
                indexes.append(0x10000000 | indices[2])
                indexes.append(0x10000000 | indices[3])
                count += 1
                
        if count > 0:
            tri_count += count
```

**Parameters:**
- `threshold`: Not given in the sources.
- `EnforceManifold`: Not given in the sources.
- `direction`: Not given in the sources.

**Edge Cases and Pitfalls:**
- If `index` exceeds the number of vertices in the cell, the cell is skipped.
- If `skip` is true due to `-2` in the transformation table, the cell is skipped.
- The use of `0x10000000` bit flag in indices suggests a specific encoding for flipped triangles, which must be handled correctly during mesh rendering or further processing.
- Precision issues with `error` comparison should be considered when setting `threshold`.

## Subtopics

## Operating characteristics
- Time complexity: Not given in the sources.
- Memory complexity: Not given in the sources.
- Measured performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths: Ensures manifold connectivity by resolving vertex sharing through parent traversal.
- Limitations: Relies on specific data structures and tables (`Utilities.*`) which may not be standard.
- When to use: When manifold connectivity is required in dual contouring.
- When not: When non-manifold meshes are acceptable or when performance is critical and manifold checks are too expensive.

## Background
This method is part of the Manifold Dual Contouring algorithm, specifically handling the index processing and triangle generation phase. It is based on the implementation found in the unofficial Java repository [S1, unofficial].

## Sources
[S1] proton2/Manifold-dual-contouring-Java, https://github.com/proton2/Manifold-dual-contouring-Java, fetched 2026-10-26 (unofficial)
