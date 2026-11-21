# Energy Region Compositing

Also called: Energy Compositing

## Overview
Energy Region Compositing is a volume rendering technique that composites multiple energy volumes within a specific region of interest defined by a magnitude volume [S1, unofficial]. It renders a volume using multiple energy volumes, blending colors based on normalized intensities above a threshold [S1, unofficial]. It differs from standard composite rendering by using a separate magnitude volume to define visibility and applying a threshold to energy intensity before compositing [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `view_matrix`: A 3x3 or 4x4 matrix containing view vectors. Indices 0:3 are the U vector, 4:7 are the V vector, and 8:11 are the View vector [S1, unofficial].
- `volume`: The main magnitude volume object with attributes `dim_x`, `dim_y`, `dim_z` and a method `get_maximum()` [S1, unofficial].
- `image_size`: Integer width and height of the output image (assumed square) [S1, unofficial].
- `image`: Output array of size `image_size * image_size * 4` (RGBA) [S1, unofficial].
- `energy_volumes`: Dictionary mapping integer energy IDs to Volume objects [S1, unofficial].
- `magnitudeVolume`: A separate volume object used to define the region of interest [S1, unofficial].
- `interactive_mode`: Boolean flag determining step size [S1, unofficial].

**Data Structures:**
- `colorDictionary`: Maps energy IDs to RGB colors. Colors are generated from permutations of `[1, 0, 0.5, 1]` taken 3 at a time [S1, unofficial].
- `voxel_cords`: 3D coordinates calculated per ray step [S1, unofficial].

**Parameters:**
- `step`: Loop step size. Set to 20 if `interactive_mode` is true, otherwise 1 [S1, unofficial]. Note: One source snippet uses 2, another 10, but the primary definition states 20 [S1, unofficial].
- `threshold`: Energy intensity threshold. Set to 0.3 (30%) [S1, unofficial].

**Formulas:**
- Ray extent diagonal:
  $$ \text{diagonal} = \lfloor \sqrt{3} \times \max(\text{dim}_x, \text{dim}_y, \text{dim}_z) / 2 \rfloor + 1 $$
- Voxel coordinates:
  $$ \text{voxel\_cords} = \mathbf{u} \times (i - \text{image\_center}) + \mathbf{v} \times (j - \text{image\_center}) + \mathbf{view} \times k + \text{volume\_center} $$
- Composite color update:
  $$ \text{current\_r} = \text{color\_a} \times \text{color\_voxel}[0] + (1 - \text{color\_a}) \times \text{initial\_r} $$
  (Same for g and b)

**Steps:**
1. Initialize `image` to zeros.
2. Generate `colorDictionary`:
   - Generate permutations of `[1, 0, 0.5, 1]` taken 3 at a time.
   - Map each energy ID in `energy_volumes` to a unique permutation.
   - Set ID 0 to `[0, 0, 0]` [S1, unofficial].
3. Extract vectors from `view_matrix`:
   - $\mathbf{u} = \text{view\_matrix}[0:3]$
   - $\mathbf{v} = \text{view\_matrix}[4:7]$
   - $\mathbf{view} = \text{view\_matrix}[8:11]$
4. Calculate centers:
   - $\text{image\_center} = \text{image\_size} / 2$
   - $\text{volume\_center} = [\text{volume.dim\_x}/2, \text{volume.dim\_y}/2, \text{volume.dim\_z}/2]$
5. Calculate `diagonal` using the formula above.
6. Set `step`: 20 if `interactive_mode` else 1.
7. Loop over pixel coordinates $(i, j)$ with step size.
8. Initialize `initial_r`, `initial_g`, `initial_b` to 0.
9. Loop along ray depth $k$ from `diagonal` down to `-diagonal`.
10. Compute `voxel_cords` using the formula above.
11. Sample `magnitudeVolume` at `voxel_cords` to get $g$.
12. If $g \neq 0$:
    - Set `color_voxel` to `[1, 1, 1]` (white) [S1, unofficial].
    - Else, set `color_voxel` to `[0, 0, 0]` [S1, unofficial].
13. Sample the main `volume` at `voxel_cords`. If value $> 0$:
    - Iterate through all `energy_volumes`.
    - For each energy, get `intensity_max` (from volume) and `energy_intensity` (interpolated at `voxel_cords`).
    - If `energy_intensity` / `intensity_max` > 0.3:
      - Update `color_voxel`:
        $$ \text{color\_voxel} = \text{color\_voxel} \times (1 - \text{ratio}) + \text{colorDictionary}[key] \times \text{ratio} $$
        where $\text{ratio} = \text{energy\_intensity} / \text{intensity\_max}$ [S1, unofficial].
14. Calculate `color_a` as the maximum component of `color_voxel` [S1, unofficial].
15. Update accumulated color:
    $$ \text{current\_r} = \text{color\_a} \times \text{color\_voxel}[0] + (1 - \text{color\_a}) \times \text{initial\_r} $$
    (Same for g and b)
    $$ \text{initial\_r}, \text{initial\_g}, \text{initial\_b} = \text{current\_r}, \text{current\_g}, \text{current\_b} $$
16. After ray loop, clamp and convert final colors to 8-bit integers:
    $$ \text{red} = \lfloor \text{current\_r} \times 255 \rfloor \text{ if } < 255 \text{ else } 255 $$
    (Same for g, b)
17. Assign RGBA values to `image` at index $(j \times \text{image\_size} + i) \times 4$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np
from itertools import permutations

def get_voxelInterpolated(volume, x, y, z):
    # Placeholder for interpolation logic not detailed in source
    # [G] Assumes trilinear interpolation or nearest neighbor
    return 0.0

def render_energy_region_compositing(view_matrix, volume, image_size, image, energy_volumes, magnitudeVolume, interactive_mode):
    # Clear image
    image.fill(0)
    
    # Generate color dictionary
    perm = np.asarray(list(permutations([1, 0, 0.5, 1], 3)))
    ids = list(energy_volumes.keys())
    colorDictionary = {0: [0, 0, 0]}
    for i in range(len(ids)):
        colorDictionary[ids[i]] = perm[i]
        
    # Extract vectors
    u_vector = view_matrix[0:3].reshape(-1, 1)
    v_vector = view_matrix[4:7].reshape(-1, 1)
    view_vector = view_matrix[8:11].reshape(-1, 1)
    
    # Centers
    image_center = image_size / 2
    volume_center = np.asarray([volume.dim_x / 2, volume.dim_y / 2, volume.dim_z / 2]).reshape(-1, 1)
    
    # Diagonal
    diagonal = np.sqrt(3) * np.max([volume.dim_x, volume.dim_y, volume.dim_z]) / 2
    diagonal = int(math.floor(diagonal)) + 1
    
    # Step size
    step = 20 if interactive_mode else 1
    
    for i in range(0, image_size, step):
        for j in range(0, image_size, step):
            initial_r, initial_g, initial_b = 0, 0, 0
            
            for k in range(diagonal, -diagonal, -1):
                # Compute voxel coordinates
                voxel_cords = (np.dot(u_vector, i - image_center) + 
                               np.dot(v_vector, j - image_center) + 
                               np.dot(view_vector, k) + volume_center)
                
                # Sample magnitude volume
                g = get_voxelInterpolated(magnitudeVolume, voxel_cords[0], voxel_cords[1], voxel_cords[2])
                
                color_voxel = np.array([0.0, 0.0, 0.0])
                if g != 0:
                    color_voxel = np.array([1.0, 1.0, 1.0])
                
                # Sample main volume
                value = get_voxelInterpolated(volume, voxel_cords[0], voxel_cords[1], voxel_cords[2])
                
                if value > 0:
                    for key, energy_vol in energy_volumes.items():
                        intensity_max = energy_vol.get_maximum()
                        energy_intensity = get_voxelInterpolated(energy_vol, voxel_cords[0], voxel_cords[1], voxel_cords[2])
                        
                        if intensity_max > 0 and (energy_intensity / intensity_max) > 0.3:
                            ratio = energy_intensity / intensity_max
                            color_voxel = np.add(np.multiply(color_voxel, 1 - ratio), 
                                                 np.multiply(colorDictionary[key], ratio))
                
                color_a = np.max(color_voxel)
                
                current_r = color_a * color_voxel[0] + (1 - color_a) * initial_r
                current_g = color_a * color_voxel[1] + (1 - color_a) * initial_g
                current_b = color_a * color_voxel[2] + (1 - color_a) * initial_b
                
                initial_r, initial_g, initial_b = current_r, current_g, current_b
            
            # Clamp and convert
            red = math.floor(current_r * 255) if current_r < 255 else 255
            green = math.floor(current_g * 255) if current_g < 255 else 255
            blue = math.floor(current_b * 255) if current_b < 255 else 255
            alpha = 255
            
            # Assign to image
            idx = (j * image_size + i) * 4
            image[idx] = red
            image[idx + 1] = green
            image[idx + 2] = blue
            image[idx + 3] = alpha
```

## Operating characteristics
- **Time Complexity:** $O(\text{image\_size}^2 \times \text{diagonal} \times N)$, where $N$ is the number of energy volumes.
- **Memory Complexity:** $O(\text{image\_size}^2)$ for the output image, plus storage for all volume data.
- **Performance:** Uses a large step size (20) in interactive mode to speed up rendering [S1, unofficial].
- **Strengths:** Allows visualization of specific energy types within a defined region of interest [S1, unofficial].
- **Limitations:** Relies on a separate magnitude volume for region definition; thresholding is fixed at 30% [S1, unofficial].

## Background
Originated in the `faviasono/raycastingrendering` project [S1, unofficial]. It is an implementation of energy-based volume compositing within a direct volume ray casting framework [S1, unofficial].

## Sources
[S1] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
