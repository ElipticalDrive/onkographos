# Generalized Voxel Coloring

Also called: GVC-IB, GVC-LDI, Voxel Colouring

## Overview
Generalized Voxel Coloring (GVC) is a voxel-based 3D reconstruction technique that computes photo-realistic volume models from multiple color images by assigning colors (radiances) to voxels while guaranteeing consistency with a set of base images [S1]. It improves upon standard Voxel Coloring by computing visibility exactly, rather than approximately, to obtain a color-consistent model [S2, unofficial]. GVC is distinguished from its parent [Voxel Coloring](4418-voxel-coloring.md) by its explicit handling of visibility via Item Buffers (IB) or Layered Depth Images (LDI), and from [Multi-view Color Consistency and Smoothing](6445-multi-view-color-consistency-and-smoothing.md) by its foundational use of exact geometric visibility checks for voxel coloring [S2, unofficial].

## How it works
**Inputs:** A set of $n+1$ calibrated input images $I_0, \dots, I_n$, camera parameters (position, focal angles), and a 3D volume subdivision into layers.
**Outputs:** A set of colored voxels $S$ representing the reconstructed scene.

**Data Structures:**
1.  **Voxel Grid:** The 3D space is subdivided into layers of voxels at uniform distance $d$ from the camera. The total volume is $\nu = \bigcup_{i=1}^{r} vd_i$, where $vd_i = \{ V \mid V = d_i \}$ and $d_1, \dots, d_r$ is a sequence of increasing numbers [S1].
2.  **Marked Pixels ($\pi$):** A global set tracking pixels in input images that have already been associated with a colored voxel to ensure each pixel contributes to at most one voxel's color consistency check [S1].
3.  **Visibility Structure:** Either an Item Buffer (IB) or a Layered Depth Image (LDI) [S2, unofficial].
    *   **IB:** Records for every pixel in an image, the single surface voxel that is visible from that pixel.
    *   **LDI:** Records for every pixel in an image, a depth-sorted list of all surface voxels that project to that pixel. LDI contains a superset of IB information but consumes considerably more memory [S2, unofficial].

**Parameters:**
*   `thresh`: The maximum allowable correlation error. A low value yields precise but incomplete reconstruction; a larger value yields complete reconstruction with potential false voxels [S1].

