# N3-tree with regular mipmapped bricks

Also called: N3-tree with regular voxel bricks, N3-tree+briques de voxels

## Overview
The N3-tree with regular mipmapped bricks is a two-level hierarchical data structure for procedural volume exploration that combines an N-ary tree (N3-tree) for hierarchy with regular voxel bricks stored in the leaves [S1, unofficial]. It links the compactness of a hierarchical structure with the efficient traversal and trilinear/quadrilinear filtering capabilities of regular grids on GPU [S1, unofficial]. It generalizes the original octree-based Brick Maps to allow arbitrary subdivision of nodes (N-ary instead of binary octree) to enable efficient GPU traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A procedural volume defined by a total maximum resolution $R_t$ on each axis, a maximum brick resolution $R_b$, and a node subdivision resolution $N$.
*   **Output:** A GPU-accessible structure consisting of an indirection texture (the tree), a brick storage texture (the data), and a brick indirection storage (mip-level mapping).

**Data Structures:**

1.  **Indirection Storage (The Tree):**
    *   **Format:** 3D texture using `GL_RGBA32I_EXT` (unsigned integer RGBA, 32 bits per component) [S1, unofficial].
    *   **Layout:** Each node in the N3-tree is stored as a small $N^3$ grid of indirection cells within this texture [S1, unofficial].
    *   **Spatial Locality:** The $N^3$ grids are organized using a Peano curve (Swizzled Walk) layout to maximize cache efficiency during tree traversal [S1, unofficial].
    *   **Cell Encoding (Sub-node information):** Each cell contains 32 bits packed as follows [S1, unofficial]:
        *   **Red (R):** 3D address (X, Y, Z) of the child node or the regular brick in the brick storage. Encoded in binary. 10 bits per coordinate (30 bits total).
        *   **Green (G):** 3D address of the parent node. Encoded similarly (10 bits per coordinate) [S1, unofficial].
        *   **Blue (B):** Reserved for future use [S1, unofficial].
        *   **Alpha (A):** Average value of all voxels in the node's covered space [S1, unofficial].
    *   **Flags:**
        *   1 bit identifies the sub-node type (brick or tree node) [S1, unofficial].
        *   1 bit identifies an empty node flag (no descendants) [S1, unofficial].
    *   **Addressable Dimension:** The 10-bit address allows addressing elements in a storage of maximum dimension $1024^3$ [S1, unofficial].

2.  **Brick Storage (The Data):**
    *   **Format:** 3D texture storing regular scalar density data (or gradients) [S1, unofficial].
    *   **Layout:** Divided into zones grouping bricks of identical resolution [S1, unofficial].
    *   **Brick Structure:** Each brick includes its data plus a 1-voxel border surrounding the data to allow trilinear filtering without artifacts from adjacent bricks [S1, unofficial].
    *   **Filtering:** Accessed in the shader by floating-point coordinates; integer coordinates from the indirection storage are transformed on the fly [S1, unofficial].

3.  **Brick Indirection Storage:**
    *   **Purpose:** Links the hierarchical tree to brick data [S1, unofficial].
    *   **Content:** A list of addresses for each brick's data for every mip-mapping level in the brick storage [S1, unofficial].
    *   **Unloaded Data:** Entries marked with a negative value indicate unloaded data [S1, unofficial].

**Manual Mip-mapping:**
Hardware mip-mapping requires duplicating the 1-voxel border $2^n$ times for a stack of $n$ levels and halving it at each sub-level, which is memory expensive [S1, unofficial]. This method implements 3D mip-mapping manually to achieve identical performance cost but with lower memory overhead [S1, unofficial].

**Number of Nodes:**
In the worst case (full resolution, no empty space skipping), the number of nodes to store in the tree is calculated as [S1, unofficial]:

$$ \sum_{i=0}^{n} \left( \frac{Nb}{N^i} \right)^3 $$

Where:
*   $n = \frac{\log(Nb)}{\log N}$
*   $Nb = \frac{R_t}{R_b}$
*   $R_t$: Total maximum resolution on each axis.
*   $R_b$: Maximum resolution of a brick.
*   $N$: Resolution of each node (e.g., $N=2$ for octree) [S1, unofficial].

