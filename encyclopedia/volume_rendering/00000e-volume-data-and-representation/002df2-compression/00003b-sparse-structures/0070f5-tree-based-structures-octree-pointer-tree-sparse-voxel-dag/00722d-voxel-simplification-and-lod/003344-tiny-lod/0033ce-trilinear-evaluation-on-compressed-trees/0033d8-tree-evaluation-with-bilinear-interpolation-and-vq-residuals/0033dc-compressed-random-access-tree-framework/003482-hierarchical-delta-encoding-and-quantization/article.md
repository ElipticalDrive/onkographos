# Hierarchical Delta Encoding and Quantization

Also called: Tree Compression

## Overview
Hierarchical Delta Encoding and Quantization is a compression method for adaptive multiresolution tree structures (such as octrees and quadtrees) used in volume rendering. It solves the problem of storing large, unbalanced hierarchical meshes by replacing explicit pointers with compact, top-down delta-encoded spatial data and quantized field values [S1, unofficial]. Unlike sibling methods that may use fixed-rate topology encoding or generic quantization, this approach exploits the geometric constraint that child bounding spheres must fit within their parent's diameter to map spatial offsets and radii into a small, fixed lookup table [S1, unofficial]. This allows for aggressive compression of the tree's geometric structure while retaining random access for rendering [S1, unofficial].

## How it works
The method compresses a hierarchical tree (Solid or Shell) into a binary stream by encoding each node's geometry, topology, and attributes into a fixed-size bit layout. The encoding is top-down: child attributes are defined relative to their parent.

### Inputs and Outputs
*   **Input:** An adaptive multiresolution tree (Solid Octree or Shell Quadtree) where each node has a bounding sphere (center $x,y,z$, radius $r$), a list of children, field values (scalar attributes), and optional normal vectors (Shell nodes).
*   **Output:** A binary data stream (file) containing the compressed nodes in breadth-first order.

### Data Structures and Layout
The core data structure is the **Solid Node**, which is 48 bits (6 bytes) long. The layout is little-endian byte order, with bits ordered MSB-leftmost within each byte.

**Solid Node Layout (48 bits / 6 bytes):**
*   **Bytes 0-2 (24 bits):**
    *   **Bits 0-12 (13 bits):** Spatial/Radius Lookup Index. This index encodes the quantized $x, y, z$ offsets and radius $r$.
    *   **Bits 13-15 (3 bits):** Number of children (0 to 8).
    *   **Bit 16 (1 bit):** Presence of grandchildren (1 if children have children, 0 otherwise).
*   **Bytes 3-4 (30 bits):**
    *   **Bits 17-31 (15 bits):** Quantized Field Value 1.
    *   **Bits 32-46 (15 bits):** Quantized Field Value 2.
*   **Byte 5 (8 bits):**
    *   **Bit 47 (1 bit):** Boundary Node Flag (1 if boundary, 0 if regular).
    *   *Note:* The remaining 7 bits of Byte 5 are unused/padding in the described 6-byte layout, or the layout description implies a specific packing where the boundary bit is the last significant bit. Based on the sum $13+3+1+15+15+1 = 48$, the bits are contiguous.

**Shell Node Layout:**
*   Same as Solid Node, but replaces the Boundary Node Flag and potentially adjusts field storage to include:
    *   **Extra 2 bytes (16 bits):** Hierarchically encoded normal vector (14 bits) and width of normal cone (2 bits).
    *   No boundary label bit.

### Parameters
*   **Spatial Quantization:**
    *   Radius $r$ is quantized to **13 discrete values** relative to the parent radius.
    *   Spatial offsets $x, y, z$ (from parent center) are quantized to **13 discrete values** relative to the parent diameter.
*   **Lookup Table:**
    *   A fixed table maps valid combinations of $(x, y, z, r)$ quantum numbers to a **13-bit index**.
    *   Only combinations satisfying the hierarchical bounding sphere condition (child sphere fits inside parent) are valid.
*   **Field Quantization:**
    *   Field values are normalized to $[0, 1]$.
    *   Quantized using a lookup table.
    *   **15 bits** per field value is sufficient for studied datasets.
    *   Arbitrary numbers of field values can be appended, each taking 2 bytes (16 bits, though only 15 are used for data).

### Steps
1.  **Build Tree:** Construct the Solid (octree) or Shell (quadtree) hierarchy. For Shell, compute normals by averaging child normals.
2.  **Normalize Fields:** For each node, normalize all scalar field values to $[0, 1]$.
3.  **Quantize Fields:** Map normalized field values to 15-bit integers using a predefined lookup table.
4.  **Compute Delta Encoding:** For each child node:
    *   Calculate offset $dx, dy, dz$ from parent center to child center.
    *   Calculate child radius $r$.
    *   Quantize $r$ to one of 13 values relative to parent radius.
    *   Quantize $dx, dy, dz$ to one of 13 values relative to parent diameter.
