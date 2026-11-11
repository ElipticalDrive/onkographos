# Image-to-Voxel Heightmap Generation

Also called: Excavate mode height clipping, Grayscale heightmap extrusion

## Overview
This method converts a 2D grayscale or RGB image into a 3D voxel structure by extruding voxel columns along the Y-axis, where pixel intensity determines the height. It is distinguished from standard volume extrusion by supporting an "excavate" mode that clips voxel columns to the minimum height of their 4-connected neighbors, creating terraced or smoothed terrain-like structures rather than vertical walls. It serves as a specific implementation within [Image-based Voxelization](12822-image-based-voxelization.md) for generating heightmaps from single images.

## How it works
**Inputs:**
- A 2D image (grayscale or RGB).
- `heightFactor` (float): Multiplier for normalized intensity. Typical value: 1.0 [S1, unofficial].
- `Excavate` (bool): Flag to enable neighbor-aware clipping [S1, unofficial].

**Outputs:**
- A list of 3D voxel coordinates $(x, y, z)$ representing the generated structure.

**Data Structures:**
- The image is treated as a 2D grid where the X-axis corresponds to image width and the Z-axis corresponds to image height. The Y-axis represents the vertical height in the voxel space.
- Pixel values are accessed by channel. For RGB images, the intensity is derived from the sum of Red, Green, and Blue channels.

**Steps:**

1. **Initialize Output:** Create an empty list for voxels.
2. **Iterate Pixels:** Loop through each pixel at image coordinates $(u, v)$, where $0 \le u < \text{Width}$ and $0 \le v < \text{Height}$.
3. **Calculate Base Height:**
   Compute the normalized intensity $I_{norm}$ and the base height $h$.
   
   $$ h = \left\lfloor \frac{\sum_{c=0}^{2} I_c}{\min(N_c, 3) \times 65535} \times H_{max} \right\rfloor $$
   
   Where:
   - $I_c$ is the value of channel $c$ (Red, Green, Blue).
   - $N_c$ is the number of channels (typically 3 for RGB, 1 for grayscale).
   - $H_{max}$ is the `heightFactor` parameter.
   - The denominator normalizes the sum of channel values against the maximum possible sum (assuming 16-bit or scaled 8-bit values where 65535 is the max, though standard 8-bit images often use 255; the source explicitly uses 65535) [S1, unofficial].
   
   *Note: If the image is 8-bit, $I_c \in [0, 255]$. The formula implies a scaling factor. If the source code uses 65535, it assumes 16-bit precision or a specific normalization scheme. Implementers should verify the bit-depth of the input image against the constant 65535.*

4. **Determine Final Height ($h_{final}$):**
   - **If `Excavate` is false:**
     $$ h_{final} = h $$
     The column extends from $y=0$ to $y=h_{final}-1$.
   
   - **If `Excavate` is true:**
     Find the minimum height $h_{min\_neighbor}$ among the 4-connected neighbors (up, down, left, right) that exist within image bounds.
     - If a neighbor is out of bounds, it is ignored for the minimum calculation.
     - If no neighbors exist (single pixel image), $h_{min\_neighbor} = 0$.
     
     $$ h_{final} = \min(h, h_{min\_neighbor}) $$
     
     *Edge Case:* If $h_{final} \le 0$, place a single voxel at $y = \max(0, h - 1)$ if $h > 0$, otherwise no voxel. Specifically, if the computed height is less than or equal to the neighbor minimum, the algorithm places a single voxel at `max(0, height - 1)` instead of an empty column [S1, unofficial].

