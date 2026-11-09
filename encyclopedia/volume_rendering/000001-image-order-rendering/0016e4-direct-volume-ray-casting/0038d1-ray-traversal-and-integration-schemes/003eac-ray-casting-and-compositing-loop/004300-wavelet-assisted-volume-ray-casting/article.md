# Wavelet-Assisted Volume Ray Casting

Also called: Index volume wavelet ray casting, Wavelet-assisted fast volume ray casting

## Overview
Wavelet-Assisted Volume Ray Casting is an acceleration technique for direct volume rendering that uses wavelet coefficients to detect local frequency content and determine an adaptive sampling distance along each ray [S1, unofficial]. It solves the problem of excessive computational cost in standard ray casting by avoiding on-the-fly reconstruction from the wavelet domain; instead, it pre-computes an "index volume" that guides the traversal of the original spatial volume [S1, unofficial]. Unlike Westermann's method, which traverses wavelet coefficients directly, this approach uses a flat index volume to skip samples, combining spatial domain rendering with wavelet-based frequency analysis [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A 3D scalar volume $V$ of size $N^3$, a user-specified error bound $E$, and a standard sampling distance $d$.
*   **Output:** A rendered 2D image.
*   **Auxiliary Output:** An index volume $I$ of size $N^3$, where each voxel stores an integer $L$.

### Data Structures

**Index Volume ($I$)**
*   **Layout:** A 3D array with the same resolution ($N^3$) as the spatial volume.
*   **Content:** Each voxel stores an integer $L$, representing the power of 2 by which the standard sampling distance is multiplied.
*   **Size:** $L$ is bounded by $\log_2 N$. The implementation uses 4 bits per voxel [S1, unofficial].
*   **Lookup Table:** A small table stores the actual sampling distance multiplier $2^L$ to avoid runtime exponentiation [S1, unofficial].

**Wavelet Coefficients**
*   The volume is decomposed using the 3D Discrete Wavelet Transform (DWT).
*   Coefficients are organized by scale level $j$ (where $j$ decreases from fine to coarse, e.g., $-1, -2, \dots, -L$).

### Formulas

**Wavelet Basis Functions**
The wavelet basis functions $\psi_{j,n}$ and scaling functions $\phi_{j,n}$ are defined as [S1, unofficial]:
$$ \psi_{j,n} = 2^{-j/2}\psi(2^j t - n) $$
$$ \phi_{j,n} = 2^{-j/2}\phi(2^j t - n) $$

**Coefficients**
The wavelet coefficients $d_{j,k}$ and scaling coefficients $c_{j,k}$ are inner products [S1, unofficial]:
$$ d_{j,k} = \langle f, \psi_{j,k} \rangle $$
$$ c_{j,k} = \langle f, \phi_{j,k} \rangle $$

**Time-Frequency Window**
For a wavelet $\psi_{j,k}$, the time-frequency window is defined as [S1, unofficial]:
$$ [2^{-j} k + 2^{-j} t^* - 2^{-j} \Delta t^*, 2^{-j} k + 2^{-j} t^* + 2^{-j} \Delta t^*] \times [2^j \omega^* - 2^j \Delta \omega^*, 2^j \omega^* + 2^j \Delta \omega^*] $$
The highest frequency represented by wavelets at level $j$ is $2^j \omega^* + 2^j \Delta \omega^*$ [S1, unofficial].

**Sampling Distance Determination**
If the standard sampling distance is $d$, and the magnitudes of all relevant wavelet coefficients from level $-1$ to $-L$ are below the error bound $E$, the appropriate sampling distance is [S1, unofficial]:
$$ d_{sample} = 2^L d $$
Relevance of coefficients is determined by the spatial window or filter support length [S1, unofficial].

### Parameters
*   **Error Bound ($E$):** A user-specified threshold for wavelet coefficient magnitudes. Typical experimental values are 0, 1, 3, 5, 10 on a scale of 0 to 255 [S1, unofficial].
*   **Wavelet Types:** Tested types include Haar (support 2), Daubechies 4 (support 4, orthogonal), and Battle-Lemarie (support 33, B-spline associated) [S1, unofficial].

### Steps

1.  **Preprocessing: Wavelet Transform**
    Apply the 3D discrete wavelet transform to the input volume data. This is implemented by applying the 1D pyramidal algorithm sequentially along the principal axes. The decomposition is separable, resulting in $O(n^3)$ time complexity [S1, unofficial].

2.  **Preprocessing: Index Volume Construction**
    Create an index volume $I$ with the same resolution as the spatial volume. For each voxel:
    a. Identify the relevant wavelet coefficients from levels $-1$ to $-L_{max}$ that affect this voxel (determined by filter support length) [S1, unofficial].
    b. Find the largest integer $L$ such that the magnitudes of all relevant coefficients from level $-1$ to $-L$ are strictly less than $E$ [S1, unofficial].
    c. Store $L$ in the corresponding voxel of $I$ [S1, unofficial].
    *Note: If no coefficients are below $E$ at level $-1$, $L=0$.*