5.  **Lookup Index:** Find the combination of $(dx, dy, dz, r)$ in the fixed 13-bit lookup table. If the combination is invalid (does not fit), it should not occur due to construction constraints.
6.  **Encode Node:**
    *   Pack the 13-bit lookup index.
    *   Pack the number of children (3 bits, max 8).
    *   Pack the grandchildren presence bit (1 bit).
    *   Pack the 15-bit quantized field values.
    *   Pack the boundary flag (1 bit) for Solid nodes.
    *   For Shell nodes, pack the 14-bit normal and 2-bit cone width.
7.  **Stream Output:** Write nodes to a binary stream in breadth-first order.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computed lookup table mapping valid (dx_idx, dy_idx, dz_idx, r_idx) to 13-bit index
# dx_idx, dy_idx, dz_idx, r_idx are integers in [0, 12]
LOOKUP_TABLE = {} 

def build_lookup_table():
    """
    Populates LOOKUP_TABLE with all valid combinations of quantized spatial offsets
    and radius that satisfy the hierarchical bounding sphere condition.
    Returns a dict: (dx, dy, dz, r) -> 13-bit index
    """
    table = {}
    index = 0
    # Iterate over all 13^4 combinations
    for dx in range(13):
        for dy in range(13):
            for dz in range(13):
                for r in range(13):
                    # Check if this combination is geometrically valid
                    # (Child sphere fits inside parent sphere)
                    if is_valid_bounding_sphere(dx, dy, dz, r):
                        table[(dx, dy, dz, r)] = index
                        index += 1
    return table

def is_valid_bounding_sphere(dx, dy, dz, r):
    """
    Check if a child sphere with offset (dx, dy, dz) and radius r
    fits inside the parent sphere.
    dx, dy, dz are quantized offsets relative to parent diameter.
    r is quantized radius relative to parent radius.
    This function implements the geometric constraint.
    """
    # Implementation depends on the specific quantization scaling.
    # Generally: distance_from_parent_center + child_radius <= parent_radius
    # Let D be parent diameter, R be parent radius. D = 2*R.
    # Offset in physical units: (dx/12) * D ? Or similar scaling.
    # Radius in physical units: (r/12) * R ?
    # The exact scaling factors are defined by the quantization scheme.
    # For the purpose of this article, we assume the condition is checked
    # against the parent's bounding sphere.
    pass

def quantize_field(value, field_lut):
    """
    Quantize a normalized field value [0, 1] to 15 bits using a lookup table.
    """
    # field_lut maps float [0,1] to int [0, 32767]
    return field_lut[value]

def encode_solid_node(node, parent_node, field_luts):
    """
    Encode a Solid tree node into 6 bytes.
    """
    # 1. Spatial/Radius Encoding
    dx = node.center.x - parent_node.center.x
    dy = node.center.y - parent_node.center.y
    dz = node.center.z - parent_node.center.z
    r = node.radius
    
    # Quantize to 13 discrete values
    dx_idx = quantize_offset(dx, parent_node.diameter)
    dy_idx = quantize_offset(dy, parent_node.diameter)
    dz_idx = quantize_offset(dz, parent_node.diameter)
    r_idx = quantize_radius(r, parent_node.radius)
    
    # Lookup index
    spatial_index = LOOKUP_TABLE[(dx_idx, dy_idx, dz_idx, r_idx)]
    
    # 2. Topology
    num_children = len(node.children)
    has_grandchildren = any(len(child.children) > 0 for child in node.children)
    
    # 3. Fields
    field1_q = quantize_field(node.field1, field_luts[0])
    field2_q = quantize_field(node.field2, field_luts[1])
    
    # 4. Boundary Flag
    is_boundary = node.is_boundary
    
    # 5. Pack Bits
    # Layout: 13 bits (spatial), 3 bits (children), 1 bit (grandchildren),
    #         15 bits (field1), 15 bits (field2), 1 bit (boundary)
    # Total 48 bits = 6 bytes
    
    bits = 0
    bits |= (spatial_index & 0x1FFF) << 35  # 13 bits at MSB of first 3 bytes? 
    # Note: Bit order is MSB-leftmost. Byte order is Little-Endian.
    # Let's construct the 6-byte buffer directly.
    
    # Byte 0-2: Spatial (13), Children (3), Grandchildren (1) -> 17 bits
    # Byte 3-4: Field1 (15), Field2 (15) -> 30 bits
    # Byte 5: Boundary (1), Padding (7) -> 8 bits
    
    # Re-evaluating bit packing for Little-Endian, MSB-first within byte
    # We will build a 48-bit integer and then unpack to bytes.
    
    packed = 0
    packed |= (spatial_index & 0x1FFF) << 35  # Bits 35-47 (13 bits)
    packed |= (num_children & 0x7) << 32      # Bits 32-34 (3 bits)
    packed |= (1 if has_grandchildren else 0) << 31 # Bit 31 (1 bit)
    packed |= (field1_q & 0x7FFF) << 16       # Bits 16-30 (15 bits)
    packed |= (field2_q & 0x7FFF) << 1        # Bits 1-15 (15 bits)
    packed |= (1 if is_boundary else 0)       # Bit 0 (1 bit)
    
    # Convert to 6 bytes, Little-Endian
    byte0 = packed & 0xFF
    byte1 = (packed >> 8) & 0xFF
    byte2 = (packed >> 16) & 0xFF
    byte3 = (packed >> 24) & 0xFF
    byte4 = (packed >> 32) & 0xFF
    byte5 = (packed >> 40) & 0xFF
    
    return bytes([byte0, byte1, byte2, byte3, byte4, byte5])

