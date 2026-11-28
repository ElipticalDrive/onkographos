# Partial Composite Storage

Also called: Partial Composite

## Overview
Partial Composite Storage is a data compression technique used in volume rendering to reduce memory footprint and communication overhead by aggregating consecutive ray samples into a single representative entity [S1]. Unlike fixed-size storage or mask-based approaches (such as those in [Packed Tensor Representation](1611-packed-tensor-representation.md)), it replaces a sequence of $N$ samples with a fixed-size structure containing the accumulated color, opacity, and depth bounds, regardless of $N$ [S1]. This method is particularly effective when large portions of the ray consist of transparent or empty space, allowing for significant reduction in data volume during the exchange phase of distributed rendering [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A sequence of consecutive samples along a ray, each having a color ($C_{in}$) and opacity ($\alpha_{in}$) derived from a transfer function [S1].
*   **Output:** A single Partial Composite structure containing accumulated color and opacity ($C_{out}$), and the depth indices of the first ($Z_s$) and last ($Z_e$) samples in the sequence [S1].

**Data Structures**
The Partial Composite structure consists of exactly 6 values:
1.  **RGBA:** 4 floating-point values representing the accumulated color and opacity ($C_{out}$) [S1].
2.  **$Z_s$:** 1 integer representing the depth index of the first sample in the sequence [S1].
3.  **$Z_e$:** 1 integer representing the depth index of the final sample in the sequence [S1].

Total size: 24 bytes (assuming 4 bytes per float and 4 bytes per integer) [S1].

**Algorithm Steps**
1.  **Sample Evaluation:** Apply a transfer function to assign color ($C_{in}$) and opacity ($\alpha_{in}$) to each sample [S1].
2.  **Sequence Identification:** Identify consecutive groups of samples to aggregate, defined by a start index $Z_s$ and an end index $Z_e$ [S1].
3.  **Front-to-Back Compositing:** Iterate through the samples in the identified sequence from $Z_s$ to $Z_e$ in front-to-back order [S1]. Accumulate the color and opacity using the equation $C_{out} = C + (1 - \alpha) \cdot C_{in} \cdot \alpha_{in}$ [S1]. In this formula, $C$ is the accumulated color so far, $\alpha$ is the accumulated opacity so far, $C_{in}$ is the current sample's color, and $\alpha_{in}$ is the current sample's opacity [S1].
4.  **Structure Construction:** Store the final accumulated $C_{out}$ (RGBA), the start index $Z_s$, and the end index $Z_e$ into the Partial Composite structure [S1].

**Pseudocode**
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

    # Initialize accumulated color and opacity [G]
    acc_color = [0.0, 0.0, 0.0, 0.0]
    acc_opacity = 0.0

    for z, c_in, a_in in samples:
        r_in, g_in, b_in, a_in_val = c_in

        # Capture state before update to match source formula variables
        alpha_old = acc_opacity
        color_old = acc_color

        # Update accumulated color: Cout = C + (1 - alpha) * Cin * alpha_in
        acc_color[0] = color_old[0] + (1.0 - alpha_old) * r_in * a_in_val
        acc_color[1] = color_old[1] + (1.0 - alpha_old) * g_in * a_in_val
        acc_color[2] = color_old[2] + (1.0 - alpha_old) * b_in * a_in_val

        # Update accumulated opacity: alpha_new = alpha_old + (1 - alpha_old) * alpha_in [G]
        acc_opacity = alpha_old + (1.0 - alpha_old) * a_in_val

    # Store final accumulated opacity in the alpha channel of the color tuple
    acc_color[3] = acc_opacity

    return tuple(acc_color), z_start, z_end
```

**Parameters and Edge Cases**
*   **Sequence Length:** The method reduces storage from $4 \times N$ bytes to 24 bytes [S1]. For example, 100 samples (400 bytes) are reduced to 24 bytes [S1].
*   **Transparent Gaps:** The source does not explicitly define the handling of fully transparent runs or gaps between valid samples, implying compositing applies only to consecutive valid samples [S1].
*   **Precision:** Floating-point accumulation may introduce precision errors over very long sequences. Not given in the sources.
*   **Sorting Requirement:** The final pixel color is obtained by accumulating the partial composites after sorting them in the correct z-depth order [S1].

## Operating characteristics
*   **Memory Reduction:** Reduces storage cost from $4 \times S_{PR} \times P$ bytes to $24 \times P$ bytes in the best case (where $S_{PR}$ is samples per ray and $P$ is pixels) [S1]. For an image with 1 million pixels and 1000 samples per ray, cost drops from 4 GB to 24 MB [S1].
*   **Communication Cost:** Significantly reduces data exchange volume in distributed rendering by sending one composite per ray segment instead of individual samples [S1].
*   **Limitations:** Requires that samples be grouped into consecutive runs [S1]. If every sample is distinct and non-composable (e.g., due to transparency gaps), the benefit is lost [S1]. The source does not specify how gaps are handled in the grouping logic, only that consecutive samples are composited [S1].
*   **Sorting Requirement:** The final pixel color requires sorting the partial composites by z-depth before accumulation [S1].

## Background
Partial composites are a technique used within scalable hybrid schemes for ray-casting of unstructured volume data [S1]. The method operates as part of a pipeline that includes object-order sampling, exchange, and image-order phases [S1]. It addresses the high memory and communication costs associated with storing and exchanging large numbers of samples in distributed volume rendering [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
