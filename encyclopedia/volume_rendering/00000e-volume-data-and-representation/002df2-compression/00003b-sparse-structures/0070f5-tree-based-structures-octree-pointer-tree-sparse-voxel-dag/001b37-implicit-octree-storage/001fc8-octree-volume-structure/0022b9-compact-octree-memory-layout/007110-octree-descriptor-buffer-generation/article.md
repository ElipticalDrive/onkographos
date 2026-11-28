# Octree Descriptor Buffer Generation

Also called: Compact Octree Buffer Generation, Recursive Octree Descriptor Builder

## Overview
Octree Descriptor Buffer Generation is a recursive algorithm that constructs a compact, linear memory representation of a sparse voxel octree. It solves the problem of efficiently storing node topology (validity and leaf status) and child pointers in a single buffer while handling large address spaces through a page-based far-pointer indirection mechanism. It differs from standard pointer-based trees by using relative offsets and from flat grids by dynamically allocating space only for non-empty branches, building on the [Compact Octree Memory Layout](2828-compact-octree-memory-layout.md) structure.

## How it works
The algorithm traverses the voxel space recursively, generating a descriptor for each node. It handles two types of pointers: relative pointers for children within the same memory page (distance $\le 0x8000$) and far (absolute) pointers for children outside this range.

### Inputs and Outputs
- **Input**: A 3D voxel grid (or heightmap/data source), the root position $(0,0,0)$, and the initial block size (typically `oct_dimensions/2`).
- **Output**: A linear buffer of `uint64_t` values (`descriptor_buffer`) containing the serialized octree descriptors.

### Data Structures
- **`descriptor_buffer`**: An array of `uint64_t`. Each element represents a node descriptor.
- **Node Descriptor Bit Layout**:
  - Bits 16–23: Valid mask. Each bit $i$ (0–7) corresponds to child $i$. If set, the child exists and is valid.
  - Bits 24–31: Leaf mask. Each bit $i$ (0–7) corresponds to child $i$. If set, the child is a leaf node.
  - Lower bits (0–15): Child pointers. Stores relative offsets or far pointer indices.

### Parameters
- `MAX_RELATIVE_PTR`: `0x8000`. The maximum distance for a relative pointer. Distances exceeding this require a far pointer.
- `PAGE_SIZE`: `0x8000` (in units of descriptors or bytes, depending on implementation; the source uses `0x8000` as the counter reset value).
- `oct_dimensions`: The side length of the initial voxel block.

### Algorithm Steps
1. **Initialization**: Start recursion at the root with position $(0,0,0)$ and `voxel_scale = oct_dimensions / 2`. Initialize `descriptor_buffer` as empty. Initialize `page_header_counter = 0x8000`. Initialize a list `descriptor_position_array` to track positions of descriptors that may need far pointers.
2. **Base Case (Leaf Voxel)**: If `voxel_scale == 1`:
   - Query the 3D grid for the 8 subvoxels at the current position.
   - Set bits 16–23 (valid mask) based on the presence of each subvoxel.
   - Set bits 24–31 (leaf mask) to `0xFF` (all children are leaves).
   - No child pointers are stored in the lower bits.
   - Push the descriptor to `descriptor_buffer`.
3. **Recursive Step**: For each of the 8 children (index $i = 0$ to $7$):
   - Calculate child position: $P_i = P_{parent} + (i \text{ in octal}) \times \text{voxel\_scale}$.
   - Call `GenerationRecursion` with $P_i$ and `voxel_scale / 2`.
   - Let the return value be the index of the child's descriptor in `descriptor_buffer`.
   - **Check Child Status**:
     - If the child is empty (no valid voxels):
       - Set the leaf bit for child $i$ (bit $i + 16 + 8$) to 1.
       - Leave the valid bit for child $i$ (bit $i + 16$) as 0.
       - Do not add to `descriptor_position_array`.
     - If the child is valid and not a leaf:
       - Set the valid bit for child $i$ (bit $i + 16$) to 1.
       - Add the child's descriptor index and position to `descriptor_position_array`.
4. **Page Boundary Handling**:
   - Calculate worst-case insertion size: $W = \text{descriptor\_position\_array.size()} \times 2$.
   - If `page_header_counter < W`:
     - Move back to the page header (reset buffer pointer to start of current page).
     - Reset `page_header_counter = 0x8000`.
     - Write the current info section position to the page header (implementation detail for indirection).
