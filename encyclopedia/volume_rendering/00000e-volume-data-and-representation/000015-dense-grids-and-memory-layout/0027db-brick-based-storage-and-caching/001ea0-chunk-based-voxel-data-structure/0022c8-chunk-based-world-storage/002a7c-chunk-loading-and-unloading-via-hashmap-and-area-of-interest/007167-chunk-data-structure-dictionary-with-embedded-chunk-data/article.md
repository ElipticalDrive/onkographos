# Chunk Data Structure — Dictionary with Embedded Chunk Data

Also called: Dictionary with Embedded Chunk Data

## Overview
This method stores voxel chunks in a hash map (dictionary) where each value is a chunk object containing its block data directly. It solves the problem of managing an infinite voxel world by keeping only active chunks in memory. It differs from [Dual Dictionary Chunk Storage](4766-chunk-loading-and-unloading-via-hashmap-and-area-of-interest.md) by avoiding separate lookups for metadata and data, though it is still reported as slower than array-based shifting approaches due to dictionary overhead [S1, unofficial].

## How it works
**Inputs:** Chunk coordinates (e.g., $x, z$), chunk size (e.g., $16 \times 16 \times 16$).
**Outputs:** A chunk object containing the voxel array.

**Data Structures:**
1.  `ChunkMap`: A hash map (dictionary) mapping chunk coordinates to `Chunk` objects.
2.  `Chunk`: An object containing:
    *   `Coordinates`: The $(x, z)$ index.
    *   `Blocks`: A contiguous array or list of voxel data (e.g., `byte` or `int` per voxel).

**Steps:**
1.  Calculate the chunk coordinates $(cx, cz)$ for the requested voxel position.
2.  Check if `ChunkMap` contains the key $(cx, cz)$.
3.  If present, return the `Chunk` object.
4.  If not present, generate or load the chunk data, create a `Chunk` object, store it in `ChunkMap`, and return it.

**Pitfalls:**
*   **Performance:** Dictionary lookups and object allocation can be slow compared to fixed arrays [S1, unofficial].
*   **Memory:** Each chunk object has overhead (object headers, references) compared to a raw array [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Chunk:
    def __init__(self, x, z, blocks):
        self.x = x
        self.z = z
        self.blocks = blocks  # Array of size 16*16*16

class World:
    def __init__(self):
        self.chunks = {}  # Dictionary mapping (x, z) to Chunk

    def get_chunk(self, x, z):
        key = (x, z)
        if key in self.chunks:
            return self.chunks[key]
        else:
            # Generate or load blocks [G]
            blocks = self.generate_blocks(x, z)
            chunk = Chunk(x, z, blocks)
            self.chunks[key] = chunk
            return chunk

    def generate_blocks(self, x, z):
        # Placeholder for generation logic [G]
        return [0] * (16 * 16 * 16)
```

## Subtopics

### Dual Dictionary Chunk Storage
Stores chunks in a dictionary and block data in a separate dictionary. This causes chunk loading speed to drop by about 10 times due to two dictionary lookups and multiple memory jumps to access chunk data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ average for dictionary lookup, but with high constant factors due to hashing and object indirection.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of active chunks, plus overhead per chunk object.
*   **Performance:** Reported as "pretty slow at chunk loading" and "unplayable" in some implementations compared to array-based shifting [S1, unofficial].
*   **Strengths:** Simple to implement; dynamic memory usage.
*   **Limitations:** High latency due to dictionary overhead and cache misses from scattered memory access [S1, unofficial].
*   **When to use:** Prototyping or when memory constraints require dynamic allocation over fixed arrays.
*   **When not to use:** High-performance real-time rendering where frame-time consistency is critical.

## Background
Originated from discussions on implementing infinite voxel terrain in Unity, specifically comparing dictionary-based approaches to array-based shifting [S1, unofficial]. It is a common pattern in early-stage voxel engine development before optimization to fixed-size arrays or more sophisticated spatial structures.

## Sources
[S1] Unity3D: Efficiently loading chunks in an infinite voxel engine, https://gamedev.stackexchange.com/questions/101463/unity3d-efficiently-loading-chunks-in-an-infinite-voxel-engine, fetched 2026-10-26 (unofficial)
