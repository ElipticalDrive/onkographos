# Deterministic Fixture Worlds

Also called: Deterministic World Generation

## Overview
Deterministic Fixture Worlds are pure functions that generate specific, small-scale voxel structures based solely on a `VolumeId`, ensuring no reliance on random number generators (RNG), system clocks, or I/O [S1, unofficial]. They serve as regression tests and acceptance criteria for voxel engines, providing predictable geometric primitives like slabs, towers, and bridges [S1, unofficial]. Unlike general procedural generation which uses noise for terrain, fixture worlds are hand-crafted logical shapes defined by exact brick coordinates [S1, unofficial].

## How it works
The method consists of four specific generator functions and a verification digest. All functions are pure and deterministic.

### Inputs and Outputs
- **Input**: A `VolumeId` (used as a seed/context identifier, though the geometry is fixed) and, for the digest, the `Volume` state.
- **Output**: A set of voxel blocks (bricks) placed in the world grid, or a hash digest of the resulting world state.

### Data Structures
- **Brick**: A unit cube in the voxel grid. Coordinates are integer triplets $(x, y, z)$.
- **Volume**: A collection of resident bricks. Each brick has fields: `coord` (integer triplet), `content hash` (identifier of block type), and `revision` (integer).
- **Hash**: BLAKE3 hash output, platform-stable.

### Steps

1. **Generate Flat Terrain**
   - Creates a $2 \times 2 \times 2$ brick bounded stone slab.
   - The slab is positioned under a resident (non-absent) air layer.
   - All bricks in this structure use uniform storage.

2. **Generate Hollow Tower**
   - Creates an 8-brick structure.
   - Forms a 2-cell-thick stone shell.
   - The structure straddles the origin, occupying negative coordinates and crossing brick boundaries.
   - Verification confirms brushes touch exactly eight bricks when centered on the corner where eight meet; a cell center exactly on the radius is included [S1, unofficial].

3. **Generate Cross Brick Bridge**
   - Creates a $2 \times 2$-cell beam.
   - The beam spans four X bricks.
   - This matches the "bridge-cross-brick" acceptance fixture [S1, unofficial].

4. **Generate Sloped Terrain**
   - Creates a small, excavatable staircase.
   - The surface is dirt, overlying stone.

