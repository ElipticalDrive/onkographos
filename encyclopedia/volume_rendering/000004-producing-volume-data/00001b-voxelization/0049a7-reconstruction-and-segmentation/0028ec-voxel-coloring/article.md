# Voxel Coloring

Also called: Photorealistic Scene Reconstruction by Voxel Coloring, Voxel Colouring Reconstruction, Voxel-based 3D reconstruction

## Overview
Voxel Coloring is a volumetric reconstruction technique that computes photo-realistic 3D models from multiple color images by assigning colors (radiances) to voxels in a 3D volume, guaranteeing consistency with the input images [S1]. It distinguishes itself from other reconstruction methods by using a "carving" approach: it starts with an opaque volume and iteratively removes (carves) voxels that are inconsistent with the observed image colors, rather than building geometry from surface points [S2, unofficial]. Unlike multi-scan methods such as Space Carving, Voxel Coloring relies on an ordinal visibility constraint to allow reconstruction in a single scan [S2, unofficial].

## How it works
**Inputs:**
- A set of $N$ color images $V_0, \dots, V_{N-1}$ [S1].
- Camera parameters for each image: position, focal angles (vertical and horizontal), and depth information (or synthesized depth maps) [S1].
- A 3D grid defining the spatial domain with resolution $(N_x, N_y, N_z)$ and a bounding box containing the scene [S1].

**Outputs:**
- A 3D volume where each voxel stores a color value (RGB) and an alpha value (transparency) [S1].

**Data Structures:**
- **Voxel Grid:** A 3D array of voxels. Each voxel contains:
  - `color`: A 3-component vector (RGB) representing the assigned radiance [S1].
  - `alpha`: A scalar value (typically 0–255) representing transparency. 0 indicates transparent (carved), 255 indicates opaque [S1].
- **Pixel Map ($\pi$):** A set or mask tracking which image pixels have been successfully mapped to voxels, used to ensure coverage [S1].

**Parameters:**
- `thresh`: The maximum allowable correlation error threshold for color consistency. A low value yields precise but potentially incomplete reconstruction; a larger value yields a more complete reconstruction but may include false voxels [S1].

**Algorithm Steps:**

1. **Initialization:**
   - Initialize a reconstruction volume of voxels as opaque (alpha = 255) to represent the initial volume containing the scene [S2, unofficial].

2. **Camera Space Transformation:**
   - For each camera, compute the change of scale matrix $M$ between object space and camera space [S1].
   - The matrix is defined as $M = R_X \times R_Y \times T$, where:
     - $R_X$ and $R_Y$ are rotation matrices that bring the vision vector to the OZ axis [S1].
     - $T$ is the translation matrix that brings the camera to the center of the scene [S1].

3. **Voxel Traversal:**
   - Traverse the voxels in a monotonic order (near-to-far relative to the cameras) to ensure correct visibility handling [S2, unofficial].
   - *Constraint:* The camera locations must satisfy an ordinal visibility constraint, meaning cameras cannot surround the scene, ensuring voxels can be visited in a single near-to-far scan relative to every camera [S2, unofficial].

4. **Projection and Consistency Test:**
   - For each voxel $v$ in the traversal order:
     - Compute the position of the corresponding voxel in the grid and project it to the image plane using the transformation matrix $M$ [S1].
     - Retrieve the color $C_{img}$ from the corresponding pixel in each input image $V_i$ [S1].
     - Test for color consistency: Check if the correlation error between the voxel's assigned color and the projected image colors exceeds `thresh` [S1].

5. **Carving:**
   - If a voxel is found to be color inconsistent across the input images, carve it by setting its alpha value to 0 (transparent) [S2, unofficial].
   - If consistent, retain its opacity (alpha = 255) and assign the pixel's color to the found voxel [S1].

