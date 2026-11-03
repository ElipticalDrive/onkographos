# OTS Marching Ray Marching Algorithm

Also called: Adaptive Binning Ray Marching, OTS Marcher, OTS Marching, Ray Marching (OTS scheme)

## Overview
The OTS Marching Ray Marching Algorithm is a volumetric rendering method for 3D Gaussians that evaluates the full Gaussian mixture to accurately handle primitive overlap and compute the volumetric integral using an extinction-based model [S1]. It distinguishes itself from splatting-based approaches by performing adaptive binning and analytical integration along rays, ensuring that overlapping Gaussians are resolved with precision governed by sample count rather than relying on rasterization approximations [S1]. Unlike raytracing solutions for Gaussians, this method explicitly accounts for primitive overlap in the integration process [S1].

## How it works
The algorithm performs adaptive sampling along each viewing ray to resolve Gaussian overlap [S1]. It iterates over the Gaussian mixture twice: first to establish bin borders adapted to the Gaussian data, and second to perform analytical integration into bins of variable extent [S1]. The bins are then blended together following the volume integral equation [S1].

### Inputs and Outputs
*   **Inputs:** A set of 3D Gaussians defined by position, scale, rotation, and color; a viewing ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a maximum ray depth or opacity threshold [S1].
*   **Outputs:** The final pixel color $I(\mathbf{x})$ and accumulated transmittance [S1].

### Data Structures
*   **Density Section Buffer:** A fixed-size buffer storing sections [S1]. Each section is defined by:
    *   `start`: The start parameter $t$ along the ray [S1].
    *   `end`: The end parameter $t$ along the ray [S1].
    *   `density`: The sampling density for this section [S1].
*   **Bin:** A segment of the ray defined by start and end $t$ values, storing integrated color and extinction [S1].

### Steps

1.  **Adaptive Binning Initialization:**
    Create a fixed-size density section buffer [S1]. Iterate over all Gaussians to fill the buffer [S1]. For each Gaussian, compute its intersection interval with the ray [S1]. Define the start and end of the sampling section using the Gaussian's position $\pm 3$ standard deviations along the ray direction [S1]. Compute the density such that a pre-defined number of bins fit within the space defined by start and end [S1].

2.  **Section Merging:**
    Add the new section to the density section buffer [S1]. Combine existing and new sections to ensure they are ordered and non-overlapping [S1]. If sections overlap, the section with the higher sampling density takes precedence in the overlap area [S1]. Combining two sections can result in up to three new sections [S1]. Sections that fall outside the buffer capacity are deferred to the next batch [S1].

3.  **Bin Creation:**
    Traverse the merged sections to create bins [S1]. The size of each bin is inversely proportional to the density of the section it belongs to [S1].

4.  **Blending Integral Computation (Batching):**
    Divide the work into batches of bins (e.g., 128 bins per batch) [S1].
    *   Initialize the first batch at $t=0$ [S1]. Subsequent batches start at the end-border of the last bin of the previous batch [S1].
    *   Iterate over the Gaussian mixture again to compute integrals for each bin in the current batch [S1].
    *   For each bin, compute 4 values: Red, Green, Blue, and integrated extinction (opacity) [S1].
    *   Colors are multiplied by the per-Gaussian integrated extinction [S1].
    *   Stop when all Gaussians are processed or accumulated opacity reaches a high threshold [S1].

5.  **Blending:**
    Blend the bin evaluations together using the following equation [S1]:

    $$
    I(\mathbf{x}) = \sum_{i=0}^{\infty} \gamma_i \prod_{j=0}^{i-1} e^{-\rho_j} + c_b \prod_{j=0}^{\infty} e^{-\rho_j}
    $$

    Where:
    *   $\gamma_i$ is the color contribution of bin $i$ [S1].
    *   $\rho_j$ is the integrated extinction (opacity) of bin $j$ [S1].
    *   The term $\prod_{j=0}^{i-1} e^{-\rho_j}$ is the transmittance factor, cached between iterations to avoid redundant computation [S1].
    *   $c_b$ is the background color [S1].

