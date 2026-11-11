# Dynamic Volume Lines (DVL)

Also called: AMR-DVL, AMR-Extended Dynamic Volume Lines, Dynamic Volume Lines (DVL) for AMR, GPU-accelerated dynamic volume lines, Interactive Volume Lines (IVL), volume lines

## Overview
Dynamic Volume Lines (DVL) visualize large-scale multi-field Adaptive Mesh Refinement (AMR) volume ensembles as 1D polylines, mapping cell IDs to the x-axis and intensity to the y-axis [S1][S2, unofficial]. This method solves the problem of visual clutter and self-occlusion inherent in 3D volume rendering by projecting non-uniform AMR cells onto a 1D plot using importance-based nonlinear scaling [S1][S2, unofficial]. It is distinguished from its siblings by its specific adaptation to AMR hierarchies (using cell size to weight importance) and its GPU-accelerated implementation for interactive transfer function updates [S1][S2, unofficial].

## How it works
**Inputs:**
*   AMR dataset: A collection of cells, each defined by a centroid, a refinement level $L_h$, and associated scalar fields.
*   Ensemble $M$: A set of volume fields to be compared.
*   Transfer Function (TF): An RGB$\alpha$ mapping used to compute intensities.
*   Plot Width $W$: The number of pixels in the x-dimension of the output plot.
*   Parameters: Exponent $P$ (steepness), minimum importance threshold.

**Outputs:**
*   A 1D polyline or bar chart where the x-axis represents spatial locality (via Hilbert codes) and the y-axis represents the average intensity of the ensemble.

**Data Structures:**
1.  **Hilbert Codes:** A 64-bit integer per cell. Computed by quantizing the cell centroid coordinates to 20 bits per dimension (60 bits total) and mapping to a Hilbert space-filling curve [S1][S2, unofficial].
2.  **Bins:** An array of size $W$ (one per pixel). Each bin stores:
    *   `accumulated_value`: Sum of intensities of cells projecting to this bin.
    *   `counter`: Number of cells projecting to this bin.
3.  **Prefix Sum Array:** Floating-point array of size $N$ (number of cells), storing the cumulative sum of importance values $F(h)$ [S1][S2, unofficial].

**Steps:**

1.  **Compute Hilbert Codes:**
    For each cell $h$, quantize its centroid $(x, y, z)$ to 20 bits per dimension. Map these quantized coordinates to a 64-bit Hilbert code $H_h$ [S1][S2, unofficial].

2.  **Compute Local Variation ($V_h$):**
    For each cell $h$, calculate the local variation across the ensemble $M$ using the transfer function intensities $I(m, h)$ [S1][S2, unofficial]:
    $$ V_h = \max_{m \in M} I(m, h) - \min_{m \in M} I(m, h) $$
    where $I(m, h)$ is the intensity of ensemble member $m$ at cell $h$ [S1][S2, unofficial].

3.  **Compute Importance ($f(h)$):**
    Calculate the importance scale for each cell, incorporating the AMR refinement level $L_h$ (where $L_h=0$ is the finest level) and cell size $C_w = 2^{L_h}$ [S1][S2, unofficial]:
    $$ f(h) = \frac{V_h}{\max V_h} \left( \frac{2^{L_h}}{\max V_h} \right)^P $$
    Note: The source text presents this formula with potential typographical ambiguity regarding the denominator's scope. The standard interpretation for AMR extension scales the normalized variation by the cell width raised to power $P$ [S1][S2, unofficial]. A minimum importance floor (e.g., 0.025) is enforced to prevent zero-width regions [S1][S2, unofficial].

4.  **Compute Prefix Sum ($F(h)$):**
    Compute the exclusive or inclusive prefix sum of the importance values $f(i)$ for all cells $i=0 \dots h$ [S1][S2, unofficial]:
    $$ F(h) = \sum_{i=0}^{h} f(i) $$
    This determines the spatial extent of each cell in the 1D plot [S1][S2, unofficial].

5.  **Compute X-Coordinates:**
    For each cell $h$, determine the sub-pixel x-coordinates $x_{f1}$ and $x_{f2}$ for the plot width $W$ [S1][S2, unofficial]:
    $$ x_{f1} = \frac{F(h)}{F(\max(h))} \times W $$
    $$ x_{f2} = \frac{F(h+1)}{F(\max(h))} \times W $$
    (Note: $F(\max(h))$ is the total sum of all importance values) [S1][S2, unofficial].