**Parameters:**
*   **Example:** $R_t = 1024^3$, $N = 4$, $R_b = 16$.
*   **Result:** An indirection storage of $64^3$ is nearly sufficient. A practical recommended size is $128^3$ [S1, unofficial].
*   **Degrees of Freedom:** Node subdivision resolution ($N$) and regular brick resolution. These impact traversal performance and update efficiency [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_DIM_BITS = 10
MAX_DIM_SIZE = 1 << MAX_DIM_BITS  # 1024
NODE_SUBDIVISION = 4  # N
BRICK_RESOLUTION = 16 # Rb
TOTAL_RESOLUTION = 1024 # Rt

def encode_address_3d(x, y, z):
    """
    Encodes a 3D address into a 30-bit integer.
    Assumes x, y, z < 1024.
    """
    return (x << 20) | (y << 10) | z

def decode_address_3d(addr_30bit):
    """
    Decodes a 30-bit integer into x, y, z.
    """
    z = addr_30bit & 0x3FF
    y = (addr_30bit >> 10) & 0x3FF
    x = (addr_30bit >> 20) & 0x3FF
    return x, y, z

def calculate_tree_size(Rt, Rb, N):
    """
    Calculates the worst-case number of nodes in the N3-tree.
    """
    Nb = Rt / Rb
    n = log(Nb) / log(N)
    total_nodes = 0
    for i in range(int(n) + 1):
        level_nodes = (Nb / (N ** i)) ** 3
        total_nodes += level_nodes
    return total_nodes

def build_indirection_texture(Rt, Rb, N):
    """
    Constructs the indirection texture structure.
    Returns the texture object and its dimensions.
    """
    # Calculate required texture size
    # Worst case size calculation
    nodes_count = calculate_tree_size(Rt, Rb, N)
    
    # Determine texture dimensions (cube root of nodes_count, rounded up to power of 2 or suitable size)
    # Example: for Rt=1024, Rb=16, N=4, nodes_count ~ 64^3. 
    # Practical size recommended: 128^3
    tex_dim = 128 
    
    # Initialize 3D texture GL_RGBA32I_EXT
    texture = create_3d_texture(width=tex_dim, height=tex_dim, depth=tex_dim, format=GL_RGBA32I_EXT)
    
    # Traverse the logical N3-tree and populate the texture
    # The texture is organized using Peano curve (Swizzled Walk) for spatial locality
    peano_coords = generate_peano_coordinates(tex_dim)
    
    node_queue = [(0, 0, 0, 0, 0, 0, None)] # (x, y, z, level, parent_x, parent_y, parent_z, parent_addr)
    
    # Note: This is a simplified representation of the tree traversal and population
    # The actual implementation requires recursive or iterative tree construction
    
    current_peano_idx = 0
    
    def populate_node(x, y, z, level, parent_addr):
        nonlocal current_peano_idx
        
        # Determine if this node is a leaf (brick) or internal node
        # Logic depends on volume data density and resolution limits
        is_leaf = (level == n) or (node_is_empty_or_constant(x, y, z))
        
        # Calculate average value for Alpha channel
        avg_value = compute_average_voxel_value(x, y, z, level, Rt, Rb, N)
        
        # Get Peano coordinate for this node's grid position
        px, py, pz = peano_coords[current_peano_idx]
        current_peano_idx += 1
        
        # Encode sub-node information
        # If leaf: address points to brick in brick storage
        # If internal: address points to child node grid in indirection texture
        
        if is_leaf:
            # Address of the brick
            brick_addr_30 = encode_address_3d(x, y, z) # Simplified brick addressing
            node_type_bit = 1 # Brick
            empty_flag = 0
        else:
            # Address of the child node grid (relative or absolute)
            # In the worst case, all children exist
            child_grid_addr = compute_child_grid_address(x, y, z, level, N, tex_dim)
            brick_addr_30 = encode_address_3d(child_grid_addr) # Simplified
            node_type_bit = 0 # Tree Node
            empty_flag = 0
            
        # Pack RGBA
        # R: Child/Brick Address (30 bits)
        # G: Parent Address (30 bits)
        # B: Reserved (0)
        # A: Average Value (8 bits? or 32 bit float/int? Source says 32 bits per component, Alpha stores average)
        
        # Note: Source specifies 32 bits per component. 
        # R and G use 30 bits for address + 1 bit type + 1 bit empty? 
        # Actually, source says: "10 bits per coordinate (X, Y, Z) = 30 bits total. 
        # 1 bit for sub-node type. 1 bit for empty node flag. Total 32 bits per sub-node."
        # This implies the 32 bits are split between R, G, B, A? 
        # Re-reading: "Red (R): 3D address... Green (G): 3D address of parent... 
        # Blue (B): Reserved... Alpha (A): Average value..."
        # And: "On the 32 bits available per sub-node..."
        # This is a contradiction. 32 bits per component * 4 components = 128 bits.
        # But the text says "On the 32 bits available per sub-node... 10 bits per coordinate...".
        # It seems the "sub-node" is stored in ONE component (likely Red or a combined structure), 
        # OR the text implies the address is packed into one 32-bit integer which is stored in one channel.
        # Let's look at: "Red (R): 3D address... Green (G): 3D address of parent...".
        # If R is 30 bits + flags, it fits in 32 bits.
        # If G is 30 bits + flags, it fits in 32 bits.
        # So R and G are 32-bit integers. B and A are 32-bit integers.
        
        r_channel = (brick_addr_30 << 2) | (node_type_bit << 1) | empty_flag # Assuming 2 bits for flags? 
        # Source: "1 bit for sub-node type... 1 bit for empty node flag."
        # So 2 bits for flags. 30 bits for address. Total 32 bits.
        
        g_channel = (encode_address_3d(parent_x, parent_y, parent_z) << 2) | 0 # Parent address, no flags mentioned for parent?
        # Source: "Green (G): 3D address of the parent node... encoded similarly."
        
        b_channel = 0 # Reserved
        
        # Alpha: Average value. Source doesn't specify bit depth for average. 
        # Assuming 32-bit integer or float representation as per texture format.
        a_channel = int(avg_value * 255) if avg_value <= 1.0 else avg_value 
        
        texture.set_pixel(px, py, pz, (r_channel, g_channel, b_channel, a_channel))
        
        if not is_leaf:
            # Recursively populate children
            for dx in range(N):
                for dy in range(N):
                    for dz in range(N):
                        child_x = x * N + dx
                        child_y = y * N + dy
                        child_z = z * N + dz
                        populate_node(child_x, child_y, child_z, level + 1, r_channel)

    # Start traversal from root
    # Root address is typically (0,0,0)
    populate_node(0, 0, 0, 0, 0)
    
    return texture

def build_brick_storage(Rt, Rb):
    """
    Constructs the brick storage texture and brick indirection storage.
    """
    # Brick storage texture
    brick_tex = create_3d_texture(width=Rb, height=Rb, depth=Rb, format=GL_RGBA32I_EXT) # Or appropriate data format
    
    # Add 1-voxel border to each brick
    # This requires duplicating border voxels for mip-mapping levels if hardware mip-mapping were used.
    # Since we use manual mip-mapping, we handle border logic in shader or pre-process.
    
    # Brick Indirection Storage
    # List of addresses for each brick's data for every mip-mapping level
    brick_indirection = []
    
    # Calculate number of bricks
    num_bricks_x = Rt / Rb
    num_bricks_y = Rt / Rb
    num_bricks_z = Rt / Rb
    
    for x in range(num_bricks_x):
        for y in range(num_bricks_y):
            for z in range(num_bricks_z):
                brick_addr = encode_address_3d(x, y, z)
                # For each mip level, store address or -1 if unloaded
                mip_levels = int(log(Rb) / log(2)) + 1
                brick_entry = [-1] * mip_levels
                brick_entry[0] = brick_addr # Level 0 is loaded
                brick_indirection.append(brick_entry)
                
    return brick_tex, brick_indirection
```

## Subtopics

### Brick Maps
The original octree-based structure for global illumination that this method generalizes. [Brick Maps](3976-brick-occupancy-recomputation.md)

### Hierarchical Grid Traversal
The broader category of ray-grid traversal techniques that use hierarchical structures to skip empty space. [Hierarchical Grid Traversal](52-name.md)

## Operating characteristics
*   **Time Complexity:** Traversal is efficient on GPU due to the regular grid structure of leaves and the use of 3D textures which exploit hardware cache spatial locality (via Peano curve layout) [S1, unofficial].
*   **Memory Complexity:** The indirection storage size is determined by the worst-case node count. For $R_t=1024^3, N=4, R_b=16$, a $64^3$ texture is nearly sufficient, but $128^3$ is recommended for practical scenes [S1, unofficial]. The manual mip-mapping reduces memory overhead compared to hardware mip-mapping by avoiding border duplication [S1, unofficial].
*   **Strengths:** Combines hierarchical compactness with regular grid filtering efficiency. Manual mip-mapping saves memory. Peano curve layout optimizes cache hits [S1, unofficial].
*   **Limitations:** Worst-case memory usage can be high if the volume is dense. Update efficiency is secondary to traversal performance but impacted by the choice of $N$ and brick resolution [S1, unofficial].
*   **When to use:** Procedural volume exploration on GPU where memory bandwidth and cache efficiency are critical.
*   **When not to use:** If hardware mip-mapping memory overhead is not a concern and the volume is sparse enough that a simpler octree suffices.

## Background
The method was developed by Sylvain Lefebvre et al. as an adaptation of the hierarchical volumetric textures model proposed by Sylvain Lefebvre [LHN05] for GPU implementation [S1, unofficial]. It generalizes the Brick Maps proposed by Christensen and Batali [CB04] for global illumination [S1, unofficial]. The work is part of the thesis "Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés" [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
