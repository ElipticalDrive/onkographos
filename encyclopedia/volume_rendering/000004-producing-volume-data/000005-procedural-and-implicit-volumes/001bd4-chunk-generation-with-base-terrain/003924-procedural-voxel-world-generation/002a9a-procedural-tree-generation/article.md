# Procedural Tree Generation

Also called: Tree Generation

## Overview
Procedural tree generation places voxel-based trees onto a terrain surface by identifying valid ground positions and filling a defined volume with trunk and leaf blocks. It differs from sibling methods like [Deterministic Fixture Worlds](3941-deterministic-fixture-worlds.md) by focusing on organic, vertical structures rather than static geometric fixtures, and from [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md) by operating on specific placement logic rather than global layer composition. The primary challenge is ensuring trees fit within chunk boundaries and do not intersect with existing geometry.

## How it works
### Inputs and Outputs
**Inputs:**
- `globalX`, `globalZ`: The integer world coordinates of the chunk or placement origin.
- `terrainHeight(x, z)`: The height of the ground block at the placement coordinate.
- `chunkWidth`, `chunkHeight`: Dimensions of the voxel chunk.
- `TREE_PROBABILITY`: A float between 0 and 1 determining the likelihood of tree generation at a valid spot.
- `TREE_HEIGHT`: The height of the trunk in blocks.
- `world.ShouldPlaceTree(globalX, globalZ)`: A boolean function determining if a tree is allowed at this location (e.g., biome checks).

**Outputs:**
- Modified voxel array where `DIRT` (base), `WOOD` (trunk), and `LEAVES` (canopy) blocks are set.

### Data Structures
- `voxels`: A 3D array or flattened 1D array representing the chunk.
- `BlockType`: Enum or integer constants for `DIRT`, `WOOD`, `LEAVES`, `AIR` (0).

### Steps
1. **Validation**: Check if the surface block at `(globalX, globalZ)` is `GRASS`. If not, abort.
2. **Probability Check**: Generate a random number `rnd`. If `rnd > TREE_PROBABILITY`, abort.
3. **Boundary Check**: Ensure the tree fits within the chunk.
   - Check `y + TREE_HEIGHT` against `chunkHeight`.
   - Check `x` and `z` offsets against chunk borders to prevent leaf overflow (e.g., `x < 2` or `x > chunkWidth - 3`).
4. **Placement Logic**:
   - **Base**: Place `DIRT` at `(x, terrainHeight, z)`.
   - **Trunk**: Place `WOOD` from `terrainHeight + 1` to `terrainHeight + TREE_HEIGHT - 1` (or similar range depending on variant).
   - **Canopy**: Place `LEAVES` in a shape defined by the specific variant (cone, box, ball).

### Formulas and Parameters
- **Trunk Height**: `trunkHeight = 4` (typical value in [S2, S3, unofficial]).
- **Canopy Shape (Cone Variant)**: Uses Manhattan distance `distance = |offsetX| + |offsetZ|` [S2, unofficial].
- **Canopy Shape (Ball Variant)**: Euclidean distance squared `distSq = (baseX - x)^2 + (baseZ - z)^2 + (baseY + height - y)^2` compared to `radius^2` [S3, unofficial].
- **Canopy Shape (Box Variant)**: Rectangular bounds `baseX ± leafWidth`, `baseZ ± leafWidth`, `baseY + trunkHeight ± leafHeight` [S3, unofficial].

### Edge Cases and Pitfalls
- **Chunk Overflow**: Trees placed near chunk edges must be clipped or prevented to avoid visual seams or out-of-bounds writes.
- **Precision**: Integer arithmetic is sufficient for block placement.
- **Degenerate Input**: If `terrainHeight` is invalid (e.g., below sea level or above chunk height), placement must fail gracefully.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def place_tree(voxels, global_x, global_z, terrain_height, chunk_width, chunk_height, tree_height, tree_probability):
    # 1. Validation
    if get_block(voxels, global_x, terrain_height, global_z) != GRASS:
        return False
    
    # 2. Probability Check
    if random() > tree_probability:
        return False
        
    # 3. Boundary Check
    if global_x < 2 or global_x >= chunk_width - 2:
        return False
    if global_z < 2 or global_z >= chunk_width - 2:
        return False
    if terrain_height + tree_height + 2 >= chunk_height:
        return False
        
    # 4. Place Base
    set_block(voxels, global_x, terrain_height, global_z, DIRT)
    
    # 5. Place Trunk
    for y in range(terrain_height + 1, terrain_height + tree_height):
        set_block(voxels, global_x, y, global_z, WOOD)
        
    # 6. Place Canopy (Generic placeholder, specific variant logic follows)
    # This section is variant-specific. See Subtopics for details.
    
    return True
```

## Subtopics

### [PlaceTreeCone](5090-placetreecone.md)
PlaceTreeCone generates a tree with a cylindrical trunk and a leaf canopy shaped as a truncated cone or inverted cone with a circular cross-section check. It is distinguished by using a radial gradient to determine leaf width based on height and a Manhattan distance heuristic for the canopy shape in some implementations. Choose this when an organic, tapered look is desired. It differs from PlaceTreeBox by having a curved canopy profile.

### PlaceTreeBall
PlaceTreeBall generates a tree with a spherical leaf canopy by checking the squared Euclidean distance from the center of the leaf volume [S3, unofficial]. It is distinguished by its use of a sphere shape, providing a more uniform, bushy appearance. Choose this for stylized or low-poly trees. It differs from PlaceTreeCone by having a constant radius canopy rather than a tapered one.

### PlaceTreeBox
PlaceTreeBox generates a tree with a rectangular prism leaf canopy by iterating over a 3D box centered on the trunk and placing leaf blocks probabilistically [S3, unofficial]. It is distinguished by its simple, blocky shape which is computationally cheaper and easier to tune. Choose this for voxel-art styles where organic shapes are not required. It differs from PlaceTreeBall and PlaceTreeCone by lacking curvature.

## Operating characteristics
- **Time Complexity**: O(V) where V is the volume of the tree canopy. For a fixed tree size, this is O(1).
- **Memory Complexity**: O(1) additional memory, as it modifies the existing voxel array in place.
- **Performance**: Very fast. The primary cost is the random number generation and boundary checks.
- **Strengths**: Simple to implement, low memory overhead, easy to tune parameters.
- **Limitations**: Trees are static and do not adapt to terrain irregularities (e.g., slopes). Can look repetitive if not randomized.
- **When to use**: When performance is critical and organic variation is not required.
- **When not to use**: When high-fidelity, adaptive tree shapes are needed.

## Background
Procedural tree generation is a staple of voxel-based games like Minecraft. The method described here is derived from various open-source implementations [S1, S2, S3, unofficial]. The specific variants (Cone, Ball, Box) represent common simplifications used in early voxel engines to balance visual appeal with computational cost.

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] philip2012/voxel-forge, https://github.com/philip2012/voxel-forge, fetched 2026-10-27 (unofficial)
[S3] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
