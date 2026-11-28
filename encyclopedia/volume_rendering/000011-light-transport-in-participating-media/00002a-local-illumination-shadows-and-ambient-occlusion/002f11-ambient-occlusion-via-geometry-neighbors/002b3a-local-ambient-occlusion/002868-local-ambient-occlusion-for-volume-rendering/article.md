# Local Ambient Occlusion for Volume Rendering

Also called: Ambient Occlusion Volume Rendering with Local Vicinity Statistics, High-Speed Ambient Occlusion Volume Rendering, Local AO, Local Occlusion, Probabilistic Ambient Occlusion for Volume Rendering, Volumetric Local Ambient Occlusion

## Overview
Local Ambient Occlusion for Volume Rendering computes shadowing by structures in the immediate vicinity of each voxel, avoiding the computational cost of global illumination while preventing regions from being fully shadowed by distant dense objects (e.g., a skull occluding a tumor) [S1]. It replaces the standard ambient term in the volume rendering equation with a local approximation that considers voxels in a neighborhood around each voxel [S2]. Unlike sibling methods that may rely on ray casting or pure geometry, this variant specifically assumes CT values in the local vicinity follow a normal distribution to estimate the probability of occlusion, allowing for high-speed rendering via piecewise linear Transfer Function (TF) integration [S3]. It provides better spatial and depth cues than local illumination alone [S3].

## How it works
The method operates in two phases: a preprocessing step to compute local statistics and a rendering step to compute the occlusion factor [S3].

### Inputs and Outputs
*   **Input:** Volume data $V$ (scalar grid, e.g., CT values), Transfer Function $TF$ (defined as $k$ connected line segments), local neighborhood window size $n \times n \times n$ [S3], and ambient bias $I_{bias}$ [S1].
*   **Output:** An occlusion intensity map $I(x)$ or per-sample occlusion factor $AO(x)$ for each voxel $x$, which modulates the final rendered color [S1, S3].

### Data Structures
1.  **Volume Data:** 3D grid of scalar values [S3].
2.  **Local Statistics Map:** For each voxel $x$, store the mean $\mu_x$ and standard deviation $\sigma_x$ of the CT values in the local vicinity [S3].
    *   *Layout:* Two 3D arrays of floats, same dimensions as $V$ [G].
    *   *Computation:* Calculated in constant time per voxel using integral images or box filters for the mean and squared mean [S3].
3.  **Transfer Function:** Defined as $k$ piecewise linear segments [S3].
    *   *Layout:* Array of $k$ structs, each containing coefficients $(a_i, b_i)$ and bounds $(n_i, n_{i+1})$ [S3].
    *   *Formula:* $TF(x) = a_i x + b_i$ for $n_i \le x < n_{i+1}$ [S3].
4.  **Lookup Table (LUT):** Precomputed values for the cumulative standard normal distribution $N(x)$ for $O(1)$ access [S3].

### Numbered Steps

1.  **Preprocessing: Compute Local Statistics** [S3]
    For each voxel $x$ at coordinates $(s, t, u)$:
    *   Define a cubic window of size $n \times n \times n$ centered at $x$ [S3].
    *   Calculate the average CT value $\mu_x$ and the average squared CT value $SqAvg_x$ within this window [S3].
    *   Compute the standard deviation $\sigma_x = \sqrt{SqAvg_x - \mu_x^2}$ [S3].
    *   Store $\mu_x$ and $\sigma_x$ in the statistics map [S3].

