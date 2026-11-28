# Improved Ray Casting Volume Rendering

Also called: Improved light projected volume rendering, Light projected volume rendering method

## Overview
Improved Ray Casting Volume Rendering is a direct volume rendering technique optimized for large-scale, regular 3D spatial data fields (specifically seismic data) by prioritizing rendering speed and high-frequency component preservation over physical lighting accuracy [S1]. It distinguishes itself from standard ray casting by reversing the classification and interpolation order (interpolating raw data before classification) and replacing floating-point arithmetic with fixed-point operations to accelerate trilinear interpolation [S1]. Unlike [Standard Ray Casting](12643-standard-ray-casting.md), which classifies data first and risks filtering high-frequency color components, this method ensures the convolution operation acts on the original discrete data field [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A regular 3D scalar data field $V$ (e.g., seismic data in SEGY format), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), a transfer function $T$ mapping scalar values to color $\mathbf{c}$ and opacity $\alpha$, and an empty voxel marker value [S1].
*   **Output:** A 2D image where each pixel contains a synthesized RGBA color value [S1].

**Data Structures:**
*   **Volume Data:** A 3D array of scalar values stored using fixed-point number representation for interpolation calculations to avoid time-consuming floating-point operations [S1].
*   **Sampling Points:** $K$ equal-interval points along the ray [S1].

**Parameters:**
*   $K$: The number of equal-interval sampling points selected along the ray for each pixel [S1]. Not given in the sources.

**Algorithm Steps:**
1.  **Ray Definition:** For each pixel, define a ray $r(t) = \mathbf{o} + t\mathbf{d}$ intersecting the volume bounding box [S1].
2.  **Sampling:** Select $K$ equal-interval sampling points along the ray segment within the volume [S1].
3.  **Interpolation (First):** For each sampling point $\mathbf{p}_i$, perform trilinear interpolation on the original discrete 3D scalar data field $V$ using the eight closest grid points [S1]. This yields an interpolated scalar value $V_{interp}$ [S1].
    *   *Note:* This step avoids the high-frequency component loss associated with classifying discrete data before interpolation [S1].
    *   *Implementation Detail:* Use fixed-point arithmetic for the interpolation weights and value accumulation to improve performance [S1].
4.  **Classification (Second):** Apply the transfer function $T$ to the interpolated scalar value $V_{interp}$ to obtain the color $\mathbf{c}_i$ and opacity $\alpha_i$ for the sample [S1].
    *   $\mathbf{c}_i, \alpha_i = T(V_{interp})$ [S1].
5.  **Shading (Ignored):** Explicitly ignore lighting effects and gradient calculations. The source states that for seismic data, gradient values change disorderly and lighting effects are weak, so this step is skipped to minimize computation [S1].
6.  **Compositing:** Synthesize the color and opacity values from front to rear using the front-to-back compositing equation [S1].
    *   Initialize accumulated color $\mathbf{C}_{acc} = (0,0,0)$ and accumulated opacity $\alpha_{acc} = 0$ [S1].
    *   For each sample $i$ from front to back:
        $$ \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{c}_i \cdot \alpha_i $$
        $$ \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_i $$
