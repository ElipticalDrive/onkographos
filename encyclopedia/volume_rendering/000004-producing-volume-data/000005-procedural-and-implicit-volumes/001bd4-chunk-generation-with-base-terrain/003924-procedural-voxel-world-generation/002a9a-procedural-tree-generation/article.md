# Procedural Tree Generation


## Overview
Procedural tree generation is a method for populating a voxel world with vegetation by algorithmically placing trunk and leaf blocks at valid surface coordinates. It solves the problem of creating natural-looking, non-uniform flora in a grid-based environment without storing explicit mesh data for every tree. It differs from its sibling [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md), which defines the static ground layer, by operating on top of that layer to inject vertical structures. It differs from [Procedural Voxel World Generation](3891-procedural-voxel-world-generation.md) by focusing specifically on the geometric construction of individual entities rather than global terrain noise.

## How it works
The method determines tree placement validity, generates the trunk, and constructs the leaf canopy using one of several geometric primitives.

**Inputs and Outputs:**
- **Inputs:** Chunk coordinates $(x, z)$, surface height $y$, random seed/state, tree parameters (height, radius, probability).
- **Outputs:** Modified voxel array where trunk and leaf block IDs are written.

**Data Structures:**
- `voxels`: A 1D array or 3D grid of `uint8` block IDs.
- `get_index(x, y, z)`: Helper to map 3D coordinates to the 1D array index.
- `RandomChance(p)`: Returns true with probability $p$.
- `Random()`: Returns a float in $[0, 1)$.

**Parameters:**
- `TREE_PROBABILITY`: Chance a valid grass block spawns a tree. Typical value: Not given in the sources.
- `TREE_HEIGHT`: Height of the trunk. Typical value: Not given in the sources.
- `radius`: Radius of the leaf canopy. Typical value: Not given in the sources.
- `leafBlock`, `fruitBlock`, `wood`, `dirt`: Block IDs.

**Algorithm Steps (Base Variant):**
1. **Validation:** Check if the voxel at $(x, y, z)$ is GRASS. Perform a random check `Random() > TREE_PROBABILITY`; if false, abort. Verify bounds: $y + TREE\_HEIGHT$ and $x, z$ must be within chunk/world limits.
2. **Trunk Base:** Set `voxels[get_index(x, y, z)] = DIRT`.
3. **Trunk Body:** Loop $iy$ from 1 to $TREE\_HEIGHT - 2$. Set `voxels[get_index(x, y + iy, z)] = WOOD`.
4. **Canopy Generation:** Use nested loops for height $iy$, x-offset $ix$, and z-offset $iz$.
   - Calculate a shaping variable $m$ based on iteration $n$.
   - Apply a pattern check: `(ix + iz) % 4`.
   - If conditions are met, set the block to LEAVES.
5. **Canopy Top:** Set `voxels[get_index(x, y + TREE_HEIGHT - 2, z)] = LEAVES`.

**Pseudocode (Base Variant):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def place_tree_base(voxels, x, y, z, TREE_HEIGHT, TREE_PROBABILITY, DIRT, WOOD, LEAVES):
    # 1. Validation
    if get_block(voxels, x, y, z) != GRASS:
        return
    if random() <= TREE_PROBABILITY:
        return
    if not is_in_bounds(x, y + TREE_HEIGHT, z):
        return

    # 2. Trunk Base
    set_block(voxels, x, y, z, DIRT)

    # 3. Trunk Body
    for iy in range(1, TREE_HEIGHT - 1):
        set_block(voxels, x, y + iy, z, WOOD)

    # 4. Canopy Generation (Simplified from source description)
    # Note: Source [S1, unofficial] describes nested loops with specific pattern logic
    # The exact formula for 'm' and the full loop bounds are not fully specified in the provided text.
    # Implementation requires iterating over a volume around the trunk top.
    for iy in range(0, TREE_HEIGHT):
        for ix in range(-radius, radius + 1):
            for iz in range(-radius, radius + 1):
                # Pattern check from source
                if (ix + iz) % 4 == 0:
                    # Shaping logic using 'm' is not fully detailed in source
                    set_block(voxels, x + ix, y + iy, z + iz, LEAVES)

    # 5. Canopy Top
    set_block(voxels, x, y + TREE_HEIGHT - 2, z, LEAVES)