2.  **Rendering: Compute Ambient Occlusion Factor** [S3]
    For each sample point $x$ during volume rendering:
    *   Retrieve $\mu_x$ and $\sigma_x$ from the statistics map [S3].
    *   Assume the CT values in the vicinity follow a normal distribution $\rho(x) = N(\mu_x, \sigma_x^2)$ [S3].
    *   Calculate the occlusion factor $AO(x)$ by integrating the TF opacity $\alpha(x)$ against the probability density $\rho(x)$:
        $$ AO(x) = \int_{-\infty}^{\infty} \alpha(x) \rho(x) dx $$
    *   Since $\alpha(x)$ is piecewise linear, decompose the integral into $k$ segments:
        $$ AO(x) = \sum_{i=1}^{k} Seg_i $$
    *   For each segment $i$ with coefficients $(a_i, b_i)$ and bounds $(n_i, n_{i+1})$:
        *   Transform the segment coefficients and bounds to the standard normal space $N(0,1)$ using the normalized variable $z = \frac{x - \mu_x}{\sigma_x}$:
            $$ a'_i = \sigma_x a_i $$
            $$ b'_i = b_i + a_i \mu_x $$
            $$ n'_i = \frac{n_i - \mu_x}{\sigma_x} $$
            $$ n'_{i+1} = \frac{n_{i+1} - \mu_x}{\sigma_x} $$
        *   Calculate the contribution $Seg_i$ using the precomputed cumulative normal distribution $N(\cdot)$:
            $$ Seg_i = \frac{a'_i}{\sqrt{2\pi}} \left( e^{-n_i'^2/2} - e^{-n_{i+1}'^2/2} \right) + b'_i \left( N(n'_{i+1}) - N(n'_i) \right) $$
            *(Note: The source paper contains a typographical error in Eq. 7 writing $2\pi$ instead of $\sqrt{2\pi}$ in the denominator; the corrected formula above is used for implementation [S3]).*
    *   The resulting $AO(x)$ is a value where closer to 1.0 means darker (more occlusion) [S3].

3.  **Final Shading**
    *   The occlusion factor modulates the lighting [S1, S3]. In the basic model, it replaces the ambient term [S2]. In the emissive model, it combines with light emission [S1].
    *   If using the ray-casting variant (LAO family general approach), the intensity $I(x)$ is computed as:
        $$ I(x) = I_{bias} + \frac{1}{K} \sum_{k=1}^{K} w_k I_k(x) $$
        where $I_k(x)$ is the incident light from ray $k$, computed via front-to-back compositing [S1].
    *   For the specific "Local Vicinity Statistics" method, the $AO(x)$ calculated in Step 2 is used directly as the occlusion multiplier for the diffuse/ambient component [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def N(x):
    """Cumulative standard normal distribution using the error function."""
    return 0.5 * (1.0 + math.erf(x / math.sqrt(2.0)))

def compute_local_stats(volume, window_size):
    """
    Preprocess volume to compute mean and std dev for each voxel's neighborhood.
    Uses a 3D integral image for O(1) sum queries.
    """
    height, width, depth = volume.shape
    half_w = window_size // 2
    
    # Initialize integral images with padding
    int_img = [[[0.0 for _ in range(depth + 1)] for _ in range(width + 1)] for _ in range(height + 1)]
    int_sq_img = [[[0.0 for _ in range(depth + 1)] for _ in range(width + 1)] for _ in range(height + 1)]
    
    # Build integral images
    for x in range(height):
        for y in range(width):
            for z in range(depth):
                val = volume[x][y][z]
                int_img[x+1][y+1][z+1] = val + int_img[x][y+1][z+1] + int_img[x+1][y][z+1] + int_img[x+1][y+1][z] - int_img[x][y][z+1] - int_img[x][y+1][z] - int_img[x+1][y][z] + int_img[x][y][z]
                int_sq_img[x+1][y+1][z+1] = val*val + int_sq_img[x][y+1][z+1] + int_sq_img[x+1][y][z+1] + int_sq_img[x+1][y+1][z] - int_sq_img[x][y][z+1] - int_sq_img[x][y+1][z] - int_sq_img[x+1][y][z] + int_sq_img[x][y][z]

    mean_map = [[[0.0 for _ in range(depth)] for _ in range(width)] for _ in range(height)]
    std_map = [[[0.0 for _ in range(depth)] for _ in range(width)] for _ in range(height)]
    
    # Query windows
    for x in range(height):
        for y in range(width):
            for z in range(depth):
                x1, x2 = max(0, x - half_w), min(height, x + half_w + 1)
                y1, y2 = max(0, y - half_w), min(width, y + half_w + 1)
                z1, z2 = max(0, z - half_w), min(depth, z + half_w + 1)
                
                count = (x2 - x1) * (y2 - y1) * (z2 - z1)
                
                # 3D integral image query
                sum_val = (int_img[x2][y2][z2] - int_img[x1][y2][z2] - int_img[x2][y1][z2] - int_img[x2][y2][z1] + int_img[x1][y1][z2] + int_img[x1][y2][z1] + int_img[x2][y1][z1] - int_img[x1][y1][z1])
                sum_sq = (int_sq_img[x2][y2][z2] - int_sq_img[x1][y2][z2] - int_sq_img[x2][y1][z2] - int_sq_img[x2][y2][z1] + int_sq_img[x1][y1][z2] + int_sq_img[x1][y2][z1] + int_sq_img[x2][y1][z1] - int_sq_img[x1][y1][z1])
                
                mu = sum_val / count
                sigma = math.sqrt((sum_sq / count) - mu**2)
                mean_map[x][y][z] = mu
                std_map[x][y][z] = sigma
    return mean_map, std_map

def compute_ao_factor(mu, sigma, tf_segments):
    """
    Compute Ambient Occlusion factor for a voxel given local statistics and TF.
    """
    ao = 0.0
    for seg in tf_segments:
        a_i, b_i = seg['a'], seg['b']
        n_i, n_next = seg['n_start'], seg['n_end']
        
        if sigma == 0.0:
            # Delta function case: distribution is a point mass at mu
            if n_i <= mu < n_next:
                ao += a_i * mu + b_i
            continue
            
        a_prime = sigma * a_i
        b_prime = b_i + a_i * mu
        n_prime_i = (n_i - mu) / sigma
        n_prime_next = (n_next - mu) / sigma
        
        pdf_i = math.exp(-n_prime_i**2 / 2.0) / math.sqrt(2.0 * math.pi)
        pdf_next = math.exp(-n_prime_next**2 / 2.0) / math.sqrt(2.0 * math.pi)
        
        term1 = a_prime * (pdf_i - pdf_next)
        term2 = b_prime * (N(n_prime_next) - N(n_prime_i))
        ao += term1 + term2
    return ao
```

### Parameters
*   **Window Size:** The size of the local vicinity ($n \times n \times n$) [S3]. A window size of 10 gradually stabilizes the trade-off between rendering speed and image quality [S3].
*   **TF Segments ($k$):** The number of line segments in the piecewise linear TF [S3]. Performance is proportional to $k$ [S3].
*   **Ambient Bias ($I_{bias}$):** Specifies the amount of minimum lighting [S1].
*   **Ray Weights ($w_k$):** In the ray-casting variant, weights in $[0, 1]$ enable directional weighting [S1].

### Edge Cases and Pitfalls
*   **Distribution Mismatch:** The method assumes CT values follow a normal distribution [S3]. If the actual data distribution is binary or bimodal with the same mean and standard deviation, the lighting will be inaccurate (e.g., regions with different transparent voxel counts but identical stats receive the same AO) [S3].
*   **Zero Standard Deviation:** If $\sigma_x = 0$ (homogeneous region), the transformation to standard normal space involves division by zero [G].
*   **TF Changes:** The multiresolution data structure (or statistics map) must be updated whenever the transfer function changes if the method relies on precomputed intensity maps [S1, S2]. However, the statistical method computes AO per-sample using the current TF, so it handles dynamic TFs naturally without full recomputation of the map, only the per-voxel stats which are TF-independent [S3].
*   **Sharp Shadows:** In ray-casting variants, contributing light only at the boundary of $\Omega$ causes sharp shadows [S1].

## Subtopics

### Ramp Function for AO Intensity
A method to scale AO intensity based on fragment opacity using a linear ramp function, replacing a single threshold value [S4, unofficial]. It uses a "Volume opacity threshold" (center of the ramp) and "Volume opacity window" (width of the ramp) [S4, unofficial]. The AO intensity factor is 0 if fragment opacity $\le$ threshold - window/2, 1 if fragment opacity $\ge$ threshold + window/2, and linearly interpolated between these bounds [S4, unofficial]. This helps prevent semi-transparent surfaces from appearing fully opaque while allowing structures behind to change color [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The statistical method reduces the number of iterations from 4096 (for 12-bit CT precision) to $k$ (the number of TF segments) [S3]. With TF simplification, $k$ can be reduced to dozens or hundreds [S3].
*   **Performance:** Achieves approximately 40 times faster rendering compared to unaccelerated methods with a window size of 8 [S3]. Signal-to-Noise Ratio (SNR) is approximately 40 dB, visually indistinguishable from the original [S3].
*   **Memory Complexity:** Requires storage for the volume data, the mean map, and the standard deviation map. The multiresolution data structure size depends on the volume size and homogeneity [S2, S3].
*   **Strengths:** Provides better spatial and depth cues (e.g., human ear depth, vertebrae spaces) than local illumination alone [S3]. Real-time rendering is possible [S3]. Handles hand-drawn TFs [S3].
*   **Limitations:** Accuracy degrades if the local CT value distribution is not normal [S3]. Image quality degrades if the window size is too large [S3].

## Background
Local Ambient Occlusion for Volume Rendering originated from the need to provide realistic lighting in volume rendering without the cost of global illumination [S1]. The concept of local occlusion was introduced by Hernell et al. in 2007 [S1]. The specific statistical approach using local vicinity statistics and normal distribution assumptions for high-speed rendering was proposed by researchers in 2019 [S3]. It builds on the general Local Ambient Occlusion family [4977-local-ambient-occlusion.md].

## Sources
[S1] Efficient Ambient and Emissive Tissue Illumination using Local Occlusion in Multiresolution Volume Rendering, https://doi.org/10.2312/vg/vg07/001-008, fetched 2026-10-26
[S2] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S3] Real-Time Computed Tomography Volume Visualization with Ambient Occlusion of Hand-Drawn Transfer Function Using Local Vicinity Statistic., https://doi.org/10.4258/hir.2019.25.4.297, fetched 2026-10-26
[S4] Improve ambient occlusion in volume rendering - 3D Slicer Community, https://discourse.slicer.org/t/improve-ambient-occlusion-in-volume-rendering/33590, fetched 2026-10-27 (unofficial)
