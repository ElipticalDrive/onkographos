# Assets Mining on Ground Feature Plane

Also called: Feature Plane Segmentation for Object Extraction, Ground Feature Plane Assets Mining

## Overview
Assets Mining on Ground Feature Plane is a post-processing technique for extracting individual object representations from a learned ground feature plane representation [S1]. It solves the problem of isolating specific scene elements (assets) from a continuous neural field by segmenting the 2D density feature plane into clusters and detecting contours to obtain bounding boxes and enclosed feature patches [S1]. It differs from its siblings by focusing on unsupervised extraction and reconfiguration of object-centric patches from the ground plane, rather than just efficient rendering or hierarchical scene modeling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A learned ground feature plane representation consisting of a density plane $M_\sigma$ and a color plane $M_{rgb}$ (or $M_c$) [S1]. Optionally, a DINO feature plane $M_{dino}$ for complex real-world scenes [S1].
*   **Output:** A set of object neural representations $P = \{(p_\sigma, p_c)\}$, where each pair represents the feature patches for density and color enclosed within the bounding box of a detected object [S1].

**Data Structures:**
*   $M_\sigma$: 2D grid storing density features.
*   $M_{rgb}$: 2D grid storing color features.
*   $M_{dino}$: 2D grid storing DINO semantic features (optional) [S1].
*   $P$: List of tuples, each containing a sub-grid (patch) from $M_\sigma$ and $M_{rgb}$ [S1].

**Steps:**
1.  **Select Density Plane:** Use $M_\sigma$ for segmentation because it exhibits sharper object boundaries compared to the color plane $M_{rgb}$ [S1]. This is attributed to the neural rendering mechanism where the model learns a clean density field first to guide color learning [S1].
2.  **Segmentation:** Apply K-means clustering to $M_\sigma$ to segment it into clusters [S1]. The number of clusters $K$ is a customizable parameter [S1].
3.  **Binary Mask Generation:** Convert the clustering result into a binary mask for each cluster of interest [S1].
4.  **Contour Detection:** Apply contour detection to the binary mask to locate each object and generate a set of bounding boxes [S1].
5.  **Patch Extraction:** For each bounding box, extract the corresponding feature patches from $M_\sigma$ and $M_{rgb}$ [S1]. These patches form the object neural representations $P = \{(p_\sigma, p_c)\}$ [S1].
6.  **Hierarchical Unpacking (Optional):** For complex hierarchical scenes (e.g., computer on table on floor), repeat the clustering step on the remaining background or specific regions to gradually unpack the scene structure [S1].
7.  **Semantic Guidance (Optional):** For real-world scenes, learn a separate DINO ground feature plane $M_{dino}$ to provide explicit indications of object presence, allowing object discovery on the field with the most distinctive features [S1].
8.  **Scene Editing (Application):** To edit the scene, "paste" an object feature patch $(p_\sigma^i, p_c^i) \in P$ to a designated location on $M_\sigma$ and $M_{rgb}$ to insert object $i$ [S1]. The edited ground feature planes $M' = (M'_\sigma, M'_c)$ are paired with the original vertical feature axes $H = (H_\sigma, H_c)$ to decode 3D information using the original decoders $Dec_\sigma$ and $Dec_{rgb}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assets_mining(M_sigma, M_rgb, M_dino=None, K=2, use_dino=False):
    """
    Extract object patches from ground feature planes.
    
    Args:
        M_sigma: 2D array, density feature plane.
        M_rgb: 2D array, color feature plane.
        M_dino: 2D array, DINO feature plane (optional).
        K: int, number of clusters for K-means.
        use_dino: bool, whether to use DINO plane for segmentation.
        
    Returns:
        P: List of tuples (patch_sigma, patch_rgb) for each object.
    """
    # Step 1: Select feature plane for segmentation
    # Use M_sigma for sharp boundaries, or M_dino if available and preferred
    if use_dino and M_dino is not None:
        feature_plane = M_dino
    else:
        feature_plane = M_sigma
        
    # Step 2: K-means Clustering
    # Labels[i, j] is the cluster ID for pixel (i, j)
    labels = kmeans_cluster(feature_plane, K)
    
    # Step 3 & 4: Generate Binary Masks and Detect Contours
    # Iterate through each cluster to find objects
    objects = []
    for cluster_id in range(K):
        # Create binary mask for current cluster
        binary_mask = (labels == cluster_id).astype(float)
        
        # Apply contour detection to find bounding boxes
        # Returns list of bounding boxes for this cluster
        bounding_boxes = find_contours(binary_mask)
        
        for bbox in bounding_boxes:
            # Step 5: Extract patches
            # bbox is typically (x_min, y_min, x_max, y_max)
            patch_sigma = M_sigma[bbox.y_min:bbox.y_max, bbox.x_min:bbox.x_max]
            patch_rgb = M_rgb[bbox.y_min:bbox.y_max, bbox.x_min:bbox.x_max]
            
            objects.append((patch_sigma, patch_rgb))
            
    return objects

