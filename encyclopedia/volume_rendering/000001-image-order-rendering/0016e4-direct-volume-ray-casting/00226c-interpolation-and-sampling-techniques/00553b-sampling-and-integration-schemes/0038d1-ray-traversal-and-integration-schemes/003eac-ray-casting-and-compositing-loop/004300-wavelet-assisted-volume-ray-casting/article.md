# Wavelet-Assisted Volume Ray Casting

Also called: Index volume wavelet ray casting, Wavelet-assisted fast volume ray casting

## Overview
Wavelet-Assisted Volume Ray Casting is an acceleration algorithm for direct volume rendering that uses wavelet coefficients to detect local signal frequency and determine the appropriate adaptive sampling distance along each ray [S1, unofficial]. It solves the computational bottleneck of standard ray casting by skipping samples in low-frequency regions while maintaining high sampling rates in high-frequency areas, avoiding the expensive on-the-fly reconstruction from the wavelet domain [S1, unofficial]. Unlike [Accumulative Rendering](14504-accumulative-rendering.md) which focuses on compositing order, or [Template-based Ray-Casting](14511-template-based-ray-casting.md) which uses geometric templates, this method relies on a precomputed "index volume" to guide spatial traversal, offering a framework for controlled approximation of rendering quality [S1, unofficial].

## How it works
**Inputs:**
- Input volume data $V$ of size $N^3$ (spatial domain).
- User-specified error bound $E$ (threshold for wavelet coefficient magnitudes).
- Standard sampling distance $d$ (base step size for uniform ray casting).
- Wavelet type (e.g., Haar, Daubechies 4, Battle-Lemarie).

**Outputs:**
- Rendered image with adaptive sampling.
- Index volume $I$ (preprocessed).

**Data Structures:**
1.  **Index Volume ($I$):** A 3D array with the same resolution as the spatial volume $V$. Each voxel stores an integer $L$, representing the power of 2 by which the standard sampling distance $d$ is multiplied.
    -   **Layout:** Integer field per voxel.
    -   **Size:** $L$ is at most $\log_2 N$. In implementation, 4 bits per voxel are sufficient [S1, unofficial].
    -   **Lookup Table:** A small table maps $L$ to the multiplier $2^L$.
2.  **Wavelet Coefficients:** Stored in memory during preprocessing to compute $I$. Not accessed during rendering [S1, unofficial].

**Formulas:**
-   **Wavelet Basis Functions:** $\psi_{j,n} = 2^{-j/2}\psi(2^j t - n)$ [S1, unofficial].
-   **Scaling Functions:** $\phi_{j,n} = 2^{-j/2}\phi(2^j t - n)$ [S1, unofficial].
-   **Coefficients:** $d_{j,k} = \langle f, \psi_{j,k} \rangle$ and $c_{j,k} = \langle f, \phi_{j,k} \rangle$ [S1, unofficial].
-   **Time-Frequency Window:** For a wavelet $\psi_{j,k}$, the window is $[2^{-j} k + 2^{-j} t^* - 2^{-j} \Delta t^*, 2^{-j} k + 2^{-j} t^* + 2^{-j} \Delta t^*] \times [2^j \omega^* - 2^j \Delta \omega^*, 2^j \omega^* + 2^j \Delta \omega^*]$ [S1, unofficial].
-   **Highest Frequency:** $2^j \omega^* + 2^j \Delta \omega^*$ [S1, unofficial].
-   **Sampling Distance Rule:** If the magnitudes of all relevant wavelet coefficients from level $-1$ to $-L$ are below $E$, the sampling distance is $2^L d$ [S1, unofficial].
-   **Relevance:** Determined by the time (space) window or filter support length [S1, unofficial].

