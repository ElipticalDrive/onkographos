# Koyamada-SBRC

Also called: K-SBRC, K-SBRCBBA, K-SBRCEA

## Overview
Koyamada-SBRC is a hybrid volume rendering algorithm that blends the Span-Buffer Ray-Casting (SBRC) approach with Koyamada’s efficient image-space algorithm to exploit the best features of both hybrid and image-space methods [S1]. It specifically addresses the overhead of scan conversion for cells with small projection areas by dynamically selecting the traversal method per cell [S1]. Unlike its parent SBRC, which activates cells based on image-space coherency, K-SBRC reduces space complexity by not activating each cell hit by a ray when the cost of activation outweighs the benefit of coherency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Unstructured grid dataset (e.g., tetrahedral cells), camera parameters, volume data.
*   **Output:** Rendered image.

**Data Structures:**
*   **Cell Projection Area ($a_c$):** Estimated area of the cell's projection in pixels.
*   **Cell Projection Height ($h_c$):** Number of spans (height) of the cell's projection in the Normalized Projection Coordinate System.
*   **Bias Parameters:** Constants $a_{IT}$, $a_{CA}$, $a_{SC}$, $a_{I'}$ representing the relative computational costs of intersection tests, cell activation, span initialization, and intersection computation respectively [S1].

**Decision Logic:**
For each cell $c$ intersected by at least one ray, the algorithm estimates the execution cost for both Koyamada’s algorithm ($t_{cKoy}$) and SBRC ($t_{cSBRC}$) [S1]. The algorithm with the smaller estimated cost is selected [S1].

The cost formulas are:
$$ t_{cKoy} = a_{IT} i_{Tc} \approx 2 a_{IT} a_c $$
$$ t_{cSBRC} = a_{CA} + a_{SC} h_c + a_{I'} a_c $$

Where:
*   $i_{Tc}$ is the expected number of intersection tests for cell $c$ in Koyamada’s algorithm. It is approximated as $2 a_c$ because Koyamada’s algorithm performs 2 intersection tests per intersection on average ($IT \approx 2I$) [S1].
*   $a_{CA}$ is the cost of cell activation in SBRC.
*   $a_{SC}$ is the cost of initializing the scan-conversion process per span in SBRC.
*   $h_c$ is the height of the projection (number of spans) [S1].
*   $a_{I'}$ is the cost of ray-face intersection computation per pixel area in SBRC [S1].

**Bias Computation:**
The components $T_{NT}$ (transform nodes), $T_R$ (generate ray-segments), and $T_S$ (sampling/composition) are ignored in the bias computation because the same routines are employed in both algorithms for the shared parts of the pipeline [S1].

**Selection Rule:**
If $2 a_{IT} a_c < a_{CA} + a_{SC} h_c + a_{I'} a_c$, use Koyamada’s algorithm; otherwise, use SBRC [S1].

**Estimation Schemes for $a_c$ and $h_c$:**
Two schemes are provided to estimate $a_c$ and $h_c$ efficiently [S1]:

1.  **Exact Area Scheme (K-SBRCEA):**
    *   Calculate the areas of all faces of the cell in the Normalized Projection Coordinate System.
    *   Sum these areas.
    *   Divide the sum by two, because the original coverage area is half the sum (both front and back faces are used in the projection calculation) [S1].
    *   Compute $h_c$ exactly by finding the height of the projection in the Normalized Projection Coordinate System [S1].

2.  **Bounding Box Area Scheme (K-SBRCBBA):**
    *   Determine the bounding box of the cell's projection in the Normalized Projection Coordinate System.
    *   Use half the area of this bounding box as the approximation for $a_c$ [S1].
    *   Compute $h_c$ exactly by finding the height of the projection in the Normalized Projection Coordinate System [S1].
    *   Note: For tetrahedral cells, the projection is either a triangle or a quad, making the bounding box easy to calculate [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def koyamada_sbrc_render(cell, camera, params):
    """
    Renders a single cell using the Koyamada-SBRC hybrid strategy.
    
    Args:
        cell: The tetrahedral cell to render.
        camera: Camera object with projection matrix.
        params: Dictionary containing cost constants a_IT, a_CA, a_SC, a_I'.
        
    Returns:
        Rendered contribution of the cell.
    """
    # 1. Project cell vertices to Normalized Projection Coordinate System (NPCS)
    # This involves transforming vertices from World Coordinate System to NPCS.
    # Note: T_NT is ignored in bias, but transformation is required for geometry.
    projected_vertices = transform_to_npcs(cell.vertices, camera.projection_matrix)
    
    # 2. Estimate projection area (a_c) and height (h_c)
    # Choose scheme based on implementation preference or resolution.
    # Here we assume Bounding Box Area scheme for efficiency, 
    # but Exact Area can be substituted.
    
    # Calculate bounding box of the projected polygon in NPCS
    min_x, max_x = min(v.x for v in projected_vertices), max(v.x for v in projected_vertices)
    min_y, max_y = min(v.y for v in projected_vertices), max(v.y for v in projected_vertices)
    
    # Bounding Box Area Scheme: a_c is half the area of the bounding box
    a_c = 0.5 * (max_x - min_x) * (max_y - min_y)
    
    # Exact Height: h_c is the height of the projection in NPCS (number of spans)
    # In NPCS, y typically corresponds to screen rows.
    h_c = max_y - min_y
    
    # 3. Compute costs for both algorithms
    a_IT = params['a_IT']
    a_CA = params['a_CA']
    a_SC = params['a_SC']
    a_I_prime = params['a_I_prime']
    
    # Cost for Koyamada's algorithm: 2 intersection tests per intersection on average
    # i_Tc approx 2 * a_c
    cost_koy = 2 * a_IT * a_c
    
    # Cost for SBRC: Cell activation + Span initialization + Intersection computation
    cost_sbrc = a_CA + (a_SC * h_c) + (a_I_prime * a_c)
    
    # 4. Decision
    if cost_koy < cost_sbrc:
        # Use Koyamada's algorithm
        return render_with_koyamada(cell, camera, params)
    else:
        # Use SBRC
        return render_with_sbrc(cell, camera, params)

def render_with_koyamada(cell, camera, params):
    """
    Renders cell using Koyamada's image-space algorithm.
    """
    # Koyamada's algorithm iterates over pixels in the cell's projection.
    # For each pixel, it performs ray-face intersection tests.
    # It does not activate cells in a span buffer.
    projected_area_pixels = estimate_pixel_count(cell, camera)
    # Perform intersection tests for each pixel in the projected area
    # Average 2 tests per intersection
    for pixel in projected_area_pixels:
        ray = camera.get_ray(pixel)
        intersections = compute_ray_face_intersections(cell, ray)
        if intersections:
            # Compose color/opacity
            color = compose_volume(cell, ray, intersections)
            return color
    return None

def render_with_sbrc(cell, camera, params):
    """
    Renders cell using Span-Buffer Ray-Casting algorithm.
    """
    # SBRC activates the cell, scan-converts front faces to create spans,
    # and then processes spans.
    # 1. Activate cell
    activate_cell(cell)
    
    # 2. Scan-convert front external faces to generate spans
    spans = scan_convert_front_faces(cell, camera)
    
    # 3. Process spans
    for span in spans:
        # Initialize scan-conversion process for this span
        init_span(span)
        
        # Sample and compose along the span
        color = sample_and_compose(span, cell, camera)
        
        # Compute ray-face intersections for the span
        compute_span_intersections(span, cell, camera)
        
        return color
    return None
```

**Edge Cases and Pitfalls:**
*   **Exact Area Cost:** Exact Area is more accurate but computationally expensive compared to Bounding Box Area [S1].
*   **Bounding Box Overestimation:** Bounding Box Area overestimates $a_c$, which favors SBRC when the difference between the actual projection area and the bounding box area is large [S1].
*   **Resolution Dependency:** At low resolutions, errors from Bounding Box Area do not exaggerate cell projection areas enough to cause erroneous bias toward SBRC. At high resolutions, these errors cause the bias to select SBRC for cells that should be processed by Koyamada’s approach, resulting in worse performance [S1].
*   **Overhead:** The algorithm introduces an overhead of taking a decision on the algorithm to be employed for each cell. If the bias chooses Koyamada’s algorithm for all cells, the result is a larger execution time than Koyamada’s algorithm alone. If it chooses SBRC for all cells, the result is a larger execution time than SBRC alone [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   The algorithm reduces the space complexity of SBRC by not activating each cell hit by a ray when Koyamada’s method is selected [S1].
*   Time complexity depends on the selected algorithm per cell. Koyamada’s algorithm is generally slower for large cells due to lack of image-space coherency benefits, while SBRC is slower for small cells due to activation overhead [S1].

**Measured Performance:**
*   On overall average across datasets and resolutions, K-SBRC yields the best performance among proposed algorithms [S1].
*   **Low Resolution (300x300):** K-SBRC based on Bounding Box Area (K-SBRCBBA) achieves the best runtime performance in all datasets except Combustion Chamber (where CP is best) [S1].
*   **Medium Resolution (600x600):** K-SBRCBBA outperforms CP in datasets with large variation in cell projection areas (Oxygen Post, Delta Wing). CP is best for Blunt Fin and Combustion Chamber [S1].
*   **High Resolution (900x900):** CP outperforms SBRC and K-SBRC in Blunt Fin and Combustion Chamber. K-SBRCEA outperforms CP in Oxygen Post, and both SBRC and K-SBRCEA outperform CP in Delta Wing [S1].
*   **Dataset Sensitivity:** K-SBRC performs considerably better than SBRC at 300x300 and 600x600 resolutions for datasets with large variation in cell projection areas (Oxygen Post, Delta Wing). It performs worse than SBRC for datasets with almost equally sized large cells (Combustion Chamber) due to bias-computation overhead [S1].

**Strengths and Limitations:**
*   **Strengths:** Exploits the best features of hybrid and image-space methods. Handles acyclic non-convex meshes and generates images of high accuracy. Supports early ray termination and cyclic meshes (when using SBRC path) [S1].
*   **Limitations:** Decision overhead. Performance degrades if the bias is inaccurate (e.g., high resolution with Bounding Box Area). Not ideal for datasets with uniformly large cells [S1].

## Background
**Origin:**
*   The Koyamada-SBRC algorithm was proposed by H. Berk et al. in 2003 [S1].
*   It emerged from the observation that the benefit of using image-space coherency for cells with small projection areas might be suppressed by the overhead of the scan conversion process in SBRC [S1].

**History and Development:**
*   The algorithm is part of a broader investigation into fast direct volume rendering of unstructured grids, comparing image-space, object-space, and hybrid methods [S1].
*   It was implemented and experimented in a common framework for relative performance analysis alongside SBRC, Koyamada’s algorithm, and the Cell Projection (CP) algorithm [S1].

**Usage:**
*   Used in direct volume rendering of unstructured grids, particularly where cell projection areas vary significantly [S1].

## Sources
[S1] Direct volume rendering of unstructured grids, https://doi.org/10.1016/s0097-8493(03)00034-7, fetched 2026-10-24
