# Fortune's Palette Compression

## Overview
Fortune's Palette Compression is a lossless data compression method that maps unique values in a flat, cubic 3D vector to a compact palette, storing indices in a bit-packed integer array [S1, unofficial]. It is designed for fast, header-only implementation in C++ to compress block data where the number of unique values is small relative to the total volume [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A flat 3D vector of type `RETURN_TYPE` (default `uint16_t`) with dimensions $N \times N \times N$, where $N$ is `vectorDiameter` [S1, unofficial].
- Output: A compressed representation stored in a `PaletteCompressedBlocks` structure, allowing random access via `GetNum` and `SetNum` [S1, unofficial].

**Data Structures**
The compressed state is held in `PaletteCompressedBlocks`:
- `SingleBlockTypeID`: `int16_t`. If the entire vector contains only one unique value, this stores that value, and `compressedBlocks` is empty [S1, unofficial]. Default is -1.
- `compressedBlocks`: `std::vector<uint64_t>`. Stores the bit-packed palette indices [S1, unofficial].
- `PaletteToBlockID`: `std::vector<RETURN_TYPE>`. Maps a palette index (0 to $K-1$) to the original value [S1, unofficial].
- `BlockToPalette`: `std::map<RETURN_TYPE, unsigned int>`. Maps an original value to its palette index [S1, unofficial].

**Parameters**
- `vectorDiameter` ($N$): Side length of the cubic vector. Typical value: 64 [S1, unofficial].
- `RETURN_TYPE`: Data type of the vector elements. Default: `uint16_t` [S1, unofficial].
- `bitBlockSize`: Bits per palette index. Calculated as $\lceil \log_2(K) \rceil$, where $K$ is the number of unique values. Minimum 1 if $K \le 1$ [S1, unofficial].
- `blocksPerInt`: Number of indices per 64-bit integer. Calculated as $\lfloor 64.0 / \text{bitBlockSize} \rfloor$ [S1, unofficial].

**Algorithm Steps**

1. **Initialization**: Calculate `vectorArea` = $N^2$ and `vectorVolume` = $N^3$ [S1, unofficial].
2. **Unique Value Collection**: Iterate through the input vector and insert all values into a `std::set<RETURN_TYPE>` to find unique values [S1, unofficial].
3. **Edge Case Handling**:
   - If the set is empty, print an error and return [S1, unofficial].
   - If the set size is 1, set `SingleBlockTypeID` to the unique value and skip compression [S1, unofficial].
4. **Palette Construction**: Iterate through the sorted unique values. For each value, append it to `PaletteToBlockID` and store the mapping from value to index in `BlockToPalette` [S1, unofficial].
5. **Bit Packing Calculation**:
   - Calculate `paletteSize` = size of unique values [S1, unofficial].
   - Calculate `bitBlockSize` = $\lceil \log_2(\text{paletteSize}) \rceil$, minimum 1 [S1, unofficial].
   - Calculate `blocksPerInt` = $\lfloor 64.0 / \text{bitBlockSize} \rfloor$ [S1, unofficial].
6. **Compression Loop**:
   - Initialize `currentPackedBlock` = 0, `currentBit` = 0, `blocksPerIntCount` = 0 [S1, unofficial].
   - Reserve space for `compressedBlocks` [S1, unofficial].
   - For each value in the input vector:
     - If `blocksPerIntCount` == `blocksPerInt`, push `currentPackedBlock` to `compressedBlocks`, reset `currentPackedBlock`, `currentBit`, and `blocksPerIntCount` to 0 [S1, unofficial].
     - Look up the palette index for the current value in `BlockToPalette` [S1, unofficial].
     - Shift the index left by `currentBit` and OR it into `currentPackedBlock` [S1, unofficial].
     - Increment `currentBit` by `bitBlockSize` and `blocksPerIntCount` by 1 [S1, unofficial].
   - If `currentBit` > 0 after the loop, push the remaining `currentPackedBlock` to `compressedBlocks` [S1, unofficial].

