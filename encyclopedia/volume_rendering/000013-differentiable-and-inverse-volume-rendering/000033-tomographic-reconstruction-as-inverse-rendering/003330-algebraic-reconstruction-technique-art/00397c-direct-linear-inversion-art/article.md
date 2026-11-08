# Direct Linear Inversion (ART)

Also called: Algebraic Reconstruction Technique (ART) - Matrix Inversion Variant, Image reconstruction via matrix inversion, Matrix Inversion, Matrix Inversion Tomography, Pseudo-inverse Reconstruction

## Overview
Direct Linear Inversion treats the discrete Radon transform as a linear system of equations $M \mathbf{v} = \mathbf{d}$, where $M$ is the forward projection matrix, $\mathbf{v}$ is the vectorized image, and $\mathbf{d}$ is the sinogram data. It solves for the image by computing the pseudo-inverse of $M$ and applying it to the data [S1, unofficial]. Unlike iterative algebraic methods such as [Simultaneous Algebraic Reconstruction Technique (SART)](10119-simultaneous-algebraic-reconstruction-technique-sart.md) which minimize residuals iteratively, this method computes a direct solution in a single matrix multiplication step [S1, unofficial]. It is distinguished by its explicit construction of the projection matrix and reliance on numerical linear algebra rather than iterative optimization [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A set of parallel-beam projections (sinogram) at various angles.
- **Output**: A reconstructed 2D image of dimensions $width \times height$.

### Data Structures
1.  **Projection Matrix ($M$)**: A dense matrix of size $(angles \times rays) \times (width \times height)$.
    -   Rows correspond to specific rays at specific angles.
    -   Columns correspond to pixels in the image.
    -   Entry $M_{ij}$ represents the intersection length (weight) between ray $i$ and pixel $j$.
2.  **Data Vector ($\mathbf{v}$)**: A column vector of size $angles \times rays$, containing the measured line integrals (sinogram values).
3.  **Reconstruction Vector ($\mathbf{r}$)**: A column vector of size $width \times height$, containing the pixel values.

### Parameters
-   **Image Dimensions**: $w$ (width), $h$ (height) in pixels.
-   **Angles**: Number of projection angles.
-   **Rays**: Number of rays per angle.
-   **Regularization Threshold ($\epsilon$)**: $10^{-6}$. Used to handle non-square or ill-conditioned matrices during pseudo-inverse computation [S1, unofficial].
-   **Angle Step Size**: $\Delta \theta = \pi / angles$ [S1, unofficial].
-   **Ray Offset**: $\sqrt{2}$ (half-width of the scan range) [S1, unofficial].
-   **Ray Step Size**: $2\sqrt{2} / (rays - 1)$ [S1, unofficial].

### Algorithm Steps
1.  **Initialize Matrix**: Create matrix $M$ of size $(angles \cdot rays) \times (w \cdot h)$ initialized to zero.
2.  **Compute Weights**: For each angle index $a \in [0, angles)$ and ray index $r \in [0, rays)$:
    a.  Calculate the ray's start and end points in normalized coordinates $[-1, 1]$.
    b.  Determine the intersection length of the ray with each pixel. The model assumes pixels are solid squares and rays are zero-width lines [S1, unofficial].
    c.  Populate $M$ at row index $k = a \cdot rays + r$ and column index $p = y \cdot w + x$ with the weight $wt$ for each pixel $(x, y)$ intersected by the ray.
3.  **Construct Data Vector**: Create vector $\mathbf{v}$ of size $angles \cdot rays$ from the sinogram data.
4.  **Compute Pseudo-Inverse**: Calculate $M^+$, the pseudo-inverse of $M$, using a regularization threshold of $\epsilon = 10^{-6}$ to stabilize the inversion [S1, unofficial].
5.  **Reconstruct**: Compute $\mathbf{r} = M^+ \mathbf{v}$.
6.  **Reshape**: Reshape vector $\mathbf{r}$ into a 2D image of dimensions $w \times h$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def direct_linear_inversion(sinogram_data, angles_count, rays_count, width, height):
    """
    Reconstructs an image from sinogram data using direct matrix inversion.
    
    Args:
        sinogram_data: Array of shape (angles_count, rays_count) containing projection values.
        angles_count: Number of projection angles.
        rays_count: Number of rays per angle.
        width: Image width in pixels.
        height: Image height in pixels.
        
    Returns:
        Reconstructed image array of shape (height, width).
    """
    # Flatten sinogram to vector v
    v = sinogram_data.flatten()
    
    # Initialize projection matrix M
    # Size: (angles * rays) x (width * height)
    num_measurements = angles_count * rays_count
    num_pixels = width * height
    M = np.zeros((num_measurements, num_pixels))
    
    # Precompute ray parameters
    # Ray offset (half-width of scan range)
    ray_offset = np.sqrt(2)
    # Ray step size
    ray_step = 2 * ray_offset / (rays_count - 1)
    # Angle step size
    angle_step = np.pi / angles_count
    
    # Build the projection matrix
    for a in range(angles_count):
        angle = a * angle_step
        for r in range(rays_count):
            # Calculate ray position offset perpendicular to the angle
            # In parallel beam geometry, the ray position t ranges from -sqrt(2) to sqrt(2)
            t = -ray_offset + r * ray_step
            
            # Determine ray start and end points in normalized coordinates [-1, 1]
            # This depends on the specific coordinate system, but typically involves
            # projecting the ray onto the image bounds.
            # For a square image of size 2x2 centered at origin (-1 to 1):
            # Ray direction vector
            cos_a = np.cos(angle)
            sin_a = np.sin(angle)
            
            # We need to find the intersection of the line defined by angle and offset t
            # with each pixel. A scan-conversion approach calculates the length of the
            # line segment crossing each pixel.
            
            # Simplified weight calculation:
            # Iterate over all pixels to find intersections
            for y in range(height):
                for x in range(width):
                    # Pixel center in normalized coordinates
                    # Assuming image spans [-1, 1] in both x and y
                    px = -1.0 + (x + 0.5) * (2.0 / width)
                    py = -1.0 + (y + 0.5) * (2.0 / height)
                    
                    # Calculate weight wt for this pixel and ray
                    # This requires a geometric intersection calculation
                    # Model: pixels are solid squares, rays are zero-width lines
                    wt = calculate_scan_weight(px, py, width, height, angle, t)
                    
                    # Populate matrix
                    row_idx = a * rays_count + r
                    col_idx = y * width + x
                    M[row_idx, col_idx] = wt

    # Compute pseudo-inverse with regularization
    # M_pseudo = (M^T M + epsilon * I)^-1 M^T  or using SVD-based pseudo-inverse
    # The source specifies a regularization threshold of 1e-6
    epsilon = 1e-6
    # Using numpy's pinv which handles regularization implicitly or explicitly
    # Note: Standard pseudo-inverse does not use a threshold in the same way as
    # truncated SVD. The source implies a specific implementation detail.
    # We assume a standard pseudo-inverse calculation here, potentially with
    # a small regularization term added to the diagonal for stability if needed.
    M_pseudo = np.linalg.pinv(M)
    
    # Compute reconstruction vector
    r = M_pseudo @ v
    
    # Reshape into image
    image = r.reshape((height, width))
    
    return image

def calculate_scan_weight(px, py, width, height, angle, t):
    """
    Calculates the intersection length of a ray with a pixel.
    This is a placeholder for the specific scan-conversion logic described
    in the source. The logic involves determining the length of the line
    segment crossing the pixel square.
    """
    # Implementation detail: [G]
    # The source describes a scan-conversion-like approach where pixels are
    # solid squares and rays are zero-width lines.
    # A robust implementation would use a line rasterization algorithm (e.g.,
    # Xiaolin Wu's or a simple DDA) to determine the intersection length.
    # For the purpose of this article, we assume a function exists that
    # returns the length of the intersection.
    
    # Simplified geometric approximation for demonstration:
    # This is not the exact algorithm from the source, but represents the
    # required operation.
    
    # Ray line equation in normal form: x*cos(a) + y*sin(a) = t
    # Pixel bounds: [px - dx, px + dx] x [py - dy, py + dy]
    # where dx = 1/width, dy = 1/height (in normalized coords)
    
    dx = 1.0 / width
    dy = 1.0 / height
    
    # Check if the ray passes through the pixel
    # Distance from pixel center to ray
    dist = abs(px * np.cos(angle) + py * np.sin(angle) - t)
    
    # If distance is less than the projected half-width of the pixel,
    # there is an intersection.
    # Projected half-width depends on angle
    proj_half_width = dx * abs(np.cos(angle)) + dy * abs(np.sin(angle))
    
    if dist < proj_half_width:
        # Calculate intersection length
        # Length = 2 * sqrt(proj_half_width^2 - dist^2) / sin(theta_intersection)
        # This is a simplification. The exact length depends on the entry and exit points.
        # A more accurate method uses the intersection of the line with the pixel edges.
        
        # For the sake of the pseudocode, we return a weight based on proximity
        # The source emphasizes the "length of the line segment crossing each pixel"
        return 2.0 * np.sqrt(max(0, proj_half_width**2 - dist**2))
    else:
        return 0.0
```

### Edge Cases and Pitfalls
-   **Discretization Effects**: Error variation occurs with odd/even ray counts due to discretization artifacts, particularly in small images [S1, unofficial].
-   **180-Degree Symmetry**: The 180-degree case is identical to the 0-degree case in parallel-beam geometry and should be avoided or handled to prevent redundant data [S1, unofficial].
-   **Matrix Shape**: The matrix $M$ is not required to be square ($width \cdot height \neq rays \cdot angles$); the pseudo-inverse handles non-square systems [S1, unofficial].
-   **Ill-Conditioning**: The matrix may be ill-conditioned. A regularization threshold of $10^{-6}$ is used to stabilize the pseudo-inverse computation [S1, unofficial].
-   **Performance**: The method is computationally expensive. Naive implementation complexity is $O((wh)^3)$ [S1, unofficial].

## Subtopics

### Simultaneous Algebraic Reconstruction Technique (SART)
An iterative method that updates the image estimate using all projections simultaneously in each iteration, offering better convergence properties than ART but requiring more memory per iteration. Choose when direct inversion is too slow or unstable for large images. [Link](10119-simultaneous-algebraic-reconstruction-technique-sart.md)

### Motion-Compensated Algebraic Reconstruction Technique
Extends ART to handle moving objects by incorporating motion models into the projection matrix. Choose when reconstructing dynamic scenes. [Link](7422-motion-compensated-algebraic-reconstruction-technique.md)

## Operating characteristics
-   **Time Complexity**: $O((wh)^3)$ for a naive implementation, where $w$ and $h$ are image dimensions [S1, unofficial]. In terms of image edge length $L$, this is roughly $O(L^6)$ [S1, unofficial].
-   **Memory Complexity**: $O(angles \cdot rays \cdot w \cdot h)$ to store the dense projection matrix $M$ [S1, unofficial].
-   **Measured Performance**: On a 35x40 scan reconstruction, RMS error was $< 1.0 \times 10^{-7}$ (test time 20s on an old MacBook) [S1, unofficial]. On a 10x10 scan reconstruction of an 8x8 synthetic image, RMS error was $< 1.0 \times 10^{-10}$ [S1, unofficial].
-   **Strengths**: Simple to implement conceptually; provides a direct solution without iterative convergence concerns; handles non-square systems naturally via pseudo-inverse [S1, unofficial].
-   **Limitations**: Extremely slow for large images due to cubic complexity; high memory usage for dense matrix storage; sensitive to discretization errors; prone to noise amplification if the matrix is ill-conditioned without regularization [S1, unofficial].
-   **When to Use**: Small images where simplicity and directness are prioritized over speed; educational purposes; scenarios where iterative methods fail to converge.
-   **When Not to Use**: Large-scale reconstructions; real-time applications; scenes with significant noise where regularization is difficult to tune.

## Background
The method originates from the observation that the Radon transform can be discretized as a linear matrix multiplication [S1, unofficial]. It was implemented as part of a project to rediscover tomographic reconstruction techniques, specifically as the `--algorithm=matrix-inversion` option in a Rust-based tomography tool [S1, unofficial]. The approach leverages standard linear algebra libraries (e.g., `nalgebra` in Rust) to compute the pseudo-inverse [S1, unofficial]. It is considered a "cheap trick" in the sense that it avoids complex algebraic theory by relying on computational power to invert the transform directly [S1, unofficial].

## Sources
[S1] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
