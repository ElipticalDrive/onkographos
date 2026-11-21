# Image-to-Voxel Heightmap Generation

Also called: Excavate mode height clipping, Grayscale heightmap extrusion

## Overview
This method converts a 2D grayscale image into a 3D voxel structure where pixel intensity determines the height of voxel columns along the Y-axis. It differs from standard rasterization by treating the image as a heightfield rather than a surface mesh, supporting both simple vertical extrusion and an 'excavate' mode that clips columns to the minimum height of their 4-connected neighbors to prevent overhangs [S1, unofficial]. It builds on [Image-based Voxelization](12822-image-based-voxelization.md) by mapping image pixels to X and Z axes while using intensity for Y.

## How it works
**Inputs:**
- A 2D grayscale or RGB image.
- `heightFactor` (float): Multiplier for normalized intensity. Typical value: 1.
- `Excavate` (bool): Flag to enable neighbor-aware clipping.

**Outputs:**
- A list of voxel coordinates $(x, y, z)$ and their associated data.

**Data Structures:**
- The voxel grid is implicit or explicit. Each voxel is defined by integer coordinates $(x, y, z)$.
- The image is accessed via $(x, z)$ indices, where $x$ corresponds to the image width and $z$ to the image height (or vice versa, depending on coordinate system convention; here we assume $x$ is width, $z$ is height, and $y$ is up).

**Steps:**
1. **Initialize** an empty list of voxels.
2. **Iterate** over each pixel $(x, z)$ in the image.
3. **Compute Height:**
   - If the image is grayscale, let $I$ be the pixel intensity.
   - If the image is RGB, compute the average intensity:
     $$ I_{avg} = \frac{I_R + I_G + I_B}{3} $$
   - Normalize the intensity to $[0, 1]$:
     $$ h_{norm} = \frac{I_{avg}}{65535} $$
     (Note: The source formula uses $\min(N_c, 3)$, implying support for other channel counts, but standard RGB uses 3. The divisor 65535 suggests 16-bit integer input, though typical images are 8-bit. If 8-bit, the divisor would be 255. We follow the source formula exactly.)
   - Calculate the target height in voxels:
     $$ H_{target} = \left\lfloor h_{norm} \times H_{max} \right\rfloor $$
     where $H_{max} = \text{heightFactor}$.
4. **Determine Clipping Height (if `Excavate` is true):**
   - Identify the 4-connected neighbors: $(x-1, z), (x+1, z), (x, z-1), (x, z+1)$.
   - For each neighbor within image bounds, retrieve its computed height $H_{neighbor}$.
   - Compute $H_{min\_neighbor} = \min(\{H_{neighbor} \mid \text{neighbor is valid}\} \cup \{0\})$.
   - If no neighbors are valid (boundary case), $H_{min\_neighbor} = 0$.
   - The effective height for this column is $H_{eff} = \min(H_{target}, H_{min\_neighbor})$.
   - If `Excavate` is false, $H_{eff} = H_{target}$.
5. **Generate Voxels:**
   - If $H_{eff} > 0$:
     - Place voxels at $(x, y, z)$ for $y$ from $0$ to $H_{eff} - 1$.
   - If $H_{eff} \le 0$:
     - According to the source, if excavating and $H_{target} \le H_{min\_neighbor}$, place a single voxel at $y = \max(0, H_{target} - 1)$. This implies a minimum height of 1 voxel if the target is positive but clipped, or potentially 0 if target is 0. The source says "places a single voxel at `max(0, height - 1)`". If height is 0, it places at -1? No, `max(0, -1)` is 0. So it places a voxel at $y=0$. If height is 1, it places at $y=0$. This logic seems to ensure a minimum presence if the target was non-zero. Let's refine: The source states: "if the computed height is less than or equal to the neighbor minimum, the algorithm places a single voxel at `max(0, height - 1)`". This is a specific edge case handling for excavation.
     - General case (non-excavate or height > neighbor min): Place voxels from $y=0$ to $y=H_{eff}-1$.
