# Region Data Layout & Serialization

Also called: Chunk Indexing, Region Data Storage and Lookup, Region File Format, Region Looktable and Chunk Data Layout, Region Save Format

## Overview
Region Data Layout & Serialization is a storage mechanism for terrain chunks that combines a lookup table and chunk data into a single byte array (`regionData`), allowing efficient retrieval and saving of chunk data without external databases [S1, unofficial]. It maps chunk coordinates $(x, z)$ to byte offsets in the array, enabling direct access to chunk data based on its position within a region [S1, unofficial]. This method is distinct from other chunk-based storage systems by its use of a single contiguous file format for both metadata and data, optimizing for simplicity and direct disk I/O [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Chunk coordinates $(x, z)$ within a region, chunk data (byte array), and region configuration constants.
*   **Output:** Updated `regionData` byte array (in memory) and optionally a compressed `.reg` file on disk.

**Data Structures:**
*   `regionData`: A single `byte[]` array containing the entire region file content.
    *   **Layout:**
        1.  **Lookup Table:** Occupies the first `Constants.REGION_LOOKTABLE_BYTES` bytes. Each entry corresponds to a chunk coordinate $(x, z)$ and stores the byte offset to the chunk's data.
        2.  **Chunk Data:** Follows the lookup table. Each chunk occupies `Constants.CHUNK_BYTES` bytes.
*   **Constants:**
    *   `REGION_SIZE`: Number of chunks per region side. Typical value: 32 [S1, unofficial].
    *   `REGION_LOOKTABLE_POS_BYTE`: Number of bytes per lookup table entry. Calculated as $\lceil \log_{256}((\text{REGION\_SIZE} \times \text{REGION\_SIZE}) + 1) \rceil$. Max value: 4. For `REGION_SIZE = 32`, this is 2 bytes [S1, unofficial].
    *   `REGION_LOOKTABLE_BYTES`: Total size of the lookup table. Calculated as `REGION_SIZE * REGION_SIZE * REGION_LOOKTABLE_POS_BYTE` [S1, unofficial].
    *   `CHUNK_BYTES`: Size of a single chunk's data in bytes [S1, unofficial].
    *   `REGION_SAVE_COMPRESSED`: Boolean flag for compression. Recommended: `TRUE` [S1, unofficial].

**Numbered Steps:**

1.  **Calculate Lookup Table Entry Position:**
    For a chunk at local region coordinates $(x, z)$, calculate the starting byte index in `regionData` for its lookup table entry:
    $$ \text{startPos} = (x + z \times \text{REGION\_SIZE}) \times \text{REGION\_LOOKTABLE\_POS\_BYTE} + \text{REGION\_LOOKTABLE\_POS\_BYTE} $$
    Note: The offset `+ REGION_LOOKTABLE_POS_BYTE` accounts for the global counter stored in the first `REGION_LOOKTABLE_POS_BYTE` bytes [S1, unofficial].

2.  **Read Existing Index:**
    Read the multi-byte index from `regionData` at `startPos`. The bytes are stored in little-endian order. Assemble the integer by iterating backwards:
    ```python
    index = 0
    for i in range(REGION_LOOKTABLE_POS_BYTE):
        index |= regionData[startPos + REGION_LOOKTABLE_POS_BYTE - i - 1] << (8 * i)
    ```
    If `index == 0`, the chunk is not saved/allocated [S1, unofficial].

3.  **Handle New Chunk Allocation:**
    If `index == 0`:
    *   Read the global chunk counter from the first `REGION_LOOKTABLE_POS_BYTE` bytes of `regionData` (little-endian).
    *   Increment the counter.
    *   **Overflow Handling:** If the most significant byte of the counter reaches 255, reset it to 0 and increment the next lower byte (carry propagation) [S1, unofficial].
    *   Write the new counter value back to the first `REGION_LOOKTABLE_POS_BYTE` bytes.
    *   The new chunk's data offset is determined by the new counter value. Let `new_counter` be the incremented value. The offset for the chunk data is:
        $$ \text{chunk\_data\_start} = \text{REGION\_LOOKTABLE\_BYTES} + (\text{new\_counter} - 1) \times \text{CHUNK\_BYTES} $$
        Note: The index stored in the lookup table is `new_counter`, not the byte offset. The lookup table entry is updated with `new_counter` [S1, unofficial].
    *   Update the lookup table entry for $(x, z)$ with `new_counter`:
        ```python
        for i in range(REGION_LOOKTABLE_POS_BYTE - 1, -1, -1):
            regionData[startPos + i] = (new_counter >> (8 * i)) & 0xFF
        ```
    *   Write the new chunk data to `regionData` starting at `chunk_data_start` [S1, unofficial].

4.  **Handle Existing Chunk Update:**
    If `index != 0`:
    *   Calculate the start position for the chunk data:
        $$ \text{startPos} = \text{REGION\_LOOKTABLE\_BYTES} + (\text{index} - 1) \times \text{CHUNK\_BYTES} $$
    *   Write the new chunk data bytes directly to `regionData` starting at `startPos` [S1, unofficial].

5.  **Save to Disk:**
    *   If `REGION_SAVE_COMPRESSED` is `TRUE`:
        *   Compress `regionData` using a compression algorithm (e.g., `CompressHelper.Compress`).
        *   Write the compressed data to the `.reg` file [S1, unofficial].
    *   If `REGION_SAVE_COMPRESSED` is `FALSE`:
        *   Write `regionData` directly to the `.reg` file [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def save_chunk(region_data, x, z, chunk_data_bytes, constants):
    """
    Saves or updates a chunk in the region data.
    
    Args:
        region_data (bytearray): The current region data buffer.
        x (int): Local x coordinate of the chunk within the region.
        z (int): Local z coordinate of the chunk within the region.
        chunk_data_bytes (bytes): The binary data for the chunk.
        constants (dict): Dictionary containing REGION_SIZE, REGION_LOOKTABLE_POS_BYTE, 
                          REGION_LOOKTABLE_BYTES, CHUNK_BYTES, REGION_SAVE_COMPRESSED.
    """
    REGION_SIZE = constants['REGION_SIZE']
    LOOKTABLE_POS_BYTE = constants['REGION_LOOKTABLE_POS_BYTE']
    LOOKTABLE_BYTES = constants['REGION_LOOKTABLE_BYTES']
    CHUNK_BYTES = constants['CHUNK_BYTES']
    COMPRESSED = constants['REGION_SAVE_COMPRESSED']
    
    # Step 1: Calculate lookup table entry position
    start_pos = (x + z * REGION_SIZE) * LOOKTABLE_POS_BYTE + LOOKTABLE_POS_BYTE
    
    # Step 2: Read existing index (little-endian)
    index = 0
    for i in range(LOOKTABLE_POS_BYTE):
        index |= region_data[start_pos + LOOKTABLE_POS_BYTE - i - 1] << (8 * i)
        
    if index == 0:
        # Step 3: New chunk allocation
        # Read global counter from first LOOKTABLE_POS_BYTE bytes
        counter = 0
        for i in range(LOOKTABLE_POS_BYTE):
            counter |= region_data[i] << (8 * i)
            
        # Increment counter with overflow handling
        counter += 1
        carry = 1
        for i in range(LOOKTABLE_POS_BYTE):
            byte_val = (counter >> (8 * i)) & 0xFF
            # Handle overflow: if byte_val is 0 after increment, it means we wrapped 255->0
            # The carry logic in the source implies a simple increment with wrap-around
            # More robustly:
            # region_data[i] = (region_data[i] + carry) & 0xFF
            # carry = 1 if (region_data[i] == 0 and carry == 1) else 0
            # But the source says: "if the most significant byte of the counter reaches 255, reset it to 0 and increment the next lower byte"
            # This is a standard multi-byte increment.
            pass
            
        # Re-implementing the counter increment as described:
        # Increment the least significant byte
        region_data[0] = (region_data[0] + 1) & 0xFF
        if region_data[0] == 0: # Overflow from 255 to 0
            # Carry to next byte
            for i in range(1, LOOKTABLE_POS_BYTE):
                region_data[i] = (region_data[i] + 1) & 0xFF
                if region_data[i] != 0:
                    break
                    
        # Read the new counter value
        new_counter = 0
        for i in range(LOOKTABLE_POS_BYTE):
            new_counter |= region_data[i] << (8 * i)
            
        # Update lookup table entry with new_counter
        for i in range(LOOKTABLE_POS_BYTE - 1, -1, -1):
            region_data[start_pos + i] = (new_counter >> (8 * i)) & 0xFF
            
        # Calculate chunk data start position
        chunk_data_start = LOOKTABLE_BYTES + (new_counter - 1) * CHUNK_BYTES
        
        # Write chunk data
        region_data[chunk_data_start : chunk_data_start + CHUNK_BYTES] = chunk_data_bytes
        
    else:
        # Step 4: Existing chunk update
        chunk_data_start = LOOKTABLE_BYTES + (index - 1) * CHUNK_BYTES
        region_data[chunk_data_start : chunk_data_start + CHUNK_BYTES] = chunk_data_bytes
        
    # Step 5: Save to disk
    if COMPRESSED:
        compressed_data = compress(region_data) # Implementation of compress not provided in source [G]
        write_file(compressed_data) # Implementation of write_file not provided in source [G]
    else:
        write_file(region_data) # Implementation of write_file not provided in source [G]

def load_chunk(region_data, x, z, constants):
    """
    Loads a chunk from the region data.
    
    Args:
        region_data (bytearray): The current region data buffer.
        x (int): Local x coordinate of the chunk within the region.
        z (int): Local z coordinate of the chunk within the region.
        constants (dict): Dictionary containing REGION_SIZE, REGION_LOOKTABLE_POS_BYTE, 
                          REGION_LOOKTABLE_BYTES, CHUNK_BYTES.
                          
    Returns:
        bytes: The chunk data, or None if the chunk is not saved.
    """
    REGION_SIZE = constants['REGION_SIZE']
    LOOKTABLE_POS_BYTE = constants['REGION_LOOKTABLE_POS_BYTE']
    LOOKTABLE_BYTES = constants['REGION_LOOKTABLE_BYTES']
    CHUNK_BYTES = constants['CHUNK_BYTES']
    
    # Calculate lookup table entry position
    start_pos = (x + z * REGION_SIZE) * LOOKTABLE_POS_BYTE + LOOKTABLE_POS_BYTE
    
    # Read index (little-endian)
    index = 0
    for i in range(LOOKTABLE_POS_BYTE):
        index |= region_data[start_pos + LOOKTABLE_POS_BYTE - i - 1] << (8 * i)
        
    if index == 0:
        return None # Chunk not saved
        
    # Calculate chunk data start position
    chunk_data_start = LOOKTABLE_BYTES + (index - 1) * CHUNK_BYTES
    
    # Return chunk data
    return region_data[chunk_data_start : chunk_data_start + CHUNK_BYTES]
```

**Parameters:**
*   `REGION_SIZE`: 32 [S1, unofficial].
*   `REGION_LOOKTABLE_POS_BYTE`: 2 for `REGION_SIZE=32` [S1, unofficial].
*   `REGION_SAVE_COMPRESSED`: `TRUE` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Chunk Marker:** The lookup table uses 0 to indicate an empty/unallocated chunk. The global counter starts at 1 [S1, unofficial].
*   **Counter Overflow:** If the global counter overflows (reaches 255 in the most significant byte), it wraps to 0 and carries over to the next byte. This can potentially conflict with the empty marker (0) if not managed carefully, although the provided code handles the carry logic [S1, unofficial].
*   **Precision:** The index is stored as a multi-byte integer. Ensure correct endianness handling (little-endian) when reading/writing [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Lookup: $O(1)$ for calculating the offset and reading the index.
    *   Read/Write Chunk Data: $O(1)$ for calculating the offset and copying data, assuming fixed-size chunks.
    *   Save: $O(N)$ where $N$ is the size of `regionData`, due to potential compression.
*   **Memory Complexity:** $O(R^2 \times C)$ where $R$ is `REGION_SIZE` and $C$ is `CHUNK_BYTES`. The entire region is loaded into memory as a single byte array [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance degrades as the region size increases due to larger memory footprint and potential I/O bottlenecks for large files. Compression can mitigate I/O time but increases CPU usage [S1, unofficial].
*   **Strengths:** Simple implementation, no external database required, efficient direct access to chunks [S1, unofficial].
*   **Limitations:** Limited by the maximum chunk index (4 bytes max), potential for counter overflow, single-file I/O bottleneck for large regions [S1, unofficial].
*   **When to Use:** Suitable for smaller worlds or regions where simplicity and direct file access are prioritized over complex caching or database management [S1, unofficial].
*   **When Not to Use:** For very large worlds requiring complex caching, incremental updates, or database-like queries [S1, unofficial].

## Background
This method is described in the context of a Unity-based voxel game engine, specifically within the `WorldManager` and `ChunkSystem` components [S1, unofficial]. It is part of a larger system for managing voxel data, including chunk generation, loading, and unloading [S1, unofficial]. The specific implementation details are found in the `WorldManager` class and related scripts in the `Javier-Garzo/Marching-cubes-on-Unity-3D` repository [S1, unofficial].

## Sources
[S1] Javier-Garzo/Marching-cubes-on-Unity-3D, https://github.com/Javier-Garzo/Marching-cubes-on-Unity-3D, fetched 2026-10-26 (unofficial)