def edit_scene(M_sigma, M_rgb, H_sigma, H_rgb, object_patches, target_location):
    """
    Insert an object into the scene by pasting its feature patches.
    
    Args:
        M_sigma: 2D array, density feature plane.
        M_rgb: 2D array, color feature plane.
        H_sigma: Vertical feature axis for density.
        H_rgb: Vertical feature axis for color.
        object_patches: Tuple (patch_sigma, patch_rgb) to insert.
        target_location: Tuple (x, y) for top-left corner of insertion.
        
    Returns:
        M_prime_sigma: Edited density plane.
        M_prime_rgb: Edited color plane.
    """
    patch_sigma, patch_rgb = object_patches
    x, y = target_location
    
    # Paste patches to designated location
    # Assuming target_location is within bounds
    M_prime_sigma = M_sigma.copy()
    M_prime_rgb = M_rgb.copy()
    
    # Calculate dimensions of the patch
    h, w = patch_sigma.shape
    
    # Paste
    M_prime_sigma[y:y+h, x:x+w] = patch_sigma
    M_prime_rgb[y:y+h, x:x+w] = patch_rgb
    
    # The edited planes are paired with original vertical axes H for decoding
    # Decoding is handled by the original decoder using M_prime and H
    return M_prime_sigma, M_prime_rgb
```

**Parameters:**
*   $K$: Number of clusters for K-means. Customizable based on desired object granularity [S1].
*   `use_dino`: Boolean flag to use DINO plane for segmentation [S1].

**Edge Cases and Pitfalls:**
*   **Hierarchical Scenes:** Objects arranged hierarchically (e.g., computer on table) may require repeated clustering to unpack [S1].
*   **Real-world Complexity:** Standard density planes may not provide clear object boundaries in complex real-world scenes; using a DINO plane $M_{dino}$ is recommended [S1].
*   **Boundary Precision:** Contour detection accuracy depends on the sharpness of the density field boundaries [S1].

## Subtopics

### [Neural Groundplans / SUDS](22036-neural-groundplans-suds.md)
Unsupervised methods deriving object-centric 3D representations from dynamic fields using connected-component labeling in feature space. They differ by relying on feature clustering and density thresholds rather than semantic labels, and explicitly disentangle static and dynamic features via ground-aligned 2D grids.

## Operating characteristics
*   **Time Complexity:** Dominated by K-means clustering on the 2D feature plane and contour detection. Complexity depends on the resolution of the feature plane and the number of clusters $K$ [S1].
*   **Memory Complexity:** Requires storing the feature planes $M_\sigma, M_{rgb}$ and the extracted patches $P$ [S1].
*   **Strengths:** Enables intuitive scene editing by allowing users to "paste" object patches [S1]. Provides unsupervised object extraction [S1].
*   **Limitations:** Relies on the quality of the learned feature planes; sharp boundaries in $M_\sigma$ are crucial [S1]. Hierarchical scenes may require multiple passes [S1].
*   **When to Use:** When object-level manipulation or extraction is needed from a ground feature plane representation [S1].
*   **When Not to Use:** When real-time rendering is the primary goal without post-processing, or when semantic labels are not required and simpler methods suffice [S1].

## Background
The method originates from the AssetField paper, which introduces ground feature plane representations for neural rendering [S1]. It builds on the principle that density fields guide color field learning, resulting in sharper boundaries in the density feature plane [S1]. The technique leverages standard computer vision tools (K-means, contour detection) to bridge the gap between continuous neural fields and discrete object representations [S1].

## Sources
[S1] AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation, https://doi.org/10.1109/iccv51070.2023.00301, fetched 2026-10-29
