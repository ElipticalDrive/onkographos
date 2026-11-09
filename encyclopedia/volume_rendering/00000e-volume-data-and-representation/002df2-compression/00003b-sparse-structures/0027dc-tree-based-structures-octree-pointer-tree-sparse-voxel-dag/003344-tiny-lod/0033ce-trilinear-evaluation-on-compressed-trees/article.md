# Trilinear Evaluation on Compressed Trees
Also called: Decompression Shader
## Overview
This method evaluates a signal value at an arbitrary point $\mathbf{x}$ within a compressed tree structure, supporting trilinear filtering and adaptive traversal [S1, unofficial]. It serves as the evaluation engine for the Compressed Random-Access Tree (CRAT) framework, enabling fine-grain random queries required by rendering algorithms [S1, unofficial]. Unlike block-based decompression schemes (e.g., DXT1) that rely on specialized hardware, this method uses adaptive tree traversal to reconstruct data on the fly, trading raw decompression speed for spatial coherence and high-quality filtering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Point $\mathbf{x}$ (2D or 3D coordinate), mipmap level $l$.
*   Output: Interpolated signal value at $\mathbf{x}$.

**Data Structures:**
*   **Tree Nodes:** Each node stores a `value` (signal magnitude) and an `address` (pointer to child node or NULL) [S1, unofficial].
*   **VQ Codebook:** A lookup table storing residual values associated with non-NULL parent addresses [S1, unofficial].
*   **Forested Mipmap:** Coarsest levels (level $\le 4$) are stored as a standard mipmap; finer levels use the tree structure [S1, unofficial].

**Parameters:**
*   Starting level for trees: $l=4$ [S1, unofficial].
*   Threshold for switching to mipmap: $l \le 4$ [S1, unofficial].

**Algorithm Steps:**
1.  **Mipmap Check:** If $l \le 4$, return the trilinearly filtered value from the mipmap [S1, unofficial].
2.  **Initialization:** Identify the square cell containing $\mathbf{x}$ at level 4 [S1, unofficial]. Retrieve the root node address and value for each of the four corners of this cell [S1, unofficial]. Re-express $\mathbf{x}$ in the cell’s local coordinates [S1, unofficial].
3.  **Traversal Loop:**
    a.  **Interpolate:** At current $\mathbf{x}$, bilinearly interpolate the four node values [S1, unofficial].
    b.  **Termination Check:** Decrement $l$ (i.e., $l \leftarrow l - 1$) [S1, unofficial]. If $l < 1$ or all four node addresses are NULL, return the interpolated value, lerped with the value from the prior level using the current $l$ [S1, unofficial].
    c.  **Descend:** Set the new cell as the quadrant containing $\mathbf{x}$ [S1, unofficial].
    d.  **Predict:** For each of the four new cell corners, predict the new node value using bilinear interpolation of the current values [S1, unofficial].
    e.  **Residual Add:** If the node parent address is non-NULL, access the VQ codebook to add the residual value to the predicted value [S1, unofficial].
    f.  **Update Address:** Update the node address to the appropriate child node [S1, unofficial].
    g.  **Re-coordinate:** Re-express $\mathbf{x}$ in the new cell’s coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate(x, l):
    if l <= 4:
        return trilinear_filter_mipmap(x, l)  # [G] Standard mipmap lookup

    cell = get_cell_containing(x, level=4)
    corners = get_cell_corners(cell)
    corner_data = []
    for c in corners:
        addr, val = retrieve_root_node(c)
        corner_data.append({'addr': addr, 'val': val})
    x_local = reexpress_in_local(x, cell)

    while True:
        current_val = bilinear_interpolate(x_local, corner_data)
        l = l - 1
        if l < 1 or all(d['addr'] is None for d in corner_data):
            prior_val = get_prior_level_value(x, l + 1)  # [G] Mipmap/parent level value
            return lerp(current_val, prior_val, l)

        quadrant = get_quadrant_containing(x_local, cell)
        new_corners = get_cell_corners(quadrant)
        new_corner_data = []

        for i, new_c in enumerate(new_corners):
            new_c_local = reexpress_in_local(new_c, quadrant)
            predicted_val = bilinear_interpolate(new_c_local, corner_data)
            parent_addr = corner_data[i]['addr']
            if parent_addr is not None:
                predicted_val += access_vq_codebook(parent_addr)
            child_addr = get_child_address(parent_addr, i)
            new_corner_data.append({'addr': child_addr, 'val': predicted_val})

        corner_data = new_corner_data
        x_local = reexpress_in_local(x, quadrant)

