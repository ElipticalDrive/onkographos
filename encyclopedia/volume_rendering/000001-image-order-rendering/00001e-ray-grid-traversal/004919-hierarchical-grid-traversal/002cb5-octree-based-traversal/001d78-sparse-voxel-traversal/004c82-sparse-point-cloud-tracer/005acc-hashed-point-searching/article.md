# Hashed-point searching

Also called: HashPoint

## Overview
Hashed-point searching is an acceleration technique for ray-tracing-based point cloud rendering that transforms the 3D search space into a 2D image plane for hash table lookup [S1]. Unlike traditional rasterization methods that use a z-buffer to retain only the nearest point per pixel, this method projects all points onto the image plane and preserves all points falling within each pixel, storing them in a hash table to enable swift location of point clouds near a camera ray [S1]. It is distinguished by its use of an adaptive searching range determined by the distance between points and the viewpoint, rather than relying on a fixed radius or K-nearest neighbors [S1].

## How it works
**Inputs:**
*   Point cloud data: A list of $n$ 3D points $\mathbf{x}^{pc}$.
*   Camera parameters: Focal length $f$, ray origin $\mathbf{o}$ (optical center), and pixel dimensions in world coordinates $\Delta x, \Delta y$ derived from calibrated camera parameters [S1].
*   Rendering configuration: Near clipping distance $t_{near}$, far clipping distance $t_{far}$, and a base radius parameter corresponding to the minimum radius at $t_{near}$ [S1].

**Outputs:**
*   A hash table mapping pixel indices to lists of point indices.
*   For a given ray, a set of candidate sample points $\mathbf{x}^{sp}_j$ projected onto the ray, filtered by an adaptive radius [S1].

**Data Structures:**
1.  **Point List:** An array of size $n$ storing the original point cloud data. The order of points may be adjusted during construction [S1].
2.  **Hash Table:** A dictionary or array where:
    *   **Key:** The integer index of a pixel on the 2D image plane.
    *   **Value:** A tuple or struct containing:
        *   `start_index`: The position (index) of the first point in the Point List that falls on this pixel.
        *   `count`: The number of points that fall on this pixel [S1].

**Steps:**

1.  **Projection and Z-ordering:**
    Project all points from the point cloud onto the 2D image plane [S1]. Arrange the points falling on the same pixel in Z-order (Morton code) to ensure they are stored as closely as possible in sequence in the Point List [S1]. This ordering facilitates efficient retrieval [S1].

2.  **Atomic Reordering (CUDA):**
    Adjust the order of points in the Point List to reflect the Z-order arrangement. This positional adjustment is achieved using atomic operations in CUDA, with a time complexity of $O(1)$ per point adjustment [S1].

3.  **Hash Table Construction:**
    Iterate through the reordered Point List. For each point, determine its pixel index. Update the Hash Table for that pixel:
    *   If the pixel index is not present in the Hash Table, initialize it with `start_index` set to the current point's index in the Point List and `count` set to 1.
    *   If the pixel index exists, increment the `count` [S1].

4.  **Ray Traversal and Kernel Definition:**
    For each pixel in the target ray's search kernel:
    *   Calculate the radius of the disc representing the pixel:
        $$ \dot{r} = \frac{\Delta x \cdot \Delta y}{\pi} $$
        where $\Delta x$ and $\Delta y$ are the width and height of the pixel in world coordinates [S1].
    *   Define a larger disc radius $\ddot{r}$ for the searching kernel to mitigate holes caused by point cloud sparsity [S1].
    *   Calculate the magnitude of the searching kernel $s$:
        $$ s = 2 \cdot \dot{r}\ddot{r} + 1 $$
        This defines the spatial extent of the search around the central pixel [S1].

5.  **Adaptive Radius Calculation:**
    For a sample point candidate $\mathbf{x}^{sp}_j$ located at distance $t$ along the ray (where $\mathbf{x}^{sp}_j = \mathbf{o} + t\mathbf{d}$), compute the adaptive radius $r$:
    $$ r = \frac {\|\mathbf {x^{sp}_j}-\mathbf {o}\|_2 \cdot f \ddot {r}}{\|\mathbf {p_o} - \mathbf {o}\|_2 \cdot \sqrt {\left (\sqrt {\|\mathbf {p_o} - \mathbf {o}\|_2^2-f^2}- \ddot {r}\right )^2+f^2}} $$
    where:
    *   $f$ is the focal length [S1].
    *   $\mathbf{o}$ is the ray origin [S1].
    *   $\mathbf{p_o}$ is the pixel center [S1].
    *   $\mathbf{x}^{sp}_j$ is the sample point [S1].
    
    The minimum radius $r_{min}$ occurs at $t_{near}$ and matches the input hyperparameter radius [S1]. As $t$ increases, $r$ increases [S1].

