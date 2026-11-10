# Object-order Binary Volume Projection

Also called: Back-to-Front Algorithm, Back-to-Front Object-Order Projection, Object space rendering, Object-order voxel projection with value-sorted array, Voxel space projection

## Overview
Object-order Binary Volume Projection is an early forward-mapping technique for rendering binary volumes, where voxels with a value of one are projected onto the image plane in back-to-front order to ensure correct occlusion without a Z-buffer [S1, S2, unofficial]. It differs from its sibling [Per-voxel Projection (Object-order)](1816-per-voxel-projection-object-order.md) by strictly restricting the input to binary data (0 for background, 1 for object) and relying on traversal order rather than a depth buffer to resolve visibility [S2, unofficial]. Unlike later object-order methods like splatting, it does not reconstruct continuous density functions or handle gray-scale values [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D binary volume grid $V$ of dimensions $D_x \times D_y \times D_z$, where $V(x,y,z) \in \{0, 1\}$. A camera model defining the projection from voxel space to the 2D image plane.
*   **Output:** An image buffer $I$ and optionally a depth buffer $D$.

**Data Structures:**
*   **Volume Grid:** A 3D array indexed by $(x, y, z)$.
*   **Image Buffer:** A 2D array indexed by pixel coordinates $(u, v)$.
*   **Depth Buffer:** A 2D array indexed by pixel coordinates $(u, v)$, storing the distance of the projected voxel from the image plane.

**Parameters:**
*   **Traversal Order:** Determined by the relative orientation of the data axes to the image plane. For each axis, the loop direction (increasing or decreasing) is chosen such that voxels farther from the image plane are processed before those closer [S1, S2, unofficial].
*   **Clipping Planes:** Optional axis-parallel clipping planes. If present, traversal bounds are modified to exclude voxels beyond the clip plane [S1, S2, unofficial].

**Steps:**
1.  **Initialize Buffers:** Clear the image buffer $I$ and depth buffer $D$ to background values.
2.  **Determine Traversal Order:** Analyze the camera orientation to determine if each axis ($x, y, z$) should be traversed in increasing or decreasing order. The order of the axes in the nested loops is arbitrary, but the direction within each axis must satisfy the back-to-front requirement [S2, unofficial].
3.  **Traverse and Project:** Iterate through the volume using three nested loops. For each voxel $(x, y, z)$:
    *   If $V(x,y,z) == 0$, skip.
    *   If clipping planes are active, check if the voxel is within the valid range. If not, skip [S2, unofficial].
    *   Project the voxel center to the image plane to obtain pixel coordinates $(u, v)$.
    *   If $(u, v)$ is within the image bounds:
        *   Write the voxel's distance from the image plane to $D(u, v)$.
        *   Write a value of 1 (or the object color) to $I(u, v)$ [S1, S2, unofficial].
4.  **Post-processing (Optional):** The generated depth image $D$ can be passed to a 2D discrete shader for further processing, such as shading or compositing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def object_order_binary_projection(volume, image, depth, camera):
    """
    volume: 3D array [Dx, Dy, Dz] with values 0 or 1
    image: 2D array [W, H]
    depth: 2D array [W, H]
    camera: object with method project(x, y, z) -> (u, v, dist)
    """
    # Initialize buffers
    for u in range(W):
        for v in range(H):
            image[u, v] = 0
            depth[u, v] = infinity

    # Determine traversal directions based on camera orientation
    # dir_x, dir_y, dir_z are +1 or -1
    # start_x, end_x, step_x define the loop range
    # ... (Implementation depends on specific camera model) ...
    
    # Example for standard axes where z is depth and we traverse z back-to-front
    # This is a simplified assumption; actual directions depend on view matrix
    for z in range(Dz - 1, -1, -1):
        for y in range(Dy):
            for x in range(Dx):
                if volume[x, y, z] == 0:
                    continue
                
                # Check clipping planes if defined
                # if is_clipped(x, y, z): continue
                
                u, v, dist = camera.project(x, y, z)
                
                if 0 <= u < W and 0 <= v < H:
                    # Overwrite with current voxel's data
                    # Since we traverse back-to-front, this is the closest object so far
                    # for this pixel in the final image, but we are writing the 
                    # farthest first? No, back-to-front means we process far first.
                    # Wait: Back-to-front means we process far voxels first.
                    # If we overwrite, the last processed (closest) wins.
                    # The source says: "first processed voxel (farther away) prevails"
                    # This implies we should NOT overwrite if already written?
                    # Let's re-read S2: "If two voxels project to the same pixel... 
                    # the first processed voxel (farther away) prevails."
                    # This implies we only write if the pixel is empty?
                    # Or does "prevails" mean it stays?
                    # S2 also says: "In a front-to-back method... once a voxel is projected... 
                    # other voxels... are ignored".
                    # S1 says: "pixel value is overwritten by the distance".
                    # Contradiction?
                    # Let's look at S1 again: "The samples are projected in back-to-front 
                    # order to ensure correct visibility. If two voxels are projected onto 
                    # the same pixel, the first processed voxel must be farther away... 
                    # pixel value is overwritten by the distance".
                    # If we overwrite, the LAST one (closest) wins.
                    # But S1 says "first processed... prevails".
                    # This suggests we should NOT overwrite if the pixel is already occupied?
                    # Or does "prevails" mean it is the one that is rendered?
                    # If I overwrite, the closest one is in the buffer.
                    # If I don't overwrite, the farthest one is in the buffer.
                    # S2: "If an image is produced by projecting all voxels... in arbitrary 
                    # order... the one that was projected later will prevail... This problem 
                    # can be solved by traversing... back-to-front."
                    # If I traverse back-to-front, and I overwrite, the closest (last) prevails.
                    # This is correct for visibility.
                    # But S1 says "first processed... prevails".
                    # This is a direct contradiction in the sources regarding the write strategy.
                    # S1: "first processed voxel (farther away) prevails" -> Do not overwrite.
                    # S2: "traversing... back-to-front... correct visibility" -> Implies closest wins.
                    # Standard object-order back-to-front: Process far, then near.
                    # If you overwrite, near wins. This is correct.
                    # If you don't overwrite, far wins. This is incorrect for visibility.
                    # However, S1 explicitly says "first processed... prevails".
                    # Let's look at S1's context: "depth image... overwritten by the distance".
                    # If depth is overwritten, the last one (closest) has the smallest distance.
                    # If the image value is overwritten, the last one (closest) is shown.
                    # So S1's "first processed... prevails" might refer to the depth calculation 
                    # or a specific interpretation.
                    # Actually, S1 says: "If two voxels are projected onto the same pixel, 
                    # the first processed voxel must be farther away from the image plane."
                    # It does NOT explicitly say "the first processed voxel's color is kept".
                    # It says "The samples are projected in back-to-front order to ensure 
                    # correct visibility."
                    # Correct visibility means the closest object is seen.
                    # Therefore, the closest object's color must be in the buffer.
                    # If we overwrite, the closest object's color is in the buffer.
                    # So we MUST overwrite.
                    # Why does S1 say "first processed... prevails"?
                    # Maybe it means "the first processed one is the one that determines 
                    # the depth for clipping"? No.
                    # Let's assume standard behavior: Overwrite. The "prevails" in S1 might 
                    # be a typo or refers to a different context (e.g. if using a Z-buffer 
                    # where far is written first and near is rejected? No, object-order 
                    # usually doesn't use Z-buffer).
                    # Let's stick to the mechanism: Back-to-front traversal + Overwrite = Correct Visibility.
                    
                    image[u, v] = 1  # Object color
                    depth[u, v] = dist
```

**Edge Cases and Pitfalls:**
*   **Arbitrary Order:** If voxels are projected in arbitrary order, the last projected voxel overwrites earlier ones, causing incorrect occlusion where closer voxels might be hidden by farther ones [S2, unofficial].
*   **Precision:** Floating-point precision in projection calculations may cause gaps or overlaps in the image, especially at high resolutions or oblique angles [G].
*   **Clipping:** Clipping planes parallel to the image plane are implemented by modifying loop bounds. Incorrect bounds can lead to artifacts or missing data [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the volume. Each voxel is processed once [S1, S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the volume storage, plus $O(W \times H)$ for the image and depth buffers [S1, S2, unofficial].
*   **Performance:** Efficient for binary volumes due to simple traversal and projection. Performance degrades if the volume is sparse and the traversal overhead is high relative to the number of projected voxels [G].
*   **Strengths:** Simple implementation; no Z-buffer required; easy to generate depth images; supports axis-parallel clipping via loop bounds [S1, S2, unofficial].
*   **Limitations:** Restricted to binary volumes; no support for gray-scale or continuous density functions; no support for arbitrary clipping planes (only axis-parallel) [S1, S2, unofficial].
*   **When to Use:** When rendering binary volumes (e.g., segmented medical data) and simplicity is prioritized over visual fidelity or flexibility [S1, S2, unofficial].
*   **When Not to Use:** For gray-scale volumes, continuous rendering, or when arbitrary clipping planes are required [S1, S2, unofficial].

## Background
Object-order Binary Volume Projection was one of the first object-order methods for volume rendering, developed in the early days of the field [S1, unofficial]. It was designed to render binary volumes, such as those derived from medical imaging, by projecting voxels in back-to-front order [S1, S2, unofficial]. The method is described in foundational texts on volume visualization, including Kaufman's "Volume Visualization: Principles and Advances" [S2, unofficial] and various dissertations on interactive volume rendering [S1, unofficial]. It serves as a precursor to more sophisticated object-order techniques like splatting [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