3.  **Rendering: Ray Casting**
    For each pixel, cast a ray through the volume.
    a. Initialize the ray position and step size.
    b. Traverse the spatial volume $V$.
    c. At each current position, read the integer $L$ from the index volume $I$ at the corresponding location [S1, unofficial].
    d. Calculate the next sampling distance as $2^L d$ using the lookup table [S1, unofficial].
    e. Sample the spatial volume $V$ at the new position.
    f. Composite the sample into the pixel buffer (e.g., using front-to-back compositing).
    g. Repeat until the ray exits the volume or early termination criteria are met [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_assisted_volume_rendering(volume, index_volume, error_bound, standard_step, image_width, image_height, camera):
    """
    Renders a volume using wavelet-assisted adaptive sampling.
    
    Args:
        volume: 3D array of voxel values (spatial domain).
        index_volume: 3D array of integers L (same resolution as volume).
        error_bound: Threshold E for coefficient magnitudes (used in preprocessing, 
                     not directly in rendering loop).
        standard_step: Base sampling distance d.
        image_width, image_height: Output image dimensions.
        camera: Object providing ray origin and direction for each pixel.
        
    Returns:
        rendered_image: 2D array of pixel colors.
    """
    # Lookup table for 2^L. Max L is log2(N). 
    # Assuming N <= 256 for 4-bit storage, L_max = 8.
    power_of_two = [2**i for i in range(9)]
    
    rendered_image = zeros((image_height, image_width))
    
    for y in range(image_height):
        for x in range(image_width):
            ray_origin, ray_direction = camera.get_ray(x, y)
            color = [0.0, 0.0, 0.0, 0.0] # RGBA
            
            # Ray traversal parameters
            t = 0.0
            # Initial step might need to be adjusted to align with grid, 
            # but standard approach uses standard_step initially or 
            # reads L from the first voxel.
            # We assume the ray starts outside or at entry.
            
            while t < ray_max_distance:
                # Get current 3D position
                pos = ray_origin + t * ray_direction
                
                # Check if pos is inside volume bounds
                if not is_inside_volume(pos):
                    break
                
                # Get voxel coordinates (integer indices)
                ix, iy, iz = floor(pos)
                
                # Read L from index volume
                L = index_volume[ix, iy, iz]
                
                # Calculate adaptive step size
                step = standard_step * power_of_two[L]
                
                # Sample the spatial volume
                # Interpolation (e.g., trilinear) is performed here
                value = interpolate_volume(volume, pos)
                
                # Compositing (Front-to-Back)
                # Assuming standard volume rendering equation components:
                # alpha = transfer_function(value)
                # color = color + alpha * transfer_color(value) * (1 - accumulated_alpha)
                # accumulated_alpha += alpha * (1 - accumulated_alpha)
                
                # [G] Standard compositing logic placeholder. 
                # The core innovation is the 'step' calculation, not the compositing.
                alpha = compute_alpha(value)
                if alpha > 0:
                    c_val = compute_color(value)
                    color = color + c_val * alpha * (1 - color[3])
                    color[3] += alpha * (1 - color[3])
                
                # Advance ray
                t += step
                
            rendered_image[y, x] = color
            
    return rendered_image

def build_index_volume(volume, error_bound, wavelet_type):
    """
    Preprocesses the volume to create the index volume.
    """
    # 1. Apply 3D DWT
    wavelet_coeffs = apply_3d_dwt(volume, wavelet_type)
    
    # 2. Determine L for each voxel
    # This requires mapping wavelet coefficients back to spatial influence.
    # The paper states relevance is decided by filter support length.
    # For each voxel (ix, iy, iz):
    #   Find max L such that |coeffs| < E for all levels -1 to -L affecting this voxel.
    
    N = volume.shape[0]
    index_volume = zeros((N, N, N), dtype=uint8)
    
    # [G] Detailed coefficient-to-voxel mapping depends on specific DWT implementation.
    # The paper simplifies this by stating the process is linear and based on support.
    # We iterate through the volume to fill the index.
    
    for ix in range(N):
        for iy in range(N):
            for iz in range(N):
                max_L = 0
                # Check levels from coarse to fine? Or fine to coarse?
                # We want the largest L (coarsest resolution) where coeffs are small.
                # Levels are -1, -2, ...
                for level in range(-1, -10, -1): # Arbitrary depth limit
                    # Check if all relevant coefficients at this level are < error_bound
                    if are_coefficients_below_threshold(wavelet_coeffs, level, ix, iy, iz, error_bound):
                        max_L = -level # L is positive integer
                    else:
                        break
                index_volume[ix, iy, iz] = max_L
                
    return index_volume
```

### Edge Cases and Pitfalls
*   **Memory Overhead:** The index volume requires approximately 3 bytes (or 4 bits) per voxel. For high-resolution volumes, this can become a memory bottleneck [S1, unofficial]. Mitigation: Reduce the resolution of the index volume, trading off acceleration rate for memory usage [S1, unofficial].
*   **Reconstruction Cost:** The index volume must be reconstructed whenever the error bound $E$ changes. However, this is a linear process and significantly faster than an inverse wavelet transform [S1, unofficial].
*   **View Independence:** The index volume does not need to be recomputed when view parameters (translation/rotation) change, allowing for fast interactive manipulation [S1, unofficial].
*   **On-the-fly Reconstruction Avoidance:** Direct evaluation in the wavelet domain is expensive due to the large number of coefficients required for reconstruction. This method avoids that cost by staying in the spatial domain for sampling [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Preprocessing:** The 3D DWT decomposition and reconstruction for a volume of size $n^3$ is $O(n^3)$, which is asymptotically optimal [S1, unofficial]. Index volume construction is also linear relative to the volume size [S1, unofficial].
*   **Rendering:** The rendering loop traverses the spatial volume. The number of samples is reduced by the factor $2^L$ in smooth regions.
*   **Memory:** Requires $O(n^3)$ for the spatial volume plus $O(n^3)$ for the index volume (4 bits per voxel) [S1, unofficial].

### Measured Performance
Tests were conducted on an SGI O2 (174MHz Mips R5000, 128MB RAM) with a $64 \times 64 \times 64$ dataset and $150 \times 150$ image resolution [S1, unofficial].

**Rendering Time (seconds):**
| Wavelet | E=0 | E=1 | E=3 | E=5 | E=10 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Standard | 9.58 | - | - | - | - |
| Haar | 6.90 | 5.20 | 3.94 | 3.23 | 2.26 |
| Daubechies 4 | 8.16 | 4.27 | 3.22 | 2.69 | 2.19 |
| Battle-Lemarie | 10.28 | 3.80 | 2.33 | 1.72 | 1.44 |

*   With Haar wavelets and $E=10$, time was 2.26s (76% saving) [S1, unofficial].
*   With Battle-Lemarie wavelets and $E=10$, time was 1.44s (88% saving) [S1, unofficial].
*   Index volume construction took 0.05s [S1, unofficial].

**Mean Square Error (MSE) vs Reference:**
| Wavelet | E=0 | E=10 |
| :--- | :--- | :--- |
| Haar | 0.044 | 1.760 |
| Daubechies 4 | 0.027 | 11.071 |
| Battle-Lemarie | 0.026 | 10.91 |

*   Smoother wavelets (larger support) generally yield faster rendering for the same error bound because they are better at detecting local frequency changes, but they may omit more energy for a given $E$ [S1, unofficial].
*   When $E=0$, images are almost identical to the reference, with maximum time savings of 28% attributed to empty space skipping [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Avoids expensive on-the-fly wavelet reconstruction; avoids octree traversal overhead; allows controlled approximation quality via $E$; index volume is view-independent.
*   **Limitations:** Memory overhead for the index volume; index volume reconstruction is needed if $E$ changes; performance depends heavily on the choice of wavelet and the error bound [S1, unofficial].

## Background

### Origin
The method was proposed by researchers from Bell Labs Research Department in a paper titled "Wavelet-Assisted Volume Ray Casting" [S1, unofficial]. It was presented in the PSB '98 proceedings (Proceedings of the Pacific Symposium on Biocomputing) [S1, unofficial].

### History and Development
Volume rendering has been critical in computational biology and medical applications, but high-resolution volumes require significant computational resources [S1, unofficial]. Previous work by Westermann used wavelet coefficients to determine sampling distances but required traversing the wavelet domain, which was slow [S1, unofficial]. This method improves upon that by pre-computing an index volume, allowing the ray casting to remain in the spatial domain while still benefiting from wavelet-based frequency analysis [S1, unofficial]. It builds on the concept that appropriate sampling rates are determined by local maximum frequency, a idea previously explored by Danskin and Hanrahan using Monte Carlo methods [S1, unofficial].

### Usage
The algorithm was implemented within VolVis, a public domain volume visualization system developed at the State University of New York at Stony Brook [S1, unofficial]. It is applicable to datasets in computational biology and medical imaging [S1, unofficial].

## Sources

[S1] Wavelet-Assisted Volume Ray Casting, https://psb.stanford.edu/psb-online/proceedings/psb98/he.pdf, fetched 2026-10-28 (unofficial)
