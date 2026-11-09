# Boundary Cell-Based Ray Casting

Also called: BC-RC, Improved PARC

## Overview
Boundary Cell-Based Ray Casting (BC-RC) is a fast serial ray casting algorithm that accelerates volume rendering by skipping empty space using boundary cells [S1, unofficial]. It improves upon the PARC algorithm by using tiny grid cells for more accurate boundary estimation and employing fast incremental parallel projection to reduce projection overhead [S1, unofficial]. Unlike PARC, which uses pre-calculated sub-volumes whose projection is expensive when small, BC-RC projects only cells containing the object boundary [S1, unofficial].

## How it works
**Inputs:**
- A regular grid of volume elements (voxels) with scalar values [S1, unofficial].
- Camera parameters for parallel projection [S1, unofficial].

**Outputs:**
- A rendered image with ray casting optimizations (adaptive sampling, early termination) [S1, unofficial].

**Data Structures:**
1.  **Volume Grid:** Regular grid of voxels.
2.  **Boundary Flags:** A binary flag $F$ for each cell $C(i, j, k)$.
    -   $F=1$: The cell contains the object boundary.
    -   $F=0$: The cell is empty or fully inside/outside (not a boundary cell) [S1, unofficial].
3.  **Z-Buffers:**
    -   $Z_n$: Front z-buffer. Stores the closest intersection distance to the object boundary for each pixel [S1, unofficial].
    -   $Z_r$: Rear z-buffer. Stores the farthest intersection distance to the object boundary for each pixel [S1, unofficial].

**Algorithm Steps:**

1.  **Boundary Cell Detection (Preprocessing):**
    Scan the volume in an off-line stage. For each cell $C(i, j, k)$, determine if it contains the object boundary. If yes, set its flag $F=1$ [S1, unofficial].

2.  **Boundary Cell Projection:**
    Project only cells with $F=1$ onto the image plane using incremental parallel projection [S1, unofficial].
    -   Project the first cell $C(0,0,0)$ to establish the base position and depth [S1, unofficial].
    -   Calculate projected positions and depths of remaining boundary cells using incremental vectors (addition only) [S1, unofficial].
    -   Update $Z_n$ and $Z_r$ for each pixel based on the depth of the projected boundary cells. $Z_n$ stores the minimum depth, $Z_r$ stores the maximum depth [S1, unofficial].

3.  **Ray Traversal:**
    Cast rays only from viable pixels where $Z_n$ and $Z_r$ indicate an intersection (i.e., $Z_n < Z_r$ and within bounds) [S1, unofficial].
    -   Traverse each ray from the closest intersection ($Z_n$) to the farthest intersection ($Z_r$) [S1, unofficial].
    -   Sample and composite the volume data starting from the intersection [S1, unofficial].
    -   Incorporate adaptive image sampling and early ray termination [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundary_cell_based_ray_casting(volume, camera_params):
    # Step 1: Boundary Cell Detection
    # Initialize flags for all cells
    num_cells_x, num_cells_y, num_cells_z = volume.shape
    boundary_flags = np.zeros((num_cells_x, num_cells_y, num_cells_z), dtype=bool)
    
    # Scan volume to find boundary cells
    # A cell is a boundary cell if it contains part of the object surface.
    # Implementation detail: Check if the cell intersects the isosurface or 
    # if adjacent cells have different material properties.
    for i in range(num_cells_x):
        for j in range(num_cells_y):
            for k in range(num_cells_z):
                if is_boundary_cell(volume, i, j, k):
                    boundary_flags[i, j, k] = True

    # Step 2: Boundary Cell Projection
    # Initialize z-buffers with infinity/empty values
    image_width, image_height = camera_params.image_size
    Z_n = np.full((image_height, image_width), np.inf)
    Z_r = np.full((image_height, image_width), -np.inf)
    
    # Project boundary cells using incremental parallel projection
    # Get projection parameters from camera_params
    # Assume parallel projection along Z-axis for simplicity in description,
    # or general parallel projection vector v.
    
    # Project first boundary cell C(0,0,0) if it is a boundary cell
    # Calculate its projected pixel coordinates (u0, v0) and depth d0
    # This requires mapping 3D cell center to 2D image plane
    
    # For all other boundary cells, use incremental updates
    # Let (u_prev, v_prev, d_prev) be the projected position/depth of the previous cell
    # Let (du, dv, dd) be the incremental vector for the next cell in scan order
    
    for i in range(num_cells_x):
        for j in range(num_cells_y):
            for k in range(num_cells_z):
                if boundary_flags[i, j, k]:
                    # Calculate projected pixel (u, v) and depth d
                    # Using incremental vectors from previous cell
                    # u = u_prev + du
                    # v = v_prev + dv
                    # d = d_prev + dd
                    
                    # Update Z-buffers
                    if d < Z_n[v, u]:
                        Z_n[v, u] = d
                    if d > Z_r[v, u]:
                        Z_r[v, u] = d

    # Step 3: Ray Traversal
    image = np.zeros((image_height, image_width, 3))
    
    for y in range(image_height):
        for x in range(image_width):
            if Z_n[y, x] < Z_r[y, x]: # Viable pixel
                # Cast ray from Z_n[y, x] to Z_r[y, x]
                # Sample volume along ray
                # Composite color and opacity
                # Apply adaptive sampling and early termination
                color = cast_ray(volume, camera_params, x, y, Z_n[y, x], Z_r[y, x])
                image[y, x] = color
                
    return image

def is_boundary_cell(volume, i, j, k):
    # [G] Implementation detail: Check if cell intersects isosurface.
    # Simple heuristic: Check if voxel values at corners differ significantly
    # or if the cell contains the isosurface value.
    # This is a placeholder for the specific boundary detection logic.
    return False # Placeholder
```

## Operating characteristics
-   **Time Complexity:** Depends on the number of boundary cells and the length of rays within the object. The projection step is $O(N_b)$ where $N_b$ is the number of boundary cells. The ray traversal is $O(N_p \cdot L)$ where $N_p$ is the number of viable pixels and $L$ is the average ray length within the object.
-   **Memory Complexity:** $O(N_v)$ for the volume data and boundary flags, plus $O(N_p)$ for the z-buffers, where $N_v$ is the number of voxels and $N_p$ is the number of pixels.
-   **Strengths:** Fast serial rendering; accurate boundary estimation; reduced projection overhead compared to PARC; compatible with existing ray casting optimizations (adaptive sampling, early termination) [S1, unofficial].
-   **Limitations:** Requires off-line preprocessing; serial algorithm (though a parallel version was proposed) [S1, unofficial].

## Background
Boundary Cell-Based Ray Casting was proposed as an improvement on the PARC algorithm [S1, unofficial]. It was developed to address the computational overhead of projecting pre-calculated sub-volumes in PARC, which becomes expensive when the sub-volumes are small [S1, unofficial]. The algorithm was part of research into time-critical volume rendering, including the development of hierarchical level-of-detail representations and user interfaces for interactive exploration [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