5. **Far Pointer Calculation**:
   - Iterate backwards through `descriptor_position_array`.
   - For each child, calculate the relative distance $D$ from the current parent descriptor's position to the child's descriptor position.
   - If $D > \text{MAX\_RELATIVE\_PTR}$:
     - Write the absolute position of the child to a separate far pointer block in the buffer.
     - Increment `far_pointer_count`.
6. **Write Child Pointers**:
   - Iterate backwards through `descriptor_position_array`.
   - For each child:
     - Calculate relative distance $D$.
     - If $D > \text{MAX\_RELATIVE\_PTR}$:
       - Set the `far_bit_mask` in the descriptor's lower bits.
       - Store the offset to the far pointer block in the lower bits.
     - Else if $D > 0$:
       - Store the relative distance $D$ directly in the lower bits.
     - Else:
       - Store 0 (or handle as invalid, though valid children should have $D > 0$).
7. **Finalize Descriptor**:
   - Combine the valid mask, leaf mask, and child pointers into a single `uint64_t`.
   - Push the descriptor to `descriptor_buffer`.
   - Update `page_header_counter` by subtracting the size of the written descriptor.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_octree_buffer(voxel_grid, oct_dimensions):
    descriptor_buffer = []
    far_pointer_block = []
    page_header_counter = 0x8000
    MAX_RELATIVE_PTR = 0x8000
    
    # Helper to query voxel presence
    def is_voxel_present(x, y, z):
        # Implementation depends on voxel_grid access
        return voxel_grid.get(x, y, z) is not None

    def generation_recursion(pos, voxel_scale):
        nonlocal page_header_counter, far_pointer_block
        
        if voxel_scale == 1:
            # Base case: leaf voxel
            valid_mask = 0
            leaf_mask = 0xFF
            
            # Query 8 subvoxels
            for i in range(8):
                # Calculate subvoxel coordinates
                x = pos[0] + ((i >> 2) & 1)
                y = pos[1] + ((i >> 1) & 1)
                z = pos[2] + (i & 1)
                
                if is_voxel_present(x, y, z):
                    valid_mask |= (1 << (i + 16))
            
            # All children are leaves
            descriptor = (valid_mask | leaf_mask)
            descriptor_buffer.append(descriptor)
            return len(descriptor_buffer) - 1

        # Recursive step
        descriptor_position_array = []
        
        for i in range(8):
            # Calculate child position
            child_pos = [
                pos[0] + ((i >> 2) & 1) * voxel_scale,
                pos[1] + ((i >> 1) & 1) * voxel_scale,
                pos[2] + (i & 1) * voxel_scale
            ]
            
            child_idx = generation_recursion(child_pos, voxel_scale // 2)
            
            # Check if child is valid
            # Note: The source implies we need to know if the child is empty.
            # This requires checking the result or a separate validity check.
            # Assuming generation_recursion returns a valid index if the child exists.
            # If the child is empty, it might return a special value or we check the grid.
            # Based on notes: "If child is a leaf of non-valid values (empty)..."
            # We need to determine if the child is empty.
            # Let's assume a helper function checks if a block is empty.
            if is_block_empty(child_pos, voxel_scale // 2, voxel_grid):
                # Child is empty
                # Set leaf bit for this child
                leaf_mask_bit = 1 << (i + 16 + 8)
                # Valid bit remains 0
                descriptor_position_array.append((child_idx, leaf_mask_bit, 0))
            else:
                # Child is valid and not a leaf (or is a leaf with data)
                # Set valid bit
                valid_bit = 1 << (i + 16)
                descriptor_position_array.append((child_idx, 0, valid_bit))
        
        # Page boundary handling
        worst_case_size = len(descriptor_position_array) * 2
        if page_header_counter < worst_case_size:
            # Move back to page header
            # Reset counter
            page_header_counter = 0x8000
            # Write current info section position to page header
            # This part is implementation-specific for page management
            pass
        
        # Calculate far pointers and write descriptors
        valid_mask = 0
        leaf_mask = 0
        
        # Iterate backwards to handle pointer offsets correctly
        for idx, (child_idx, leaf_bit, valid_bit) in enumerate(reversed(descriptor_position_array)):
            if valid_bit:
                valid_mask |= valid_bit
            if leaf_bit:
                leaf_mask |= leaf_bit
            
            # Calculate relative distance
            # Current descriptor position is len(descriptor_buffer)
            # Child descriptor position is child_idx
            # Distance is absolute difference
            current_pos = len(descriptor_buffer)
            dist = abs(current_pos - child_idx)
            
            if dist > MAX_RELATIVE_PTR:
                # Far pointer
                far_ptr_offset = len(far_pointer_block)
                far_pointer_block.append(child_idx)
                # Store far pointer index in lower bits
                # Assuming lower bits store the index into far_pointer_block
                # The exact bit packing for far pointers is not fully specified in notes
                # but notes say "store the offset to the far pointer block"
                # We'll assume a specific bit range for far pointer index
                # Let's assume bits 0-15 store the far pointer index
                # And a flag bit indicates it's a far pointer
                # Note: "set the far_bit_mask in the descriptor"
                # We need to define far_bit_mask. Let's assume bit 0 is far flag.
                # This is a gap filled by [G]
                pass 
            else:
                # Relative pointer
                # Store relative distance in lower bits
                pass
        
        # Construct final descriptor
        # This part is incomplete due to lack of exact bit packing for pointers
        # The notes say "Lower bits store relative or far pointers"
        # We need to pack valid_mask, leaf_mask, and pointers into uint64_t
        # Bits 16-23: valid_mask
        # Bits 24-31: leaf_mask
        # Bits 0-15: pointers
        
        # Re-iterate to pack pointers
        pointer_data = 0
        for idx, (child_idx, leaf_bit, valid_bit) in enumerate(reversed(descriptor_position_array)):
            current_pos = len(descriptor_buffer)
            dist = abs(current_pos - child_idx)
            
            if dist > MAX_RELATIVE_PTR:
                # Far pointer
                far_ptr_offset = len(far_pointer_block)
                far_pointer_block.append(child_idx)
                # Set far bit flag (assuming bit 0 of pointer field)
                # And store offset
                # This is a guess for bit packing
                pointer_data |= (1 << (idx * 2)) # Far flag
                pointer_data |= (far_ptr_offset << (idx * 2 + 1)) # Offset
            else:
                # Relative pointer
                pointer_data |= (dist << (idx * 2)) # Relative distance
        
        descriptor = (valid_mask | leaf_mask | pointer_data)
        descriptor_buffer.append(descriptor)
        page_header_counter -= 1 # Assuming 1 descriptor per step
        return len(descriptor_buffer) - 1

    def is_block_empty(pos, scale, grid):
        # Check if all 8 subvoxels are empty
        for i in range(8):
            x = pos[0] + ((i >> 2) & 1) * scale
            y = pos[1] + ((i >> 1) & 1) * scale
            z = pos[2] + (i & 1) * scale
            if is_voxel_present(x, y, z):
                return False
        return True

    generation_recursion([0, 0, 0], oct_dimensions // 2)
    return descriptor_buffer, far_pointer_block
```

### Edge Cases and Pitfalls
- **Precision**: Relative pointers are limited to `0x8000`. If the tree is very deep or sparse, far pointers are required.
- **Boundaries**: Page boundaries must be handled to ensure far pointers are correctly resolved.
- **Degenerate Input**: Empty grids result in a single root descriptor with no valid children.

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of nodes in the octree. Each node is visited once.
- **Memory Complexity**: $O(N)$ for the descriptor buffer. Far pointer block adds overhead for large address spaces.
- **Performance**: Efficient for sparse data. Compact layout reduces memory footprint.
- **Strengths**: Handles large address spaces with far pointers. Compact memory layout.
- **Limitations**: Complex pointer management. Page boundary handling adds overhead.
- **When to Use**: Sparse voxel octrees with large spatial extent.
- **When Not to Use**: Dense grids or small address spaces where relative pointers suffice.

## Background
The method is described in the GitHub repository [voxel-raycaster](https://github.com/MitchellHansen/voxel-raycaster) by Mitchell Hansen [S1, unofficial]. It is part of a hardware-accelerated voxel raycasting implementation.

## Sources
[S1] GitHub - MitchellHansen/voxel-raycaster: Hardware accelerated voxel ray ..., https://github.com/MitchellHansen/voxel-raycaster, fetched 2026-10-25 (unofficial)
