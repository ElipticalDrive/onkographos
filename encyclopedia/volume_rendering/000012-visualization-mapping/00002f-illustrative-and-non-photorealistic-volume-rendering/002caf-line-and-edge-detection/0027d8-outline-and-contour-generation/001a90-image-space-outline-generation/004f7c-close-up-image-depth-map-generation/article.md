# Close-Up Image Depth Map Generation

Also called: Local Edge Histogram CGDM, Occlusion-based CGDM

## Overview
A method to generate a Close-Up Image Depth Map (CGDM) for images lacking a single vanishing point or containing multiple convergence points, relying on occlusion cues and local edge histograms to infer depth [S1]. It differs from perspective view methods by using local edge density rather than global geometric projection, where regions with fewer edges are interpreted as farther away [S1].

## How it works
**Inputs:** A 2D grayscale or luminance image.
**Outputs:** An 8-bit CGDM image $D_f$ of the same dimensions as the input.

**Data Structures:**
*   **Edge Image:** Binary or grayscale map resulting from edge detection.
*   **Block Grid:** The image divided into non-overlapping $5 \times 5$ pixel blocks.
*   **Main Blocks:** A subset of blocks where the edge count $N_{ij}$ exceeds the average edge count $N_{av}$.
*   **CGDM Block ($D_i$):** An $R \times R$ circular gradient map centered at the block center, where $R$ is the radius.

**Parameters:**
*   $M$: Total number of main blocks.
*   $N_{i0}$: Number of edges in the $i$-th main block.
*   $N_{av}$: Average number of edges per block across the entire image.
*   $R$: Radius of the circular depth gradient. The optimized value is half the length (or width) of the image [S1].
*   $D_i$: The CGDM generated for the $i$-th main block.
*   $D_f$: The final fused CGDM.

**Steps:**
1.  **Edge Extraction:** Extract edges from the input 2D image using the Canny algorithm [S1].
2.  **Block Division:** Divide the edge image into a grid of $5 \times 5$ pixel blocks.
3.  **Edge Counting:** Count the number of edge pixels $N_{ij}$ in each block $(i, j)$.
4.  **Average Calculation:** Compute the average edge count $N_{av}$ across all blocks.
5.  **Main Block Identification:** Identify blocks where $N_{ij} > N_{av}$ as "main blocks." Let the total count of these blocks be $M$.
6.  **Local CGDM Generation:** For each main block $i$:
    a.  Define a circular region centered at the block's center.
    b.  Set the pixel value at the center to 255.
    c.  Set the pixel value at the circumference (radius $R$) to 0.
    d.  Interpolate pixel values linearly from 255 at the center to 0 at the circumference.
