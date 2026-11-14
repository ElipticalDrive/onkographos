# Fast Maximum Intensity Projection (MIP)

Also called: Accelerated MIP, MIP via shear-warp

## Overview
Fast Maximum Intensity Projection (MIP) is a method to accelerate the computation of Maximum Intensity Projections by decomposing the density domain into intervals and using shear-warp projection optimized for finding the maximum density along rays [S1, unofficial]. Unlike classical MIP, which uses brute-force resampling at evenly spaced points, this method avoids unnecessary resampling by determining the interval containing the maximum density for each ray and resampling only at voxel locations where densities reside in that interval [S1, unofficial]. It builds on [Maximum Intensity Projection](902-maximum-intensity-projection.md) and differs from [Lacroute's shear-warp](902-maximum-intensity-projection.md) by using completely different preprocessing optimized for MIP rather than alpha-blending [S1, unofficial].

## How it works
**Inputs:** 3D volume data (voxel densities), viewing direction.
**Outputs:** 2D projection image where each pixel contains the maximum density value along the corresponding ray.

**Data Structures:**
*   **Density Interval Map:** A decomposition of the global density range $[D_{min}, D_{max}]$ into $N$ finite intervals.
*   **Binary Mask/Code Array:** For each voxel (or cell in the bilinear extension), a binary code indicating which density interval(s) the voxel's density falls into. In the basic method, this is a single bit per interval or a compacted integer mask. In the bilinear extension, two codes (max and min) are assigned per cell.

**Steps:**

1.  **Preprocessing (Interval Encoding):**
    *   Decompose the density domain into a finite number of intervals [S1, unofficial].
    *   For each voxel, determine which interval its density belongs to.
    *   Store this information in a binary mask or code array. For the basic method, a bit is set if the voxel density is in the current interval being tested.

2.  **Shear-Warp Transformation:**
    *   Generalize the method to arbitrary viewing directions using binary shear transformation and discrete translations of slices [S1, unofficial].
    *   Apply a binary shear transformation to the volume data and masks. To avoid cutting of binary codes stored in the integer array mask during the shift, the bits of the two-bit long segments in the array shift must be set to the same value [S1, unofficial].
    *   Generate an intermediate image in the sheared space. This involves projecting the sheared slices onto the image plane.

3.  **Ray Casting with Interval Pruning:**
    *   For each ray (pixel in the final image), determine the interval that contains the maximum density [S1, unofficial].
    *   This is done by checking the binary masks in the sheared space. The algorithm identifies the highest interval index $i$ such that the mask indicates the presence of a voxel in that interval along the ray.
    *   Resample the volume *only* at voxel locations where densities reside in the determined interval [S1, unofficial].
    *   Use nearest neighbor resampling for faster rendering with slight quality reduction, or trilinear interpolation for higher accuracy [S1, unofficial].

4.  **Refinement (Optional for Higher Accuracy):**
    *   If higher accuracy is required, determine exact entry and exit points for each cell intersected by the ray and find the local maximum in the bounded segment [S1, unofficial].
    *   If the volume is treated as a continuous 3D function with trilinear interpolation, the density along the ray segment is a third-degree polynomial [S1, unofficial].
    *   Calculate the local maximum analytically or heuristically [S1, unofficial].
    *   Optimize by only calculating local maxima for cells where the maximum corner density is greater than the current ray density [S1, unofficial].

5.  **Unprojection:**
    *   Produce the final image from the intermediate image in the sheared space using the standard shear-warp unprojection technique [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_mip(volume, view_direction, num_intervals):
    """
    volume: 3D array of voxel densities
    view_direction: vector (dx, dy, dz)
    num_intervals: number of density intervals to decompose the domain into
    """
    # 1. Preprocessing: Encode density intervals into binary masks
    # Determine global min/max density
    d_min = min(volume.flatten())
    d_max = max(volume.flatten())
    interval_width = (d_max - d_min) / num_intervals
    
    # Create binary masks for each interval
    # mask[i] is a 3D boolean array where True if voxel density is in interval i
    masks = []
    for i in range(num_intervals):
        lower = d_min + i * interval_width
        upper = lower + interval_width
        # Create mask for interval i
        mask_i = (volume >= lower) & (volume < upper)
        masks.append(mask_i)
        
    # 2. Shear-Warp Transformation
    # Apply binary shear transformation to volume and masks
    # This step transforms the volume so that rays are parallel to the z-axis in sheared space
    sheared_volume, sheared_masks = apply_shear_transformation(volume, masks, view_direction)
    
    # 3. Ray Casting with Interval Pruning
    # Initialize output image with -infinity
    output_image = np.full((height, width), -np.inf)
    
    # For each pixel (x, y) in the output image
    for y in range(height):
        for x in range(width):
            # Determine the interval containing the maximum density
            # Iterate from highest interval to lowest to find the first one present
            max_density = -np.inf
            found_interval = -1
            
            for i in range(num_intervals - 1, -1, -1):
                # Check if any voxel in the ray path has density in interval i
                # In sheared space, this corresponds to checking the mask along the z-axis
                if np.any(sheared_masks[i][y, x, :]):
                    found_interval = i
                    break
            
            # If a density was found, resample only in that interval
            if found_interval != -1:
                # Resample the volume in the found interval
                # Use nearest neighbor for speed
                samples = sheared_volume[found_interval][y, x, :]
                # Filter samples to only those in the interval (already implied by mask)
                # Find max density in these samples
                current_max = np.max(samples)
                
                # Update output image if this is the new global max
                if current_max > output_image[y, x]:
                    output_image[y, x] = current_max
                    
            # Optional: Refinement step for higher accuracy
            # If trilinear interpolation is used, find exact local maxima in cells
            # This is time-demanding and not suitable for interactive applications [S1, unofficial]
            
    # 4. Unprojection
    # Apply inverse shear transformation to get the final image
    final_image = apply_unprojection(output_image, view_direction)
    
    return final_image

def apply_shear_transformation(volume, masks, view_direction):
    """
    Applies binary shear transformation to volume and masks.
    To avoid cutting of binary codes, bits of two-bit long segments in the array shift
    must be set to the same value.
    """
    # Implementation details of shear transformation
    # This involves discrete translations of slices using a Bresenham-like symmetric 3D DDA line drawing algorithm [S1, unofficial]
    # Return sheared volume and masks
    pass

def apply_unprojection(image, view_direction):
    """
    Applies inverse shear transformation to produce the final image.
    """
    # Implementation details of unprojection
    pass
```

**Parameters:**
*   `num_intervals`: Number of density intervals. Typical values depend on the dynamic range of the volume and desired precision. Not given in the sources.
*   `view_direction`: The direction of projection. Can be arbitrary.

**Edge Cases and Pitfalls:**
*   **Interactive Performance:** Algorithms investigating local maximum locations inside a cell are time-demanding and not suitable for interactive applications [S1, unofficial].
*   **Binary Code Cutting:** During binary shear transformation, if the bits of the two-bit long segments in the array shift are not set to the same value, the binary codes stored in the integer array mask may be cut, leading to incorrect results [S1, unofficial].
*   **Approximation:** Since rays are approximated by 3D discrete lines, the algorithm does not generate exact maximum intensity projections [S1, unofficial].

## Subtopics

### Bilinear Interpolation Extension for Shear-Warp MIP
An extension to improve accuracy by using bilinear interpolation, assigning binary density codes to rectangular cells on planes perpendicular to the viewing direction's principal component rather than voxels [S1, unofficial]. It assigns two codes to each cell: one for the maximum density and one for the minimum density of the four corner voxels. The lower bound of the global maximum is determined as the highest minimum density. Resampling is performed only for cells where the encoded maximum density is not smaller than the lower bound or the current maximum [S1, unofficial]. Using four-bit density codes makes the shear operation slower but allows more exact information, resulting in fewer samples taken in the projection step [S1, unofficial]. It can be used as a fast MIP previewer by displaying highest density indices directly, yielding an image with 16 gray levels [S1, unofficial].

### Shear-warp Projection for MIP
A generalization of the MIP technique to arbitrary viewing directions using binary shear transformation and discrete translations of slices [S1, unofficial]. It applies a binary shear transformation, ensuring that the bits of the two-bit long segments in the array shift are set to the same value to avoid cutting of binary codes stored in the integer array mask [S1, unofficial]. It generates an intermediate image in sheared space and produces the final image using standard shear-warp unprojection [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Faster than classical brute-force MIP because it avoids unnecessary resampling. The complexity depends on the number of intervals and the efficiency of the shear-warp transformation. The refinement step (local maximum calculation) is time-demanding [S1, unofficial].
*   **Memory Complexity:** Requires storage for binary masks or codes for each interval. The bilinear extension requires more memory for two codes per cell and potentially more complex data structures [S1, unofficial].
*   **Measured Performance:** Tested on an SGI Indy workstation. Average shearing, MIP, and LMIP rendering times are summarized in Table 4.5 of the source, but specific numbers are not provided in the text [S1, unofficial].
*   **Strengths:** Significant time savings compared to classical MIP. Suitable for medical data visualization where speed is important. Can be used as a fast previewer.
*   **Limitations:** Approximates rays with 3D discrete lines, so it does not generate exact MIPs. The refinement step is too slow for interactive applications. The bilinear extension makes the shear operation slower.
*   **When to Use:** When fast MIP is needed for medical data visualization and exact precision is not critical, or when a preview is sufficient.
*   **When Not to Use:** When exact maximum intensity projection is required, or when interactive performance is needed and the refinement step is necessary.

## Background
Maximum Intensity Projection (MIP) is a useful technique for the analysis of medical data, such as the visualization of location, shape, and topology of blood vessels, and is part of almost every commercial medical-imaging system [S1, unofficial]. However, not many efficient algorithms existed for finding the maximum intensity along a mathematical ray [S1, unofficial]. The fast MIP method was developed to address this inefficiency, building on the shear-warp factorization algorithm proposed by Lacroute for fast alpha-blending evaluation [S1, unofficial]. The new method uses the shear-warp projection approach but with completely different preprocessing optimized for maximum intensity projection [S1, unofficial]. It was implemented in C++ and tested on an SGI Indy workstation [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
