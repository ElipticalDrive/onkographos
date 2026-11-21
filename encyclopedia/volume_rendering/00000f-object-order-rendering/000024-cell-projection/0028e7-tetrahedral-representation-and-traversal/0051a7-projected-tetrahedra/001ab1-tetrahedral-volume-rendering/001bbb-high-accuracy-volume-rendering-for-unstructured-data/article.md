# High Accuracy Volume Rendering for Unstructured Data

Also called: Exact Volume Rendering for Unstructured Data, HIAC System, High Accuracy Volume Renderer for Unstructured Data

## Overview
High Accuracy Volume Rendering (HIAC) is a software-based cell projection renderer designed for high-quality batch-mode generation of images from unstructured finite element data [S1, unofficial]. It solves the problem of rendering complex, non-convex, or disconnected meshes with mathematical exactness, avoiding the approximations typically introduced for efficiency [S2, unofficial]. Unlike its siblings, which may prioritize interactivity or approximate sampling, HIAC guarantees high accuracy by using exact solutions for radiance integrals and interpolation, extending hardware-assisted projection to arbitrary convex polyhedra [S2, unofficial].

## How it works
**Inputs:** Unstructured mesh data containing cells of type linear or quadratic tetrahedra, bricks, prisms, wedges, or pyramids; scalar or vector field data associated with vertices; camera parameters; and lighting model parameters [S2, unofficial].
**Outputs:** A high-fidelity 2D image [S2, unofficial].

**Data Structures:**
The system relies on a visibility ordering algorithm to determine the front-to-back or back-to-front traversal of cells [S2, unofficial]. For mixed cell types, the system handles tetrahedra with high accuracy, while other convex polyhedra (bricks, prisms, wedges, pyramids) are projected and composited, but not with high accuracy [S2, unofficial].

**Steps:**
1. **Visibility Ordering:** Compute the exact visibility order of all cells relative to the viewpoint using a revised algorithm that includes a correction and a method for dramatically increasing its efficiency [S2, unofficial].
2. **Load Balancing:** Partition the volumetric data set using a Kd-tree structure to distribute cells across processing nodes, ensuring balanced workload [S1, unofficial].
3. **Parallelization:** Distribute the sorting step of the visibility ordering among several nodes to parallelize the rendering [S1, unofficial].
4. **Projection:** Project the 3D cells onto the 2D image plane [S2, unofficial].
5. **Interpolation and Integration:** Use exact mathematical solutions for interpolation and radiance integrals whenever possible [S2, unofficial].
6. **Subpixel Accumulation:** For very small cells, use subpixel accumulation by splatting to avoid sampling error [S2, unofficial].
7. **Compositing:** Composite the projected and shaded cells according to the visibility order [S2, unofficial].
8. **Isosurface Embedding:** Accurate semi-transparent shaded isosurfaces may be embedded within the volume rendering process [S2, unofficial].

