# Adapted Splatting for Medical Applications

Also called: Adapted Splatting, Optimized Splatting

## Overview
Adapted Splatting is a volume rendering method that optimizes standard splatting by culling voxels invisible to the observer and using a compressed data format to reduce memory usage for free voxels [S1, unofficial]. It solves the performance bottleneck of standard splatting, which processes all voxels in a model regardless of visibility, and the memory overhead of storing sparse medical datasets where large volumes are empty [S1, unofficial]. It sets itself apart from its siblings by combining a specific run-length-like file format for free/occupied voxel segments with a two-pass CPU-based occlusion culling buffer before applying the GPU rasterization [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A 3D voxel grid representing a medical volume.
*   Camera parameters (view matrix, projection matrix, screen resolution).
*   Kernel function parameters (e.g., Gaussian radius).

**Outputs:**
*   A 2D rendered image.
*   A compressed file representation of the voxel data.

### Data Structures

#### File Format (Model Segments)
The model is stored as a sequence of bytes. The structure is defined as follows [S1, unofficial]:
1.  **Segment Definition**: Each segment consists of two integers:
    *   $f_i$: The count of consecutive free (empty) voxels.
    *   $n_i$: The count of occupied voxels following the free voxels.
2.  **Voxel Data**: Following $n_i$, there are $n_i$ voxels. Each voxel includes its core attributes (coordinates) and any additional application-specific attributes. The size of additional attributes is defined during model description.
3.  **Termination**: A segment description ends when $n_i = 0$.
4.  **Storage Rule**: Segments where $n_i = 0$ (i.e., segments containing only free voxels) are not stored in the file or memory [S1, unofficial].

#### In-Memory Data Structure
The model in RAM consists of two parts [S1, unofficial]:
1.  **Byte Sequence**: The concatenated sequence of segment descriptions (as defined in the file format).
2.  **Offset Array**: An array storing the byte offset (start index) in the byte sequence for each segment. If a segment contains no occupied voxels, the corresponding value in this array is $-1$.

### Algorithm Steps

#### 1. Preprocessing
*   Calculate the kernel function values (footprint function) and store them in a lookup table [S1, unofficial].
*   The kernel is identical for all voxels and independent of the view direction. The Gaussian function is most common due to ease of table lookup [S1, unofficial].
*   Higher quality can be achieved by increasing the resolution of the lookup table [S1, unofficial].

#### 2. Occlusion Culling (Optimization Step 1 & 2)
This step determines which voxels are potentially visible by simulating depth on a 2D buffer [S1, unofficial].

1.  Create a **Buffer Plane** with the same dimensions as the screen plane. Initialize this buffer with maximum depth values (or infinity) [S1, unofficial].
2.  Iterate through the entire model (reading from the in-memory structure).
3.  For each occupied voxel:
    *   Project the voxel center onto the buffer plane.
    *   Determine the footprint radius based on the kernel radius.
    *   For each pixel in the buffer plane covered by this footprint:
        *   Compare the voxel's Z-value (depth) with the current value in the buffer plane.
        *   If the voxel's Z-value is **less** than the current buffer value (closer to the camera), update the buffer plane with the voxel's Z-value.
        *   Otherwise, leave the current buffer value unchanged [S1, unofficial].

#### 3. Selection (Optimization Step 3)
*   Select only those voxels whose projection onto the buffer plane exists at least partially (i.e., they were not fully occluded by closer voxels in Step 2) [S1, unofficial].
*   A voxel is discarded if its projected radius is completely covered by the radii of other points closer to the screen (smaller Z) [S1, unofficial].

#### 4. Rendering
*   Apply the standard splatting algorithm only to the selected visible voxels [S1, unofficial].
*   For each selected voxel, convolve it with the kernel to form a 3D image, project it onto the screen plane, and distribute energy to pixels within the footprint [S1, unofficial].
*   Form the final image by integrating energy over all footprints [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adapted_splatting_rendering(voxel_model, camera, screen_width, screen_height, kernel_radius):
    """
    Renders a volume using adapted splatting with occlusion culling.
    
    Args:
        voxel_model: Object containing the byte_sequence and offset_array.
        camera: Object with project_point() and get_z() methods.
        screen_width: int
        screen_height: int
        kernel_radius: float
        
    Returns:
        image: 2D array of pixel colors/intensities
    """
    
    # Step 1: Preprocessing (Lookup Table Generation)
    # In a real implementation, this is done once per kernel type.
    lookup_table = generate_gaussian_lookup_table(kernel_radius)
    
    # Step 2: Occlusion Culling Buffer Initialization
    # Buffer stores the minimum Z-depth seen so far for each screen pixel.
    # Initialize with infinity (or max float).
    depth_buffer = np.full((screen_height, screen_width), np.inf)
    
    # Helper to parse voxel data from the compressed format
    def get_occupied_voxels(model):
        voxels = []
        # Iterate through segments using offset_array
        for i in range(len(model.offset_array)):
            offset = model.offset_array[i]
            if offset == -1:
                continue # Segment has no occupied voxels
            
            # Parse segment at offset
            # Format: [f_i (int), n_i (int), n_i * voxel_data]
            # Note: Exact integer size depends on implementation, assumed 4-byte here
            f_i = read_int(model.byte_sequence, offset)
            n_i = read_int(model.byte_sequence, offset + 4)
            
            current_pos = offset + 8
            for _ in range(n_i):
                voxel = parse_voxel(model.byte_sequence, current_pos)
                voxels.append(voxel)
                current_pos += voxel_size # Includes core + additional attributes
        return voxels

    occupied_voxels = get_occupied_voxels(voxel_model)
    
    # Step 3: Project and Fill Depth Buffer
    visible_voxels = []
    
    for voxel in occupied_voxels:
        # Project voxel center to screen coordinates
        screen_x, screen_y = camera.project_point(voxel.center)
        voxel_z = camera.get_z(voxel.center)
        
        # Check if projection is within screen bounds
        if not (0 <= screen_x < screen_width and 0 <= screen_y < screen_height):
            continue
            
        # Determine footprint area on the buffer
        # Assuming circular footprint for simplicity, or square bounding box
        radius_pixels = kernel_radius_to_pixels(kernel_radius, camera)
        
        min_x = max(0, int(screen_x - radius_pixels))
        max_x = min(screen_width - 1, int(screen_x + radius_pixels))
        min_y = max(0, int(screen_y - radius_pixels))
        max_y = min(screen_height - 1, int(screen_y + radius_pixels))
        
        is_visible = False
        
        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                # Depth comparison: if voxel is closer, it updates the buffer
                if voxel_z < depth_buffer[y, x]:
                    depth_buffer[y, x] = voxel_z
                    is_visible = True
        
        # A voxel is selected if its projection exists at least partially
        # i.e., it updated at least one pixel in the depth buffer
        if is_visible:
            visible_voxels.append(voxel)
            
    # Step 4: Standard Splatting for Visible Voxels
    image = np.zeros((screen_height, screen_width))
    
    for voxel in visible_voxels:
        screen_x, screen_y = camera.project_point(voxel.center)
        
        # Distribute energy to pixels within footprint
        radius_pixels = kernel_radius_to_pixels(kernel_radius, camera)
        min_x = max(0, int(screen_x - radius_pixels))
        max_x = min(screen_width - 1, int(screen_x + radius_pixels))
        min_y = max(0, int(screen_y - radius_pixels))
        max_y = min(screen_height - 1, int(screen_y + radius_pixels))
        
        for y in range(min_y, max_y + 1):
            for x in range(min_x, max_x + 1):
                # Calculate distance from center
                dist = np.sqrt((x - screen_x)**2 + (y - screen_y)**2)
                
                # Lookup kernel value
                weight = lookup_table[dist]
                
                # Add energy to pixel
                # Assuming voxel has an intensity/value attribute
                image[y, x] += voxel.value * weight
                
    return image

def generate_gaussian_lookup_table(radius):
    # Placeholder for actual Gaussian calculation
    # Returns array of weights for distances 0 to radius
    pass

def parse_voxel(byte_seq, pos):
    # Placeholder for parsing specific voxel structure
    pass

def read_int(byte_seq, pos):
    # Placeholder for reading 4-byte integer
    pass

def kernel_radius_to_pixels(radius, camera):
    # Convert world-space radius to screen-space pixels
    pass
```

### Parameters
*   **Kernel**: A function used to convolve each voxel. The Gaussian function is the most common [S1, unofficial].
*   **Additional Attributes**: Each voxel can store extra data. The number of bytes for each additional attribute is defined during model description [S1, unofficial].

### Edge Cases and Pitfalls
*   **Occlusion Culling Logic**: A voxel is discarded if its projected radius is completely covered by the radii of other points closer to the screen (smaller Z) [S1, unofficial].
*   **Precision**: The depth comparison in the buffer plane must use sufficient precision to avoid z-fighting or incorrect culling of partially visible voxels.
*   **Boundary Conditions**: Voxels near the edge of the screen or volume must be handled correctly during projection and footprint calculation to avoid out-of-bounds array access.

## Operating characteristics

### Time and Memory Complexity
*   **Memory**: The adapted format significantly reduces memory usage for sparse data by grouping free voxels. The file size reduction is substantial: on a 'Tibia' model, reduced by 2x; on a 'Chest' model, from 8,411,000 to 38,231 bytes; on a large 'Aircraft' model, reduction reached 22x [S1, unofficial].
*   **Time**: Rendering time is reduced because the kernel and projection are applied only to visible voxels. On an Intel Core2 CPU E8400 3.0 GHz with GTX 470:
    *   Chest: Original 8.4ms vs Optimized 1.9ms.
    *   Tibia: Original 3.4ms vs Optimized 1.1ms.
    *   Skull: Original 4.1ms vs Optimized 1.3ms [S1, unofficial].

### Strengths and Limitations
*   **Strengths**: Significant reduction in file size and rendering time for sparse medical volumes. Allows for additional voxel attributes.
*   **Limitations**: Requires a preprocessing step for occlusion culling. The compression format is specific to this method and may not be compatible with standard voxel viewers.

### When to Use
*   Use for medical applications with sparse voxel data (e.g., bone structures, segmented organs) where memory bandwidth and rendering speed are critical.
*   Do not use for dense, fully occupied volumes where the overhead of culling may not justify the gains, or if standard compatibility is required.

## Background
The method was proposed by S. I. Vyatkin and A. N. Romanyuk. It addresses the high memory requirements and computational cost of standard volume graphics by optimizing splatting through voxel culling and a specialized data storage format [S1, unofficial].

## Sources
[S1] An adapted splatting method for medical applications, https://ir.lib.vntu.edu.ua/bitstream/123456789/25853/1/33-40.pdf, fetched 2026-10-26 (unofficial)
