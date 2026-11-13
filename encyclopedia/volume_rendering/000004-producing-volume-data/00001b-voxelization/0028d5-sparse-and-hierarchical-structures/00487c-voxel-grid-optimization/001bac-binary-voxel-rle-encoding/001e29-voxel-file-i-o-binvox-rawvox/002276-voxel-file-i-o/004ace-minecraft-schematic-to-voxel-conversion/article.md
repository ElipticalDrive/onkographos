# Minecraft Schematic to Voxel Conversion

Also called: SchematicToSchematic

## Overview
Converts a Minecraft `.schematic` file (NBT format) into a `FileToVox` Schematic structure by parsing block IDs, metadata, and colors, then iterating through the voxel grid to populate the output [S1, unofficial]. It differs from generic voxelization methods by directly interpreting Minecraft's specific block ID and metadata system rather than reconstructing geometry from meshes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A `.schematic` file containing NBT data with dimensions and block arrays.
- Output: A `FileToVox` Schematic structure containing a list of Voxel objects (position and color).

**Data Structures:**
- `RawSchematic`:
  - `Width`: Short
  - `Height`: Short
  - `Length`: Short
  - `Blocks`: ByteArray (block IDs)
  - `Data`: ByteArray (block metadata)
- `ColorMapping`: Dictionary mapping `Tuple<int, int>` (block ID, metadata) to `Color` (A, R, G, B). Loaded from `./schematics/config.txt` [S1, unofficial].
- `Voxel`: Contains coordinates $(x, y, z)$ and a `Color`.

**Parameters:**
- `mExcavate` (Boolean): If true, only add voxels connected to air. If false, add all non-zero blocks [S1, unofficial].
- `Schematic.MAX_WORLD_HEIGHT`, `Schematic.MAX_WORLD_LENGTH`, `Schematic.MAX_WORLD_WIDTH`: Maximum allowed dimensions [S1, unofficial].

**Steps:**
1. **Validate Dimensions:** Check if `Height > Schematic.MAX_WORLD_HEIGHT`, `Length > Schematic.MAX_WORLD_LENGTH`, or `Width > Schematic.MAX_WORLD_WIDTH`. Throw an exception if any exceed limits [S1, unofficial].
2. **Iterate Voxels:** Loop through the voxel grid in the order: $y$ from $0$ to $Height-1$, $z$ from $0$ to $Length-1$, $x$ from $0$ to $Width-1$ [S1, unofficial].
3. **Calculate Index:** For each $(x, y, z)$, calculate the linear index in the `Blocks` array:
   $$ index = (y \times Length + z) \times Width + x $$
4. **Check Block Existence:** If `Blocks[index] == 0`, skip to the next voxel [S1, unofficial].
5. **Excavation Check (if `mExcavate` is true):**
   - Check the 6 neighbors: Left, Right, Top, Bottom, Ahead, Behind [S1, unofficial].
   - Calculate neighbor indices with boundary checks:
     - Left X: $(y \times Length + z) \times Width + (x - 1)$
     - Right X: $(y \times Length + z) \times Width + (x + 1)$
     - Top: $((y + 1) \times Length + z) \times Width + x$
     - Bottom: $((y - 1) \times Length + z) \times Width + x$
     - Ahead: $(y \times Length + (z + 1)) \times Width + x$
     - Behind: $(y \times Length + (z - 1)) \times Width + x$
   - Ensure indices are within bounds ($x-1 \ge 0$, $x+1 < Width$, etc.) before accessing [S1, unofficial].
   - If any neighbor has a block ID of 0 (air), 8 (water), or 9 (lava), the current block is considered connected to air [S1, unofficial].
   - If `mExcavate` is true and the block is NOT connected to air, skip adding it [S1, unofficial].
6. **Color Lookup:**
   - Attempt to find the color in `mColors` using the key `(blockID, Data[index])` [S1, unofficial].
   - If not found, fallback to `(blockID, 0)` [S1, unofficial].
   - If still not found, use a default color (implementation detail not specified in sources) [G].