7.  **Early Termination:** If $\alpha_{acc}$ approaches 1 (indicating the pixel is effectively opaque), skip the remaining voxels/samples for that ray to eliminate invalid computation [S1].
8.  **Empty Voxel Skipping:** Skip sampling points that fall within empty voxels (voxels with no data contribution) to accelerate rendering speed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def improved_ray_casting(volume_data, volume_bounds, camera, transfer_function, K, empty_value):
    """
    volume_data: 3D list/array of scalar values (fixed-point representation preferred)
    volume_bounds: tuple (min_x, max_x, min_y, max_y, min_z, max_z)
    camera: object with origin o, direction d, resolution, and get_ray(x, y) method [G]
    transfer_function: callable scalar -> (color_tuple, opacity_float)
    K: int, number of sampling points per ray
    empty_value: scalar value representing empty space in volume_data
    """
    height, width = camera.resolution
    # Initialize output image with RGBA tuples
    output = [[(0.0, 0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]

    def intersect_volume(ray_o, ray_d, bounds):
        # Slab method to find entry (t_near) and exit (t_far) parameters
        t_min, t_max = 0.0, float('inf')
        for i in range(3):
            if ray_d[i] == 0.0:
                if ray_o[i] < bounds[2*i] or ray_o[i] > bounds[2*i+1]:
                    return None, None
            else:
                t1 = (bounds[2*i] - ray_o[i]) / ray_d[i]
                t2 = (bounds[2*i+1] - ray_o[i]) / ray_d[i]
                t_min = max(t_min, min(t1, t2))
                t_max = min(t_max, max(t1, t2))
        if t_min > t_max:
            return None, None
        return t_min, t_max

    def is_empty_voxel(p, bounds, data, empty_val):
        # Check if point is within volume bounds
        if not (bounds[0] <= p[0] <= bounds[1] and bounds[2] <= p[1] <= bounds[3] and bounds[4] <= p[2] <= bounds[5]):
            return False
        # Map continuous coordinate to integer grid indices
        ix, iy, iz = int(p[0]), int(p[1]), int(p[2])
        # Clamp to valid array bounds
        ix = max(0, min(ix, len(data)-1))
        iy = max(0, min(iy, len(data[0])-1))
        iz = max(0, min(iz, len(data[0][0])-1))
        return data[ix][iy][iz] == empty_val

    for y in range(height):
        for x in range(width):
            ray_o, ray_d = camera.get_ray(x, y)
            t_near, t_far = intersect_volume(ray_o, ray_d, volume_bounds)
            if t_near is None:
                continue

            dt = (t_far - t_near) / K
            acc_color = (0.0, 0.0, 0.0)
            acc_opacity = 0.0

            for k in range(K):
                t = t_near + k * dt
                p = (ray_o[0] + t * ray_d[0], ray_o[1] + t * ray_d[1], ray_o[2] + t * ray_d[2])

                # Safety bounds check
                if not (volume_bounds[0] <= p[0] <= volume_bounds[1] and
                        volume_bounds[2] <= p[1] <= volume_bounds[3] and
                        volume_bounds[4] <= p[2] <= volume_bounds[5]):
                    continue

                # Skip empty voxels
                if is_empty_voxel(p, volume_bounds, volume_data, empty_value):
                    continue

                # Step 3: Interpolate original data first (fixed-point arithmetic)
                v_interp = trilinear_interpolate_fixed_point(p, volume_data)

                # Step 4: Classify after interpolation
                color, opacity = transfer_function(v_interp)

                # Step 7: Early termination check
                if acc_opacity >= 0.99:
                    break

                # Step 6: Front-to-back compositing
                weight = (1.0 - acc_opacity) * opacity
                acc_color = (acc_color[0] + weight * color[0],
                             acc_color[1] + weight * color[1],
                             acc_color[2] + weight * color[2])
                acc_opacity = acc_opacity + weight

            output[y][x] = (acc_color[0], acc_color[1], acc_color[2], acc_opacity)

    return output

def trilinear_interpolate_fixed_point(p, volume_data):
    """
    Performs trilinear interpolation using fixed-point arithmetic logic.
    p: 3D coordinate (x, y, z)
    volume_data: 3D array of scalars
    """
    ix, iy, iz = int(p[0]), int(p[1]), int(p[2])
    fx, fy, fz = p[0] - ix, p[1] - iy, p[2] - iz

    # Boundary clamping for indices
    ix = max(0, min(ix, len(volume_data)-1))
    iy = max(0, min(iy, len(volume_data[0])-1))
    iz = max(0, min(iz, len(volume_data[0][0])-1))

    # Fetch 8 surrounding voxels
    v000 = volume_data[ix][iy][iz]
    v100 = volume_data[min(ix+1, len(volume_data)-1)][iy][iz]
    v010 = volume_data[ix][min(iy+1, len(volume_data[0])-1)][iz]
    v110 = volume_data[min(ix+1, len(volume_data)-1)][min(iy+1, len(volume_data[0])-1)][iz]
    v001 = volume_data[ix][iy][min(iz+1, len(volume_data[0][0])-1)]
    v101 = volume_data[min(ix+1, len(volume_data)-1)][iy][min(iz+1, len(volume_data[0][0])-1)]
    v011 = volume_data[ix][min(iy+1, len(volume_data[0])-1)][min(iz+1, len(volume_data[0][0])-1)]
    v111 = volume_data[min(ix+1, len(volume_data)-1)][min(iy+1, len(volume_data[0])-1)][min(iz+1, len(volume_data[0][0])-1)]

    # Interpolation using fixed-point math (logical structure)
    # In practice, weights and values are scaled integers to avoid floating-point arithmetic [S1]
    w000 = (1-fx)*(1-fy)*(1-fz)
    w100 = fx*(1-fy)*(1-fz)
    w010 = (1-fx)*fy*(1-fz)
    w110 = fx*fy*(1-fz)
    w001 = (1-fx)*(1-fy)*fz
    w101 = fx*(1-fy)*fz
    w011 = (1-fx)*fy*fz
    w111 = fx*fy*fz

    return (w000*v000 + w100*v100 + w010*v010 + w110*v110 + 
            w001*v001 + w101*v101 + w011*v011 + w111*v111)
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$ per pixel, where $N$ is the number of pixels and $K$ is the number of samples. The use of fixed-point arithmetic reduces the constant factor for interpolation compared to floating-point [S1].
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the number of voxels [S1].
*   **Strengths:** Significantly faster rendering speed for large-scale seismic data due to skipped lighting calculations, fixed-point interpolation, and early termination [S1]. Preserves high-frequency color components better than standard ray casting by interpolating before classification [S1].
*   **Limitations:** Not physically accurate due to ignored lighting/shading [S1].
*   **When to use:** Ideal for real-time or near-real-time visualization of large-scale, regular 3D scalar fields (like seismic data) where interpretive clarity and speed are prioritized over photorealistic lighting [S1].
*   **When not to use:** Not suitable for scenes requiring accurate surface shading, specular highlights, or where gradient-based normal estimation is critical for visual understanding [S1].

## Background
The method was introduced in the context of 3D visualization for seismic data processing [S1]. It addresses the inherent defects of standard ray casting, specifically its heavy calculation overhead and time/space complexity which increases with data volume [S1]. Seismic data is characterized as large-scale, distributed, scattered material with different eigenvalues, no clear interfaces, and disorderly gradient directions, making traditional lighting models ineffective [S1]. The method was implemented using the Visualization Toolkit (VTK) and MFC under Microsoft Windows to display SEGY format data [S1].

## Sources
[S1] Research on 3D Visualization Method of Seismic Data, https://doi.org/10.14257/ijsip.2016.9.5.39, fetched 2026-10-26
