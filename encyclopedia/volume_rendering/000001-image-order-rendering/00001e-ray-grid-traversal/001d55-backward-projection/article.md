# Backward Projection


## Overview
Backward Projection is a rendering technique that maps volumetric data attributes (such as intensity, scale, or opacity) from the 3D voxel grid onto a 2D image plane by accumulating contributions along rays. It solves the problem of visualizing volumetric features by aggregating weighted voxel values into image pixels, effectively performing a weighted sum of the volume data along the viewing direction. Unlike forward projection methods that project image pixels into the volume, backward projection traverses the volume for each pixel, allowing for flexible compositing and transfer function application. It is distinct from its siblings in [Ray-grid traversal](17-ray-grid-traversal.md) by focusing on the accumulation of scalar or vector attributes into a buffer rather than just geometric intersection, and differs from [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by typically using a uniform accumulation strategy without hierarchical acceleration structures in its basic form.

## How it works
**Inputs and Outputs:**
- Input: A 3D volumetric dataset $V$ of dimensions $W \times H \times D$, a set of rays defined by camera parameters (origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel $(i, j)$), and a transfer function or weighting scheme.
- Output: A 2D image buffer $I$ of dimensions $W_{img} \times H_{img}$.

**Data Structures:**
- The volume $V$ is stored as a 3D array. In implementation, it is often flattened to 1D for memory efficiency.
- The output buffer $I$ is a 2D array (or 3D if multi-channel) initialized to zero.
- Ray parameters are computed per pixel: step size $\Delta t$ and current position $\mathbf{p}$.

**Steps:**
1. **Initialization**: For each pixel $(i, j)$ in the output image, initialize the accumulator $C_{i,j} = 0$. Compute the ray origin $\mathbf{o}_{i,j}$ and direction $\mathbf{d}_{i,j}$. Determine the entry and exit points of the ray with the volume bounding box.
2. **Ray Traversal**: For each step $k$ along the ray:
   - Compute the current 3D coordinate $\mathbf{p}_k = \mathbf{o}_{i,j} + k \cdot \Delta t \cdot \mathbf{d}_{i,j}$.
   - Check if $\mathbf{p}_k$ is within the volume bounds. If not, terminate the ray for this pixel.
   - Sample the volume value $v$ at $\mathbf{p}_k$. This may involve trilinear interpolation if $\mathbf{p}_k$ is not aligned with voxel centers.
   - Compute the weight $w$ for this sample based on the transfer function or visibility criteria.
   - Accumulate the contribution: $C_{i,j} \leftarrow C_{i,j} + w \cdot v$.
3. **Finalization**: After traversing the entire ray, store $C_{i,j}$ in the output image buffer $I_{i,j}$.

**Formulas:**
The accumulation for a pixel $(i, j)$ is given by:
$$ I_{i,j} = \sum_{k=0}^{N-1} w_k \cdot V(\mathbf{p}_k) $$
where:
- $N$ is the number of steps along the ray.
- $w_k$ is the weight at step $k$, derived from opacity or visibility thresholds.
- $V(\mathbf{p}_k)$ is the interpolated volume value at position $\mathbf{p}_k$.

**Parameters:**
- $\Delta t$: Step size, typically set to the voxel size or a fraction thereof for accuracy.
- $\tau$: Visibility threshold, used in advanced variants to filter contributions.

**Edge Cases and Pitfalls:**
- **Precision**: Floating-point precision errors can accumulate in the accumulator. Use double precision for accumulation if necessary.
- **Boundaries**: Rays may miss the volume entirely or only graze it. Proper bounding box checks are required.
- **Degenerate Input**: Empty volumes or volumes with no significant values should be handled to avoid unnecessary computation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_projection(volume, camera_params, image_width, image_height):
    """
    Perform backward projection to render a volume to an image.
    
    Args:
        volume: 3D numpy array of shape (D, H, W) representing the volumetric data.
        camera_params: Object containing ray origins and directions for each pixel.
        image_width: Width of the output image.
        image_height: Height of the output image.
        
    Returns:
        image: 2D numpy array of shape (image_height, image_width) representing the rendered image.
    """
    # Initialize output image buffer
    image = np.zeros((image_height, image_width), dtype=np.float32)
    
    # Get volume dimensions
    D, H, W = volume.shape
    
    # Iterate over each pixel in the output image
    for j in range(image_height):
        for i in range(image_width):
            # Get ray parameters for this pixel
            ray_origin = camera_params.origins[j, i]
            ray_direction = camera_params.directions[j, i]
            
            # Compute ray entry and exit points with volume bounding box
            # Assuming volume is centered at origin with size 1x1x1 for simplicity
            # In practice, this involves solving for t_min and t_max
            t_min, t_max = compute_ray_box_intersection(ray_origin, ray_direction, (0,0,0), (1,1,1))
            
            # If ray does not intersect the volume, skip
            if t_min > t_max:
                continue
            
            # Determine step size (e.g., voxel size)
            step_size = 1.0 / max(D, H, W)
            num_steps = int((t_max - t_min) / step_size)
            
            # Accumulate contributions along the ray
            accumulator = 0.0
            for k in range(num_steps):
                t = t_min + k * step_size
                # Compute current 3D position
                pos = ray_origin + t * ray_direction
                
                # Check if position is within volume bounds
                if not is_inside_volume(pos, (0,0,0), (1,1,1)):
                    break
                
                # Sample volume value at current position
                # Using trilinear interpolation for smoothness
                v = trilinear_sample(volume, pos)
                
                # Compute weight (e.g., based on opacity or visibility)
                # For basic backward projection, weight might be 1 or based on transfer function
                w = compute_weight(v)
                
                # Accumulate contribution
                accumulator += w * v
            
            # Store result in image buffer
            image[j, i] = accumulator
            
    return image

def compute_ray_box_intersection(origin, direction, box_min, box_max):
    """
    Compute the intersection of a ray with an axis-aligned bounding box.
    
    Args:
        origin: Ray origin (x, y, z).
        direction: Ray direction (x, y, z).
        box_min: Minimum corner of the box (x, y, z).
        box_max: Maximum corner of the box (x, y, z).
        
    Returns:
        t_min, t_max: Entry and exit distances along the ray.
    """
    t_min = -np.inf
    t_max = np.inf
    
    for i in range(3):
        if direction[i] == 0:
            if origin[i] < box_min[i] or origin[i] > box_max[i]:
                return np.inf, -np.inf  # Ray is parallel and outside box
        else:
            t1 = (box_min[i] - origin[i]) / direction[i]
            t2 = (box_max[i] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
    return t_min, t_max

def is_inside_volume(pos, box_min, box_max):
    """
    Check if a point is inside the volume bounding box.
    
    Args:
        pos: Point coordinates (x, y, z).
        box_min: Minimum corner of the box (x, y, z).
        box_max: Maximum corner of the box (x, y, z).
        
    Returns:
        True if inside, False otherwise.
    """
    return (box_min[0] <= pos[0] <= box_max[0] and
            box_min[1] <= pos[1] <= box_max[1] and
            box_min[2] <= pos[2] <= box_max[2])

def trilinear_sample(volume, pos):
    """
    Perform trilinear interpolation on the volume at the given position.
    
    Args:
        volume: 3D numpy array of shape (D, H, W).
        pos: Position (x, y, z) in normalized coordinates [0, 1].
        
    Returns:
        Interpolated value.
    """
    # Convert normalized position to voxel indices
    D, H, W = volume.shape
    x = pos[0] * (W - 1)
    y = pos[1] * (H - 1)
    z = pos[2] * (D - 1)
    
    # Get integer and fractional parts
    x0, y0, z0 = int(x), int(y), int(z)
    dx, dy, dz = x - x0, y - y0, z - z0
    
    # Handle boundary conditions
    x0 = max(0, min(x0, W - 2))
    y0 = max(0, min(y0, H - 2))
    z0 = max(0, min(z0, D - 2))
    
    # Get 8 neighboring values
    v000 = volume[z0, y0, x0]
    v100 = volume[z0, y0, x0 + 1]
    v010 = volume[z0, y0 + 1, x0]
    v110 = volume[z0, y0 + 1, x0 + 1]
    v001 = volume[z0 + 1, y0, x0]
    v101 = volume[z0 + 1, y0, x0 + 1]
    v011 = volume[z0 + 1, y0 + 1, x0]
    v111 = volume[z0 + 1, y0 + 1, x0 + 1]
    
    # Interpolate along x
    v00 = v000 * (1 - dx) + v100 * dx
    v01 = v010 * (1 - dx) + v110 * dx
    v10 = v001 * (1 - dx) + v101 * dx
    v11 = v011 * (1 - dx) + v111 * dx
    
    # Interpolate along y
    v0 = v00 * (1 - dy) + v01 * dy
    v1 = v10 * (1 - dy) + v11 * dy
    
    # Interpolate along z
    return v0 * (1 - dz) + v1 * dz

def compute_weight(v):
    """
    Compute weight for a volume value.
    
    Args:
        v: Volume value.
        
    Returns:
        Weight.
    """
    # Basic weight function (e.g., identity or transfer function)
    return v
```

## Subtopics

### [GPU Backprojection](5843-gpu-backprojection.md)
GPU Backprojection is a rendering pass that visualizes detected scale-space blobs by generating small slices for each blob, computing weighted contributions, and blending them to form the final image, followed by normalization. It decouples the expensive scale-space computation from the frequent backprojection, allowing interactive adjustment of smoothness and visibility parameters.

### Visibility Thresholding for Scale Backprojection
Visibility Thresholding for Scale Backprojection is a method to prevent scale information from being backprojected to voxels that do not correspond to the target feature (e.g., vessels), improving classification accuracy by using a visibility parameter [S2]. A voxel is determined to be not visible from the Laplacian of Gaussian (LoG) extrema if, at some point along the ray connecting the two positions, the intensity is below the threshold $\tau$ [S2]. Alternatively, thresholding can be based on gradient magnitude [S2]. The parameter $\tau$ (tau) is a visibility parameter used to control the sensitivity of the visibility test, and can be based on scalar value or gradient magnitude [S2]. This is a fast approximation; more accurate results can be obtained with a region growing approach added as a post-process based on the initial size-based classification [S2].

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels in the output image and $M$ is the average number of steps per ray.
- **Memory Complexity**: $O(V + I)$, where $V$ is the size of the volume data and $I$ is the size of the output image.
- **Measured Performance**: Performance depends heavily on the volume resolution and image size. High-resolution volumes and images require more computation.
- **Scaling**: Scales linearly with the number of pixels and steps. Parallelization on GPUs can significantly improve performance.
- **Strengths**: Simple to implement, flexible with transfer functions, suitable for real-time visualization with optimizations.
- **Limitations**: Computationally expensive for high-resolution volumes, sensitive to noise without preprocessing.
- **When to Use**: Suitable for interactive visualization of volumetric data, especially when combined with scale-space analysis.
- **When Not to Use**: Not ideal for very large volumes where hierarchical methods might be more efficient.

## Background
Backward Projection is a fundamental technique in volume rendering, originating from the need to visualize 3D volumetric data on 2D displays. It has been widely used in medical imaging, scientific visualization, and computer graphics. The method is closely related to ray casting and ray marching techniques. The specific variant involving scale-space analysis and visibility thresholding was introduced in the context of size-based transfer functions for volume exploration [S2].

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S2] PDF Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
