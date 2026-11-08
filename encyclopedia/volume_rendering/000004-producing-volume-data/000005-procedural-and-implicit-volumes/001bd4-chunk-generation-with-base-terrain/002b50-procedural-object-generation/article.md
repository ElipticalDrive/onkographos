# Procedural Object Generation

Also called: PlaceHouseBasic

## Overview
Constructs a basic rectangular house with a floor, roof, walls, a door on a specified side, and supporting legs extending down to solid ground [S1, unofficial].

## How it works
**Inputs:**
- `cornerX`, `cornerY`, `cornerZ`: Integer coordinates of the house's front-left corner [S1, unofficial].
- `length`: Integer width of the house along the X-axis [S1, unofficial].
- `width`: Integer depth of the house along the Z-axis [S1, unofficial].
- `height`: Integer height of the house walls [S1, unofficial].
- `doorSide`: Integer 0-3 indicating the wall side for the door [S1, unofficial].
- `floorID`, `roofID`, `wallID`: Integer block types for respective components [S1, unofficial].
- `GetBlockData(x, y, z)`: Function returning the block ID at coordinates [S1, unofficial].
- `SetBlockData(x, y, z, id)`: Function setting the block ID at coordinates [S1, unofficial].

**Data Structures:**
- The volume is a 3D grid of block IDs [S1, unofficial].
- Block ID `0` represents air [S1, unofficial].
- Block ID `4` represents liquid [S1, unofficial].
- Block ID `6` represents wood, used for legs [S1, unofficial].

**Steps:**
1. **Clear Volume:** Set `pad = 2` [S1, unofficial]. Call `SetRectVolume(cornerX - pad, cornerX + length + pad - 1, cornerY, cornerY + height + pad, cornerZ - pad, cornerZ + width + pad - 1, 0)` to clear the target bounding box [S1, unofficial].
2. **Place Floor:** Loop `x` from `cornerX - pad` to `cornerX + length + pad` (exclusive) and `z` from `cornerZ - pad` to `cornerZ + width + pad` (exclusive) [S1, unofficial]. Set block at `(x, cornerY, z)` to `floorID` [S1, unofficial].
3. **Place Roof:** Loop `x` from `cornerX` to `cornerX + length` (exclusive) and `z` from `cornerZ` to `cornerZ + width` (exclusive) [S1, unofficial]. Set block at `(x, cornerY + height - 1, z)` to `roofID` [S1, unofficial].
4. **Place X-Side Walls:** Loop `y` from `cornerY + 1` to `cornerY + height - 1` (exclusive) and `x` from `cornerX` to `cornerX + length` (exclusive) [S1, unofficial]. Set blocks at `(x, y, cornerZ)` and `(x, y, cornerZ + width - 1)` to `wallID` [S1, unofficial].
5. **Place Z-Side Walls:** Loop `y` from `cornerY + 1` to `cornerY + height - 1` (exclusive) and `z` from `cornerZ` to `cornerZ + width` (exclusive) [S1, unofficial]. Set blocks at `(cornerX, y, z)` and `(cornerX + length - 1, y, z)` to `wallID` [S1, unofficial].
6. **Carve Door:** Based on `doorSide` (0-3), set specific blocks to `0` (air) to create a 2×3 opening (width 2, height 3) on the respective wall side, centered horizontally [S1, unofficial].
7. **Place Legs:** For each of the four corners (adjusted by `pad`), start at `y = cornerY - 1` [S1, unofficial]. While `y > 0` and the block below is air (`0`) or liquid (`4`), set block at `(x, y, z)` to `6` (wood) and decrement `y` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def PlaceHouseBasic(cornerX, cornerY, cornerZ, length, width, height, doorSide, floorID, roofID, wallID, GetBlockData, SetBlockData):
    pad = 2
    # 1. Clear volume
    for x in range(cornerX - pad, cornerX + length + pad):
        for y in range(cornerY, cornerY + height + pad):
            for z in range(cornerZ - pad, cornerZ + width + pad):
                SetBlockData(x, y, z, 0)

    # 2. Place floor
    for x in range(cornerX - pad, cornerX + length + pad):
        for z in range(cornerZ - pad, cornerZ + width + pad):
            SetBlockData(x, cornerY, z, floorID)

    # 3. Place roof
    for x in range(cornerX, cornerX + length):
        for z in range(cornerZ, cornerZ + width):
            SetBlockData(x, cornerY + height - 1, z, roofID)

    # 4. Place x-side walls
    for y in range(cornerY + 1, cornerY + height - 1):
        for x in range(cornerX, cornerX + length):
            SetBlockData(x, y, cornerZ, wallID)
            SetBlockData(x, y, cornerZ + width - 1, wallID)

    # 5. Place z-side walls
    for y in range(cornerY + 1, cornerY + height - 1):
        for z in range(cornerZ, cornerZ + width):
            SetBlockData(cornerX, y, z, wallID)
            SetBlockData(cornerX + length - 1, y, z, wallID)

    # 6. Carve door (2x3 opening, centered horizontally)
    # doorSide: 0=-Z, 1=+X, 2=+Z, 3=-X
    if doorSide == 0:  # -Z wall
        x_start = cornerX + (length // 2) - 1
        for x in range(x_start, x_start + 2):
            for y in range(cornerY + 1, cornerY + 4):
                SetBlockData(x, y, cornerZ, 0)
    elif doorSide == 1:  # +X wall
        z_start = cornerZ + (width // 2) - 1
        for z in range(z_start, z_start + 2):
            for y in range(cornerY + 1, cornerY + 4):
                SetBlockData(cornerX + length - 1, y, z, 0)
    elif doorSide == 2:  # +Z wall
        x_start = cornerX + (length // 2) - 1
        for x in range(x_start, x_start + 2):
            for y in range(cornerY + 1, cornerY + 4):
                SetBlockData(x, y, cornerZ + width - 1, 0)
    elif doorSide == 3:  # -X wall
        z_start = cornerZ + (width // 2) - 1
        for z in range(z_start, z_start + 2):
            for y in range(cornerY + 1, cornerY + 4):
                SetBlockData(cornerX, y, z, 0)

    # 7. Place legs
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
            if block_below == 0 or block_below == 4:
                SetBlockData(x, y, z, 6)
                y -= 1
            else:
                break
```

**Parameters:**
- `pad = 2`: Padding around the house footprint for floor and leg placement [S1, unofficial].
- `doorSide`: 0-3, mapping to specific walls [S1, unofficial].

**Edge Cases:**
- Not given in the sources.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