6.  **Search and Filtering:**
    *   Traverse each pixel in the search kernel defined by $s$ [S1].
    *   Assess the number of associated points per pixel using the Hash Table (`count`) to determine if a search is needed [S1].
    *   If required, look up the point list for the pixel using the Hash Table (`start_index` and `count`) [S1].
    *   For each point in the list, calculate its distance to the ray. If the distance is within the adaptive radius $r$ calculated for the point's depth, include it as a candidate sample point $\mathbf{x}^{sp}_j$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def build_hash_table(points, camera_params, image_width, image_height):
    """
    Constructs the hash table for hashed-point searching.
    
    Args:
        points: List of 3D points (n, 3).
        camera_params: Dict containing 'f' (focal length), 'o' (ray origin),
                       'K' (intrinsic matrix or equivalent for projection),
                       'delta_x', 'delta_y' (pixel world dimensions).
        image_width: Width of the image plane in pixels.
        image_height: Height of the image plane in pixels.
        
    Returns:
        hash_table: Dict mapping pixel_index -> {'start_index': int, 'count': int}
        point_list: Reordered list of point indices.
    """
    n = len(points)
    # Step 1 & 2: Project and Z-order
    # Note: Z-ordering requires mapping 3D coordinates to 2D pixel indices
    # and then sorting points within each pixel by their Morton code.
    # This implementation assumes a helper function 'project_and_get_morton'
    # which returns (pixel_index, morton_code) for each point.
    
    point_morton_data = []
    for i, p in enumerate(points):
        pixel_idx, morton_code = project_and_get_morton(p, camera_params, image_width, image_height)
        point_morton_data.append((pixel_idx, morton_code, i))
        
    # Sort by pixel index, then by morton code
    point_morton_data.sort(key=lambda x: (x[0], x[1]))
    
    # Step 3: Build Hash Table
    hash_table = {}
    point_list = [] # Stores original indices in the new Z-ordered sequence
    
    current_pixel_idx = -1
    start_index = 0
    
    for idx, (pixel_idx, morton_code, original_idx) in enumerate(point_morton_data):
        if pixel_idx != current_pixel_idx:
            # New pixel
            if current_pixel_idx != -1:
                # Finalize previous pixel
                hash_table[current_pixel_idx]['count'] = idx - start_index
            
            current_pixel_idx = pixel_idx
            start_index = idx
            hash_table[current_pixel_idx] = {
                'start_index': idx,
                'count': 0 # Will be updated or calculated at end
            }
        
        point_list.append(original_idx)
        
    # Finalize last pixel
    if current_pixel_idx != -1:
        hash_table[current_pixel_idx]['count'] = n - start_index
        
    return hash_table, point_list

