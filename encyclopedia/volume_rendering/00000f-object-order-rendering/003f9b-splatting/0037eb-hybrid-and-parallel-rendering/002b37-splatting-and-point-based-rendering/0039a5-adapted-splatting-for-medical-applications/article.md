# Adapted Splatting for Medical Applications

Also called: Adapted Splatting, Optimized Splatting

## Overview
Adapted Splatting is a volume rendering method that optimizes standard splatting by culling voxels invisible to the observer and using a compressed data format to reduce memory usage for free voxels [S1, unofficial]. It solves the high computational cost of processing all voxels in a model and the large memory footprint of storing sparse volumetric data. It sets itself apart from standard splatting by introducing a two-pass optimization: a depth-buffer-based occlusion culling pass that identifies only visible voxels, followed by rendering only those selected voxels, while simultaneously employing a run-length-like encoding for storage that groups consecutive free voxels [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A voxel model defined in the adapted file format (see Data Structures).
*   Camera parameters (view direction, projection matrix).
*   Kernel function parameters (e.g., Gaussian radius).
*   Screen dimensions (width $W$, height $H$).

**Outputs:**
*   A 2D image (framebuffer) with color/intensity values.
*   (Implicitly) A set of "visible" voxels identified during the culling phase.

### Data Structures

#### File Format (On-Disk)
The model is stored as a sequence of bytes. The structure is defined as follows [S1, unofficial]:

1.  **Segment Definition:** A segment consists of two integers:
    *   $f_i$: The count of consecutive *free* (empty) voxels preceding the occupied ones in this segment.
    *   $n_i$: The count of *occupied* voxels following $f_i$.
2.  **Voxel Data:** Immediately following $n_i$, there are $n_i$ voxels. Each voxel includes its coordinates and any additional application-specific attributes. The size of these additional attributes is defined during model description (number of bytes per attribute is fixed per attribute type).
3.  **Termination:** A segment description ends when $n_i = 0$. This sentinel value indicates the end of the occupied voxel list for that segment.
4.  **Free Segment Handling:** Segments where $n_i = 0$ (i.e., no occupied voxels) are **not** stored in the file sequence.

#### In-Memory Data Structure
To allow efficient access during rendering, the in-memory representation consists of two parts [S1, unofficial]:

1.  **Byte Sequence:** A contiguous array of bytes representing the concatenated segments from the file format (excluding empty segments).
2.  **Offset Array:** An array of indices (byte offsets) into the Byte Sequence.
    *   The $k$-th entry in the Offset Array stores the starting byte index in the Byte Sequence for the $k$-th segment.
    *   If a segment contains no occupied voxels (which are excluded from the file, but conceptually exist in the grid), the corresponding value in the Offset Array is $-1$.

*Note: The exact integer size for $f_i$ and $n_i$ is not specified in the sources [S1, unofficial].*

### Algorithm Steps

#### 1. Preprocessing
*   Calculate the kernel function values for the chosen footprint function.
*   Store these values in a lookup table to avoid computing the footprint function during rendering [S1, unofficial].
*   The Gaussian function is the most common kernel due to the ease of table lookup [S1, unofficial].

#### 2. Occlusion Culling (Buffer Plane Pass)
This pass identifies which voxels contribute to the final image by simulating a depth test.

1.  Initialize a **Buffer Plane** (depth buffer) with dimensions equal to the screen plane ($W \times H$). Initialize all entries to $+\infty$ (or the maximum possible Z value).
2.  Iterate through the entire voxel model (reading from the In-Memory Data Structure).
3.  For each occupied voxel $v$:
    a. Project the voxel center onto the screen plane to get pixel coordinates $(u, v)$.
    b. Determine the footprint radius $R$ based on the kernel and projection.
    c. Iterate over all pixels $(x, y)$ within the bounding box of the footprint centered at $(u, v)$.
    d. For each pixel $(x, y)$, compare the voxel's depth $Z_v$ with the current value in the Buffer Plane $B[x, y]$.
    e. If $Z_v < B[x, y]$, update $B[x, y] = Z_v$. Otherwise, leave $B[x, y]$ unchanged.
    f. Track whether any pixel in the footprint was updated or if the footprint overlaps with any pixel that has a finite depth value. *Correction based on source logic:* The source states: "Select only those voxels whose projection onto the buffer plane exists at least partially (i.e., they were not fully occluded by closer voxels in Step 2)." This implies we must check if the voxel's footprint is *completely* covered by closer voxels.
    g. **Refined Logic for Selection:**
        i. Before updating, check if the voxel's footprint is *fully* occluded. A voxel is fully occluded if, for all pixels $(x, y)$ in its footprint, the current Buffer Plane value $B[x, y]$ is less than $Z_v$ (meaning a closer voxel already covers that area).
        ii. If the voxel is **not** fully occluded, mark it as **Visible**.
        iii. If the voxel is **not** fully occluded, perform the depth update: for all $(x, y)$ in footprint, if $Z_v < B[x, y]$, set $B[x, y] = Z_v$.
        iv. If the voxel **is** fully occluded, discard it (do not update buffer, do not render).

*Note: The source description of Step 2 says "Fill pixels... if Z is less... otherwise leave current value." Step 3 says "Select only those voxels whose projection... exists at least partially." This implies the culling decision is based on whether the voxel contributes to the depth buffer or overlaps with existing depth. A voxel is discarded if its projected radius is completely covered by the radii of other points closer to the screen [S1, unofficial].*

#### 3. Rendering (Splatting Pass)
1.  Initialize the final Image Framebuffer to transparent/background color.
2.  Iterate through the list of **Visible** voxels identified in Step 2.
3.  For each visible voxel $v$:
    a. Project $v$ onto the screen plane.
    b. For each pixel $(x, y)$ in the footprint:
        i. Retrieve the kernel weight $K$ from the lookup table based on the distance from the voxel center to the pixel center.
        ii. Compute the contribution $C = K \times \text{Value}(v)$.
        iii. Composite $C$ into the Image Framebuffer at $(x, y)$ using standard volume compositing (e.g., front-to-back alpha blending or ray-marching integration equivalent).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adapted_splatting_render(voxel_model, camera, screen_width, screen_height, kernel_lookup_table):
    """
    Renders a voxel model using Adapted Splatting.
    
    Args:
        voxel_model: Object containing 'byte_sequence' and 'offset_array'.
                     Each segment in the sequence starts with f_i, n_i, followed by n_i voxels.
        camera: Object with methods to project voxel to screen.
        screen_width: int
        screen_height: int
        kernel_lookup_table: dict or array mapping distance -> weight.
        
    Returns:
        framebuffer: 2D array of shape (screen_height, screen_width, channels)
    """
    
    # 1. Initialize Buffer Plane (Depth Buffer)
    # Initialize with infinity to represent empty space
    buffer_plane = [[float('inf')] * screen_width for _ in range(screen_height)]
    
    # List to store visible voxels for the second pass
    visible_voxels = []
    
    # 2. Parse and Cull Pass
    # Iterate through segments
    for seg_idx, start_offset in enumerate(voxel_model.offset_array):
        if start_offset == -1:
            continue # Segment has no occupied voxels
            
        # Read segment header: f_i and n_i
        # Note: Exact byte parsing depends on integer size, assumed 4-byte ints here
        f_i = read_int(voxel_model.byte_sequence, start_offset)
        n_i = read_int(voxel_model.byte_sequence, start_offset + 4)
        
        voxel_data_start = start_offset + 8
        
        # Iterate through occupied voxels in this segment
        for k in range(n_i):
            voxel = parse_voxel(voxel_model.byte_sequence, voxel_data_start + k * voxel_size)
            
            # Project voxel to screen
            screen_coords = camera.project(voxel.center)
            u, v = screen_coords.x, screen_coords.y
            z_v = voxel.center.z # Depth value
            
            # Determine footprint radius R based on kernel and projection
            R = get_footprint_radius(voxel, camera)
            
            # Check for full occlusion and update buffer
            is_fully_occluded = True
            pixels_to_update = []
            
            # Define bounding box of footprint
            min_x = max(0, int(u - R))
            max_x = min(screen_width - 1, int(u + R))
            min_y = max(0, int(v - R))
            max_y = min(screen_height - 1, int(v + R))
            
            for y in range(min_y, max_y + 1):
                for x in range(min_x, max_x + 1):
                    current_depth = buffer_plane[y][x]
                    
                    # If current buffer depth is greater than voxel depth, 
                    # this pixel is not fully occluded by a closer voxel
                    if z_v < current_depth:
                        is_fully_occluded = False
                        pixels_to_update.append((x, y))
            
            # If the voxel is not fully occluded, it is visible
            if not is_fully_occluded:
                visible_voxels.append(voxel)
                
                # Update the buffer plane with the voxel's depth
                for (x, y) in pixels_to_update:
                    buffer_plane[y][x] = z_v

    # 3. Rendering Pass
    framebuffer = [[background_color] * screen_width for _ in range(screen_height)]
    
    for voxel in visible_voxels:
        screen_coords = camera.project(voxel.center)
        u, v = screen_coords.x, screen_coords.y
        R = get_footprint_radius(voxel, camera)
        
        min_x = max(0, int(u - R))
        max_x = min(screen_width - 1, int(u + R))
        min_y = max(0, int(v - R))
        max_y = min(screen_height - 1, int(v + R))
        
        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                # Calculate distance from voxel center to pixel center in screen space
                dist = sqrt((x - u)**2 + (y - v)**2)
                
                # Get kernel weight from lookup table
                weight = kernel_lookup_table.get(dist, 0.0)
                
                # Compute contribution
                # Assuming voxel.value is the scalar attribute (e.g., density)
                contribution = weight * voxel.value
                
                # Composite into framebuffer
                # Standard front-to-back alpha blending or accumulation
                # Note: Exact compositing formula (e.g., alpha blending vs ray marching) 
                # is not specified beyond "integration of energy". 
                # We assume simple accumulation or alpha blending.
                # Let's assume standard alpha blending for simplicity, 
                # though volume rendering often uses ray-marching integration.
                # Given the source says "energy is distributed... integration", 
                # and splatting often implies additive or alpha-blended accumulation.
                
                # Placeholder for specific compositing equation not detailed in source
                # framebuffer[y][x] = composite(framebuffer[y][x], contribution, weight)
                pass # Implementation detail of compositing depends on specific kernel/alpha model

    return framebuffer

def read_int(byte_sequence, offset):
    # Assuming 4-byte little-endian integers
    return int.from_bytes(byte_sequence[offset:offset+4], byteorder='little')

def parse_voxel(byte_sequence, offset):
    # Parses voxel data based on predefined attribute sizes
    # Returns a voxel object with center coordinates and additional attributes
    pass

def get_footprint_radius(voxel, camera):
    # Returns the radius R of the kernel footprint in screen pixels
    # Depends on voxel size, kernel width, and camera projection
    pass
```

### Parameters
*   **Kernel:** A function used to convolve each voxel to create its 3D image. It is identical for all voxels, independent of view direction, and calculated during preprocessing [S1, unofficial]. The Gaussian function is the most common [S1, unofficial].
*   **Additional Attributes:** Each voxel can store extra data specific to the application. The number of bytes for each additional attribute is defined during model description [S1, unofficial].

### Edge Cases and Pitfalls
*   **Occlusion Culling Logic:** A voxel is discarded if its projected radius is completely covered by the radii of other points closer to the screen (smaller Z) [S1, unofficial]. Care must be taken to ensure the depth comparison is strict ($Z_v < B[x,y]$) to handle co-planar or touching voxels correctly.
*   **Precision:** The source mentions increasing the table resolution for higher quality and to prevent artifacts [S1, unofficial].
*   **Empty Segments:** Segments with no occupied voxels are not stored in memory, but their existence in the logical grid is tracked via the offset array value of $-1$ [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** The adapted format significantly reduces memory usage for sparse data by grouping free voxels. The in-memory structure requires storing the byte sequence and an offset array. The size of the byte sequence is proportional to the number of occupied voxels and their attributes, plus the overhead of $f_i$ and $n_i$ integers per segment [S1, unofficial].
*   **Time:** The culling pass requires iterating through all voxels and potentially updating the buffer plane. The rendering pass only processes visible voxels. This reduces the number of kernel evaluations and compositing operations compared to standard splatting, which processes all voxels [S1, unofficial].

### Measured Performance
Tests were conducted on an Intel Core2 CPU E8400 3.0 GHz with a GTX 470 GPU [S1, unofficial]:

| Model | Original Splatting Time | Adapted Splatting Time | Speedup |
| :--- | :--- | :--- | :--- |
| Chest | 8.4 ms | 1.9 ms | ~4.4x |
| Tibia | 3.4 ms | 1.1 ms | ~3.1x |
| Skull | 4.1 ms | 1.3 ms | ~3.2x |

### File Size Reduction
| Model | Standard Storage Size | Adapted Storage Size | Reduction Factor |
| :--- | :--- | :--- | :--- |
| Chest | 8,411,000 bytes | 38,231 bytes | ~220x |
| Tibia | 72,457 bytes | 36,220 bytes | ~2x |
| Skull | 122,600 bytes | 56,487 bytes | ~2.2x |
| Aircraft (Large) | Not specified | Not specified | ~22x |

*Note: The "Aircraft" model is mentioned in the text as having a 22x reduction, but is not in the table [S1, unofficial].*

### Strengths and Limitations
*   **Strengths:** Significant reduction in file size for sparse models (especially large ones with many free voxels). Reduced rendering time by avoiding computation for occluded voxels. Supports application-specific additional attributes [S1, unofficial].
*   **Limitations:** Requires a preprocessing step to build the lookup table and parse the custom format. The culling pass adds an initial overhead, though it is offset by the reduced rendering work. The method is tailored for voxel-based data and may not directly apply to surface-based representations [S1, unofficial].

## Background
The method was proposed by S. I. Vyatkin and A. N. Romanyuk [S1, unofficial]. It addresses the limitations of standard volume graphics, which require large amounts of memory and significant computation for filtering [S1, unofficial]. The work builds on the foundational splatting algorithm where voxels are treated as 3D points convolved with a kernel [S1, unofficial].

## Sources

[S1] An adapted splatting method for medical applications, https://ir.lib.vntu.edu.ua/bitstream/123456789/25853/1/33-40.pdf, fetched 2026-10-26 (unofficial)