# Helper definitions
def get_cell_containing(p, level):
    cell_size = 1.0 / (2 ** level)
    return {'origin': (p[0] // cell_size) * cell_size, 'size': cell_size}

def get_cell_corners(cell):
    o, s = cell['origin'], cell['size']
    return [(o[0], o[1]), (o[0]+s, o[1]), (o[0], o[1]+s), (o[0]+s, o[1]+s)]

def reexpress_in_local(p, cell):
    o, s = cell['origin'], cell['size']
    return ((p[0] - o[0]) / s, (p[1] - o[1]) / s)

def bilinear_interpolate(uv, corner_data):
    v00, v10, v01, v11 = [d['val'] for d in corner_data]
    u, v = uv
    return v00*(1-u)*(1-v) + v10*u*(1-v) + v01*(1-u)*v + v11*u*v

def get_quadrant_containing(uv, cell):
    u, v = uv
    s = cell['size'] / 2.0
    o = cell['origin']
    if u < 0.5 and v < 0.5: return {'origin': (o[0], o[1]), 'size': s}
    if u >= 0.5 and v < 0.5: return {'origin': (o[0]+s, o[1]), 'size': s}
    if u < 0.5 and v >= 0.5: return {'origin': (o[0], o[1]+s), 'size': s}
    return {'origin': (o[0]+s, o[1]+s), 'size': s}

def retrieve_root_node(corner): return None, 0.0  # [G] Standard tree root lookup
def access_vq_codebook(addr): return 0.0  # [G] Standard VQ residual lookup
def get_child_address(parent_addr, corner_idx): return None  # [G] Standard child pointer retrieval
def get_prior_level_value(x, level): return 0.0  # [G] Standard mipmap/parent level fetch
def lerp(a, b, t): return a + t * (b - a)
def trilinear_filter_mipmap(x, l): return 0.0  # [G] Standard mipmap lookup
```

**Edge Cases and Pitfalls:**
*   **NULL Addresses:** If all four node addresses are NULL, the traversal stops, and the value is interpolated with the prior mipmap level [S1, unofficial].
*   **Precision:** Integer arithmetic is used for decoding to enable better caching behavior on GPU [S1, unofficial].
*   **Caching:** The linear memory layout of the tree nodes is preferred over 2D texture unfolding for better caching [S1, unofficial].

## Subtopics

### [Tree Evaluation with Bilinear Interpolation and VQ Residuals](7078-tree-evaluation-with-bilinear-interpolation-and-vq-residuals.md)
This method evaluates a signal value at a point $\mathbf{x}$ by traversing an adaptive quadtree structure, using bilinear interpolation of node values and Vector Quantization (VQ) residuals to reconstruct the signal. It serves as the evaluation engine for the Compressed Random-Access Tree (CRAT) framework, enabling fine-grain random queries required by rendering algorithms. Unlike its sibling [Trilinear Evaluation on Compressed Trees](7074-trilinear-evaluation-on-compressed-trees.md), which operates in 3D with trilinear filtering, this method is designed for 2D spatially coherent data using bilinear filtering and primal subdivision logic.

## Operating characteristics
*   **Performance:** On a GeForce 8800 GTX, rendering performance varies by dataset: Land (lightmap) at 48 frames/sec, Lady (matte) at 60 frames/sec, Teapot (dist) at 115 frames/sec, Bull (dist) at 122 frames/sec, and Ennis (HDR) at 34 frames/sec [S1, unofficial].
*   **Decompression Rate:** The decompression rates are about 20X slower than DXT1/BC4U schemes [S1, unofficial].
*   **Hardware Dependency:** Block-based schemes benefit from specialized GPU hardware and optimized texture caching [S1, unofficial]. This method allows real-time rendering even without such assistance, but is slower [S1, unofficial].
*   **Instruction Count:** The decompression shader compiles to 298 instructions [S1, unofficial].
*   **Strengths:** Supports full filtering and adaptive traversal; enables real-time rendering of compressed data without specialized hardware decompression units [S1, unofficial].
*   **Limitations:** Significantly slower than hardware-accelerated block compression; complexity of tree traversal may impact cache coherence if not implemented with linear memory layouts [S1, unofficial].

## Background
This method was introduced by S. Lefebvre and H. Hoppe in the paper "Compressed Random-Access Trees for Spatially Coherent Data" published in the Eurographics Association 2007 proceedings [S1, unofficial]. It was developed to address the need for efficient random access to compressed spatial data, particularly for rendering applications requiring high-quality filtering and adaptive detail [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
