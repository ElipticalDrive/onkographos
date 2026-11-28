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
For each splat $s_i$, identify the set of $k$-nearest neighboring splats $N_k(s_i)$ based on Euclidean distance $d(s_i, s_j) = \|c_j - c_i\|$ [S1]. The radius $r_i$ is determined by the distance to the $k$-th nearest neighbor to ensure sufficient overlap for hole-free rendering [S1]. The specific equation for $r_i$ involves the splat's center $c$ and normal vector $n$ in eye coordinates, and the 3D point $q_n$ on the near plane that intersects the ray [S1].

### Step 2: Determine Splat Variance ($v_i$)
The variance $v_i$ is calculated such that the Gaussian function value at the boundary (distance $r_i$) is negligible, ensuring smooth blending [S1]. The 1D Gaussian function $G_i(r)$ for splat $s_i$ with respect to distance $r$ from center $c_i$ is defined as:
$$ G_i(r) = A \cdot \exp\left(-\frac{r^2}{2v_i}\right) $$
where $A$ is the amplitude [S1]. To find $v_i$, the method sets the Gaussian value at the boundary $r_i$ to a threshold (effectively zero or negligible) [S1]. This leads to the relationship:
$$ v_i = \frac{r_i^2}{2 \ln(A / \text{threshold})} $$
The source states this simplifies to an expression where $v_i$ depends only on the splat's size $r_i$ [S1]. The specific simplified equation is referenced as equation (5) in the source, involving a constant $k$ [S1]. In implementation, a variable constant $k_c$ may be used to adjust the variance, varying around the base constant $k$ by offsets (e.g., -3, 0, +3) [S1].

### Step 3: Visibility Pass (Geometry Pass 1)
1.  Send one vertex per splat [S1].
2.  Transform the center $c_i$ to window coordinates and the normal $n_i$ to eye-space [S1].
3.  Compute the image-space square size (projected size) using perspective foreshortening [S1]. The formula uses the near ($n$), top ($t$), and bottom ($b$) of the view frustum, viewport height ($h_{vp}$), and the distance between the camera and the splat center ($z_{eye}$) [S1].
4.  Discard back-facing splats [S1].
5.  In the fragment shader, perform the On Splat Test (OST) [S1]. Cast a ray from the pixel location towards the splat and find the intersection point $q$ between the ray and the splat using the equation $(q - c) \cdot n = 0$ [S1].
6.  Determine if the pixel is within the screen-space projection of the splat [S1]. The OST condition for a circular splat is checked using the computed intersection [S1].
7.  For pixels passing OST, correct the depth value $z_{vp}$ to prevent incorrect blending of overlapping splats, following the method in [Botsch et al. 2004] [S1].
8.  Compute a perspective-correct depth value and store it in the depth image [S1].

### Step 4: Attribute Pass
1.  Disable depth writing [S1].
2.  Enable alpha blending with `glBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE, GL_ONE, GL_ONE)` [S1].
3.  Send all splat attributes (center, color, normal, size, variance) to the pipeline [S1].
4.  In the fragment shader, perform OST on the generated fragment [S1]. Discard back-facing splats [S1].
5.  Compute weighted normal, color, and position $q$ for fragments passing OST [S1].
6.  Store these weighted attributes in separate color buffers in an OpenGL Frame Buffer Object (FBO) [S1].

