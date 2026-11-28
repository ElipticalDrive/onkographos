# Dynamic Octree Growth Algorithm

Also called: Jan Schultke's construction method, Octree Construction from Voxels, Octree expansion method, SVO Builder, Voxel to Octree Conversion

## Overview
The Dynamic Octree Growth Algorithm constructs a Sparse Voxel Octree (SVO) by dynamically increasing its depth and spatial extent as new voxels are processed, rather than pre-calculating the size from the input data [S1, unofficial]. It handles both positive and negative coordinates by centering the root at the boundary between -1 and 0, ensuring the tree grows to encompass all input data without pre-calculating the scene's bounding box [S1, unofficial]. It differs from static voxelization by handling out-of-bounds coordinates on-the-fly [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A stream of voxel positions (coordinates).
- Output: A Sparse Voxel Octree structure with dynamically grown depth.

**Data Structures:**
- **Node**: Represents a voxel or a parent of 8 children.
- **Octree**: Contains a root node and manages the current depth.
- **Child Indexing**: Child 0 (binary `000`) is the lowest position; Child 7 (binary `111`) is the highest position [S1, unofficial].

**Initialization:**
1. Initialize the Octree with a root node.
2. Define the coordinate system: The root node covers the range $[-1, 0)$ for each axis [S1, unofficial].
3. A voxel $v$ is contained in an octree of depth $d$ if $v$ is in the range $[-2^{d-1}, 2^{d-1})$ [S1, unofficial].

**Processing Steps:**
For each incoming voxel $v$:
1. **Check Containment**: Determine if voxel $v$ is contained within the current octree depth $d$.
   - If yes, insert $v$ into the leaf node corresponding to its position.
   - If no, trigger **Growth**.

2. **Growth Loop**:
   The algorithm expands the octree by creating a new layer of nodes. For each child index $i$ from 1 to 8 of the root node $r$ (Note: The source uses 1-based indexing for the loop but 0-7 for child indices, implying $i$ iterates through the *slots* being replaced or shifted. The pseudocode below follows the source's logic exactly):
   1. Create a new node $n'$.
   2. Move the existing child $n$ (at index $i$) to become the child at index $8 - i + 1$ of $n'$.
   3. Make $n'$ the new child of root $r$ at index $i$.
   4. Return the updated root.

   *Pseudocode for Growth:*
   ```python
   def grow_octree(octree):
       r = octree.root
       # Iterate i from 1 to 8 (inclusive)
       for i in range(1, 9):
           # Get current child at index i
           # Note: Source uses 1-based indexing for loop variable i
           # but 0-based for child access. Assuming i corresponds to child index i-1?
           # The source pseudocode says: "for all i such that 1 <= i <= 8 do n <- child(r, i)"
           # And "n' <- set child(n', n, 8-i+1)"
           # Let's stick to the source's variable names and logic.
           
           n = get_child(r, i) # Assuming 1-based indexing for child access as per source loop
           
           n_prime = create_node()
           
           # Set n as child of n' at position 8-i+1
           set_child(n_prime, n, 8 - i + 1)
           
           # Set n' as child of r at position i
           set_child(r, n_prime, i)
           
       octree.root = r
       return octree
   ```
   *Correction on Indexing*: The source text says "Child 0... is lowest... Child 7... is highest". The loop is `1 <= i <= 8`. The shift is `8-i+1`. If $i=1$, shift is 8. If $i=8$, shift is 1. This suggests the children are being rotated or shifted to make room for a new root layer. The source pseudocode is:
   ```python
   r ← root(o)
   for all i such that 1 ≤ i ≤ 8 do
     n ← child(r, i)
     n′ ← node()
     n′ ← set child(n′ , n, 8−i+1)
     r ← set child(r, n′ , i)
   end for
   return set root(o, r)
   ```
   This implies the children are accessed via 1-based indexing in this specific algorithm description.

3. **Bitstream Generation**:
   After growth ensures containment, generate a bitstream based on the normalized position of the voxel.
   - Normalize the voxel position so no component is negative [S1, unofficial].
   - Divide the bitstream into strips of 3 bits.
   - Each strip indicates which child to follow (0 to 7).
   - The leftmost strip represents the root's child; the rightmost represents the leaf [S1, unofficial].

4. **Insertion**:
   Traverse the octree using the bitstream to find the correct leaf node and insert the voxel data.

**Parameters:**
- **Coordinate System**: Root covers $[-1, 0)$ per axis [S1, unofficial].
- **Containment Range**: Voxel $v$ is in octree depth $d$ if $v \in [-2^{d-1}, 2^{d-1})$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Negative Coordinates**: The algorithm specifically handles negative coordinates by centering the root at the boundary between -1 and 0 [S1, unofficial].
- **Precision**: Normalization of voxel positions is required to generate the correct bitstream [S1, unofficial].

## Subtopics

### [Octree Insertion with Growth](2387-octree-insertion-with-growth.md)
Provides the specific recursive insertion logic and boundary checks required for incremental construction, differing from static voxelization by handling out-of-bounds coordinates on-the-fly.

### [Sparse Voxel Octree (Dynamic, Origin-Centered)](3622-sparse-voxel-octree-dynamic-origin-centered.md)
Enforces a strict origin-centered coordinate system and power-of-two voxel dimensions, growing depth only when necessary.

### [Regular-Adaptive Octree Construction](6560-regular-adaptive-octree-construction.md)
Enforces a fixed regular structure at shallow depths and uses explicit topological/curvature criteria for subdivision rather than simple occupancy checks.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Handles positive and negative coordinates without pre-calculating bounding boxes; dynamically adjusts to input data size [S1, unofficial].
- **Limitations**: Not given in the sources.
- **When to use**: When the scene's bounding box is unknown or when memory efficiency is critical for sparse data [S1, unofficial].
- **When not to use**: When static bounding boxes are known and fixed, or when simpler structures suffice [S1, unofficial].

## Background
The algorithm was published by Jan Schultke [S1, unofficial]. It is based on the methodology published by Jan Schultke [Eis21] [S1, unofficial]. The approach was adopted for projects requiring efficient ray tracing and memory savings, as regular grids consume considerable memory and require intersection tests for every voxel [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