6. **Termination:**
   - Stop when all remaining opaque voxels are color consistent [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_coloring(images, cameras, grid_resolution, bounding_box, thresh):
    """
    Reconstructs a 3D volume using Voxel Coloring.
    
    Args:
        images: List of N color images.
        cameras: List of N camera objects with position, rotation, and depth info.
        grid_resolution: Tuple (Nx, Ny, Nz) defining the voxel grid size.
        bounding_box: Tuple defining the 3D space bounds.
        thresh: Maximum allowable correlation error threshold.
        
    Returns:
        volume: 3D array of voxels with 'color' and 'alpha' fields.
    """
    # Initialize volume with opaque voxels (alpha = 255) [S2, unofficial]
    volume = [[[{'color': [0.0, 0.0, 0.0], 'alpha': 255} for _ in range(grid_resolution[2])]
                for _ in range(grid_resolution[1])] for _ in range(grid_resolution[0])]

    for x in range(grid_resolution[0]):
        for y in range(grid_resolution[1]):
            for z in range(grid_resolution[2]):
                voxel = volume[x][y][z]
                if voxel['alpha'] == 0:
                    continue

                consistent_colors = []
                for i, cam in enumerate(cameras):
                    # Compute transformation matrix M = RX * RY * T [S1]
                    # RX, RY rotate vision vector to OZ axis; T translates camera to scene center [S1]
                    M = cam.RX @ cam.RY @ cam.T

                    # Project voxel center to image plane [S1]
                    # Homogeneous coordinate: p_h = M @ [x, y, z, 1]^T
                    p_h = M @ [x, y, z, 1.0]
                    # Perspective division to get normalized device coordinates
                    if p_h[3] == 0:
                        continue
                    p_ndc = [p_h[0]/p_h[3], p_h[1]/p_h[3], p_h[2]/p_h[3]]

                    # Map to pixel coordinates (u, v) using image dimensions [G]
                    img_w, img_h = images[i].shape[1], images[i].shape[0]
                    u = int((p_ndc[0] + 1.0) * img_w / 2.0)
                    v = int((1.0 - p_ndc[1]) * img_h / 2.0)

                    # Check bounds
                    if not (0 <= u < img_w and 0 <= v < img_h):
                        continue

                    # Retrieve pixel color [S1]
                    pixel_color = images[i].get_color(u, v)
                    consistent_colors.append(pixel_color)

                if not consistent_colors:
                    voxel['alpha'] = 0
                    continue

                # Assign average color from consistent views [G]
                avg_color = [sum(c[j] for c in consistent_colors) / len(consistent_colors) for j in range(3)]

                # Test consistency: max allowable correlation error threshold [S1]
                # Using Euclidean distance as a standard proxy for correlation error [G]
                max_deviation = max(sum((c[j] - avg_color[j])**2 for j in range(3))**0.5 for c in consistent_colors)

                if max_deviation <= thresh:
                    voxel['color'] = avg_color
                    voxel['alpha'] = 255
                else:
                    voxel['alpha'] = 0

    return volume
```

**Edge Cases and Pitfalls:**
- **Ordinal Visibility Constraint:** If cameras surround the scene, the monotonic traversal order cannot be established, leading to incorrect visibility handling and reconstruction artifacts [S2, unofficial].
- **Threshold Selection:** A low `thresh` may result in an incomplete reconstruction (holes), while a high `thresh` may include false voxels (noise) [S1].
- **Depth Information:** The method relies on depth information or synthesized depth maps to project voxels accurately. Lack of accurate depth can lead to misalignment [S1].

## Subtopics

### [Generalized Voxel Coloring](5526-generalized-voxel-coloring.md)
Improves upon standard Voxel Coloring by computing visibility exactly using Item Buffers or Layered Depth Images, allowing for more accurate color-consistent models.

### [Multi-view Color Consistency and Smoothing](6445-multi-view-color-consistency-and-smoothing.md)
A post-processing method that refines voxel colors to handle noise and non-Lambertian reflectance, preserving sharp edges.

### [Volumetric Texture Rendering via Image Layers](11395-volumetric-texture-rendering-via-image-layers.md)
Renders volumetric textures by converting the voxel volume into 2D image layers for real-time rendering on standard graphics hardware.

## Operating characteristics
- **Time Complexity:** The algorithm traverses all voxels in the volume and checks consistency against all camera views. Complexity is $O(V \times N)$, where $V$ is the number of voxels and $N$ is the number of cameras [S2, unofficial].
- **Memory Complexity:** Requires storage for the 3D volume ($O(V)$) and the input images ($O(N \times I)$, where $I$ is image size) [S1].
- **Performance:** The method is computationally intensive due to the voxel traversal and consistency checks. However, it allows for a single scan reconstruction if the ordinal visibility constraint is met, which is faster than multi-scan methods like Space Carving [S2, unofficial].
- **Strengths:** Produces photo-realistic volume models with consistent colors across views. Simple to implement conceptually [S1].
- **Limitations:** Restricted by the ordinal visibility constraint; cannot handle scenes surrounded by cameras. Sensitive to the threshold parameter [S1][S2, unofficial].

## Background
Voxel Coloring was introduced by Seitz and Dyer in 1997 in the paper "Photorealistic Scene Reconstruction by Voxel Coloring" [S1]. The method laid the foundation for subsequent voxel-based reconstruction techniques [S1].

## Sources
[S1] Integration of Voxel Colouring Technique in the Volumetric Textures Representation Based on Image Layers, https://doi.org/10.3844/jcssp.2006.600.606, fetched 2026-10-26
[S2] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