5. **Compute Digest**
   - Computes a platform-stable BLAKE3 hash over the `Volume`.
   - Iterates over resident bricks in canonical order $(z, y, x)$.
   - Includes fields: `coord`, `content hash`, and `revision`.
   - This digest is a local regression check and is explicitly **not** the replicated `spall.topology.v1` hash [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_flat_terrain(volume_id: str) -> List[Brick]:
    """
    Generates a 2x2x2-brick bounded stone slab under a resident air layer.
    Returns a list of Brick objects with uniform storage.
    """
    bricks = []
    # Define the 2x2x2 slab coordinates relative to a base point
    # Assuming base at (0,0,0) for simplicity, actual implementation may offset
    for x in range(0, 2):
        for y in range(0, 2):
            for z in range(0, 2):
                bricks.append(Brick(coord=(x, y, z), content="stone", revision=1))
    return bricks

def generate_hollow_tower(volume_id: str) -> List[Brick]:
    """
    Generates an 8-brick, 2-cell-thick stone shell straddling the origin.
    Occupies negative coordinates and crosses brick boundaries.
    """
    bricks = []
    # Straddling origin implies coordinates like (-1, -1, -1) to (0, 0, 0)
    # A 2-cell thick shell around the origin corner involves specific boundary bricks
    # Based on "touches exactly eight bricks when centred on the corner"
    # Coordinates: (-1, -1, -1), (-1, -1, 0), (-1, 0, -1), (-1, 0, 0),
    #               (0, -1, -1), (0, -1, 0), (0, 0, -1), (0, 0, 0)
    coords = [
        (-1, -1, -1), (-1, -1, 0), (-1, 0, -1), (-1, 0, 0),
        (0, -1, -1), (0, -1, 0), (0, 0, -1), (0, 0, 0)
    ]
    for coord in coords:
        bricks.append(Brick(coord=coord, content="stone", revision=1))
    return bricks

def generate_cross_brick_bridge(volume_id: str) -> List[Brick]:
    """
    Generates a 2x2-cell beam spanning four X bricks.
    """
    bricks = []
    # Spanning four X bricks implies a length of 4 in X, width 2 in Y, height 2 in Z
    # Or similar configuration matching "2x2-cell beam"
    # Assuming beam along X axis, 2 cells high, 2 cells wide
    for x in range(0, 4):
        for y in range(0, 2):
            for z in range(0, 2):
                bricks.append(Brick(coord=(x, y, z), content="stone", revision=1))
    return bricks

def generate_sloped_terrain(volume_id: str) -> List[Brick]:
    """
    Generates a small excavatable staircase with dirt surface over stone.
    """
    bricks = []
    # Simple staircase: 3 steps
    # Step 1: z=0, x=0..2, y=0..1 (stone)
    # Step 2: z=1, x=1..3, y=0..1 (stone)
    # Step 3: z=2, x=2..4, y=0..1 (stone)
    # Surface: dirt on top of each step
    # Step 1 surface: z=1, x=0..2, y=0..1 (dirt)
    # Step 2 surface: z=2, x=1..3, y=0..1 (dirt)
    # Step 3 surface: z=3, x=2..4, y=0..1 (dirt)
    
    # Step 1
    for x in range(0, 3):
        for y in range(0, 2):
            bricks.append(Brick(coord=(x, y, 0), content="stone", revision=1))
            bricks.append(Brick(coord=(x, y, 1), content="dirt", revision=1))
            
    # Step 2
    for x in range(1, 4):
        for y in range(0, 2):
            bricks.append(Brick(coord=(x, y, 1), content="stone", revision=1))
            bricks.append(Brick(coord=(x, y, 2), content="dirt", revision=1))
            
    # Step 3
    for x in range(2, 5):
        for y in range(0, 2):
            bricks.append(Brick(coord=(x, y, 2), content="stone", revision=1))
            bricks.append(Brick(coord=(x, y, 3), content="dirt", revision=1))
            
    return bricks

def digest_volume(volume: Volume) -> str:
    """
    Computes a platform-stable BLAKE3 hash over resident bricks.
    Order: (z, y, x).
    Fields: coord, content hash, revision.
    """
    import hashlib
    
    # Get resident bricks in canonical order (z, y, x)
    bricks = sorted(volume.resident_brick_coords(), key=lambda b: (b.coord.z, b.coord.y, b.coord.x))
    
    # Create a string representation of the brick data
    data_parts = []
    for brick in bricks:
        # coord is integer triplet
        coord_str = f"{brick.coord.x},{brick.coord.y},{brick.coord.z}"
        # content hash is a hash identifier
        content_hash = brick.content_hash
        # revision is an integer
        revision = brick.revision
        
        data_parts.append(f"{coord_str}:{content_hash}:{revision}")
    
    data_string = "|".join(data_parts)
    
    # Compute BLAKE3 hash
    # Note: BLAKE3 is not in standard Python hashlib, using a placeholder for the algorithm
    # In implementation, use a BLAKE3 library
    hash_obj = hashlib.blake3(data_string.encode('utf-8'))
    return hash_obj.hexdigest()
```

### Parameters
- **VolumeId**: String identifier for the volume context [S1, unofficial].
- **Noise**: Not used in fixture worlds (used in general procedural generation) [S2, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Ray results are explicitly not required to be bit-identical across machines; hit validation is server-side [S1, unofficial]. Integer brush plans stay fully deterministic [S1, unofficial].
- **Boundary**: A cell center exactly on the radius is included in the hollow tower generation [S1, unofficial].
- **Digest Confusion**: The digest is a local regression check and is explicitly **not** the replicated `spall.topology.v1` hash [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ for generation, as the structures are fixed-size and small. $O(N)$ for digest, where $N$ is the number of resident bricks.
- **Memory Complexity**: $O(1)$ for generation, as the structures are fixed-size. $O(N)$ for digest, where $N$ is the number of resident bricks.
- **Strengths**: Deterministic, platform-stable (for digest), easy to test, no RNG dependencies.
- **Limitations**: Only suitable for small, fixed structures. Not suitable for large-scale terrain generation.
- **When to use**: Regression testing, acceptance criteria, debugging voxel engine geometry.
- **When not to use**: Large-scale world generation, dynamic content.

## Background
Deterministic Fixture Worlds were introduced in the `spall` voxel engine project [S1, unofficial]. They are part of a broader effort to ensure voxel engine correctness through pure functions and deterministic testing [S1, unofficial]. The concept is distinct from general procedural generation, which often relies on noise and hashing for terrain and object placement [S2, unofficial].

## Sources
[S1] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
[S2] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