**Formulas:**
The radiance $L$ along a ray is computed using exact polynomial integration over linear and quadratic tetrahedral cells, where the density $\rho$ and emission $\epsilon$ are interpolated using exact basis functions [S2, unofficial]. For mixed convex polyhedra, approximate integration is applied [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hiach(mesh, fields, camera, light_model):
    # 1. Visibility Ordering: Compute exact visibility order of cells relative to viewpoint [S2, unofficial]
    # Source describes a revised algorithm with a correction and efficiency method [S2, unofficial]
    ordered_cells = sorted(mesh.cells, key=lambda c: c.depth(camera)) # [G]

    # 2. Load Balancing: Partition volumetric data using Kd-tree [S1, unofficial]
    partitions = kd_partition_cells(ordered_cells) # [G]

    # 3. Parallelization: Distribute sorting/compositing across nodes [S1, unofficial]
    results = []
    for partition in partitions:
        results.append(render_partition(partition, fields, camera, light_model))

    # 4-7. Composite Results
    final_image = front_to_back_composite(results) # [G]
    return final_image

def render_partition(cells, fields, camera, light_model):
    image = create_empty_image(camera.resolution) # [G]
    for cell in cells:
        # 4. Projection: Project 3D cells to 2D image plane [S2, unofficial]
        projected = project_to_screen(cell, camera) # [G]

        # 5. Interpolation & Integration: Exact solutions for radiance/interpolation [S2, unofficial]
        if cell.cell_type in ('linear_tet', 'quadratic_tet'):
            radiance = exact_radiance_integral(cell, fields, light_model) # [G]
        else:
            radiance = approximate_radiance_integral(cell, fields, light_model) # [S2, unofficial]

        # 6. Subpixel Accumulation: Splatting for very small cells [S2, unofficial]
        if projected.area < SUBPIXEL_THRESHOLD: # [G]
            splat_subpixel_accumulation(image, projected, radiance) # [G]
        else:
            rasterize_and_composite(image, projected, radiance) # [G]

    # 8. Isosurface Embedding: Embed accurate semi-transparent shaded isosurfaces [S2, unofficial]
    embed_isosurfaces(image, mesh, fields) # [G]
    return image

# Standard helper implementations [G]
def kd_partition_cells(cells):
    return [cells[i::2] for i in range(2)] # Simplified spatial split

def front_to_back_composite(images):
    result = images[0]
    for img in images[1:]:
        result = alpha_composite(result, img)
    return result

def create_empty_image(res):
    return [[0.0, 0.0, 0.0, 0.0] for _ in range(res[0] * res[1])]

def project_to_screen(cell, camera):
    return cell.transform(camera.projection_matrix)

def exact_radiance_integral(cell, fields, light_model):
    return integrate_polynomial(cell, fields, light_model)

def approximate_radiance_integral(cell, fields, light_model):
    return sample_and_average(cell, fields, light_model)

def splat_subpixel_accumulation(image, projected, radiance):
    image.add_splat(projected, radiance)

def rasterize_and_composite(image, projected, radiance):
    image.rasterize(projected, radiance)

def embed_isosurfaces(image, mesh, fields):
    image.overlay_isosurfaces(mesh, fields)
```

**Parameters:**
*   **Accuracy Mode:** High accuracy is enforced for linear and quadratic tetrahedra [S2, unofficial]. Mixed cell types (bricks, prisms, wedges, pyramids) are supported but not with high accuracy [S2, unofficial].
*   **Subpixel Threshold:** Not given in the sources [G].

**Edge Cases:**
*   **Non-convex/Disconnected Meshes:** The system handles these, but requires conversion to convex meshes (e.g., via Delaunay triangulation) for accurate processing [S2, unofficial].
*   **Very Small Cells:** Handled via subpixel splatting to prevent sampling error [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The system is computationally intensive [S1, unofficial]. Rendering a 512×512 viewport takes up to one minute on an SGI Onyx with four processors [S1, unofficial]. Even with 128 processors on an SGI Onyx2, it takes over two seconds [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [G].
*   **Strengths:** Very high accuracy; exact mathematical solutions for integrals and interpolation; handles complex unstructured meshes (non-convex, disconnected, mixed types) [S1, unofficial][S2, unofficial].
*   **Limitations:** Not intended for interactive purposes; slow performance even on high-end hardware; approximations are introduced for mixed cell types (not high accuracy) [S1, unofficial][S2, unofficial].
*   **When to Use:** Batch-mode rendering where image quality is paramount and performance is secondary [S1, unofficial].
*   **When Not to Use:** Interactive applications or real-time rendering [S1, unofficial].

## Background
The HIAC system was developed by Williams, Max, and Stein in 1998 [S2, unofficial]. It was designed as a benchmark system for comparison, where all mathematical operations are described in detail [S1, unofficial]. Later, Bennett et al. parallelized the HIAC system by distributing the sorting step of the visibility ordering among several nodes, using Kd-partitioning for load balancing [S1, unofficial]. The system is particularly suited for finite element data [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) Tetrahedral projection using vertex shaders - ResearchGate, https://www.researchgate.net/publication/221247894_Tetrahedral_projection_using_vertex_shaders, fetched 2026-10-28 (unofficial)
