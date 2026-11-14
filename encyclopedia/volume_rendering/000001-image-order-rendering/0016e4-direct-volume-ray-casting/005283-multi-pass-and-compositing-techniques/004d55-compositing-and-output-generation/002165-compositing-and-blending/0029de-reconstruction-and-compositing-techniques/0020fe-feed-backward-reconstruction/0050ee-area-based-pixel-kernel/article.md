# Area-based Pixel Kernel

Also called: Area kernel, Noyau de Pixel basé sur l’aire (Area-based Pixel Kernel), Noyau d’interpolation sur l’aire des pixels

## Overview
The Area-based Pixel Kernel is a reconstruction kernel used in feed-backward reconstruction that assigns pixel contributions proportional to the geometric area of intersection between a pixel and a thick projection line [S1, unofficial]. It solves the artifact issues inherent in Dirac and B-Spline kernels, which leave unreconstructed pixels during subsampling, and linear kernels, which fail to fully exploit available data during oversampling [S1, unofficial]. It is distinguished from its sibling [Spatial Convolution](2513-spatial-convolution.md) by its specific geometric interpretation of projection thickness and its optimization via rotation-based approximation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Projection data $p(\theta, \rho)$, where $\theta$ is the angle and $\rho$ is the radial coordinate; pixel grid coordinates $(i, j)$; projection line thickness $d\rho$; pixel size $\Delta p$ (often assumed normalized or equal to $d\rho$ in specific derivations, but distinct in the general formula).
*   **Outputs:** Reconstructed image intensity $I(i, j)$.

**Data Structures:**
*   **Projection Buffer:** Array of values indexed by $(\theta, \rho)$.
*   **Temporary Image $I_\theta$:** A 2D array used to store the backprojected contributions for a specific angle $\theta$ before rotation.
*   **Result Image $I$:** The final accumulated 2D array.

**Parameters:**
*   $d\rho$: Thickness of the projection line.
*   $\Delta p$: Pixel width (used in the boundary case formula).
*   $\theta$: Current projection angle.
*   $\rho$: Radial position of the projection line.
*   $i, j$: Integer pixel coordinates.

**Numbered Steps:**
1.  Initialize the result image $I$ to zero.
2.  For each projection angle $\theta$:
    a.  Create a temporary image $I_\theta$ of the same dimensions as $I$, initialized to zero.
    b.  For each radial sample $\rho$ in the projection $p(\theta, \cdot)$:
        i.  Calculate the kernel weight $\Lambda(\phi=0, \rho, i, j)$ for every pixel $(i, j)$ in $I_\theta$ using the formula below. Note that for the optimization step, the kernel is calculated assuming $\phi=0$ (horizontal/vertical alignment).
        ii. Add the contribution $p(\theta, \rho) \times \Lambda(\phi=0, \rho, i, j)$ to $I_\theta(i, j)$.
    c.  Rotate the temporary image $I_\theta$ by the angle $-\theta$.
    d.  Add the rotated $I_\theta$ to the result image $I$.

**Formula:**
The kernel $\Lambda(\phi=0, \rho, i, j)$ for the approximation where the projection is aligned with the axes ($\phi=0$) is defined as [S1, unofficial]:

$$
\Lambda(\phi=0, \rho, i, j) = 
\begin{cases} 
d\rho & \text{if } [\rho - d\rho/2] = [\rho + d\rho/2] = [\rho] = i \\
\frac{d\rho + \Delta p}{2} + |\rho - i| & \text{if } [\rho - d\rho/2] < i \land [\rho + d\rho/2] = i \\
1 & \text{if } [\rho - d\rho/2] < i \land [\rho + d\rho/2] > i \\
0 & \text{otherwise}
\end{cases}
$$

