# Image-based Voxelization

Also called: Multiple Image to Schematic Conversion

## Overview
Image-based Voxelization converts a sequence of 2D images into a 3D volumetric schematic where the image index maps to the depth axis (Y), and pixel coordinates map to the spatial axes (X and Z) [S1, unofficial]. It is distinguished from standard volume extrusion by its specific handling of surface boundaries via an "excavate" mode, which clips voxel columns based on neighbor transparency to create terraced structures rather than solid vertical walls [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A list of 2D images (grayscale or RGB). An optional 256x1 pixel color gradient file (`mInputColorFile`).
*   **Output**: A `Schematic` object containing a list of voxels with coordinates $(x, y, z)$ and associated color data.

**Data Structures**
*   **Voxel**: Defined by integer coordinates $(x, y, z)$ and a `Color` object.
    *   $x$: Image pixel column index.
    *   $y$: Image index in the input list (depth).
    *   $z$: Image pixel row index.
*   **Schematic**: Contains a list of voxels. Maximum world width and length are constrained by `Schematic.MAX_WORLD_WIDTH` and `Schematic.MAX_WORLD_LENGTH` [S1, unofficial].

**Parameters**
*   `mExcavate` (boolean): If `true`, applies neighbor checking to identify surface voxels; if `false`, fills all non-transparent pixels [S1, unofficial].
*   `mInputColorFile` (string path): Optional path to a 256x1 pixel color gradient. If present, overrides pixel colors based on distance from the image center [S1, unofficial].
*   `mColorLimit` (integer): Limit for color quantization applied to the final voxel list [S1, unofficial].
*   `loadImageParam.Height`: Determines the vertical extent of voxels when not in single-voxel mode [S1, unofficial].

**Steps**
1.  **Validation**: Verify input image width $\le 256$ and height $= 1$ if `mInputColorFile` is provided. If `mInputColorFile` is present, verify its dimensions are exactly 256x1; otherwise, throw an `ArgumentException` [S1, unofficial].
2.  **Color Quantization**: If `mInputColorFile` is present and the image has more than 256 unique colors, or if `loadImageParam.EnableColor` is true, quantize the image colors to `mColorLimit` [S1, unofficial].
3.  **Iteration**: For each image index $i$ (representing the Y-axis):
    a. Load the image.
    b. Iterate through each pixel $(x, y)$ where $x \in [0, \text{width}-1]$ and $y \in [0, \text{height}-1]$.
    c. Retrieve pixel color channels (RGB and Alpha).
    d. **Skip Condition**: If the pixel is Empty, Transparent, Black, or has all RGB channels equal to zero, skip this pixel [S1, unofficial].
    e. **Color Override**: If `mInputColorFile` is present:
        i. Calculate distance $d$ from the image center $(\text{height}/2, \text{height}/2)$ to pixel $(x, y)$.
        ii. Normalize $d$ to range $[0, 1]$.
        iii. Sample color from `mInputColorFile` at index $\lfloor \text{normalized\_d} \times (\text{colorWidth}-1) \rfloor$.
    f. **Voxel Placement**:
        i. If `mExcavate` is `true`: Call `CheckNeighbor(x, y)`. If any of the four 4-connected neighbors (left, top, right, bottom) is empty/transparent/black or has zero RGB, add a voxel at $(x, i, y)$ with the determined color.
        ii. If `mExcavate` is `false`: Add a voxel at $(x, i, y)$ with the determined color [S1, unofficial].
4.  **Quantization**: Apply quantization to the final list of voxels using `mColorLimit` [S1, unofficial].
5.  **Finalization**: Create and return the `Schematic` from the quantized voxel list [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_based_voxelization(images, input_color_file=None, excavate=False, color_limit=256):
    """
    Convert a list of 2D images into a 3D schematic.
    
    Args:
        images: List of 2D image arrays (height, width, channels).
        input_color_file: Optional 256x1 color gradient array.
        excavate: Boolean flag for surface-only voxelization.
        color_limit: Integer for color quantization.
        
    Returns:
        List of Voxel objects: (x, y, z, color)
    """
    voxels = []
    
    # Validate input color file dimensions if provided
    if input_color_file is not None:
        if input_color_file.shape != (1, 256, 3): # Assuming 256x1x3 RGB
            raise ValueError("Input color file must be exactly 256x1 pixels")
            
    # Process each image (depth axis Y)
    for y_idx, image in enumerate(images):
        height, width = image.shape[:2]
        
        # Validate image size constraints
        if width > 256 or height > 256: # Assuming MAX_WORLD constraints
            raise ValueError("Image dimensions exceed maximum limits")
            
        for x in range(width):
            for z in range(height):
                pixel = image[z, x]
                r, g, b, a = pixel[:4]
                
                # Skip empty/transparent/black pixels
                if is_empty_or_black(r, g, b, a):
                    continue
                
                # Determine color
                final_color = pixel[:3]
                
                # Apply color override if gradient file is present
                if input_color_file is not None:
                    center_x = height / 2
                    center_z = height / 2 # Assuming square center logic or width/2
                    # Note: Source uses height/2 for both, assuming square or specific center logic
                    dist = math.sqrt((x - center_x)**2 + (z - center_z)**2)
                    max_dist = math.sqrt((center_x)**2 + (center_z)**2)
                    normalized_dist = dist / max_dist if max_dist > 0 else 0
                    normalized_dist = max(0.0, min(1.0, normalized_dist))
                    
                    color_idx = int(normalized_dist * 255)
                    final_color = input_color_file[0, color_idx]
                
                # Place voxel based on excavate mode
                if excavate:
                    if check_neighbor_empty(images, y_idx, x, z, width, height):
                        voxels.append(Voxel(x, y_idx, z, final_color))
                else:
                    voxels.append(Voxel(x, y_idx, z, final_color))
                    
    # Quantize final colors
    voxels = quantize_colors(voxels, color_limit)
    
    return voxels

def is_empty_or_black(r, g, b, a):
    """Check if pixel is transparent, empty, or black."""
    if a == 0:
        return True
    if r == 0 and g == 0 and b == 0:
        return True
    return False

def check_neighbor_empty(images, y_idx, x, z, width, height):
    """Check if any 4-connected neighbor is empty/transparent/black."""
    neighbors = [
        (x - 1, z), (x + 1, z),
        (x, z - 1), (x, z + 1)
    ]
    
    for nx, nz in neighbors:
        # Check bounds
        if nx < 0 or nx >= width or nz < 0 or nz >= height:
            # Out of bounds is considered empty for surface detection
            return True
            
        # Check if neighbor image exists (for depth)
        if y_idx < 0 or y_idx >= len(images):
            return True
            
        neighbor_pixel = images[y_idx][nz, nx]
        r, g, b, a = neighbor_pixel[:4]
        
        if is_empty_or_black(r, g, b, a):
            return True
            
    return False

def quantize_colors(voxels, color_limit):
    """Simplified quantization placeholder."""
    # In practice, this would use a k-means or median-cut algorithm
    # to reduce the number of unique colors in the voxel list.
    return voxels
```

## Subtopics

### [Image-to-Voxel Heightmap Generation](12828-image-to-voxel-heightmap-generation.md)
Converts a single 2D image into a 3D voxel structure by extruding columns based on pixel intensity. It supports an "excavate" mode to create terraced structures by clipping columns to the minimum height of 4-connected neighbors, distinguishing it from standard vertical extrusion.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot H \cdot W)$, where $N$ is the number of images, and $H, W$ are image height and width. Neighbor checking in excavate mode adds a constant factor of 4 checks per voxel.
*   **Memory Complexity**: $O(N \cdot H \cdot W)$ for storing the voxel list.
*   **Strengths**: Simple implementation; supports color mapping via gradients; excavate mode allows for stylized terrain-like outputs.
*   **Limitations**: Input images are limited to 256x256 pixels; requires exact 256x1 color gradient if used; no anti-aliasing or sub-voxel precision mentioned.
*   **When to use**: When converting 2D sprite sheets or sequential frames into a 3D schematic for voxel-based rendering or editing.
*   **When not to use**: For high-resolution textures or when precise geometric accuracy is required over stylistic voxel representation.

## Background
Originating from the FileToVox tool, this method is designed to convert 2D images into 3D Minecraft-style schematics [S1, unofficial]. It is part of a pipeline for generating voxel assets from image data, specifically targeting users who want to create 3D structures from 2D art or heightmaps.

## Sources
[S1] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