**Algorithm Steps:**
1.  Initialize the set of colored voxels $S \leftarrow \emptyset$ and the global marked pixel set $\pi \leftarrow \emptyset$ [S1].
2.  Iterate through layers $i$ from $1$ to $r$:
    3.  For each voxel $V$ in layer $vd_i$:
        4.  For each input image $j$ from $0$ to $n$:
            5.  Project voxel $V$ onto image $I_j$ and compute its footprint $\rho$ [S1].
            6.  Identify the subset of pixels $\pi_j$ within the footprint $\rho$ that are not yet marked: $\pi_j \leftarrow \{ p \in \rho \mid p \notin \pi \}$ [S1].
        7.  Compute the consistency metric $\lambda_V$ for voxel $V$ based on the unmarked pixels in the footprints across all images [S1].
        8.  Let $m$ be the number of consistent pixels found (where consistency is determined by $\lambda_V$) [S1].
        9.  If $m > 0$ AND $\lambda_V < \text{thresh}$:
            10. Color the voxel $V$ and add it to $S$: $S \leftarrow S \cup \{V\}$ [S1].
        11. Update the global marked pixel set: $\pi \leftarrow \pi \cup \bigcup_{j} \pi_j$ [S1].
        12. Mark the individual pixels in the current footprints $\pi_j$ as used [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generalized_voxel_coloring(images, cameras, layers, thresh):
    """
    images: list of n+1 images I_0...I_n
    cameras: list of n+1 camera parameters
    layers: list of distances d_1...d_r
    thresh: maximum allowable correlation error
    Returns: set of colored voxels S
    """
    S = set()
    marked_pixels = set() # Global set of (image_index, pixel_coords)
    
    # Precompute visibility structures (IB or LDI) if not using exact ray casting per voxel
    # For exact visibility, we check occlusion during the loop or use precomputed structures
    # Note: The source algorithm iterates voxels and checks images. 
    # Exact visibility implies checking if V is the closest surface voxel to the camera 
    # along the ray for each pixel in the footprint, or using IB/LDI to know which voxels are visible.
    
    for i in range(1, len(layers) + 1):
        d = layers[i-1]
        layer_voxels = get_voxels_at_distance(d) # vd_i
        
        for V in layer_voxels:
            # Collect unmarked pixels from all images that cover V
            all_unmarked_pixels = []
            
            for j in range(len(images)):
                # Project V onto image I_j
                footprint = project_onto_image(V, cameras[j], images[j].resolution)
                
                # Find pixels in footprint not yet marked
                unmarked_in_footprint = []
                for p in footprint:
                    if (j, p) not in marked_pixels:
                        unmarked_in_footprint.append(p)
                        all_unmarked_pixels.append((j, p))
                
                # Note: The source marks pixels *after* consistency check for the voxel
                # but accumulates them. The order in source:
                # 1. Compute pi_j (unmarked)
                # 2. Compute lambda_V
                # 3. If consistent, color V
                # 4. Mark pi (union of all pi_j)
                
            # Compute consistency metric lambda_V
            # lambda_V is typically a correlation error or variance across the unmarked pixels
            # The source does not give the exact formula for lambda_V, only that it is computed
            # and compared to thresh. [G] Assuming lambda_V is the mean squared error or 
            # correlation coefficient of the colors of V projected onto the unmarked pixels.
            lambda_V = compute_consistency_metric(V, all_unmarked_pixels, images, cameras)
            
            # Count consistent pixels m
            # The source says "If m > 0 and lambda_V < thresh". 
            # This implies m is the count of pixels that satisfy the consistency condition.
            # [G] Assuming a pixel is consistent if its color matches V's color within tolerance.
            m = 0
            for j, p in all_unmarked_pixels:
                if is_pixel_consistent(V, p, images[j], cameras[j]):
                    m += 1
            
            if m > 0 and lambda_V < thresh:
                # Color the voxel
                # [G] Assign average color of consistent pixels to V
                V.color = compute_average_color(all_unmarked_pixels, images, cameras)
                S.add(V)
                
                # Mark pixels as used
                for j, p in all_unmarked_pixels:
                    marked_pixels.add((j, p))
                    
    return S

def is_pixel_consistent(V, p, image, camera):
    # [G] Standard implementation: project V to pixel, compare color
    projected_color = project_color(V, camera)
    actual_color = image.get_pixel_color(p)
    # Simple Euclidean distance or correlation
    return color_distance(projected_color, actual_color) < tolerance
```

## Subtopics

### Roxels
An algorithm that attempts to reconstruct semi-transparent voxels, unlike previous algorithms (including standard GVC) that assumed opaque object voxels [S2, unofficial]. It is chosen when the scene contains transparent or translucent materials that standard carving cannot represent [S2, unofficial].

### 6D Voxel Coloring
A method for reconstructing time-varying scenes by linking two time-consecutive 3D voxel spaces together to form a 6D space [S2, unofficial]. It is chosen for dynamic scene reconstruction where temporal coherence is required [S2, unofficial].

## Operating characteristics
**Time Complexity:** The algorithm iterates through all voxels in all layers and all input images. For each voxel, it computes footprints and consistency metrics. Exact visibility computation (via IB/LDI or ray casting) adds overhead compared to approximate methods. The source notes that voxel carving is computationally expensive, typically requiring tens of seconds or more [S2, unofficial].
**Memory Complexity:** Depends on the visibility structure. LDI consumes considerably more memory than IB because it stores a depth-sorted list of all surface voxels per pixel, whereas IB stores only the single visible voxel [S2, unofficial].
**Strengths:** Produces numerically more consistent reconstructions with input images compared to approximate visibility methods like Space Carving [S2, unofficial]. Provides photo-realistic volume models [S1].
**Limitations:** Computationally expensive. High memory usage if LDI is used. Sensitive to the `thresh` parameter; low values yield incomplete reconstructions, high values yield false voxels [S1]. Requires accurately calibrated cameras [S2, unofficial].
**When to use:** When high geometric and photometric consistency is required and computational resources are sufficient. When exact visibility is critical for quality.
**When not to use:** For real-time applications or very large scenes where memory is constrained (prefer IB over LDI, or approximate methods). For scenes with significant calibration errors (use Approximate Space Carving) [S2, unofficial].

## Background
Generalized Voxel Coloring was introduced by Culbertson et al. in 1999 (cited as [CMS99] in sources) and further detailed in subsequent works [S2, unofficial]. It is an extension of the original Voxel Coloring technique introduced by Seitz in 2000 (cited as [21] in [S1]), which itself is based on the Space Carving paradigm. The method was developed to address the shortcomings of approximate visibility in standard voxel coloring by computing visibility exactly using Item Buffers or Layered Depth Images [S2, unofficial]. It is used in volumetric reconstruction from uncalibrated or calibrated camera views, often as a core component in multiresolution carving methods [S2, unofficial].

## Sources
[S1] Integration of Voxel Colouring Technique in the Volumetric Textures Representation Based on Image Layers, https://doi.org/10.3844/jcssp.2006.600.606, fetched 2026-10-26
[S2] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