def quantize_offset(offset, parent_diameter):
    """
    Quantize offset to 13 discrete values of parent diameter.
    """
    # Normalize offset to [-0.5, 0.5] relative to diameter?
    # Or [0, 1]? The text says "offsets from parent center".
    # Assuming symmetric quantization.
    normalized = offset / parent_diameter
    # Map to 13 levels. E.g., -0.5 to 0.5 mapped to 0..12.
    # Level = floor((normalized + 0.5) * 13)
    level = int((normalized + 0.5) * 13)
    return max(0, min(12, level))

def quantize_radius(radius, parent_radius):
    """
    Quantize radius to 13 discrete values of parent radius.
    """
    # Assuming radius <= parent_radius.
    # Map 0..1 to 0..12.
    normalized = radius / parent_radius
    level = int(normalized * 13)
    return max(0, min(12, level))
```

### Edge Cases and Pitfalls
*   **Invalid Combinations:** The lookup table only contains valid combinations. If a child node does not fit within the parent's bounding sphere due to construction errors, the encoding will fail. The tree construction must guarantee this constraint.
*   **Precision:** 15 bits for field values is sufficient for studied datasets but may not be for all. Users should verify quantization error for their data.
*   **Grandchildren Bit:** The 1-bit grandchildren flag is a simplification. It does not encode the *number* of grandchildren, only their presence. This is sufficient for traversal if the topology is encoded separately or if the tree structure is known.
*   **Shell Node Normals:** Normal vectors are hierarchically encoded. The 14-bit normal and 2-bit cone width must be decoded correctly to reconstruct the shell geometry.

## Subtopics

### [Snapped Grid Extent Quantization](19480-snapped-grid-extent-quantization.md)
A BVH data layout technique that compresses AABB coordinates into fixed-point integers relative to a global coordinate frame. It achieves aggressive compression (12 bytes per node) but relies on directed rounding modes for watertight dequantization, making it less portable than standard arithmetic methods.

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N)$ where $N$ is the number of nodes, as each node is processed once. Decoding is also $O(N)$ for traversal.
*   **Memory Complexity:** Each Solid node is 6 bytes. Each Shell node is 8 bytes (6 + 2). This is significantly smaller than storing full floating-point coordinates and pointers.
*   **Performance:** The method enables fast top-down multi-resolution rendering by flushing the tree to a binary stream in breadth-first order, facilitating efficient memory access patterns.
*   **Strengths:** High compression ratio for spatial data; simple lookup-based quantization; supports arbitrary field values.
*   **Limitations:** Fixed lookup table size limits the flexibility of spatial quantization; requires pre-computation of the lookup table; unbalanced trees still require embedded pointers (though the text mentions "minimize the number of pointers", it does not eliminate them entirely in the described layout, unlike the parent framework's goal).

## Background
The method was introduced by Museth and Lombeyda in 2004 in the paper "TetSplat: Real-time Rendering and Volume Clipping of Large ..." [S1, unofficial]. It builds on earlier work in splatting and hierarchical volume rendering, aiming to reduce the memory footprint of large adaptive meshes for real-time visualization. The approach is part of the TetSplat system, which uses tetrahedral splatting for volume rendering.

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
