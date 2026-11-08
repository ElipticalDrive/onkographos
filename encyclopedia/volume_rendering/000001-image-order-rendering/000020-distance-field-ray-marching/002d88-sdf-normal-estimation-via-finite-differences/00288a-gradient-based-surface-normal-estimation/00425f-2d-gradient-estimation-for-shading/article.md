# 2D Gradient Estimation for Shading

Also called: Depth Image Gradient Shading, Post-processing Normal Estimation from Depth

## Overview
Estimates surface normals from a rendered depth image (distance from the view plane to the surface) using 2D central differences, enabling post-processing shading without recomputing the volume render [S1, book]. It sets itself apart from 3D gradient estimation by operating on the final 2D image plane rather than the volumetric data, offering significantly faster performance and allowing quick lighting/surface property changes without depth image recomputation, though at the cost of lower accuracy and potential edge blurring [S1, book].

## How it works
**Inputs and Outputs:**
- Input: A depth image $Z$, where $Z(x, y)$ represents the distance from the view plane to the surface at pixel coordinates $(x, y)$.
- Output: A normal map (unit normal vectors) for each pixel.

**Data Structures:**
- The depth image is typically a 2D array of floating-point values.
- Pixel coordinates $(x_p, y_p)$ index into this array.

**Steps:**
1. **Compute Gradient Components:** Calculate the partial derivatives of the depth image with respect to $x$ and $y$ using central differences. The $z$-component of the gradient is assumed to be constant.
   - $\frac{\partial Z}{\partial x} \simeq \frac{Z(x_p + \Delta x, y_p) - Z(x_p - \Delta x, y_p)}{2 \Delta x}$
   - $\frac{\partial Z}{\partial y} \simeq \frac{Z(x_p, y_p + \Delta y) - Z(x_p, y_p - \Delta y)}{2 \Delta y}$
   - $\frac{\partial Z}{\partial z} \simeq 1$
   Here, $x_p, y_p$ are the pixel coordinates, and $\Delta x, \Delta y$ are the step sizes [S1, book].
2. **Normalize:** Normalize the resulting gradient vector $\vec{n} = (\frac{\partial Z}{\partial x}, \frac{\partial Z}{\partial y}, \frac{\partial Z}{\partial z})$ to produce a unit normal vector [S1, book].

**Parameters:**
- $\Delta x, \Delta y$: Typically set to the pixel spacing in $x$ and $y$ (i.e., 1 pixel) to use neighboring pixel values. Larger values can be used to smooth the image [S1, book].

**Pitfalls:**
- **Disjoint Regions:** Normals are computed from depth values that may represent disjoint regions in the volume (e.g., background and foreground object edges). This leads to blurring of sharp features on object edges [S1, book].
- **Boundary Conditions:** Central differences require neighbors on both sides. Pixels at the image boundary must be handled (e.g., clamping, reflection, or using one-sided differences) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normals_2d(depth_image, dx=1, dy=1):
    """
    Computes surface normals from a depth image using 2D central differences.
    
    Args:
        depth_image: 2D array of floats, Z(x, y).
        dx: Step size in x direction.
        dy: Step size in y direction.
        
    Returns:
        normals: 2D array of 3D unit normal vectors.
    """
    height, width = depth_image.shape
    normals = np.zeros((height, width, 3))
    
    for y in range(height):
        for x in range(width):
            # Handle boundaries: clamp to nearest valid neighbor
            x_left = max(0, x - dx)
            x_right = min(width - 1, x + dx)
            y_up = max(0, y - dy)
            y_down = min(height - 1, y + dy)
            
            # Central differences
            dz_dx = (depth_image[x_right, y] - depth_image[x_left, y]) / (2 * dx)
            dz_dy = (depth_image[x, y_down] - depth_image[x, y_up]) / (2 * dy)
            dz_dz = 1.0
            
            # Gradient vector
            grad = np.array([dz_dx, dz_dy, dz_dz])
            
            # Normalize
            norm = np.linalg.norm(grad)
            if norm > 0:
                normals[y, x] = grad / norm
            else:
                normals[y, x] = np.array([0.0, 0.0, 1.0]) # Default normal
                
    return normals
```

## Subtopics

### Curvature-Constrained 2D Gradient Estimation
A refinement of 2D gradient estimation that restricts normal computation to pixels within the same continuous curvature region to prevent blurring across disjoint object boundaries [S1, book]. It locates regions of continuous curvature in the depth image and estimates the normal for a pixel using only other pixel values that fall within the same curvature region [S1, book]. This may require reducing $\Delta x$ and $\Delta y$ values or using off-centered differences (forward/backward) to estimate gradient components [S1, book]. For example, the x-component of the gradient could be computed with a forward difference: $\frac{\partial Z}{\partial x} \simeq \frac{Z(x_p + \Delta x, y_p) - Z(x_p, y_p)}{\Delta x}$, or a backward difference: $\frac{\partial Z}{\partial x} \simeq \frac{Z(x_p, y_p) - Z(x_p - \Delta x, y_p)}{\Delta x}$ [S1, book].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of pixels in the depth image. Each pixel requires a constant number of arithmetic operations.
- **Memory Complexity:** $O(N)$ to store the input depth image and output normal map.
- **Performance:** Generally faster than 3D gradient estimation [S1, book].
- **Accuracy:** Less accurate than 3D gradient estimation [S1, book].
- **Strengths:** Allows quick lighting and surface property changes without requiring the depth image to be recomputed [S1, book].
- **Limitations:** Prone to blurring sharp features on object edges due to disjoint regions in the volume [S1, book]. Not suitable for shading effects in images computed with compositing techniques where gradients are needed at many locations within the volume for each pixel; a 3D gradient estimation technique is more suitable for this purpose [S1, book].

## Background
Originated in the context of volume rendering and surface illumination models, specifically for capturing ambient, diffuse, and specular highlights from depth images [S1, book]. The technique is described in the VTK Book, Chapter 7, referencing earlier work on curvature-constrained estimation [Yagel92a] [S1, book]. It is used in post-processing stages of volume rendering pipelines where a depth image is available but full volumetric shading is too expensive.

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
