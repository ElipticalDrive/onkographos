# Validator Reference Face Reconstruction

## Overview
This method reconstructs a canonical, sorted list of individual voxel faces from a voxel subchunk to serve as a ground-truth reference for validating GPU-based greedy meshing algorithms [S1, unofficial]. It solves the problem of verifying bitwise reproducibility by generating a detailed, unoptimized face list that can be compared against the output of optimized extractors [S1, unofficial]. It differs from its parent [GPU Face Extraction](3760-gpu-face-extraction.md) by prioritizing correctness and canonical ordering over performance, and from its sibling [Occlusion-Based Face Generation](13226-occlusion-based-face-generation.md) by explicitly handling the full set of visible faces without occlusion culling optimizations [G].

## How it works
**Inputs and Outputs:**
- Input: `voxels`, a flat array of `uint8_t` representing material IDs in a $16 \times 16 \times 16$ chunk [S1, unofficial].
- Output: `faces`, a sorted `std::vector<uint64_t>` where each element is a packed 64-bit face token [S1, unofficial].

**Data Structures:**
- `face_token`: A logical representation of a single voxel face with fields `axis` (0-2), `positive` (boolean), `depth` (int), `u` (int), `v` (int), and `material` (uint8_t) [S1, unofficial].
- `faces`: A vector of `uint64_t` storing the generated tokens [S1, unofficial].

**Numbered Steps:**
1. Initialize an empty vector `faces`.
2. Iterate through all voxel coordinates $(x, y, z)$ where $0 \le x, y, z < 16$ [S1, unofficial].
3. For each voxel, retrieve `material = voxels[voxel_index(x, y, z)]` [S1, unofficial]. If `material == 0`, skip to the next voxel [S1, unofficial].
4. For each of the 6 directions $d \in \{0, \dots, 5\}$:
   a. Calculate neighbor coordinates using direction vectors $dx = \{-1, +1, 0, 0, 0, 0\}$, $dy = \{0, 0, -1, +1, 0, 0\}$, $dz = \{0, 0, 0, 0, -1, +1\}$ [S1, unofficial].
   b. Compute $nx = x + dx[d]$, $ny = y + dy[d]$, $nz = z + dz[d]$ [S1, unofficial].
   c. Check if the neighbor is within chunk bounds and has `material != 0` [S1, unofficial]. If the neighbor is valid and non-empty, this face is internal and is skipped [S1, unofficial].
   d. Determine face attributes:
      - $axis = d / 2$
      - $positive = (d \& 1) \neq 0$
      - $depth = (axis == 0) ? x : (axis == 1) ? y : z$
      - $u = (axis == 0) ? y : (axis == 1) ? z : x$
      - $v = (axis == 0) ? z : (axis == 1) ? x : y$ [S1, unofficial]
   e. Push a face token `(axis, positive, depth, u, v, material)` to `faces` [S1, unofficial].
5. Sort the `faces` vector using standard lexicographical comparison of the `uint64_t` values [S1, unofficial].
6. Return `faces` [S1, unofficial].

**Parameters:**
- `chunk_size`: Fixed at 16 for the reference implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- Boundary voxels have neighbors outside the chunk, which are treated as empty (material 0), so faces on the chunk boundary are always visible [S1, unofficial].
- The method assumes valid input; invalid quads are handled by the validator, not the reference generator [S1, unofficial].
- Precision: Integer arithmetic is used throughout, avoiding floating-point inaccuracies [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_reference_faces(voxels: List[int], chunk_size: int = 16) -> List[int]:
    faces = []
    
    # Direction vectors for 6 faces: -X, +X, -Y, +Y, -Z, +Z
    dx = [-1, 1, 0, 0, 0, 0]
    dy = [0, 0, -1, 1, 0, 0]
    dz = [0, 0, 0, 0, -1, 1]
    
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                material = voxels[x * chunk_size * chunk_size + y * chunk_size + z]
                if material == 0:
                    continue
                
                for d in range(6):
                    nx = x + dx[d]
                    ny = y + dy[d]
                    nz = z + dz[d]
                    
                    # Check bounds and material
                    if (0 <= nx < chunk_size and 
                        0 <= ny < chunk_size and 
                        0 <= nz < chunk_size):
                        neighbor_material = voxels[nx * chunk_size * chunk_size + ny * chunk_size + nz]
                        if neighbor_material != 0:
                            continue # Not a surface face
                    
                    # Determine face attributes
                    axis = d // 2
                    positive = d % 2 != 0
                    
                    # Map to u, v, depth
                    if axis == 0: # X-axis
                        depth = x
                        u = y
                        v = z
                    elif axis == 1: # Y-axis
                        depth = y
                        u = z
                        v = x
                    else: # Z-axis
                        depth = z
                        u = x
                        v = y
                    
                    # Encode token
                    # Exact bit-packing for individual face tokens is not specified in the sources [S1, unofficial].
                    token = (material << 32) | (depth << 24) | (u << 16) | (v << 8) | (axis << 2) | int(positive)
                    faces.append(token)
    
    faces.sort()
    return faces
```

## Subtopics

### [Validator Comparison Logic](3797-validator-comparison-logic.md)
This method provides the verification logic for GPU-based greedy meshing by comparing generated face vectors against reference meshes [G]. It differs from the parent [Validator Reference Face Reconstruction](3794-validator-reference-face-reconstruction.md) by focusing on the comparison and error reporting phase rather than the generation of the reference data itself [G]. It is chosen when bitwise reproducibility of the mesh extraction process must be verified [G].

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the number of voxels ($16^3 = 4096$), plus $O(F \log F)$ for sorting, where $F$ is the number of faces [G].
- Memory Complexity: $O(F)$ to store the face list [G].
- Measured Performance: Not given in the sources.
- Strengths: Provides a deterministic, correct ground truth for validation [G]. Simple to implement [G].
- Limitations: Slow compared to GPU extraction [G]. Not suitable for real-time rendering [G].
- When to use: During development and testing of GPU meshing algorithms to ensure correctness [G].
- When not to use: In production rendering pipelines where performance is critical [G].

## Background
Originated in the context of streaming binary greedy meshing for voxel engines, specifically in the `streaming-binary-greedy-meshing` project [S1, unofficial]. It is used to validate the output of GPU-based face extraction kernels [S1, unofficial]. Author, date, and specific paper details are not given in the sources.

## Sources
[S1] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