Where $[\cdot]$ is the round function returning the nearest integer to the real parameter [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def area_kernel_weight(rho, d_rho, delta_p, pixel_i):
    """
    Calculates the kernel weight for a single pixel i given rho, d_rho, and delta_p.
    Assumes phi=0 alignment.
    """
    # Helper for round to nearest integer
    def round_to_int(x):
        return int(math.floor(x + 0.5))

    rho_low = rho - d_rho / 2.0
    rho_high = rho + d_rho / 2.0
    
    i_low = round_to_int(rho_low)
    i_high = round_to_int(rho_high)
    i_mid = round_to_int(rho)
    
    # Case 1: The entire thick line falls within the single pixel i
    if i_low == i_high == i_mid == pixel_i:
        return d_rho
    
    # Case 2: The thick line starts in pixel i-1 (or lower) and ends exactly at the boundary of pixel i
    # Condition: [rho - d_rho/2] < i AND [rho + d_rho/2] == i
    if i_low < pixel_i and i_high == pixel_i:
        return (d_rho + delta_p) / 2.0 + abs(rho - pixel_i)
    
    # Case 3: The thick line spans across pixel i (starts before i and ends after i)
    # Condition: [rho - d_rho/2] < i AND [rho + d_rho/2] > i
    if i_low < pixel_i and i_high > pixel_i:
        return 1.0
    
    # Case 4: No intersection
    return 0.0

def backproject_area_kernel(projections, image_shape, d_rho, delta_p):
    """
    Reconstructs an image using the Area-based Pixel Kernel approximation.
    
    Args:
        projections: List of tuples (theta, rho_values) where rho_values is an array of projection values.
        image_shape: Tuple (height, width) of the output image.
        d_rho: Thickness of the projection line.
        delta_p: Pixel size.
        
    Returns:
        Reconstructed image as a 2D array.
    """
    height, width = image_shape
    result_image = [[0.0 for _ in range(width)] for _ in range(height)]
    
    for theta, rho_values in projections:
        # Step 1: Create temporary image for current angle
        temp_image = [[0.0 for _ in range(width)] for _ in range(height)]
        
        # Step 2: Backproject each rho into temp_image assuming phi=0
        for rho, value in rho_values:
            # Iterate over all pixels to calculate weights
            # Optimization: Only iterate pixels where weight > 0
            # Based on formula, weight is non-zero only for i in [i_low - 1, i_high + 1] roughly
            # But for correctness, we check the condition logic
            
            # Determine range of pixels that might have non-zero weight
            # i_low = round(rho - d_rho/2), i_high = round(rho + d_rho/2)
            # Non-zero if i_low < i AND i_high > i OR boundary cases.
            # Safe range: from min(i_low, i_high) - 1 to max(i_low, i_high) + 1
            # However, the formula is strict. We iterate all pixels or a tight bounding box.
            
            # Tight bounding box for non-zero weights:
            # Case 1: i == round(rho). Range is just i.
            # Case 2: i == round(rho + d_rho/2). Range is i.
            # Case 3: i == round(rho - d_rho/2). Range is i.
            # Case 4: i between round(rho - d_rho/2) and round(rho + d_rho/2).
            
            # Let's just iterate the relevant integer indices
            rho_low = rho - d_rho / 2.0
            rho_high = rho + d_rho / 2.0
            
            start_i = int(math.floor(min(rho_low, rho_high) - 1))
            end_i = int(math.ceil(max(rho_low, rho_high) + 1))
            
            for i in range(max(0, start_i), min(width, end_i)):
                weight = area_kernel_weight(rho, d_rho, delta_p, i)
                if weight > 0:
                    # In the approximation, the contribution is spread along the axis perpendicular to theta=0
                    # i.e., along the j-axis (vertical) for all j? 
                    # The source says "backproject ... into a temporary image". 
                    # Usually, for a line at rho, it affects all pixels with that radial coordinate.
                    # In the phi=0 approximation, the line is horizontal (or vertical depending on coord system).
                    # The formula depends on i (row or col index corresponding to rho).
                    # Assuming i corresponds to the radial dimension.
                    # The contribution is uniform across the other dimension (j) in the temp image?
                    # The source implies the weight is calculated for pixel (i,j).
                    # If the line is horizontal (phi=0), it spans all j.
                    # So we add to all j for the calculated i.
                    
                    for j in range(height):
                        temp_image[j][i] += value * weight
                        
        # Step 3: Rotate temp_image by -theta and add to result_image
        # Rotation implementation:
        # Map each pixel (r, c) in result_image to (r', c') in temp_image
        # Or map temp_image pixels to result_image.
        # Standard rotation of image by -theta:
        # x' = x cos(-theta) - y sin(-theta)
        # y' = x sin(-theta) + y cos(-theta)
        
        # We iterate over the result image coordinates to sample from temp_image
        for r in range(height):
            for c in range(width):
                # Center coordinates
                cx = c - width / 2.0
                cy = r - height / 2.0
                
                # Rotate coordinates by -theta to find source in temp_image
                # cos(-theta) = cos(theta), sin(-theta) = -sin(theta)
                cos_t = math.cos(theta)
                sin_t = math.sin(theta)
                
                src_x = cx * cos_t - cy * (-sin_t) # cx cos + cy sin
                src_y = cx * (-sin_t) + cy * cos_t # -cx sin + cy cos
                
                # Convert back to pixel indices
                src_c = src_x + width / 2.0
                src_r = src_y + height / 2.0
                
                # Bilinear interpolation or nearest neighbor for sampling temp_image
                # For simplicity and exactness of "addition", we assume nearest or bilinear.
                # The source does not specify interpolation for the rotation step, 
                # only that the image is rotated and added.
                # We use nearest neighbor for discrete implementation.
                
                src_c_int = int(math.floor(src_c + 0.5))
                src_r_int = int(math.floor(src_r + 0.5))
                
                if 0 <= src_r_int < height and 0 <= src_c_int < width:
                    result_image[r][c] += temp_image[src_r_int][src_c_int]
                    
    return result_image
```

**Edge Cases and Pitfalls:**
*   **Complex Geometry:** The exact calculation of the common area between a thick line and a pixel generates complex shapes requiring non-trivial decomposition into simple shapes, which is why the rotation approximation is used [S1, unofficial].
*   **Precision:** The use of the round function $[\cdot]$ introduces discrete boundaries. Small changes in $\rho$ can cause jumps in the kernel weight, potentially leading to artifacts if not handled with care in floating-point arithmetic [S1, unofficial].
*   **Boundary Conditions:** The formula assumes the pixel grid aligns with the coordinate system used for $\rho$. Misalignment requires careful offset handling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The approximation reduces the complexity of the area calculation. For each projection, it requires $O(N_\rho \cdot N_{pixels})$ for the backprojection into the temporary image, plus $O(N_{pixels})$ for the rotation and accumulation. The rotation step dominates if implemented naively, but can be optimized.
*   **Memory Complexity:** Requires an additional temporary image of size $O(N_{pixels})$ per projection angle, or a single temporary image reused for all angles.
*   **Strengths:** Eliminates unreconstructed pixels (holes) present in Dirac and B-Spline kernels during subsampling. Fully exploits data in oversampling scenarios, unlike linear kernels [S1, unofficial].
*   **Limitations:** The rotation approximation introduces its own interpolation errors. The exact area calculation is avoided, which may lead to slight inaccuracies compared to a true geometric integration [S1, unofficial].
*   **When to Use:** When high geometric fidelity is required and subsampling artifacts are a concern, and computational resources allow for the rotation step.

## Background
The Area-based Pixel Kernel was introduced in the context of tomographic reconstruction to address the limitations of standard interpolators. It is described in the thesis "Précision et qualité en reconstruction tomographique : algorithmes et applications" [S1, unofficial]. The method is part of the feed-backward reconstruction framework, where convolution is viewed as gathering contributions from neighboring input samples for each output sample [S1, unofficial].

## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
