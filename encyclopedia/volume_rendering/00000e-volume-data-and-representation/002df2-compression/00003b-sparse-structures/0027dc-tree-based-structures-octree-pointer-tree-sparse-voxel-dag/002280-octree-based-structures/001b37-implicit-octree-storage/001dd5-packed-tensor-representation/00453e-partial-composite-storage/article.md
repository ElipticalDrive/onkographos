# Partial Composite Storage

Also called: Partial Composite

## Overview
Partial Composite Storage is a data compression technique used in volume rendering to reduce memory footprint and communication overhead by aggregating consecutive ray samples into a single representative entity [S1]. Unlike fixed-size storage or mask-based approaches (such as those in [Packed Tensor Representation](1611-packed-tensor-representation.md)), it replaces a sequence of $N$ samples with a fixed-size structure containing the accumulated color, opacity, and depth bounds, regardless of $N$ [S1]. This method is particularly effective when large portions of the ray consist of transparent or empty space, allowing for significant reduction in data volume during the exchange phase of distributed rendering [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A sequence of consecutive samples along a ray, each having a color ($C_{in}$) and opacity ($\alpha_{in}$) derived from a transfer function.
*   **Output:** A single Partial Composite structure containing accumulated color ($C_{out}$), accumulated opacity ($\alpha_{out}$), and the depth indices of the first ($Z_s$) and last ($Z_e$) samples in the sequence.

**Data Structures**
The Partial Composite structure consists of exactly 6 values:
1.  **RGBA:** 4 floating-point values representing the accumulated color and opacity ($C_{out}$).
2.  **$Z_s$:** 1 integer representing the depth index of the first sample in the sequence.
3.  **$Z_e$:** 1 integer representing the depth index of the final sample in the sequence.

Total size: 24 bytes (assuming 4 bytes per float and 4 bytes per integer) [S1].

**Algorithm Steps**
1.  **Sample Evaluation:** For each sample along the ray, apply the transfer function to determine its color ($C_{in}$) and opacity ($\alpha_{in}$) [S1].
2.  **Sequence Identification:** Identify consecutive groups of samples that will be aggregated. A group is defined by a start index $Z_s$ and an end index $Z_e$ [S1].
3.  **Front-to-Back Compositing:** Iterate through the samples in the identified sequence from $Z_s$ to $Z_e$ in front-to-back order. Accumulate the color and opacity using the standard volume rendering equation [S1]:
    $$C_{out} = C + (1 - \alpha) \cdot C_{in} \cdot \alpha_{in}$$
    Where:
    *   $C_{out}$ is the RGBA value of the output partial composite.
    *   $C$ is the color of the current partial composite (accumulated so far).
    *   $\alpha$ is the accumulated opacity (accumulated so far).
    *   $C_{in}$ is the color of the current sample.
    *   $\alpha_{in}$ is the opacity of the current sample.
    
    *Note: The source formula [S1] uses $C$ and $\alpha$ as the accumulated state variables. Standard front-to-back compositing usually updates $\alpha$ as $\alpha_{new} = \alpha_{old} + (1-\alpha_{old})\alpha_{in}$. The source text explicitly defines the color update equation but implies standard accumulation for opacity to define the "accumulated opacity" term $\alpha$ [S1].*
4.  **Structure Construction:** Store the final accumulated $C_{out}$ (RGBA), the start index $Z_s$, and the end index $Z_e$ into the Partial Composite structure [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_partial_composite(samples):
    """
    Creates a Partial Composite from a list of consecutive samples.
    samples: List of tuples (z_index, color_rgba, opacity_alpha)
             sorted by z_index.
    Returns: Tuple (rgba_out, z_start, z_end)
    """
    if not samples:
        return None, None, None

    z_start = samples[0][0]
    z_end = samples[-1][0]
    
    # Initialize accumulated color and opacity
    # Assuming standard front-to-back initialization
    acc_color = (0.0, 0.0, 0.0, 0.0)
    acc_opacity = 0.0
    
    for z, c_in, a_in in samples:
        # Extract components
        r_in, g_in, b_in, a_in_val = c_in
        
        # Update accumulated opacity
        # Standard formula: alpha_new = alpha_old + (1 - alpha_old) * alpha_in
        acc_opacity = acc_opacity + (1.0 - acc_opacity) * a_in_val
        
        # Update accumulated color using source formula:
        # Cout = C + (1 - alpha) * Cin * alpha_in
        # Note: 'alpha' in source formula refers to accumulated opacity before this step
        # 'C' refers to accumulated color before this step
        r_out = acc_color[0] + (1.0 - acc_opacity + a_in_val) * r_in * a_in_val # Wait, source formula is Cout = C + (1 - alpha) * Cin * alpha_in
        # Let's strictly follow source formula: Cout = C + (1 - alpha) * Cin * alpha_in
        # Here 'alpha' is the accumulated opacity BEFORE processing the current sample?
        # Or is it the current sample's alpha? Source says: "alpha is the accumulated opacity, and alpha_in is the opacity of the current sample."
        # And "C is the color of the current partial composite" (accumulated so far).
        
        # Correct interpretation of source formula (1):
        # Cout_new = C_old + (1 - alpha_old) * C_in * alpha_in
        
        r_out = acc_color[0] + (1.0 - acc_opacity) * r_in * a_in_val
        g_out = acc_color[1] + (1.0 - acc_opacity) * g_in * a_in_val
        b_out = acc_color[2] + (1.0 - acc_opacity) * b_in * a_in_val
        # Alpha component of Cout is just the new acc_opacity
        
        acc_color = (r_out, g_out, b_out, acc_opacity)

    return acc_color, z_start, z_end
```

**Parameters and Edge Cases**
*   **Sequence Length:** The method reduces storage from $4 \times N$ bytes to 24 bytes. For example, 100 samples (400 bytes) are reduced to 24 bytes [S1].
*   **Empty Sequences:** If a sequence contains no samples (fully transparent), the composite should ideally represent zero opacity. The source does not explicitly define the handling of fully transparent runs, but implies compositing only valid samples [S1].
*   **Precision:** Floating-point accumulation may introduce precision errors over very long sequences. Not given in the sources.

## Operating characteristics
*   **Memory Reduction:** Reduces storage cost from $4 \times S_{PR} \times P$ bytes to $24 \times P$ bytes in the best case (where $S_{PR}$ is samples per ray and $P$ is pixels) [S1]. For an image with 1 million pixels and 1000 samples per ray, cost drops from 4 GB to 24 MB [S1].
*   **Communication Cost:** Significantly reduces data exchange volume in distributed rendering by sending one composite per ray segment instead of individual samples [S1].
*   **Limitations:** Requires that samples be grouped into consecutive runs. If every sample is distinct and non-composable (e.g., due to transparency gaps), the benefit is lost. The source does not specify how gaps (transparent regions) are handled in the grouping logic, only that consecutive samples are composited [S1].
*   **Sorting Requirement:** The final pixel color requires sorting the partial composites by z-depth before accumulation [S1].

## Background
Partial composites were introduced in the context of scalable hybrid schemes for ray-casting of unstructured volume data [S1]. The technique is part of a broader pipeline including object-order sampling, exchange, and image-order phases [S1]. It addresses the high memory and communication costs associated with storing and exchanging large numbers of samples in distributed volume rendering [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
