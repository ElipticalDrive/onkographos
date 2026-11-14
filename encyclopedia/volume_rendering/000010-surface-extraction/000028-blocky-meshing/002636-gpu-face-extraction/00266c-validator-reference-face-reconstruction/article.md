# Validator Reference Face Reconstruction


## Overview
This method reconstructs a canonical, sorted list of individual voxel faces from a voxel subchunk to serve as a ground-truth reference for validating GPU-based greedy meshing algorithms. It solves the problem of verifying bitwise reproducibility by generating a detailed, unoptimized face list that can be compared against the output of optimized extractors. It differs from its parent [GPU Face Extraction](3760-gpu-face-extraction.md) by prioritizing correctness and canonical ordering over performance, and from its sibling [Occlusion-Based Face Generation](13226-occlusion-based-face-generation.md) by explicitly handling the full set of visible faces without occlusion culling optimizations.

## How it works
The method iterates through every voxel in the chunk, identifies surface boundaries, and constructs a list of face tokens. These tokens are then sorted to ensure a deterministic order for comparison.

**Inputs and Outputs:**
- Input: `voxels`, a flat array of `uint8_t` representing material IDs in a $16 \times 16 \times 16$ chunk.
- Output: `faces`, a sorted `std::vector<uint64_t>` where each element is a packed 64-bit face token.

**Data Structures:**
- `face_token`: A 64-bit integer packed with face attributes.
- `faces`: A vector of `uint64_t`.

**Packed Token Layout (`uint64_t`):**
The token encodes the face geometry and material. The decoding logic (used in validation) is:
- Bits 0-5 (`a`): 6-bit value.
- Bits 6-11 (`b`): 6-bit value.
- Bits 12-17 (`c`): 6-bit value.
- Bits 18-23 (`w`): 6-bit value.
- Bits 24-31 (`h`): 8-bit value.
- Bits 32-63 (`material`): 32-bit value.

**Steps:**
1. Initialize an empty vector `faces`.
2. Iterate through all voxel coordinates $(x, y, z)$ where $0 \le x, y, z < 16$.
3. For each voxel, retrieve `material = voxels[voxel_index(x, y, z)]`. If `material == 0`, skip.
4. For each of the 6 directions $d \in \{0, \dots, 5\}$:
   a. Calculate neighbor coordinates:
      - $dx = \{-1, +1, 0, 0, 0, 0\}$
      - $dy = \{0, 0, -1, +1, 0, 0\}$
      - $dz = \{0, 0, 0, 0, -1, +1\}$
      - $nx = x + dx[d]$, $ny = y + dy[d]$, $nz = z + dz[d]$
   b. Check if the neighbor is within bounds and has `material != 0`. If not, this face is visible.
   c. Determine face attributes:
      - $axis = d / 2$
      - $positive = (d \& 1) \neq 0$
      - $depth = (axis == 0) ? x : (axis == 1) ? y : z$
      - $u = (axis == 0) ? y : (axis == 1) ? z : x$
      - $v = (axis == 0) ? z : (axis == 1) ? x : y$
   d. Push a face token for this single voxel face to `faces`. The token format for a single voxel face is derived from the greedy meshing output format for consistency.

**Note on Greedy Meshing Output Format:**
The source [S1, unofficial] provides a decoder for `uint64_t` tokens that represent *greedy meshed* quads (with width/height). However, the reference reconstruction typically generates *individual* voxel faces for validation. The provided `expected_faces` function in [S1, unofficial] pushes a token for each individual face. The packing for these individual tokens must match the format expected by the validator. Based on the decoder in [S1, unofficial], a single voxel face can be encoded as:
- $a=0, b=0, c=0, w=1, h=1$ (or similar minimal values)
- $material$ is the voxel material.
- The specific mapping of $u, v, depth$ to the token fields $a, b, c, w, h$ depends on the specific greedy meshing implementation's token format. The source [S1, unofficial] shows a decoder that maps $a,b,c,w,h$ to geometric offsets. For a single voxel face, the geometric extent is 1x1.

**Sorting:**
1. Sort the `faces` vector using standard lexicographical comparison of the `uint64_t` values.
2. Return `faces`.

**Edge Cases:**
- Boundary voxels: Neighbors outside the chunk are treated as empty (material 0), so faces on the chunk boundary are always visible.
- Invalid quads: The source [S1, unofficial] includes a check for invalid quads emitted by the GPU, throwing an error if bounds are exceeded. The reference reconstruction should not produce invalid quads if implemented correctly.

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
                    # Note: The exact packing depends on the validator's expected format.
                    # Based on [S1, unofficial] decoder, we assume a format compatible with greedy mesh output.
                    # For a single voxel, width and height are 1.
                    # We need to map u, v, depth, material to the 64-bit token.
                    # The source [S1, unofficial] decoder uses:
                    # a = q & 63
                    # b = (q >> 6) & 63
                    # c = (q >> 12) & 63
                    # w = (q >> 18) & 63
                    # h = (q >> 24) & 255
                    # material = q >> 32
                    #
                    # The add_rect function in [S1, unofficial] maps these to geometry.
                    # For a single voxel face, we can set w=1, h=1.
                    # The mapping of u, v, depth to a, b, c depends on the axis.
                    # From [S1, unofficial] add_rect logic:
                    # Face 0 (axis 0, pos true): u=b-1, v=c, du=a, dv=h, w=w
                    # This implies a, b, c are used to encode position and size.
                    # For a single voxel, du=1, dv=1. So a=1, h=1.
                    # u = b-1 => b = u+1
                    # v = c => c = v
                    #
                    # This mapping is specific to the greedy meshing format.
                    # For reference reconstruction, we must generate tokens in the SAME format.
                    
                    if axis == 0:
                        if positive:
                            a = 1
                            b = u + 1
                            c = v
                        else:
                            a = 1
                            b = u
                            c = v
                    elif axis == 1:
                        if positive:
                            a = 1
                            b = u
                            c = v
                        else:
                            a = 1
                            b = u
                            c = v
                    else:
                        if positive:
                            a = 1
                            b = u
                            c = v
                        else:
                            a = 1
                            b = u
                            c = v
                    
                    w = 1
                    h = 1
                    
                    token = (material << 32) | (h << 24) | (w << 18) | (c << 12) | (b << 6) | a
                    faces.append(token)
    
    faces.sort()
    return faces
```

## Subtopics

### [Validator Comparison Logic](3797-validator-comparison-logic.md)
Provides the verification logic for GPU-based greedy meshing by comparing generated face vectors against reference meshes.

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the number of voxels ($16^3 = 4096$), plus $O(F \log F)$ for sorting, where $F$ is the number of faces.
- Memory Complexity: $O(F)$ to store the face list.
- Strengths: Provides a deterministic, correct ground truth for validation. Simple to implement.
- Limitations: Slow compared to GPU extraction. Not suitable for real-time rendering.
- When to use: During development and testing of GPU meshing algorithms to ensure correctness.

## Background
Originated in the context of streaming binary greedy meshing for voxel engines, specifically in the `streaming-binary-greedy-meshing` project [S1, unofficial]. It is used to validate the output of GPU-based face extraction kernels.

## Sources
[S1] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