**Parameters:**
-   **Error Bound ($E$):** Threshold for coefficient magnitudes. Typical values: 0, 1, 3, 5, 10 (on a scale of 0–255) [S1, unofficial].
-   **Wavelet Types:** Haar (support 2), Daubechies 4 (support 4, orthogonal), Battle-Lemarie (support 33, B-spline associated) [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing: Wavelet Transform**
    Apply the 3D discrete wavelet transform (DWT) to the input volume $V$. This decomposes the volume into smooth approximations and detail signals along seven orientations [S1, unofficial]. The decomposition is implemented in $O(N^3)$ time [S1, unofficial].

2.  **Preprocessing: Index Volume Construction**
    Create the index volume $I$ with the same resolution as $V$. For each voxel, determine the maximum level $L$ such that the magnitudes of all relevant wavelet coefficients from level $-1$ to $-L$ affecting that voxel are below the error bound $E$.
    -   Relevance is determined by the filter support length or time window [S1, unofficial].
    -   Store $L$ in $I$. If no coefficients below level $-1$ are below $E$, $L=0$ (standard distance $d$) [S1, unofficial].
    -   This process is linear and fast compared to inverse DWT [S1, unofficial].

3.  **Rendering: Ray Traversal**
    For each pixel, cast a ray through the volume.
    -   Initialize ray position and step counter.
    -   While the ray is within the volume bounds:
        a.  Read the index value $L$ from the index volume $I$ at the current ray position.
        b.  Calculate the sampling distance: $step = 2^L \times d$.
        c.  Sample the original spatial volume $V$ at the current position using standard interpolation (e.g., trilinear) [S1, unofficial].
        d.  Perform compositing (e.g., front-to-back alpha blending) [S1, unofficial].
        e.  Advance the ray position by $step$.
        f.  Check for early termination (e.g., opacity threshold) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_assisted_volume_ray_casting(volume, index_volume, error_bound, standard_step, ray_origin, ray_direction, image_width, image_height):
    """
    Renders a volume using wavelet-assisted adaptive sampling.
    
    Args:
        volume: 3D array of voxel values (spatial domain).
        index_volume: 3D array of integers L (same resolution as volume).
        error_bound: Threshold E (not used in rendering, only for index construction).
        standard_step: Base sampling distance d.
        ray_origin: 3D vector.
        ray_direction: 3D vector (normalized).
        image_width: int.
        image_height: int.
        
    Returns:
        rendered_image: 2D array.
    """
    rendered_image = zeros((image_height, image_width))
    
    # Precompute power lookup table for 2^L
    # L is at most log2(N). Assuming N <= 256 for 4-bit storage, max L is 8.
    power_table = [1 << i for i in range(9)] 
    
    for y in range(image_height):
        for x in range(image_width):
            # Compute ray parameters for this pixel
            # Note: Exact ray generation depends on camera model, assumed here as generic
            current_ray_origin = ray_origin
            current_ray_direction = ray_direction
            
            # Initialize compositing state
            accumulated_color = 0.0
            accumulated_opacity = 0.0
            
            # Ray traversal
            t = 0.0
            while t < max_distance: # max_distance depends on volume bounds
                # Get current 3D position
                pos = current_ray_origin + t * current_ray_direction
                
                # Check if pos is inside volume bounds
                if not is_inside_volume(pos):
                    break
                
                # Get index L from index volume
                # Interpolate index volume if necessary, or use nearest neighbor
                # The source implies direct access, assuming integer coordinates or simple mapping
                voxel_coords = get_voxel_coords(pos)
                L = index_volume[voxel_coords]
                
                # Calculate step size
                step = power_table[L] * standard_step
                
                # Sample spatial volume
                voxel_value = sample_volume(volume, pos)
                
                # Compositing step (simplified front-to-back)
                # Assuming constant opacity per sample for simplicity, 
                # real implementation uses transfer function
                alpha = compute_alpha(voxel_value)
                color = compute_color(voxel_value)
                
                # Standard compositing equation: C_out = C_in + (1 - O_in) * C_sample * alpha_sample
                # O_out = O_in + (1 - O_in) * alpha_sample
                # Note: The source does not specify the exact compositing formula, 
                # but states it is "volume ray casting" which implies standard compositing.
                # [G] Using standard front-to-back compositing.
                weight = (1.0 - accumulated_opacity) * alpha
                accumulated_color += weight * color
                accumulated_opacity += weight
                
                # Early termination check
                if accumulated_opacity > 0.99:
                    break
                
                # Advance ray
                t += step
                
            rendered_image[y, x] = accumulated_color
            
    return rendered_image

def is_inside_volume(pos):
    # Placeholder for volume boundary check
    # [G] Implementation depends on volume bounding box
    return True

def get_voxel_coords(pos):
    # Placeholder for coordinate to voxel index mapping
    # [G] Implementation depends on volume grid alignment
    return (0, 0, 0)

def sample_volume(volume, pos):
    # Placeholder for trilinear interpolation
    # [G] Standard interpolation
    return 0.0

def compute_alpha(value):
    # Placeholder for transfer function
    # [G] Standard transfer function
    return 0.0

def compute_color(value):
    # Placeholder for transfer function
    # [G] Standard transfer function
    return 0.0
```

## Subtopics

### [Direct Summation of Wavelets](15650-direct-summation-of-wavelets.md)
Evaluates the radiative transfer equation by directly summing wavelet coefficients during ray casting, avoiding resampling to a uniform grid. It uses wavelet coefficients as the primary data structure, offering better image quality than VTK's projected tetrahedra but slower speeds than VTK's ZSweep renderer. Choose this when memory overhead of unstructured grids is a concern and direct wavelet evaluation is feasible.

## Operating characteristics
-   **Time Complexity:** The 3D DWT decomposition and reconstruction is $O(N^3)$, which is asymptotically optimal [S1, unofficial]. The rendering speed depends on the error bound $E$ and wavelet type.
-   **Memory Complexity:** Requires approximately 3 bytes (or 4 bits per voxel) of extra memory for the index volume per $N^3$ volume [S1, unofficial]. For high-resolution volumes, this can be a bottleneck; mitigation includes reducing the index volume resolution [S1, unofficial].
-   **Measured Performance:** On an SGI O2 (174MHz Mips R5000, 128MB RAM) with a 64x64x64 dataset and 150x150 image resolution:
    -   Standard rendering: 9.58s [S1, unofficial].
    -   Haar wavelets, $E=10$: 2.26s (76% saving) [S1, unofficial].
    -   Battle-Lemarie wavelets, $E=10$: 1.44s (88% saving) [S1, unofficial].
    -   Index volume construction: 0.05s [S1, unofficial].
-   **Image Quality (MSE vs Reference):**
    -   Haar, $E=0$: 0.044; $E=10$: 1.760 [S1, unofficial].
    -   Daubechies 4, $E=0$: 0.027; $E=10$: 11.071 [S1, unofficial].
    -   Battle-Lemarie, $E=0$: 0.026; $E=10$: 10.91 [S1, unofficial].
-   **Strengths:** Avoids expensive on-the-fly reconstruction from the wavelet domain; avoids octree traversal overhead; provides controlled approximation of quality via $E$; index volume does not need recomputation for view changes (translation/rotation) [S1, unofficial].
-   **Limitations:** Memory overhead for index volume; reconstruction of index volume is required when $E$ changes (though linear and fast); smoother wavelets with larger support may omit more energy for a given $E$ [S1, unofficial].
-   **When to Use:** When rendering large volumes where adaptive sampling can significantly reduce sample counts, and memory allows for an auxiliary index volume.
-   **When Not to Use:** When memory is extremely constrained and cannot accommodate the index volume, or when view-dependent features require frequent recomputation of the sampling strategy (though view rotation/translation does not require recomputation) [S1, unofficial].

## Background
The method was proposed by researchers from Bell Labs and the State University of New York at Stony Brook in 1998 [S1, unofficial]. It builds on the concept of using wavelet transforms for volume visualization, previously explored by Muraki and Gross et al. [S1, unofficial]. The algorithm addresses the limitations of Westermann's method, which required traversing wavelet coefficients and on-the-fly reconstruction, and traditional pyramid/octree structures which involved expensive traversal [S1, unofficial]. It leverages the time-frequency locality of wavelets, which provide variable window widths (narrow for high frequencies, wide for low frequencies), making them superior to Short-Time Fourier Transform (STFT) for detecting local frequency changes [S1, unofficial].

## Sources
[S1] Wavelet-Assisted Volume Ray Casting, https://psb.stanford.edu/psb-online/proceedings/psb98/he.pdf, fetched 2026-10-28 (unofficial)