6.  **Rasterize Cells (Binning):**
    Initialize $W$ bins to zero. For each cell, project the interval $[x_{f1}, x_{f2}]$ to integer bin indices in $[0, W-1]$. For each overlapping bin, add the cell's intensity value to `accumulated_value` and increment `counter` [S1][S2, unofficial]. This step uses atomic operations and is the computational bottleneck [S1][S2, unofficial].

7.  **Apply Weights:**
    For each bin $j$, compute the average intensity:
    $$ \text{Avg}_j = \frac{\text{accumulated\_value}_j}{\text{counter}_j} $$
    If `counter_j` is 0, $\text{Avg}_j$ is 0 [S1][S2, unofficial].

8.  **Apply Transfer Function:**
    Apply the RGB$\alpha$ transfer function to $\text{Avg}_j$ to obtain the y-coordinate (height) and color for the polyline at pixel $j$ [S1][S2, unofficial]. The alpha value determines the height; RGB determines the color [S1][S2, unofficial].

9.  **Rasterize Lines:**
    Draw the polylines or bar charts using the computed $(x, y)$ coordinates and colors [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_hilbert_code(centroid, bits_per_dim=20):
    """
    Quantize centroid to 20 bits per dimension and return 64-bit Hilbert code.
    Implementation details of Hilbert mapping omitted as per standard libraries.
    """
    # Quantize
    q_x = int(centroid.x * (2**bits_per_dim - 1))
    q_y = int(centroid.y * (2**bits_per_dim - 1))
    q_z = int(centroid.z * (2**bits_per_dim - 1))
    # Map to Hilbert code (placeholder for actual Hilbert encoding logic)
    return hilbert_encode_3d(q_x, q_y, q_z)

def compute_local_variation(cells, ensemble, transfer_function):
    """
    Compute V_h for each cell.
    """
    variations = []
    for cell in cells:
        intensities = []
        for m in ensemble:
            # Get intensity from transfer function applied to cell value in field m
            val = get_cell_value(cell, m)
            intensity = transfer_function(val)
            intensities.append(intensity)
        
        if not intensities:
            variations.append(0.0)
        else:
            v_h = max(intensities) - min(intensities)
            variations.append(v_h)
    return variations

def compute_importance(variations, max_variation, refinement_levels, P, min_importance=0.025):
    """
    Compute importance f(h) for each cell.
    """
    importances = []
    for i, v_h in enumerate(variations):
        L_h = refinement_levels[i]
        # Cell size C_w = 2^L_h
        # Formula: f(h) = (V_h / max_V) * ((2^L_h) / max_V)^P
        # Note: Source text implies scaling by cell width relative to max variation or similar.
        # Based on Eq 3 in [S1][S2, unofficial]: f(h) = (V_h / max(V_h)) * ( (2^L_h) / max(V_h) )^P
        # However, dimensional analysis suggests (2^L_h) might be scaled differently.
        # We follow the literal structure provided in sources.
        
        # Correction: Source [S1] Eq 3: f(h) = (V_h / max V_h) * ( (2^L_h) / max V_h )^P
        # This seems to divide cell width by max variation again. 
        # Let's stick to the most robust interpretation: 
        # f(h) = (V_h / max_V) * (2^L_h)^P * (1/max_V)^P ? 
        # The source text says: "extend Eq. (2) to include the AMR cell width as follows: f(h) = Vh ( 2Lh max Vh )P"
        # Interpreting "Vh ( 2Lh max Vh )P" as: (V_h / max_V_h) * ( (2^L_h) / max_V_h )^P
        
        term1 = v_h / max_variation if max_variation > 0 else 0.0
        term2_base = (2 ** L_h) / max_variation if max_variation > 0 else 0.0
        term2 = term2_base ** P
        
        f_h = term1 * term2
        if f_h < min_importance:
            f_h = min_importance
        importances.append(f_h)
    return importances

def compute_prefix_sum(importances):
    """
    Compute inclusive prefix sum F(h).
    """
    F = [0.0] * len(importances)
    current_sum = 0.0
    for i, imp in enumerate(importances):
        current_sum += imp
        F[i] = current_sum
    return F

def rasterize_cells(cells, prefix_sum, total_importance, plot_width, ensemble, transfer_function):
    """
    Project cells to bins and compute average intensities.
    """
    W = plot_width
    bins_value = [0.0] * W
    bins_count = [0] * W
    
    for i, cell in enumerate(cells):
        # Get x-coordinates
        F_h = prefix_sum[i]
        F_h_next = prefix_sum[i+1] if i+1 < len(prefix_sum) else total_importance
        
        x_f1 = (F_h / total_importance) * W
        x_f2 = (F_h_next / total_importance) * W
        
        # Project to integer bins
        bin_start = int(math.floor(x_f1))
        bin_end = int(math.ceil(x_f2))
        
        # Clamp to [0, W-1]
        bin_start = max(0, min(bin_start, W - 1))
        bin_end = max(0, min(bin_end, W - 1))
        
        # Compute average intensity for this cell across ensemble
        # Using the intensity that generated the variation? 
        # Source says: "increase the overlapping bins by the value of the corresponding AMR cell"
        # And later: "divide each by its bin counter, obtaining the average intensity"
        # We use the mean intensity of the ensemble for this cell for the y-value.
        intensities = []
        for m in ensemble:
            val = get_cell_value(cell, m)
            intensities.append(transfer_function(val))
        cell_intensity = sum(intensities) / len(intensities) if intensities else 0.0
        
        for b in range(bin_start, bin_end + 1):
            if 0 <= b < W:
                bins_value[b] += cell_intensity
                bins_count[b] += 1
                
    # Apply weights
    avg_intensities = []
    for b in range(W):
        if bins_count[b] > 0:
            avg_intensities.append(bins_value[b] / bins_count[b])
        else:
            avg_intensities.append(0.0)
            
    return avg_intensities

def render_volume_lines(avg_intensities, transfer_function, plot_width):
    """
    Apply TF to get Y coordinates and render.
    """
    y_coords = []
    for intensity in avg_intensities:
        # TF returns RGB and Alpha. Alpha is Y height.
        rgb, alpha = transfer_function(intensity)
        y_coords.append(alpha)
    return y_coords
```

**Parameters:**
*   $P$: User-defined parameter controlling the steepness of the importance curve. Typical range: 0.0 to 5.0 [S2, unofficial].
*   `min_importance`: Minimum enforced importance to prevent zero-width regions. Typical value: 0.025 [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Overdraw:** Multiple cells projecting to single pixels cause overdraw. The binning method averages these, which may obscure fine details [S1][S2, unofficial].
*   **Step Function Artifacts:** Using box-shaped basis functions (simple binning) instead of tent-shaped functions results in step-function artifacts when line segments span multiple bins [S1][S2, unofficial].
*   **ROI Selection in AMR:** For AMR data, the Hilbert code of the centroid is insufficient to determine the Region of Interest (ROI) size. Explicit cell location is required to check if a sample falls inside an ROI [S2, unofficial].
*   **Max Variation Approximation:** Computing $\max V_h$ exactly via parallel reduce is costly. It is approximated using transfer function and field data ranges on the CPU [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the `RasterizeCells` kernel due to atomic operations. Other kernels (`InclusiveScan`, `ApplyWeights`, `ApplyTransFunc`) are $< 0.01$ ms for 35.8M cells [S2, unofficial].
*   **Memory Complexity:** GPU memory usage for 35.8M cells including auxiliary data is 3.6 GB [S2, unofficial].
*   **Performance:** On an NVIDIA A6000 GPU, the method supports interactive updates. Parallelizing the `RasterizeCells` kernel with multiple CUDA streams did not reduce execution time, as the GPU was already fully occupied [S2, unofficial].
*   **Strengths:** Interactive updates of transfer functions; handles large-scale AMR data; avoids 3D occlusion [S1][S2, unofficial].
*   **Limitations:** Approximation of $\max V_h$ may be conservative; step-function artifacts from binning; bottlenecked by atomic operations in rasterization [S1][S2, unofficial].

## Background
Dynamic Volume Lines were originally proposed by Weissenböck et al. for structured-regular volumetric data [S1][S2, unofficial]. The method was extended to support large-scale multi-field AMR data and GPU acceleration (Interactive Volume Lines) by Zellmann et al. in 2023 [S1][S2, unofficial]. The implementation uses CUDA and CUB libraries [S1][S2, unofficial].

## Sources

[S1] Visual Analysis of Large Multi-Field AMR Data on GPUs Using Interactive Volume Lines, https://doi.org/10.1109/vis54172.2023.00020, fetched 2026-10-26
[S2] PDF Visual Analsyis of Large Multi-Field AMR Data on GPUs Using Interactive ..., https://www.cs.bilkent.edu.tr/~gudukbay/publications/papers/conf_papers/Zellmann_Et_Al_IEEE_VIS_2023.pdf, fetched 2026-10-27 (unofficial)