6.  **Gradient Computation:**
    Compute gradients manually using a front-to-back strategy [S1]. Two variants exist:
    *   **Attached:** Considers the numerical integration procedure and border positions in the gradient calculation [S1].
    *   **Detached:** Assumes the numerical method can be ignored, keeping sampling and border positions constant with respect to differentiation [S1]. The detached variant performs significantly better [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ots_marching_ray_march(ray_origin, ray_direction, gaussians, background_color, max_sections_buffer, bins_per_batch=128, opacity_threshold=0.99, bins_per_gaussian=10):
    """
    Renders a single ray using the OTS Marching Algorithm.
    """
    # 1. Adaptive Binning Initialization
    sections = []
    for g in gaussians:
        # Compute intersection interval along ray using position +/- 3 standard deviations
        t_start = g.position - 3.0 * g.scale
        t_end = g.position + 3.0 * g.scale
        if t_start > t_end:
            t_start, t_end = t_end, t_start

        # Compute density such that a pre-defined number of bins fit in the space
        length = t_end - t_start
        density = bins_per_gaussian / length if length > 0 else 0.0

        new_section = Section(start=t_start, end=t_end, density=density)
        sections = merge_sections(sections, new_section, max_sections_buffer)

    # 2. Bin Creation
    bins = create_bins(sections)

    # 3. Blending Integral Computation (Batching)
    total_color = [0.0, 0.0, 0.0]
    transmittance = 1.0

    for batch_start in range(0, len(bins), bins_per_batch):
        batch_end = min(batch_start + bins_per_batch, len(bins))
        current_batch_bins = bins[batch_start:batch_end]

        for bin in current_batch_bins:
            bin_color = [0.0, 0.0, 0.0]
            bin_extinction = 0.0

            # Iterate over Gaussians again to compute integrals for each bin
            for g in gaussians:
                # Check overlap and compute analytical integral
                if g.overlaps_segment(bin.start, bin.end):
                    c, e = g.integrate_over_segment(bin.start, bin.end)
                    bin_color[0] += c[0]
                    bin_color[1] += c[1]
                    bin_color[2] += c[2]
                    bin_extinction += e

            # Front-to-back blending matching Eq. 3
            transmittance_factor = exp(-bin_extinction)
            total_color[0] += bin_color[0] * transmittance
            total_color[1] += bin_color[1] * transmittance
            total_color[2] += bin_color[2] * transmittance
            transmittance *= transmittance_factor

            # Check accumulated opacity threshold
            if 1.0 - transmittance >= opacity_threshold:
                break

        if 1.0 - transmittance >= opacity_threshold:
            break

    # Add background
    total_color[0] += background_color[0] * transmittance
    total_color[1] += background_color[1] * transmittance
    total_color[2] += background_color[2] * transmittance

    return total_color, transmittance

def merge_sections(existing, new_sec, max_size):
    """
    Combines new_sec with existing sections ensuring order and no overlap.
    Higher density wins in overlap. Combining 2 sections creates up to 3 new sections.
    """
    merged = existing + [new_sec]
    merged.sort(key=lambda s: s.start)

    result = []
    for sec in merged:
        if not result:
            result.append(sec)
            continue
        last = result[-1]
        if sec.start < last.end:
            # Overlap exists
            if sec.density >= last.density:
                # Higher density wins: extend last section, discard lower density part
                last.end = max(last.end, sec.end)
            else:
                # Lower density wins in overlap: split last section
                last.end = sec.start
                result.append(sec)
        else:
            result.append(sec)

    # Defer excess sections to next batch if buffer full
    if len(result) > max_size:
        result = result[:max_size]
    return result

def create_bins(sections):
    """
    Traverse sections to create bins. Bin size is inversely proportional to density.
    """
    bins = []
    for sec in sections:
        if sec.density <= 0:
            continue
        bin_width = 1.0 / sec.density
        t = sec.start
        while t < sec.end:
            bin_end = min(t + bin_width, sec.end)
            bins.append(Bin(start=t, end=bin_end))
            t = bin_end
    return bins

class Section:
    def __init__(self, start, end, density):
        self.start = start
        self.end = end
        self.density = density

class Bin:
    def __init__(self, start, end):
        self.start = start
        self.end = end
```

### Parameters
*   **Batch Size:** Typically 128 bins per batch [S1].
*   **Activation Function:** Softplus with $\beta = 2$ is used instead of sigmoid to allow Gaussians to become fully opaque [S1].
*   **Standard Deviation Factor:** 3 standard deviations are used to define the start and end of sampling sections [S1].

### Edge Cases and Pitfalls
*   **Buffer Overflow:** Sections that fall out of the fixed-size density section buffer are deferred to the next batch [S1].
*   **Opacity Convergence:** Using a constant opacity initialization can prevent convergence for large Gaussian counts [S1]. A power function fitting opacity to the number of Gaussians is recommended for initialization [S1].
*   **Gradient Choice:** The attached gradient variant is computationally more expensive and performs worse than the detached variant [S1].

## Operating characteristics
*   **Performance:** The algorithm is considerably slower than splatting, typically between one and two orders of magnitude slower, depending on the scene [S1].
*   **Accuracy:** It provides accurate resolution of Gaussian overlap and precise perspective projection, serving as a testbed to examine the importance of these factors in 3DGS [S1].
*   **Complexity:** The algorithm requires iterating over the Gaussian mixture twice per batch (once for binning, once for integration), leading to higher computational cost compared to single-pass splatting [S1].
*   **Strengths:** Accurate volumetric rendering, handles overlap correctly, accounts for primitive overlap in integration unlike some raytracing solutions [S1].
*   **Limitations:** Low performance compared to splatting; implementation complexity is higher [S1].

## Background
The OTS Marching Ray Marching Algorithm was introduced in the paper "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" [S1]. It was developed as part of an implementation to study the impact of approximations like spatial overlap and sorting in 3D Gaussian Splatting [S1]. The implementation builds on the original 3DGS code base to enable meaningful comparisons [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