**Random Access (GetNum)**
1. Calculate linear index: $idx = y + (x \cdot N) + (z \cdot N^2)$ [S1, unofficial].
2. If `SingleBlockTypeID` != -1, return `SingleBlockTypeID` [S1, unofficial].
3. Calculate `intIndex` = $idx / \text{blocksPerInt}$ and `bitOffset` = $(idx \% \text{blocksPerInt}) \cdot \text{bitBlockSize}$ [S1, unofficial].
4. If `intIndex` is out of bounds of `compressedBlocks`, return 0 [S1, unofficial].
5. Extract block data: `blockData` = `(compressedBlocks[intIndex] >> bitOffset) & ((1ULL << bitBlockSize) - 1)` [S1, unofficial].
6. Return `PaletteToBlockID[blockData]` cast to `uint8_t` [S1, unofficial].

**Random Access (SetNum)**
1. Calculate linear index $idx$ as above [S1, unofficial].
2. If `SingleBlockTypeID` != -1, the vector must be decompressed and recompressed because the palette structure does not support adding new unique values in-place [S1, unofficial].
3. Calculate `intIndex` and `bitOffset` as above [S1, unofficial].
4. If `intIndex` is out of bounds, do nothing [S1, unofficial].
5. Retrieve `currentPackedBlock` = `compressedBlocks[intIndex]` [S1, unofficial].
6. Clear the bits for the target block: `currentPackedBlock &= ~(mask << bitOffset)`, where `mask` = $(1ULL << \text{bitBlockSize}) - 1$ [S1, unofficial].
7. Look up the palette index for `newNum` in `BlockToPalette` [S1, unofficial]. If `newNum` is not in the palette, the entire vector must be decompressed and recompressed [S1, unofficial].
8. Set the new block data: `currentPackedBlock |= (palette_index << bitOffset)` [S1, unofficial].
9. Store `currentPackedBlock` back to `compressedBlocks[intIndex]` [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Dict, Set

RETURN_TYPE = int  # Typically uint16_t

class PaletteCompressedBlocks:
    def __init__(self):
        self.SingleBlockTypeID: int = -1
        self.compressedBlocks: List[int] = []
        self.PaletteToBlockID: List[RETURN_TYPE] = []
        self.BlockToPalette: Dict[RETURN_TYPE, int] = {}

class PaletteCompression:
    def __init__(self, vectorDiameter: int):
        self.vectorDiameter = vectorDiameter
        self.vectorArea = vectorDiameter * vectorDiameter
        self.vectorVolume = vectorDiameter * vectorDiameter * vectorDiameter
        self.blocks = PaletteCompressedBlocks()

    def compress(self, uncompressedBlocks: List[RETURN_TYPE]) -> None:
        self.blocks.compressedBlocks.clear()
        self.blocks.PaletteToBlockID.clear()
        self.blocks.BlockToPalette.clear()

        if self.blocks.SingleBlockTypeID != -1:
            raise ValueError("SingleBlockTypeID is not -1")

        # 1. Collect unique values
        uniqueBlocks: Set[RETURN_TYPE] = set(uncompressedBlocks)

        # 2. Edge cases
        if len(uniqueBlocks) == 0:
            raise ValueError("Empty vector")
        if len(uniqueBlocks) == 1:
            self.blocks.SingleBlockTypeID = next(iter(uniqueBlocks))
            return

        # 3. Build Palette
        for block in uniqueBlocks:
            self.blocks.PaletteToBlockID.append(block)
            self.blocks.BlockToPalette[block] = len(self.blocks.BlockToPalette)

        # 4. Calculate bit parameters
        paletteSize = len(uniqueBlocks)
        bitBlockSize = 1 if paletteSize <= 1 else math.ceil(math.log2(paletteSize))
        blocksPerInt = 64 // bitBlockSize

        # 5. Compress
        currentPackedBlock = 0
        currentBit = 0
        blocksPerIntCount = 0
        
        # Reserve space
        num_ints = (len(uncompressedBlocks) + blocksPerInt - 1) // blocksPerInt
        self.blocks.compressedBlocks = [0] * num_ints

        for block in uncompressedBlocks:
            if blocksPerIntCount == blocksPerInt:
                self.blocks.compressedBlocks.append(currentPackedBlock)
                currentPackedBlock = 0
                currentBit = 0
                blocksPerIntCount = 0
            
            # Get palette index
            if block not in self.blocks.BlockToPalette:
                raise ValueError("Block not in palette")
            palette_index = self.blocks.BlockToPalette[block]
            
            # Pack
            packed_block = palette_index << currentBit
            currentPackedBlock |= packed_block
            
            currentBit += bitBlockSize
            blocksPerIntCount += 1

        # 6. Finalize
        if currentBit > 0:
            self.blocks.compressedBlocks.append(currentPackedBlock)

    def get_num(self, x: int, y: int, z: int) -> int:
        if self.blocks.SingleBlockTypeID != -1:
            return self.blocks.SingleBlockTypeID

        index = y + (x * self.vectorDiameter) + (z * self.vectorArea)
        
        paletteSize = len(self.blocks.PaletteToBlockID)
        bitBlockSize = 1 if paletteSize <= 1 else math.ceil(math.log2(paletteSize))
        blocksPerInt = 64 // bitBlockSize

        intIndex = index // blocksPerInt
        bitOffset = (index % blocksPerInt) * bitBlockSize

        if intIndex >= len(self.blocks.compressedBlocks):
            return 0

        mask = (1 << bitBlockSize) - 1
        blockData = (self.blocks.compressedBlocks[intIndex] >> bitOffset) & mask
        
        return self.blocks.PaletteToBlockID[blockData]

    def set_num(self, x: int, y: int, z: int, newNum: RETURN_TYPE) -> None:
        if self.blocks.SingleBlockTypeID != -1:
            raise ValueError("Cannot modify single block type vector in place")

        index = y + (x * self.vectorDiameter) + (z * self.vectorArea)
        
        paletteSize = len(self.blocks.PaletteToBlockID)
        bitBlockSize = 1 if paletteSize <= 1 else math.ceil(math.log2(paletteSize))
        blocksPerInt = 64 // bitBlockSize

        intIndex = index // blocksPerInt
        bitOffset = (index % blocksPerInt) * bitBlockSize

        if intIndex >= len(self.blocks.compressedBlocks):
            return

        if newNum not in self.blocks.BlockToPalette:
            raise ValueError("New value not in palette; requires full decompression and recompression")

        mask = (1 << bitBlockSize) - 1
        currentPackedBlock = self.blocks.compressedBlocks[intIndex]

        # Clear bits
        currentPackedBlock &= ~(mask << bitOffset)

        # Set new value
        packedBlock = self.blocks.BlockToPalette[newNum]
        currentPackedBlock |= (packedBlock << bitOffset)

        self.blocks.compressedBlocks[intIndex] = currentPackedBlock
```

## Operating characteristics
- **Time Complexity**: Compression is $O(V)$ where $V$ is the vector volume, due to single-pass iteration and set insertion [S1, unofficial]. Decompression/Access is $O(1)$ per element [S1, unofficial].
- **Memory Complexity**: The compressed size is $V \cdot \lceil \log_2(K) \rceil$ bits, where $K$ is the number of unique values [S1, unofficial]. The overhead includes the palette ($K \cdot \text{sizeof(RETURN\_TYPE)}$) and the `BlockToPalette` map [S1, unofficial].
- **Performance**: Designed for speed with header-only C++ implementation. No external dependencies [S1, unofficial].
- **Strengths**: Very fast for low-entropy data (few unique values). Simple API. Supports in-place modification of existing values [S1, unofficial].
- **Limitations**: Only supports flat, cubic vectors ($N \times N \times N$) [S1, unofficial]. Cannot add new unique values without full recompression [S1, unofficial]. Not suitable for high-entropy data where $K$ is large, as bit overhead approaches original size [S1, unofficial].

## Background
Fortune's Palette Compression is an unofficial implementation found in a GitHub repository [S1, unofficial]. It is designed as a fast, header-only C++ library for compressing block data, inspired by techniques used in voxel-based games like Minecraft [S1, unofficial]. It is not a peer-reviewed academic method but a practical engineering solution for specific use cases [S1, unofficial].

## Sources
[S1] Finding-Fortune/Fortunes-Palette-Compression, https://github.com/Finding-Fortune/Fortunes-Palette-Compression, fetched 2026-10-28 (unofficial)