7.  **Fusion:** Combine the CGDMs of all main blocks into the final depth map $D_f$ using the weighted sum formula:
    $$ D_f = \sum_{i=1}^{M} \left( D_i \times \frac{N_{i0}}{\sum_{k=1}^{M} N_{k0}} \right) $$
    where $N_{i0}$ is the edge count of the $i$-th main block [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_close_up_cgdm(image):
    # Step 1: Extract edges
    edge_image = canny_edge_detection(image)
    
    # Step 2 & 3: Divide into 5x5 blocks and count edges
    block_size = 5
    height, width = edge_image.shape
    blocks = []
    edge_counts = []
    
    for y in range(0, height, block_size):
        for x in range(0, width, block_size):
            # Extract block, handling boundary conditions if necessary
            block = edge_image[y:y+block_size, x:x+block_size]
            # Count non-zero pixels (edges)
            n_ij = count_non_zero(block)
            blocks.append({'x': x, 'y': y, 'block': block})
            edge_counts.append(n_ij)
            
    # Step 4: Calculate average
    if not edge_counts:
        return np.zeros_like(image, dtype=np.uint8)
        
    n_av = np.mean(edge_counts)
    
    # Step 5: Identify main blocks
    main_blocks = []
    for idx, count in enumerate(edge_counts):
        if count > n_av:
            main_blocks.append(blocks[idx])
            
    m = len(main_blocks)
    if m == 0:
        return np.zeros_like(image, dtype=np.uint8)
        
    # Step 6: Generate local CGDMs
    # Optimized radius: half the image width or height
    r = min(width, height) // 2
    
    # Precompute circle mask and gradient values
    # Center of the circle is at (0,0) relative to the block center for generation,
    # then shifted to block center.
    y_coords, x_coords = np.ogrid[-r:r+1, -r:r+1]
    dist = np.sqrt(x_coords**2 + y_coords**2)
    
    # Normalize distance to [0, 1] where 0 is center, 1 is circumference
    # Avoid division by zero if r=0
    if r == 0:
        max_dist = 1.0
    else:
        max_dist = r
        
    normalized_dist = dist / max_dist
    # Gradient: 255 at center (dist=0), 0 at circumference (dist=1)
    gradient = 255 * (1 - normalized_dist)
    # Clip to valid range
    gradient = np.clip(gradient, 0, 255).astype(np.uint8)
    
    # Create a mask for the circle to ignore pixels outside radius
    circle_mask = (dist <= r).astype(np.uint8)
    
    # Initialize final depth map
    d_f = np.zeros((height, width), dtype=np.float32)
    
    # Sum of edge counts for normalization
    sum_n_i0 = sum(b['count'] for b in main_blocks) # Note: need to store count in block dict or re-access
    # Re-accessing counts for main blocks
    main_counts = []
    for b in main_blocks:
        # Find index in original edge_counts
        idx = blocks.index(b)
        main_counts.append(edge_counts[idx])
        
    sum_n_i0 = sum(main_counts)
    
    # Step 7: Fuse
    for i, b in enumerate(main_blocks):
        n_i0 = main_counts[i]
        
        # Create a blank canvas for this block's contribution
        d_i = np.zeros((height, width), dtype=np.float32)
        
        # Place the gradient circle at the block center
        # Block center coordinates
        cx = b['x'] + block_size // 2
        cy = b['y'] + block_size // 2
        
        # Define region of interest in the final map
        # The circle extends from cx-r to cx+r
        y_start = max(0, cy - r)
        y_end = min(height, cy + r + 1)
        x_start = max(0, cx - r)
        x_end = min(width, cx + r + 1)
        
        # Corresponding slice of the gradient and mask
        # Adjust gradient indices to match the ROI
        # Gradient is centered at 0,0. ROI is offset by (cx-r, cy-r)
        # So gradient index (dy, dx) corresponds to image index (cy-r+dy, cx-r+dx)
        # We need to map image indices back to gradient indices
        # dy_img = y - cy + r
        # dx_img = x - cx + r
        
        # Efficiently place the gradient
        # Create a temporary array for the circle
        circle_h = y_end - y_start
        circle_w = x_end - x_start
        
        # Calculate relative coordinates within the circle placement
        # y_rel = y - (cy - r)
        # x_rel = x - (cx - r)
        # But gradient is defined from -r to r.
        # So gradient index = y_rel - r = y - cy
        # Let's just slice the precomputed gradient and mask
        
        # Gradient shape is (2r+1, 2r+1)
        # We need to place it such that center (r, r) in gradient aligns with (cx, cy) in image
        # So gradient row 0 aligns with image row cy-r
        # gradient row r aligns with image row cy
        
        y_grad_start = 0
        y_grad_end = 2*r + 1
        
        # Clip to image bounds
        if y_start < 0:
            y_grad_start = -y_start
            y_start = 0
        if y_end > height:
            y_grad_end = y_grad_end - (y_end - height)
            y_end = height
            
        if x_start < 0:
            x_grad_start = -x_start
            x_start = 0
        if x_end > width:
            x_grad_end = x_grad_end - (x_end - width)
            x_end = width
            
        # Extract the relevant part of the gradient and mask
        # Note: The gradient/mask arrays are full size (2r+1, 2r+1)
        # We need to slice them based on the overlap with the image ROI
        # The ROI in the image is [y_start:y_end, x_start:x_end]
        # The corresponding indices in the gradient array are:
        # y_grad_idx = y_img - (cy - r)
        # So for y_img = y_start, y_grad_idx = y_start - cy + r
        # Since y_start >= cy-r (after clipping), this is >= 0
        
        y_g_start = y_start - (cy - r)
        y_g_end = y_end - (cy - r)
        x_g_start = x_start - (cx - r)
        x_g_end = x_end - (cx - r)
        
        # Ensure indices are within gradient bounds
        y_g_start = max(0, y_g_start)
        y_g_end = min(2*r + 1, y_g_end)
        x_g_start = max(0, x_g_start)
        x_g_end = min(2*r + 1, x_g_end)
        
        local_gradient = gradient[y_g_start:y_g_end, x_g_start:x_g_end]
        local_mask = circle_mask[y_g_start:y_g_end, x_g_start:x_g_end]
        
        # Calculate the weight for this block
        weight = n_i0 / sum_n_i0
        
        # Add to the final map
        # d_f[y_start:y_end, x_start:x_end] += local_gradient * local_mask * weight
        # Only add where mask is 1
        roi = d_f[y_start:y_end, x_start:x_end]
        roi_masked = roi * (1 - local_mask.astype(np.float32)) # Zero out existing values where mask is 1? 
        # The source implies fusion by sum. If blocks overlap, they sum. 
        # The mask ensures only the circle area is affected.
        # However, standard fusion usually just adds. The mask prevents adding outside the circle.
        
        # Correct logic:
        # d_f[y_start:y_end, x_start:x_end] += (local_gradient * local_mask * weight)
        
        d_f[y_start:y_end, x_start:x_end] += local_gradient.astype(np.float32) * local_mask.astype(np.float32) * weight
        
    # Convert to uint8
    d_f_uint8 = np.clip(d_f, 0, 255).astype(np.uint8)
    return d_f_uint8
```

## Subtopics

### Perspective View Depth Map Generation
A variant for images with a clear vanishing point (perspective view, $p \le 0.5$). It detects the vanishing point $(x_{vp}, y_{vp})$ via Hough transform on Canny edges and calculates separate horizontal and vertical depth gradients. Horizontal depth is assigned along columns from the VP to the edge, and vertical depth along rows [S1]. Pitfall: The vanishing point may not be central, requiring image expansion by zero-padding to avoid depth errors in reconstruction [S1].

## Operating characteristics
*   **Complexity:** Linear with respect to the number of pixels for edge detection and block counting. The fusion step is proportional to the number of main blocks $M$ and the area of the circles.
*   **Strengths:** Effective for close-up images where perspective cues are absent or ambiguous. Uses local occlusion cues (edge density) which are robust for object-centric scenes.
*   **Limitations:** Relies on the Canny algorithm's quality; noisy edges can distort block counts. The assumption that "fewer edges = farther" may not hold for all textures. The fixed $5 \times 5$ block size may not adapt well to varying scene scales.
*   **Performance:** Not given in the sources.

## Background
Originated in the context of holographic 3D display using 2D-to-3D rendering approaches [S1]. The method classifies images into perspective view and close-up types to apply appropriate depth map generation strategies for computer-generated holography (CGH) [S1].

## Sources
[S1] Holographic 3D Display Using Depth Maps Generated by 2D-to-3D Rendering Approach, https://doi.org/10.3390/app11219889, fetched 2026-10-26