7. **Create Voxel:** Create a `Voxel` with coordinates $(x, y, z)$ and the resolved color. Add to the output list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_schematic_to_voxels(raw_schematic: RawSchematic, excavate: bool, color_map: Dict[Tuple[int, int], Color]) -> List[Voxel]:
    width = raw_schematic.Width
    height = raw_schematic.Height
    length = raw_schematic.Length
    
    # Validate dimensions
    if height > MAX_WORLD_HEIGHT or length > MAX_WORLD_LENGTH or width > MAX_WORLD_WIDTH:
        raise Exception("Dimensions exceed maximum allowed size")
        
    voxels = []
    
    for y in range(height):
        for z in range(length):
            for x in range(width):
                index = (y * length + z) * width + x
                
                block_id = raw_schematic.Blocks[index]
                
                # Skip air blocks
                if block_id == 0:
                    continue
                
                # Excavation logic
                if excavate:
                    if not is_connected_to_air(raw_schematic, x, y, z, width, height, length):
                        continue
                
                # Color lookup
                metadata = raw_schematic.Data[index]
                color = color_map.get((block_id, metadata))
                if color is None:
                    color = color_map.get((block_id, 0))
                    if color is None:
                        # Fallback to default color if not specified in sources
                        color = Color(255, 255, 255, 255) 
                
                voxels.append(Voxel(x, y, z, color))
                
    return voxels

def is_connected_to_air(raw_schematic: RawSchematic, x: int, y: int, z: int, width: int, height: int, length: int) -> bool:
    neighbors = [
        (x - 1, y, z), (x + 1, y, z),
        (x, y + 1, z), (x, y - 1, z),
        (x, y, z + 1), (x, y, z - 1)
    ]
    
    for nx, ny, nz in neighbors:
        # Boundary checks
        if nx < 0 or nx >= width: continue
        if ny < 0 or ny >= height: continue
        if nz < 0 or nz >= length: continue
        
        n_index = (ny * length + nz) * width + nx
        n_block_id = raw_schematic.Blocks[n_index]
        
        # Check if neighbor is air, water, or lava
        if n_block_id in [0, 8, 9]:
            return True
            
    return False
```

## Subtopics

### Minecraft-style Block World Loading
Loads a 3D block world from a text file where each line contains 6 floats: $x, z, y, r, g, b$. Constructs a `WorldModel` vector of position-color pairs for rendering [S2, unofficial].
- **Steps:**
  1. Open file `assets/{argv[1]}.txt` [S2, unofficial].
  2. Read all lines into `splitFileStr` [S2, unofficial].
  3. For each line, parse 6 floats [S2, unofficial].
  4. Map to `Vec3<float> pos = {x, z, y}` (note: $y$ and $z$ are swapped from file order) [S2, unofficial].
  5. Map to `Color color = {r, g, b}` [S2, unofficial].
  6. Push to `worldModel` vector [S2, unofficial].
  7. Compute bounds: `boundsMin = {-32 + padding, -32 + padding, 0 + padding}`, `boundsMax = {32 - padding, 32 - padding, 64 - padding}` [S2, unofficial].
- **Parameters:** `padding`: Integer offset for world bounds. Default 2. Can be overridden via `-padding` CLI flag [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \times H \times L)$ where $W, H, L$ are dimensions. Excavation adds a constant factor of 6 neighbor checks per voxel.
- **Memory Complexity:** $O(W \times H \times L)$ for the input arrays and output voxel list.
- **Strengths:** Directly handles Minecraft's specific data format; supports excavation to remove internal blocks.
- **Limitations:** Requires a color mapping configuration file; limited by maximum world height/length/width constraints.
- **When to use:** When converting Minecraft builds to voxel formats for rendering or 3D printing.
- **When not to use:** For general 3D mesh voxelization or other file formats.

## Background
Originated from the FileToVox project by Zarbuz [S1, unofficial]. FileToVox is a console program that converts various file formats into Magicavoxel (.vox) files, supporting multiple input types including `.schematic` [S1, unofficial].

## Sources
[S1] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
[S2] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
