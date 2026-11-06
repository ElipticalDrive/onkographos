# Automatic Size and Variance Determination for Surface Splatting
Also called: Optimal Splat Size and Variance

## Overview
This method improves the visual quality of surface splatting on close-up views by automatically computing the radius ($r_i$) and variance ($v_i$) for each splat to ensure hole-free rendering and smooth blending, comparable to polygon rendering [S1]. Unlike methods that use global or fixed local variance and size, this approach calculates these parameters per-splat based on local point density, balancing blurriness and aliasing without manual user adjustment [S1]. It builds on [Splat Radius Adjustment Based on Normal Angle](1264-splat-radius-adjustment-based-on-normal-angle.md) by providing the specific automatic formulas for size and variance that prevent the discontinuities and artifacts associated with improper parameter choices [S1].

## How it works
### Inputs and Outputs
**Input:** A surface $S$ defined by a set of points $P = \{p_i\}$ satisfying the sampling criterion from [Amenta et al. 1998] [S1]. For each point $p_i$, the center $c_i$ is set to $p_i$, and the normal $n_i$ is assumed to be pre-computed [S1].
**Output:** A set of circular Gaussian splats $s_i = (c_i, n_i, r_i, v_i)$, where $r_i$ is the radius (size) and $v_i$ is the variance, along with color attributes [S1].

### Data Structures
Each splat requires 11 floating-point values packed into an OpenGL Vertex Buffer Object (VBO) to avoid data transfer overhead [S1]:
1.  Center $c_i$: 3 floats (x, y, z)
2.  Normal $n_i$: 3 floats (x, y, z)
3.  Color: 3 floats (R, G, B)
4.  Size $r_i$: 1 float
5.  Variance $v_i$: 1 float

### Step 1: Determine Splat Size ($r_i$)
For each splat $s_i$, identify the set of $k$-nearest neighboring splats $N_k(s_i)$ based on Euclidean distance $d(s_i, s_j) = \|c_j - c_i\|$ [S1]. The radius $r_i$ is determined by the distance to the $k$-th nearest neighbor to ensure sufficient overlap for hole-free rendering [S1].

The specific equation for $r_i$ is not fully visible in the source snippet, but it is derived from the distance to the $k$-th nearest neighbor in eye coordinates [S1]. Let $c$ be the splat's center and $n$ be the normal vector in eye coordinates. Let $q_n$ be the corresponding 3D point on the near plane that intersects the ray from the pixel [S1]. The distance calculation involves these eye-space coordinates [S1].

### Step 2: Determine Splat Variance ($v_i$)
The variance $v_i$ is calculated such that the Gaussian function value at the boundary (distance $r_i$) is negligible, ensuring smooth blending [S1].

The 1D Gaussian function $G_i(r)$ for splat $s_i$ with respect to distance $r$ from center $c_i$ is defined as:
$$ G_i(r) = A \cdot \exp\left(-\frac{r^2}{2v_i}\right) $$
where $A$ is the amplitude [S1].

To find $v_i$, the method sets the Gaussian value at the boundary $r_i$ to a threshold (effectively zero or negligible). This leads to the relationship:
$$ v_i = \frac{r_i^2}{2 \ln(A / \text{threshold})} $$
The text states this simplifies to an expression where $v_i$ depends only on the splat's size $r_i$ [S1]. The specific simplified equation is referenced as equation (5) in the source, involving a constant $k$ [S1]. In implementation, a variable constant $k_c$ may be used to adjust the variance, varying around the base constant $k$ by offsets (e.g., -3, 0, +3) [S1].

### Step 3: Visibility Pass (Geometry Pass 1)
1.  Send one vertex per splat.
2.  Transform the center $c_i$ to window coordinates and the normal $n_i$ to eye-space [S1].
3.  Compute the image-space square size (projected size) using perspective foreshortening. The formula uses the near ($n$), top ($t$), and bottom ($b$) of the view frustum, viewport height ($h_{vp}$), and the distance between the camera and the splat center ($z_{eye}$) [S1].
4.  Discard back-facing splats [S1].
5.  In the fragment shader, perform the On Splat Test (OST). Cast a ray from the pixel location towards the splat and find the intersection point $q$ between the ray and the splat using the equation $(q - c) \cdot n = 0$ [S1].
6.  Determine if the pixel is within the screen-space projection of the splat. The OST condition for a circular splat is checked using the computed intersection [S1].
7.  For pixels passing OST, correct the depth value $z_{vp}$ to prevent incorrect blending of overlapping splats, following the method in [Botsch et al. 2004] [S1].
8.  Compute a perspective-correct depth value and store it in the depth image [S1].

