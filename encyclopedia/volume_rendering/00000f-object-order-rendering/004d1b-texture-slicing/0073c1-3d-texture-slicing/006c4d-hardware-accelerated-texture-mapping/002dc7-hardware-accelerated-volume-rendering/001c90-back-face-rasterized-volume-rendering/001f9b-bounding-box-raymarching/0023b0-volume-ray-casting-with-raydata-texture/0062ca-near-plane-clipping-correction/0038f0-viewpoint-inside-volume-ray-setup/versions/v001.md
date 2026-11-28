# Viewpoint Inside Volume Ray Setup

Also called: Endoscopic Raycasting, Hybrid ray setup, Polygonal geometry intersection

## Overview
This method enables volume ray casting when the camera viewpoint is located inside the volume (e.g., virtual colonoscopy), a scenario where standard bounding-box ray setups fail because the near clipping plane intersects the front faces of the volume, creating holes in the ray entry data [S1, S2, unofficial]. It extends the image-based ray setup by intersecting rays with explicit polygonal geometry to correctly determine entry points and clip internal surfaces, allowing for hybrid rendering of isosurfaces and direct volume rendering [S1, S2, unofficial].

## How it works
The method builds on the image-based ray setup described in [Near Plane Clipping Correction](4848-near-plane-clipping-correction.md). It computes ray parameters per pixel using a two-pass pre-rendering step followed by the ray casting pass.

**Inputs:**
- Volume texture $V$.
- Polygonal geometry mesh $M$ representing internal surfaces or the volume boundary.
- Camera parameters (view and projection matrices).
- Ray step size $s$ and maximum ray length $L_{max}$.

**Outputs:**
- Final color image $C$.

**Data Structures:**
1.  **Ray Start Position Image ($I_{start}$):** A 2D texture storing the 3D world-space starting position of the ray for each pixel. Format: RGBA, where RGB is the position and A is unused or used for flags.
2.  **Ray Length Image ($I_{len}$):** A 2D texture storing the total length of the ray segment to be marched. Format: R32F (single float).
3.  **Modified Ray Length Image ($I_{len}'$):** Intermediate texture storing the clipped ray length after polygon intersection.
4.  **Compositing Buffer:** The final rendered image buffer.

**Steps:**

1.  **Compute Starting Position Image ($I_{start}$):**
    For each pixel $(u, v)$, compute the ray origin. If the viewpoint is inside the volume, the origin is the camera position. If outside, it is the intersection with the near plane or bounding box. This step is identical to the base image-based ray setup [S1, S2, unofficial].
    $$ \mathbf{p}_0 = \text{Unproject}(u, v, \text{near\_plane\_depth}) $$
    [G] If the camera is inside, $\mathbf{p}_0$ is the camera eye position.

2.  **Compute Ray Length Image ($I_{len}$):**
    For each pixel, compute the initial ray length. This is typically the distance from the ray origin to the far plane or the bounding box exit point.
    $$ L = \| \mathbf{p}_{far} - \mathbf{p}_0 \| $$
    Store $L$ in $I_{len}[u, v]$ [S1, S2, unofficial].

3.  **Render Polygonal Geometry to Modify Ray Length:**
    This step clips the rays at internal surfaces or corrects entry points.
    a.  **Fill Holes (if necessary):** If the near clipping plane clips into front faces, rasterize the near clipping plane geometry into $I_{len}$ (or a temporary buffer) to fill gaps where no ray origin was defined [S1, S2, unofficial].
    b.  **Intersect with Polygonal Mesh:** Rasterize the polygonal mesh $M$ using a depth buffer. For each pixel, if a polygon is visible, compute the intersection distance $d_{int}$ from the ray origin $\mathbf{p}_0$ along the ray direction $\mathbf{d}$.
    c.  **Clip Length:** Update the ray length in $I_{len}'$ to the minimum of the current length and the intersection distance.
        $$ L' = \min(L, d_{int}) $$
        If no intersection occurs, $L' = L$ [S1, S2, unofficial].
    [G] The rasterization must use the same view/projection matrices as the ray casting to ensure alignment. The depth buffer ensures only the nearest surface is considered.

4.  **Raycasting:**
    For each pixel, retrieve $\mathbf{p}_0$ from $I_{start}$ and $L'$ from $I_{len}'$. March the ray from $\mathbf{p}_0$ along $\mathbf{d}$ for length $L'$, sampling the volume $V$ and compositing the result into the compositing buffer.
    $$ \mathbf{p}(t) = \mathbf{p}_0 + t \cdot \mathbf{d}, \quad t \in [0, L'] $$
    Standard front-to-back or back-to-front compositing is applied [S1, unofficial].

5.  **Blending:**
    Composite the result into the final image. If hybrid rendering is used (isosurface + DVR), the polygonal geometry may be rendered separately as an opaque surface, and the volume ray casting is performed only for the interior or exterior regions as defined by the geometry [S1, unofficial].

**Pitfalls:**
-   **Precision:** Floating-point precision errors in ray origin and intersection calculations can cause gaps or overlaps, especially in endoscopic views with wide fields of view [S1, unofficial].
-   **Geometry Alignment:** The polygonal mesh must be perfectly aligned with the volume data and camera projection. Misalignment causes visual artifacts [S1, unofficial].
-   **Performance:** Rasterizing polygonal geometry adds overhead. Complex meshes can slow down the pre-pass [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N)$ for ray casting, where $N$ is the number of pixels. The pre-pass is $O(P)$, where $P$ is the number of polygons in the mesh.
-   **Memory Complexity:** $O(N)$ for the ray start and length images. Additional memory for the depth buffer during the polygon rasterization pass.
-   **Strengths:** Enables realistic endoscopic visualization; handles complex internal structures; allows hybrid rendering [S1, S2, unofficial].
-   **Limitations:** Requires explicit polygonal geometry; sensitive to geometry precision; increased memory usage for pre-pass buffers [S1, S2, unofficial].
-   **When to use:** Virtual colonoscopy, internal organ visualization, any scenario where the camera is inside the volume [S1, unofficial].
-   **When not to use:** Simple bounding-box volumes where the camera is outside; real-time applications with very limited memory and complex internal geometry [S2, unofficial].

## Background
The method was developed as an extension of image-based volume ray casting to support viewpoints inside the volume. It was presented in the context of advanced illumination techniques for GPU-based volume ray casting [S1, S2, unofficial]. The approach builds on earlier work by Scharsach et al. (2006) for handling near clipping plane issues [S1, unofficial]. It is particularly relevant for medical visualization, such as virtual colonoscopy, where the camera navigates inside the body [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