def search_points(ray_origin, ray_direction, pixel_center, t_near, t_far, 
                  hash_table, point_list, points, camera_params, search_kernel_radius):
    """
    Searches for points near a ray using the hashed-point structure.
    
    Args:
        ray_origin: 3D vector o.
        ray_direction: Normalized 3D vector d.
        pixel_center: 3D vector p_o (center of the primary pixel).
        t_near, t_far: Floats for clipping.
        hash_table: The constructed hash table.
        point_list: The reordered point indices.
        points: Original point cloud data.
        camera_params: Dict with 'f', 'delta_x', 'delta_y'.
        search_kernel_radius: The radius r_double_dot (r̈) for the kernel.
        
    Returns:
        candidates: List of sample point candidates (3D vectors).
    """
    f = camera_params['f']
    delta_x = camera_params['delta_x']
    delta_y = camera_params['delta_y']
    
    # Calculate pixel disc radius r_dot
    r_dot = (delta_x * delta_y) / math.pi
    
    # Calculate search kernel magnitude s (not strictly needed for lookup logic 
    # if we iterate pixels, but defines the scope)
    # s = 2 * r_dot * r_double_dot + 1
    
    candidates = []
    
    # Determine which pixels to check based on the search kernel
    # This requires projecting the search kernel onto the image plane
    # and iterating over those pixel indices.
    # For simplicity, we assume a helper 'get_kernel_pixels' returns 
    # a list of pixel indices within the search kernel.
    kernel_pixel_indices = get_kernel_pixels(pixel_center, ray_origin, ray_direction, 
                                             r_dot, search_kernel_radius, 
                                             camera_params, image_width, image_height)
    
    for pixel_idx in kernel_pixel_indices:
        if pixel_idx not in hash_table:
            continue
            
        info = hash_table[pixel_idx]
        start = info['start_index']
        count = info['count']
        
        # Iterate over points in this pixel's bucket
        for i in range(start, start + count):
            point_idx = point_list[i]
            point_pos = points[point_idx]
            
            # Project point onto ray to find t
            # x_sp_j = o + t * d
            # We need to check if the point is within the adaptive radius r
            # First, find the projection of the point onto the ray
            # t = dot(point - o, d)
            t = dot(point_pos - ray_origin, ray_direction)
            
            if t < t_near or t > t_far:
                continue
                
            # Calculate the closest point on the ray to the 3D point
            closest_point_on_ray = ray_origin + t * ray_direction
            distance_to_ray = norm(point_pos - closest_point_on_ray)
            
            # Calculate adaptive radius r for this t
            # r = ( ||x_sp_j - o|| * f * r_double_dot ) / 
            #     ( ||p_o - o|| * sqrt( (sqrt(||p_o - o||^2 - f^2) - r_double_dot)^2 + f^2 ) )
            
            # Note: x_sp_j in the formula refers to the sample point on the ray.
            # However, the formula in [S1] uses ||x_sp_j - o|| which is simply t.
            # Let's verify the formula components.
            # ||x_sp_j - o||_2 is the distance from origin to the sample point on the ray.
            # Since x_sp_j = o + t*d, ||x_sp_j - o|| = t.
            
            dist_o_to_point = t
            dist_o_to_pixel_center = norm(pixel_center - ray_origin)
            
            # Denominator term
            inner_sqrt_arg = (dist_o_to_pixel_center**2 - f**2)
            if inner_sqrt_arg < 0:
                inner_sqrt_arg = 0 # Guard against numerical error
            inner_sqrt = math.sqrt(inner_sqrt_arg)
            
            denominator_term = math.sqrt((inner_sqrt - search_kernel_radius)**2 + f**2)
            
            denominator = dist_o_to_pixel_center * denominator_term
            numerator = dist_o_point * f * search_kernel_radius
            
            if denominator == 0:
                continue
                
            r_adaptive = numerator / denominator
            
            if distance_to_ray <= r_adaptive:
                candidates.append(closest_point_on_ray)
                
    return candidates

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def norm(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def project_and_get_morton(point, camera_params, width, height):
    # Placeholder for actual projection and Morton code calculation
    # This function maps 3D point to 2D pixel index and computes Z-order
    pass

def get_kernel_pixels(pixel_center, ray_origin, ray_direction, r_dot, r_double_dot, 
                      camera_params, width, height):
    # Placeholder for determining which pixels fall within the search kernel cone
    pass
```

**Edge Cases and Pitfalls:**
*   **Sparsity Holes:** If the point cloud is sparse, a ray cone might contain no points, causing holes in rendering. The method mitigates this by using a larger disc radius $\ddot{r}$ for the searching kernel, expanding the search area beyond the immediate pixel [S1].
*   **Precision:** The adaptive radius calculation involves square roots and divisions. Numerical stability should be considered, particularly when $\|\mathbf{p_o} - \mathbf{o}\|$ is close to $f$ or when $t$ is very small [S1].
*   **Boundary Conditions:** Points exactly on the boundary of the image plane or at $t_{near}/t_{far}$ must be handled consistently with the projection and clipping logic [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Construction: $O(n)$ for $n$ points, with $O(1)$ per point adjustment during reordering [S1].
    *   Query: $O(1)$ per pixel lookup. Total search complexity is $O(n + mq)$, where $m$ is the number of rays and $q$ is the average number of points falling in the radius of each ray [S1].
*   **Memory Complexity:** $O(n)$ to store the point list and hash table entries [S1].
*   **Strengths:**
    *   Shifts the search paradigm from "ray seeking point" to "point seeking ray," which is efficient for sparse point clouds [S1].
    *   Adaptive radius reduces noise from irrelevant points and avoids the fixed radius/K-NN limitations [S1].
    *   High efficiency due to hash table lookup and atomic operations [S1].
*   **Limitations:**
    *   Requires CUDA for efficient atomic reordering (as per the source implementation) [S1].
    *   Performance depends on the distribution of points; highly clustered points may lead to hash collisions or large buckets, though the hash table structure handles this via linked lists or arrays [S1].
    *   The method is designed for neural rendering contexts where feature aggregation is performed on the primary surface [S1].

## Background
Hashed-point searching, also known as HashPoint, was introduced in the paper "HashPoint: Accelerated Point Searching and Sampling for Neural Rendering" [S1]. The method was developed to address the inefficiencies of traditional 3D point cloud searching in neural rendering pipelines. It draws inspiration from Morton code (Z-order) for spatial indexing and volume rendering techniques for sample point selection [S1]. The approach is particularly relevant for novel view synthesis and 3D Gaussian Splatting variants that rely on sparse point cloud representations [S1].

## Sources
[S1] HashPoint: Accelerated Point Searching and Sampling for Neural Rendering, https://doi.org/10.1109/cvpr52733.2024.00427, fetched 2026-10-27