```

**Edge Cases and Pitfalls:**
- **Boundary Checks:** Failure to check chunk/world boundaries when placing blocks at $y + TREE\_HEIGHT$ can cause out-of-bounds memory access [S1, unofficial].
- **Precision:** Integer arithmetic is used for all coordinates; floating-point noise is not used in the base variant [S1, unofficial].
- **Degenerate Input:** If `TREE_HEIGHT` is less than 3, the trunk loop range may be invalid or empty.

## Subtopics

### PlaceTreeBall
Generates a tree with a spherical leaf canopy by checking the squared Euclidean distance from the center of the leaf volume [S2, unofficial].
- **Trunk:** Loop $y$ from $baseY$ to $baseY + height$ (exclusive), setting block at $(baseX, y, baseZ)$ to ID 6 [S2, unofficial].
- **Leaves:** Loop $y$ from $baseY + height - radius$ to $baseY + height + radius$ (inclusive), $x$ from $baseX - radius$ to $baseX + radius$, and $z$ from $baseZ - radius$ to $baseZ + radius$ [S2, unofficial].
- **Logic:** Calculate squared distances: $xDistSq = (baseX - x)^2$, $zDistSq = (baseZ - z)^2$, $yDistSq = (baseY + height - y)^2$. If $xDistSq + zDistSq + yDistSq \le radius^2$ and the block is air (ID 0), apply `RandomChance` for leaf and fruit blocks [S2, unofficial].

### PlaceTreeBox
Generates a tree with a rectangular prism leaf canopy by iterating over a 3D box centered on the trunk and placing leaf blocks probabilistically [S2, unofficial].
- **Trunk:** Loop $y$ from $baseY$ to $baseY + trunkHeight$ (exclusive), setting block at $(baseX, y, baseZ)$ to ID 6 [S2, unofficial].
- **Leaves:** Loop $x$ from $baseX - leafWidth$ to $baseX + leafWidth$, $y$ from $baseY + trunkHeight$ to $baseY + trunkHeight + leafHeight$, and $z$ from $baseZ - leafWidth$ to $baseZ + leafWidth$ [S2, unofficial].
- **Logic:** If the target block is air (ID 0), call `RandomChance(leafPercent)` (default 7) for leaves and `RandomChance(fruitPercent)` (default 18) for fruit [S2, unofficial].

### PlaceTreeCone
Generates a tree with a conical leaf canopy where the radius decreases linearly with height, using a circular cross-section check [S2, unofficial].
- **Trunk:** Loop $y$ from $baseY$ to $baseY + height$ (exclusive), setting block at $(baseX, y, baseZ)$ to ID 6 [S2, unofficial].
- **Leaves:** Loop $y$ from $baseY + leafBase$ to $baseY + height + 1$ (inclusive) [S2, unofficial].
- **Logic:** Calculate radial gradient: $radGradient = (baseY + height + 1 - y) / height$. Calculate current width: $thisWidth = \text{round}(radGradient \times radius)$. Iterate $x$ and $z$ within this width. If inside the circle and block is air (ID 0), apply `RandomChance` for leaf and fruit blocks [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(H \cdot R^2)$ for the base variant, where $H$ is tree height and $R$ is canopy radius. The box and cone variants are $O(W \cdot H \cdot W)$ where $W$ is leaf width/radius.
- **Memory Complexity:** $O(1)$ auxiliary space, modifying the voxel array in place.
- **Performance:** Fast for small trees; performance degrades with large canopy radii due to the volume of checks.
- **Strengths:** Simple to implement; low memory overhead; supports various shapes.
- **Limitations:** Rigid geometry; no inter-tree collision detection mentioned; relies on random chance for density in box/cone variants.

## Background
Procedural tree generation is a staple of voxel-based games like Minecraft. The base variant described in [S1, unofficial] is part of an unofficial Python implementation of Minecraft world generation. The variants described in [S2, unofficial] are from another unofficial JavaScript/TypeScript implementation ("bad-minecraft"). These methods predate modern GPU-based terrain generation and are often used for client-side or simple server-side chunk generation.

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