5. **Generate Voxels:**
   For each $y$ from $0$ to $h_{final} - 1$:
   - Add voxel at $(x=u, y=y, z=v)$ to the output list.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_heightmap_voxels(image, height_factor=1.0, excavate=False):
    """
    image: 2D array of pixels, where each pixel is a tuple/list of channel values.
           Channels are assumed to be 16-bit or scaled such that max value is 65535.
           If 8-bit, scale up by 256 or adjust denominator.
    height_factor: float, H_max
    excavate: bool
    """
    width = image.width
    height = image.height
    voxels = []
    
    # Precompute neighbor heights if excavating
    neighbor_heights = None
    if excavate:
        neighbor_heights = [[0] * width for _ in range(height)]
        for y in range(height):
            for x in range(width):
                # Calculate raw height for this pixel first to determine neighbor min
                # Note: This requires a two-pass or careful single-pass logic.
                # The source implies checking neighbors' *existing* heights.
                # In a single pass, we might not have processed neighbors yet.
                # However, 'excavate' usually implies a global constraint or 
                # a specific local smoothing. The source code snippet suggests
                # checking bounds explicitly during the voxel placement loop.
                pass

    for y in range(height):
        for x in range(width):
            pixel = image[y][x]
            
            # Calculate intensity
            # Assuming RGB or Grayscale. N_c is number of channels.
            channels = pixel
            n_c = len(channels)
            sum_channels = sum(channels)
            
            # Normalization factor from source
            max_val = 65535
            denominator = min(n_c, 3) * max_val
            
            if denominator == 0:
                h = 0
            else:
                h = int((sum_channels / denominator) * height_factor)
            
            if not excavate:
                h_final = h
            else:
                # Find min height of 4-connected neighbors
                # Note: This logic depends on whether 'neighbor height' refers to
                # the *computed* height of the neighbor or the *final* height.
                # The source says "clips voxel columns to the minimum height of their 4-connected neighbors".
                # This implies we need the final heights of neighbors.
                # A simple implementation might use the raw calculated height of neighbors.
                
                min_neighbor_h = float('inf')
                has_neighbor = False
                
                # Check 4 neighbors
                neighbors = [
                    (x - 1, y), (x + 1, y),
                    (x, y - 1), (x, y + 1)
                ]
                
                for nx, ny in neighbors:
                    if 0 <= nx < width and 0 <= ny < height:
                        has_neighbor = True
                        # Get neighbor's calculated height
                        n_pixel = image[ny][nx]
                        n_channels = n_pixel
                        n_n_c = len(n_channels)
                        n_sum = sum(n_channels)
                        n_denom = min(n_n_c, 3) * max_val
                        if n_denom == 0:
                            n_h = 0
                        else:
                            n_h = int((n_sum / n_denom) * height_factor)
                        
                        if n_h < min_neighbor_h:
                            min_neighbor_h = n_h
                
                if not has_neighbor:
                    min_neighbor_h = 0
                
                h_final = min(h, min_neighbor_h)
            
            # Edge case handling from source
            if h_final <= 0:
                if h > 0:
                    # Place single voxel at max(0, h - 1)
                    vy = max(0, h - 1)
                    voxels.append((x, vy, y)) # Note: Source uses param.X, param.Y for image coords
                    # Mapping: Image X -> Voxel X, Image Y -> Voxel Z, Height -> Voxel Y
                continue
            
            # Extrude voxels from 0 to h_final - 1
            for vy in range(h_final):
                voxels.append((x, vy, y))
                
    return voxels
```

**Parameters:**
- `heightFactor`: Controls the vertical scale. Default 1.0 [S1, unofficial].
- `Excavate`: Boolean. Enables clipping [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Pixels:** Pixels on the edge of the image lack neighbors on one or more sides. The implementation must check bounds explicitly (`if (param.X - 1 >= 0 && param.X + 1 < param.GrayscaleImage.Width && param.Y - 1 >= 0 && param.Y + 1 < param.GrayscaleImage.Height)`). If out of bounds, the neighbor is ignored for the minimum calculation [S1, unofficial].
- **Negative/Zero Height:** If the computed height is less than or equal to the neighbor minimum, the algorithm places a single voxel at `max(0, height - 1)` instead of an empty column or negative height [S1, unofficial].
- **Precision:** The formula uses integer division/flooring. Ensure channel values are scaled correctly relative to the 65535 constant. If the input image is 8-bit (0-255), the intensity will be very low unless `heightFactor` is adjusted or the denominator is changed to 255.

## Operating characteristics
- **Time Complexity:** $O(W \times H)$, where $W$ and $H$ are the image dimensions. If `Excavate` is true, the neighbor lookup is constant time (4 neighbors), so it remains linear with respect to the number of pixels.
- **Memory Complexity:** $O(W \times H)$ to store the output voxel list. The maximum number of voxels is $W \times H \times H_{max}$.
- **Strengths:** Simple to implement; generates terrain-like structures efficiently; "Excavate" mode reduces vertical artifacts and creates smoother transitions.
- **Limitations:** Resolution is limited by the input image resolution; height resolution is limited by `heightFactor` and pixel precision; boundary effects may cause uneven edges if not handled carefully.
- **When to use:** When converting 2D heightmaps or grayscale images into 3D voxel data for games or simulations where performance is critical and high-fidelity surface detail is not required.
- **When not to use:** When high-resolution surface geometry is needed (use mesh extraction instead); when the input image is not a heightmap (e.g., color textures should be mapped differently).

## Background
This method is implemented in the **FileToVox** tool, an unofficial utility for converting various file formats (including QB files and images) into Minecraft-style `.vox` files. The specific "Excavate" mode and heightmap generation logic are part of its image processing pipeline. The tool is hosted on GitHub and is not peer-reviewed [S1, unofficial].

## Sources
[S1] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