6. **Handle Boundaries:**
   - Pixels on the edge of the image lack neighbors on one or more sides. The implementation checks bounds explicitly: `if (param.X - 1 >= 0 && param.X + 1 < param.GrayscaleImage.Width && param.Y - 1 >= 0 && param.Y + 1 < param.GrayscaleImage.Height)`. If out of bounds, it defaults to adding voxels from 0 to the computed height (treating missing neighbors as having height 0 or ignoring them for the min calculation? The source says "defaults to adding voxels from 0 to the computed height", implying no clipping from missing neighbors, i.e., they are not considered in the min, or considered as infinity? The code snippet implies checking bounds. If a neighbor is out of bounds, it is likely not included in the min set. If the set is empty, min is undefined, so it defaults to no clipping).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxels(image, height_factor, excavate):
    voxels = []
    width = image.width
    height = image.height
    
    # Precompute heights for all pixels
    heights = [[0 for _ in range(height)] for _ in range(width)]
    
    for x in range(width):
        for z in range(height):
            pixel = image.get_pixel(x, z)
            # Assuming 16-bit RGB for formula fidelity, or scale appropriately
            # Source formula: floor( (sum(Ic) / (min(Nc, 3) * 65535)) * Hmax )
            # For standard 8-bit RGB, divisor is 3 * 255 = 765. 
            # We stick to the source's 65535, assuming 16-bit input or normalized float input scaled.
            # If input is 0-255, 65535 is wrong. We assume input is 0-65535 or normalized.
            # Let's assume normalized float [0,1] for simplicity in implementation, 
            # but the formula uses 65535. We will implement the formula as written.
            
            r, g, b = pixel.r, pixel.g, pixel.b
            n_channels = 3
            intensity_sum = r + g + b
            # Normalize
            h_norm = intensity_sum / (n_channels * 65535.0)
            h_target = int(h_norm * height_factor)
            heights[x][z] = h_target

    for x in range(width):
        for z in range(height):
            h_target = heights[x][z]
            
            if not excavate:
                h_eff = h_target
            else:
                # Find min neighbor height
                neighbors = []
                if x > 0: neighbors.append(heights[x-1][z])
                if x < width - 1: neighbors.append(heights[x+1][z])
                if z > 0: neighbors.append(heights[x][z-1])
                if z < height - 1: neighbors.append(heights[x][z+1])
                
                if neighbors:
                    h_min_neighbor = min(neighbors)
                else:
                    # Boundary pixel, no neighbors to clip against
                    h_min_neighbor = float('inf') 
                
                h_eff = min(h_target, h_min_neighbor)
            
            # Generate voxels
            if h_eff > 0:
                for y in range(h_eff):
                    voxels.append((x, y, z))
            elif h_eff <= 0 and excavate and h_target <= h_min_neighbor:
                # Specific edge case from source
                y_pos = max(0, h_target - 1)
                if y_pos >= 0:
                    voxels.append((x, y_pos, z))
    
    return voxels
```

## Subtopics

### [Height Curve Activation & Voxel Column Generation](17940-height-curve-activation-voxel-column-generation.md)
This method transforms a normalized 2D noise heightmap into a 3D voxel column structure by applying a hyperbolic tangent activation function to smooth terrain features, clamping values to a water line, and stacking voxels from a defined ground offset to the calculated height. It differs from simple linear extrusion by using a non-linear activation curve to compress extreme noise values, providing more natural-looking terrain profiles, and explicitly handles color mapping based on discrete terrain bands.

## Operating characteristics
- **Time Complexity:** $O(W \times H)$, where $W$ and $H$ are the image dimensions. Each pixel is processed once, and neighbor checks are constant time (4 neighbors).
- **Memory Complexity:** $O(W \times H)$ to store the intermediate height map if precomputed, or $O(V)$ for the output voxel list, where $V$ is the number of voxels.
- **Strengths:** Simple to implement; efficient for generating terrain-like structures from heightmaps; excavate mode prevents floating islands.
- **Limitations:** Resolution is limited by image resolution; no sub-voxel precision; excavate mode can flatten steep features.
- **When to use:** When converting 2D heightmaps (e.g., from noise functions or images) into 3D voxel data for games or simulations.
- **When not to use:** When high-resolution smooth surfaces are required (use mesh-based approaches) or when complex 3D geometry is needed beyond heightfields.

## Background
This method is part of the voxelization pipeline in tools like FileToVox, which converts various file formats into voxel data. It is commonly used in Minecraft modding, voxel-based games, and procedural terrain generation. The 'excavate' mode is a specific heuristic to maintain structural integrity in voxel art.

## Sources
[S1] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