### Step 4: Attribute Pass
1.  Disable depth writing.
2.  Enable alpha blending with `glBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE, GL_ONE, GL_ONE)` [S1].
3.  Send all splat attributes (center, color, normal, size, variance) to the pipeline [S1].
4.  In the fragment shader, perform OST on the generated fragment. Discard back-facing splats.
5.  Compute weighted normal, color, and position $q$ for fragments passing OST.
6.  Store these weighted attributes in separate color buffers in an OpenGL Frame Buffer Object (FBO) [S1].

### Step 5: Shading Pass
1.  Render a window-size rectangle.
2.  Enable depth writing and disable alpha blending.
3.  Enable multitexturing to access FBO color buffers.
4.  In the fragment shader, normalize the weighted normal, color, and position from the color buffers.
5.  Perform per-pixel shading [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
# k: number of nearest neighbors for size determination
# k_c: constant for variance calculation (typically close to k)
# A: Gaussian amplitude (typically 1.0)
# threshold: Gaussian value at boundary (e.g., 0.0 or small epsilon)

def compute_splat_attributes(points, normals):
    """
    Computes radius and variance for each splat.
    points: List of 3D points P = {p_i}
    normals: List of 3D normals N = {n_i}
    Returns: List of splat attributes (center, normal, radius, variance, color)
    """
    splats = []
    n = len(points)
    
    # Precompute distances to find k-nearest neighbors
    # This is a placeholder for the actual k-NN search algorithm
    # which is not detailed in the source snippet but is required.
    # [G] Implementation note: Use a spatial data structure (e.g., KD-tree) 
    # for efficient k-NN search.
    k_nn_distances = find_k_nearest_neighbor_distances(points, k)

    for i in range(n):
        center = points[i]
        normal = normals[i]
        
        # Step 1: Determine Radius
        # r_i is based on the distance to the k-th nearest neighbor.
        # The exact formula involves eye-space coordinates and near plane intersection.
        # [G] Implementation note: Transform center and normal to eye space.
        # Compute distance to k-th neighbor in eye space.
        # The source implies r_i is set using this distance.
        # Let d_k be the distance to the k-th nearest neighbor in eye space.
        # r_i = d_k * scaling_factor # [G] Scaling factor depends on the specific 
        # geometric derivation in Eq (1) which is not fully visible.
        # However, the core concept is r_i ~ distance to k-th neighbor.
        radius = k_nn_distances[i] 
        
        # Step 2: Determine Variance
        # v_i = r_i^2 / (2 * ln(A / threshold))
        # The source mentions a constant k in the simplified equation.
        # Let's assume the simplified form is v_i = r_i^2 / (2 * k_c)
        # where k_c is a constant related to the threshold.
        variance = (radius ** 2) / (2.0 * k_c)
        
        # Color is assumed to be associated with the point
        color = get_point_color(points[i])
        
        splats.append({
            'center': center,
            'normal': normal,
            'radius': radius,
            'variance': variance,
            'color': color
        })
        
    return splats

def render_splats(splats, camera):
    """
    Renders splats using the 3-pass approach.
    """
    # Pass 1: Visibility Pass
    depth_image = initialize_depth_image()
    visible_splats = []
    
    for splat in splats:
        # Transform to eye space
        center_eye = transform_to_eye_space(splat['center'], camera)
        normal_eye = transform_to_eye_space(splat['normal'], camera)
        
        # Discard back-facing
        if dot(normal_eye, -center_eye) < 0:
            continue
            
        # Compute projected size
        z_eye = length(center_eye)
        projected_size = compute_projected_size(splat['radius'], z_eye, camera)
        
        # Set OpenGL point size to projected_size
        # Perform OST in fragment shader
        # [G] The OST logic is implemented in the fragment shader.
        # It checks if the pixel is within the splat's projection.
        # If passed, compute perspective-correct depth and store in depth_image.
        if passes_ost(center_eye, normal_eye, projected_size, camera):
            visible_splats.append(splat)
            
    # Pass 2: Attribute Pass
    fbo_color_buffers = initialize_fbo_buffers()
    enable_alpha_blending(GL_SRC_ALPHA, GL_ONE, GL_ONE, GL_ONE)
    disable_depth_write()
    
    for splat in visible_splats:
        # Send all attributes
        # In fragment shader, compute weighted normal, color, position
        # Store in FBO color buffers
        pass
        
    # Pass 3: Shading Pass
    enable_depth_write()
    disable_alpha_blending()
    render_shading_pass(fbo_color_buffers)
```

### Parameters
-   $k$: The number of nearest neighbors used to determine splat size. The text mentions 'k-th nearest neighboring splat' but does not specify a typical value in this snippet [S1].
-   $r_i$: The optimal base radius for the splat size, determined by the method's automatic calculation (referenced as equation (1) in the text) [S1].
-   $k_c$: A variable constant used to adjust the variance value. In the experimental configurations, $k_c$ varies around $k$ by offsets of -3, 0, or +3 [S1].

### Edge Cases and Pitfalls
-   **Variance too large:** Exhibits noticeable splat discontinuity [S1].
-   **Size or variance too small:** Causes aliasing effects [S1].
-   **Small overlapping area:** Results in less blending, leading to disturbing visual artifacts on close-up views [S1].
-   **Size < 75% of $r_i$:** Model exhibits disturbing aliasing effects [S1].
-   **Size > 125% of $r_i$:** Model shows a high level of blurriness [S1].
-   **Variance > optimal:** Model displays noticeable discontinuities among overlapping splats [S1].
-   **Variance < optimal:** Model causes aliasing effects [S1].
-   **Sharp features:** The optimal splat size does not augur well with sharp features on the surface. This can be improved by clipping splats near corners and edges [S1].

## Operating characteristics
-   **Time Complexity:** The method requires finding k-nearest neighbors for each splat, which is typically $O(N \log N)$ or $O(N^2)$ depending on the implementation, plus the rendering passes. The rendering itself is hardware-accelerated.
-   **Memory Complexity:** Each splat requires 11 floating-point values. The FBO stores weighted attributes for each pixel.
-   **Measured Performance:** On Dragon and Hand models, surface splatting with this method took less than 0.3 milliseconds per frame, while polygon rendering took more than 20 milliseconds [S1].
-   **Strengths:** Improves rendering quality on close-up views to be comparable to polygon rendering; automates parameter determination; significantly faster than polygon rendering for highly detailed surfaces [S1].
-   **Limitations:** Not theoretically proven to be optimal for all models; struggles with sharp features without additional clipping; Gaussian blending can result in noticeable blurriness if parameters are not optimal [S1].
-   **When to use:** When high-quality close-up views of highly detailed surfaces are required, and performance is critical.
-   **When not to use:** When sharp features are prominent and clipping is not applied; when polygon rendering performance is acceptable and quality is paramount for non-close-up views.

## Background
The method is presented in the paper 'Automatic size and variance determination for better surface splatting quality on close-up view' [S1]. It builds on foundational work by Zwicker et al. (2001) on Surface Splatting and Pfister et al. (2000) on Surfel rendering [S1]. Previous works by Zwicker et al. (2001) and Ren et al. (2002) used global or local variance and fixed size, which could exhibit discontinuities on irregularly sampled models [S1]. Pajarola et al. (2004) required user adjustment of blending kernel parameters [S1]. Botsch et al. (2004/2005) applied per-pixel Phong shading, which is unsuitable for non-shiny materials and depends heavily on variance for normal interpolation [S1]. This method automates the determination of size and variance to address these issues [S1]. It has been tested on 3D models including Bunny, Dragon, and Hand, sourced from the Stanford 3D Scanning Repository, Digital Michelangelo projects, and Cyberware [S1].

## Sources
[S1] Automatic size and variance determination for better surface splatting quality on close-up view, https://doi.org/10.1145/2087756.2087792, fetched 2026-11-05