### Step 5: Shading Pass
1.  Render a window-size rectangle [S1].
2.  Enable depth writing and disable alpha blending [S1].
3.  Enable multitexturing to access FBO color buffers [S1].
4.  In the fragment shader, normalize the weighted normal, color, and position from the color buffers [S1].
5.  Perform per-pixel shading [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def length(v):
    return math.sqrt(dot(v, v))

def compute_splat_attributes(points, normals, k, A=1.0, threshold=1e-6):
    n = len(points)
    k_nn_dists = []
    for i in range(n):
        dists = sorted([length([points[i][j] - points[j][j] for j in range(3)]) for j in range(n)])
        k_nn_dists.append(dists[k-1])

    splats = []
    for i in range(n):
        c = points[i]
        n_vec = normals[i]
        r_i = k_nn_dists[i]
        v_i = (r_i ** 2) / (2.0 * math.log(A / threshold))
        splats.append({'center': c, 'normal': n_vec, 'radius': r_i, 'variance': v_i, 'color': [0.5, 0.5, 0.5]})
    return splats

def visibility_pass(splats, camera):
    depth_image = [float('inf')] * (camera.width * camera.height)
    visible = []
    for s in splats:
        c_eye = [s['center'][j] - camera.eye_pos[j] for j in range(3)]
        n_eye = [s['normal'][j] for j in range(3)]
        if dot(n_eye, [-c_eye[j] for j in range(3)]) <= 0:
            continue
        z_eye = length(c_eye)
        proj_size = (camera.h_vp * s['radius']) / z_eye * (1.0 / (camera.t - camera.b))
        visible.append({'splat': s, 'proj_size': proj_size, 'z_eye': z_eye})
    return visible

def attribute_pass(visible_splats, camera):
    fbo_normal = [[0.0, 0.0, 0.0] for _ in range(camera.width * camera.height)]
    fbo_color = [[0.0, 0.0, 0.0] for _ in range(camera.width * camera.height)]
    fbo_position = [[0.0, 0.0, 0.0] for _ in range(camera.width * camera.height)]
    for v in visible_splats:
        s = v['splat']
        c_eye = [s['center'][j] - camera.eye_pos[j] for j in range(3)]
        n_eye = [s['normal'][j] for j in range(3)]
        # Fragment shader computes weighted attributes for pixels passing OST
        # and accumulates them into FBO buffers using glBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE, GL_ONE, GL_ONE)
        pass
    return fbo_normal, fbo_color, fbo_position

def shading_pass(fbo_buffers):
    # Fragment shader normalizes weighted normal, color, position from FBO buffers
    # and performs per-pixel shading
    pass
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
-   **Size set to 125% of $r_i$:** Model shows a high level of blurriness [S1].
-   **Variance > optimal:** Model displays noticeable discontinuities among overlapping splats [S1].
-   **Variance < optimal:** Model causes aliasing effects [S1].
-   **Sharp features:** The optimal splat size does not augur well with sharp features on the surface [S1]. This can be improved by clipping splats near corners and edges [S1].

## Operating characteristics
-   **Time Complexity:** The method requires finding k-nearest neighbors for each splat, which is typically $O(N \log N)$ or $O(N^2)$ depending on the implementation, plus the rendering passes [S1].
-   **Memory Complexity:** Each splat requires 11 floating-point values [S1].
-   **Measured Performance:** On Dragon and Hand models, surface splatting took less than 0.3 milliseconds per frame, while polygon rendering took more than 20 milliseconds [S1].
-   **Strengths:** Improves rendering quality on close-up views to be comparable to polygon rendering; automates parameter determination [S1].
-   **Limitations:** Not theoretically proven to be optimal for all models; struggles with sharp features without additional clipping; Gaussian blending can result in noticeable blurriness if parameters are not optimal [S1].
-   **When to use:** When high-quality close-up views of highly detailed surfaces are required, and performance is critical [S1].
-   **When not to use:** When sharp features are prominent and clipping is not applied; when polygon rendering performance is acceptable and quality is paramount for non-close-up views [S1].

## Background
The method is presented in the paper 'Automatic size and variance determination for better surface splatting quality on close-up view' [S1]. It builds on foundational work by Zwicker et al. (2001) on Surface Splatting and Pfister et al. (2000) on Surfel rendering [S1]. Previous works by Zwicker et al. (2001) and Ren et al. (2002) used global or local variance and fixed size, which could exhibit discontinuities on irregularly sampled models [S1]. Pajarola et al. (2004) required user adjustment of blending kernel parameters [S1]. Botsch et al. (2004/2005) applied per-pixel Phong shading, which is unsuitable for non-shiny materials and depends heavily on variance for normal interpolation [S1]. This method automates the determination of size and variance to address these issues [S1]. It has been tested on 3D models including Bunny, Dragon, and Hand, sourced from the Stanford 3D Scanning Repository, Digital Michelangelo projects, and Cyberware [S1].

## Sources
[S1] Automatic size and variance determination for better surface splatting quality on close-up view, https://doi.org/10.1145/2087756.2087792, fetched 2026-11-05
