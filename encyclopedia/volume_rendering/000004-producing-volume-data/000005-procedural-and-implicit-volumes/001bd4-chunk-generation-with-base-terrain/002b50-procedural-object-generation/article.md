# Procedural Object Generation

Also called: PlaceHouseBasic

## Overview
Constructs a basic rectangular house with a floor, roof, walls, a door on a specified side, and supporting legs extending down to solid ground [S1, unofficial]. It differs from sibling methods like [Procedural Terrain Heightmap Generation](1850-procedural-terrain-heightmap-generation.md) by placing discrete, structured architectural primitives rather than continuous terrain surfaces, and from [Procedural Cave Generation](2579-procedural-cave-generation.md) by generating enclosed, man-made geometry rather than voids.

## How it works
**Inputs:**
- `cornerX`, `cornerZ`: Integer coordinates of the house's front-left corner on the ground plane.
- `length`: Integer width of the house along the X-axis.
- `width`: Integer depth of the house along the Z-axis.
- `height`: Integer height of the house walls.
- `doorSide`: Integer 0-3 indicating the wall side for the door (0: -Z, 1: +X, 2: +Z, 3: -X).
- `floorID`, `roofID`, `wallID`: Integer block types for respective components.
- `GetBlockData(x, y, z)`: Function returning the block ID at coordinates.
- `SetBlockData(x, y, z, id)`: Function setting the block ID at coordinates.

**Data Structures:**
- The volume is a 3D grid of block IDs.
- Block ID `0` represents air.
- Block ID `4` represents liquid.
- Block ID `6` represents wood (used for legs).

**Steps:**
1. **Clear Volume:** Define a padding `pad = 2`. Call `SetRectVolume(cornerX - pad, cornerX + length + pad - 1, cornerY, cornerY + height + pad, cornerZ - pad, cornerZ + width + pad - 1, 0)` to clear the area where the house will be built [S1, unofficial].
2. **Place Floor:** Loop `x` from `cornerX - pad` to `cornerX + length + pad` (exclusive) and `z` from `cornerZ - pad` to `cornerZ + width + pad` (exclusive). Set block at `(x, cornerY, z)` to `floorID` [S1, unofficial].
3. **Place Roof:** Loop `x` from `cornerX` to `cornerX + length` (exclusive) and `z` from `cornerZ` to `cornerZ + width` (exclusive). Set block at `(x, cornerY + height - 1, z)` to `roofID` [S1, unofficial].
4. **Place X-Side Walls:** Loop `y` from `cornerY + 1` to `cornerY + height - 1` (exclusive) and `x` from `cornerX` to `cornerX + length` (exclusive). Set blocks at `(x, y, cornerZ)` and `(x, y, cornerZ + width - 1)` to `wallID` [S1, unofficial].
5. **Place Z-Side Walls:** Loop `y` from `cornerY + 1` to `cornerY + height - 1` (exclusive) and `z` from `cornerZ` to `cornerZ + width` (exclusive). Set blocks at `(cornerX, y, z)` and `(cornerX + length - 1, y, z)` to `wallID` [S1, unofficial].
6. **Carve Door:** Create a 2x3 opening (width 2, height 3) centered horizontally on the wall specified by `doorSide`. Set these blocks to `0` (air) [S1, unofficial].
7. **Place Legs:** For each of the four corners (adjusted by `pad`), start at `y = cornerY - 1`. While `y > 0` and the block below `(x, y-1, z)` is air (`0`) or liquid (`4`), set block at `(x, y, z)` to `6` (wood) and decrement `y` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def PlaceHouseBasic(cornerX, cornerZ, length, width, height, doorSide, floorID, roofID, wallID, GetBlockData, SetBlockData):
    pad = 2
    # Clear volume
    for x in range(cornerX - pad, cornerX + length + pad):
        for y in range(cornerY, cornerY + height + pad):
            for z in range(cornerZ - pad, cornerZ + width + pad):
                SetBlockData(x, y, z, 0)

    # Place floor
    for x in range(cornerX - pad, cornerX + length + pad):
        for z in range(cornerZ - pad, cornerZ + width + pad):
            SetBlockData(x, cornerY, z, floorID)

    # Place roof
    for x in range(cornerX, cornerX + length):
        for z in range(cornerZ, cornerZ + width):
            SetBlockData(x, cornerY + height - 1, z, roofID)

    # Place x-side walls
    for y in range(cornerY + 1, cornerY + height - 1):
        for x in range(cornerX, cornerX + length):
            SetBlockData(x, y, cornerZ, wallID)
            SetBlockData(x, y, cornerZ + width - 1, wallID)

    # Place z-side walls
    for y in range(cornerY + 1, cornerY + height - 1):
        for z in range(cornerZ, cornerZ + width):
            SetBlockData(cornerX, y, z, wallID)
            SetBlockData(cornerX + length - 1, y, z, wallID)

    # Carve door
    # Determine door position based on doorSide (0: -Z, 1: +X, 2: +Z, 3: -X)
    # Centered horizontally, 2 wide, 3 high
    # Implementation details for specific coordinates depend on doorSide mapping [S1, unofficial]
    # For example, if doorSide == 0 (-Z wall):
    #   x_start = cornerX + (length // 2) - 1
    #   z = cornerZ
    #   for x in range(x_start, x_start + 2):
    #       for y in range(cornerY + 1, cornerY + 4):
    #           SetBlockData(x, y, z, 0)
    # Similar logic for other sides [S1, unofficial]

    # Place legs
    corners = [
        (cornerX - pad, cornerZ - pad),
        (cornerX + length + pad - 1, cornerZ - pad),
        (cornerX - pad, cornerZ + width + pad - 1),
        (cornerX + length + pad - 1, cornerZ + width + pad - 1)
    ]
    for (x, z) in corners:
        y = cornerY - 1
        while y > 0:
            block_below = GetBlockData(x, y - 1, z)
            if block_below == 0 or block_below == 4: # Air or Liquid
                SetBlockData(x, y, z, 6) # Wood
                y -= 1
            else:
                break
```

**Parameters:**
- `pad = 2`: Padding around the house footprint for floor and leg placement [S1, unofficial].
- `doorSide`: 0-3, mapping to specific walls [S1, unofficial].

**Edge Cases:**
- If `height` is small, the wall loops may not execute, resulting in no walls [S1, unofficial].
- If the ground is not reachable (e.g., floating island), legs may not extend fully [S1, unofficial].
- Precision issues are minimal as coordinates are integers [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(L \cdot W + H \cdot (L + W) + C)$, where $L$ is length, $W$ is width, $H$ is height, and $C$ is the number of legs (constant 4). The dominant term is wall placement $O(H \cdot (L + W))$.
- **Memory Complexity:** $O(1)$ auxiliary space, as it operates directly on the volume grid.
- **Strengths:** Simple, fast, deterministic structure generation.
- **Limitations:** Only generates a single house type. No variation in size or shape.
- **When to use:** When a simple, recognizable house structure is needed in a voxel world.
- **When not to use:** When complex, varied, or organic structures are required.

## Background
Originated in the "bad-minecraft" project by mayakeele [S1, unofficial]. It is part of a procedural generation pipeline for voxel worlds, specifically handling object placement after terrain generation.

## Sources
[S1] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
