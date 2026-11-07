# Automated Classification Widget Placement

Also called: Automated Widget Placement

## Overview
Automated Classification Widget Placement is an interaction technique that automatically positions and sizes classification widgets within a multi-dimensional transfer function (TF) domain based on feedback from user probes and the statistical distribution of re-projected voxels [S1, book][S2]. It solves the difficulty of manually configuring effective higher-dimensional transfer functions by tying widget geometry directly to data characteristics [S1, book][S2]. Unlike manual editing methods, it reduces user effort by deriving widget parameters from probe location and voxel variance [S1, book][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A data probe object (defining a query location in the TF domain), the current transfer function domain (e.g., 2D intensity-opacity or intensity-gradient space), and the volume dataset.
*   **Output:** Updated widget parameters (center, width, height) for rectangular or triangular classification widgets.

**Data Structures:**
*   **Data Probe:** An interactive handle in the TF domain. It has a "tip" position (query point) and a "slider" value (scaling factor).
*   **Re-projected Voxels:** A set of 2D points $(u_i, v_i)$ representing the transfer function domain coordinates of volume voxels that are spatially near the probe's current 3D position in the volume.

**Steps:**
1.  **Probe Interaction:** The user moves the data probe to a location in the TF domain. This defines the query point $\mathbf{p}_{probe} = (u_{probe}, v_{probe})$ and a scale factor $s$ from the probe's slider.
2.  **Voxel Selection:** Identify voxels in the volume dataset that are spatially proximate to the probe's 3D position.
3.  **Re-projection:** For each selected voxel, compute its value in the TF domain (e.g., intensity and gradient magnitude) to obtain a set of 2D points $\mathcal{V} = \{(u_i, v_i)\}$.
4.  **Statistical Analysis:** Calculate the mean and variance of the re-projected voxel distribution $\mathcal{V}$.
    *   Mean: $\mu_u = \frac{1}{N}\sum u_i$, $\mu_v = \frac{1}{N}\sum v_i$.
    *   Variance: $\sigma_u^2 = \frac{1}{N}\sum (u_i - \mu_u)^2$, $\sigma_v^2 = \frac{1}{N}\sum (v_i - \mu_v)^2$.
5.  **Widget Placement:**
    *   **Rectangular Widget:**
        *   Center: $(\mu_u, \mu_v)$.
        *   Size: Controlled by the probe's slider $s$. The width $W$ and height $H$ are proportional to $s$ and potentially the variances. The sources state the size is "controlled by the data probe's slider" [S1, book][S2]. A common implementation is $W = k \cdot s \cdot \sigma_u$ and $H = k \cdot s \cdot \sigma_v$ for some constant $k$, or simply $W, H \propto s$.
    *   **Triangular Widget:**
        *   Horizontal Location: $u_{center} = u_{probe}$ (the data value queried by the probe) [S1, book][S2].
        *   Width: Determined by the horizontal variance $\sigma_u$ of the re-projected voxels [S1, book][S2].
        *   Height: Determined by the vertical variance $\sigma_v$ of the re-projected voxels [S1, book][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def place_widgets(probe, volume, tf_domain):
    # probe: object with attributes probe_pos_3d, probe_tf_pos (u, v), slider_scale
    # volume: dataset with get_voxel_value(pos)
    # tf_domain: function to map voxel value to TF coordinates (u, v)
    
    # 1. Get spatially proximate voxels
    proximate_voxels = get_proximate_voxels(probe.probe_pos_3d, volume)
    
    # 2. Re-project voxels to TF domain
    reprojected_points = []
    for voxel in proximate_voxels:
        val = volume.get_voxel_value(voxel.pos)
        u, v = tf_domain(val)
        reprojected_points.append((u, v))
        
    if not reprojected_points:
        return None
        
    # 3. Calculate statistics
    N = len(reprojected_points)
    mean_u = sum(p[0] for p in reprojected_points) / N
    mean_v = sum(p[1] for p in reprojected_points) / N
    
    var_u = sum((p[0] - mean_u)**2 for p in reprojected_points) / N
    var_v = sum((p[1] - mean_v)**2 for p in reprojected_points) / N
    
    std_u = sqrt(var_u)
    std_v = sqrt(var_v)
    
    # 4. Place Rectangular Widget
    # Centered at mean of re-projected voxels
    rect_center = (mean_u, mean_v)
    # Size controlled by probe slider
    # Implementation detail: scale variances by slider
    rect_width = probe.slider_scale * std_u
    rect_height = probe.slider_scale * std_v
    
    # 5. Place Triangular Widget
    # Horizontal location at probe's TF query position
    tri_u_center = probe.probe_tf_pos[0]
    # Width and height determined by variances
    tri_width = std_u
    tri_height = std_v
    
    return {
        'rectangular': {'center': rect_center, 'width': rect_width, 'height': rect_height},
        'triangular': {'u_center': tri_u_center, 'width': tri_width, 'height': tri_height}
    }
```

**Edge Cases:**
*   **Empty Probe Region:** If no voxels are proximate to the probe, statistics cannot be computed. The widget placement should default to the probe's current position with minimal size [G].
*   **Zero Variance:** If all re-projected voxels have identical TF values, variance is zero. The widget collapses to a point. A minimum size threshold should be applied [G].

## Operating characteristics
*   **Complexity:** Time complexity is $O(N)$ where $N$ is the number of proximate voxels. Memory complexity is $O(N)$ to store re-projected points.
*   **Performance:** Fast for small probe neighborhoods. Performance degrades if the probe covers a large volume or if voxel selection is inefficient.
*   **Strengths:** Reduces manual effort; adapts to local data distribution; intuitive link between spatial location and TF domain [S1, book][S2].
*   **Limitations:** Relies on accurate spatial proximity; may produce small widgets if data is sparse; does not handle global TF structure [S1, book][S2].

## Background
Originated in the context of multi-dimensional transfer functions for volume rendering [S1, book][S2]. It addresses the usability challenges of high-dimensional TF editing by automating widget placement based on data feedback [S1, book][S2].

## Sources
[S1] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
