# Direct Summation of Wavelets

Also called: Direct Wavelet Summation, Direct summation approach, Direct summation of wavelet coefficients, Direct wavelet summation on adaptive mesh, Wavelet coefficient summation during volumetric ray casting, Wavelet space ray casting, Wavelet-based ray casting

## Overview
Direct Summation of Wavelets is a volume rendering technique that evaluates the radiative transfer equation by directly summing wavelet coefficients during ray casting, avoiding the need to convert the wavelet-based grid into an unstructured mesh [S1][S2, unofficial]. Unlike [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md), which uses wavelets primarily for adaptive sampling rate decisions, this method reconstructs the field value $f(x)$ on-the-fly by summing the contributions of scaling and wavelet functions, allowing for interactive rendering of coarse details with optional fine-level detail addition [S1][S2, unofficial]. It provides better image quality than VTK's projected tetrahedra renderer and comparable quality to ZSweep, while requiring significantly less memory and working successfully on large data sets where VTK-based approaches fail due to memory constraints [S1][S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Wavelet-compressed volume data represented on an adaptive mesh, consisting of scaling coefficients at a coarse level and wavelet coefficients at finer levels [S2, unofficial].
*   Ray parameters: origin, direction, and step size $d_i$ [S1][S2, unofficial].
*   Threshold value $\epsilon$ for coefficient filtering [S1][S2, unofficial].
*   Cutoff limits for function support: $\xi_0 = 1$ for scaling functions, $\xi_0 = 2$ for wavelet functions [S1][S2, unofficial].

**Outputs:**
*   Rendered image pixels, where each pixel value $I$ is the result of the discretized radiative transfer equation [S1][S2, unofficial].

### Data Structures
The data is stored in a linear array of coarse mesh nodes. The coarse grid resolution is typically $N^3$ with $N=64$ [S1][S2, unofficial].

**CoarseMeshNode** (array of size $N \times N \times N$):
*   `float scaling_coefficients[8]`: Values of the 8 nearest scaling coefficients. This redundancy improves cache coherence, providing ~30% acceleration [S1][S2, unofficial].
*   `WaveletNode* wavelet_node_array`: Pointer to a dynamically allocated array of wavelet coefficients associated with this coarse block [S1][S2, unofficial].
*   `int number_of_wlt_nodes_of_level[J-1]`: Array storing the count of wavelet nodes per level [S1]. *Note: [S2, unofficial] describes this as a single length field for the adaptive array, sorted by level and type. We implement the structure described in [S1] as it provides more specific layout details, but note the sorting requirement from [S2, unofficial].*

**WaveletNode** (elements in `wavelet_node_array`):
*   `float wavelet_coefficient`: The coefficient value $d_{k\mu,j}$ [S1][S2, unofficial].
*   `float wavelet_coordinates[3]`: The spatial coordinates of the wavelet center [S1][S2, unofficial].
*   `int wavelet_type`: The type of the wavelet (e.g., family index $\mu$). Storing this is redundant but saves computation time during summation, providing ~10% additional speedup [S1][S2, unofficial].

The `wavelet_node_array` is sorted by wavelet levels and types to increase cache coherence [S1][S2, unofficial].

### Algorithm Steps

1.  **Preprocessing (Partial Inverse Transform):**
    Perform a partial inverse wavelet transform to obtain scaling functions on a coarse grid at a user-prescribed resolution (e.g., $64^3$) [S1][S2, unofficial]. This step is performed in a fraction of a second [S1][S2, unofficial].

2.  **Coefficient Filtering:**
    Identify wavelet coefficients to include in the summation. A coefficient is included if its magnitude $|d_{k\mu,j}| > \epsilon \cdot \|f\|$ (where $\|f\|$ is the dimensional scale of $f$) [S2, unofficial]. The threshold $\epsilon$ is typically 0.01 [S1][S2, unofficial]. The error in determining $f(x)$ using this truncated sum is $O(\epsilon)$ [S1][S2, unofficial].

3.  **Ray Initialization:**
    For each pixel in the image (e.g., $500 \times 500$), cast a ray through the volume [S1][S2, unofficial].

4.  **Coarse Image Computation (Interactive):**
    For each point $x$ along the ray, compute the base function value $f_{coarse}(x)$ using linear interpolation from the nearest 8 coarse mesh scaling coefficients stored in `CoarseMeshNode` [S1][S2, unofficial].
    *   Access the `scaling_coefficients[8]` from the relevant coarse mesh node.
    *   Perform trilinear interpolation to obtain $f_{coarse}(x)$ [S1][S2, unofficial].
    *   This step is interactive [S1][S2, unofficial].

5.  **Detail Addition (Non-Interactive in current implementation):**
    Compute the additional detail $\Delta f(x)$ by summing the contributions of fine-level wavelet coefficients whose support overlaps with $x$ and whose magnitude exceeds $\epsilon$ [S1][S2, unofficial].
    *   Iterate through the `wavelet_node_array` of the coarse block containing $x$.
    *   For each `WaveletNode`, check if the wavelet support overlaps with $x$. The support is defined by cutoff limits: scaling functions are zero for argument $> 1$, wavelet functions are zero for argument $> 2$ [S1][S2, unofficial].
    *   If overlapping, compute the wavelet function value $\psi(x)$ using table lookup [S1][S2, unofficial].
    *   Add $d_{k\mu,j} \cdot \psi(x)$ to $\Delta f(x)$ [S1][S2, unofficial].
    *   The total function value is $f(x) = f_{coarse}(x) + \Delta f(x)$ [S1][S2, unofficial].

6.  **Radiative Transfer Integration:**
    Compute the color $C_i$ and opacity $\alpha_i$ from $f(x_i)$ using transfer functions $C(f(x))$ and $\alpha(f(x))$ [S1][S2, unofficial].
    Approximate the rendering integral $I$ using front-to-back compositing:
    $$ I \approx \sum_{i=0}^{n(\alpha)} C_i \alpha_i d_i \prod_{j=0}^{i-1} (1 - \alpha_j d_j) $$
    where:
    *   $i$ is the ray point index.
    *   $d_i$ is the constant step size along the ray [S1][S2, unofficial].
    *   $n(\alpha)$ is the total number of points along the ray [S1][S2, unofficial].
    *   $C_i$ is the color at point $i$ [S2, unofficial].
    *   $\alpha_i$ is the opacity at point $i$ [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
COARSE_GRID_SIZE = 64
THRESHOLD_EPSILON = 0.01
SCALING_CUTOFF = 1.0
WAVELET_CUTOFF = 2.0

# Data Structures (Conceptual)
class WaveletNode:
    def __init__(self, coefficient, coordinates, wavelet_type):
        self.coefficient = coefficient
        self.coordinates = coordinates # [x, y, z]
        self.wavelet_type = wavelet_type

class CoarseMeshNode:
    def __init__(self, scaling_coeffs, wavelet_nodes):
        self.scaling_coefficients = scaling_coeffs # Array of 8 floats
        self.wavelet_nodes = wavelet_nodes # List of WaveletNode, sorted by level/type

# Global Data
coarse_mesh = [None] * (COARSE_GRID_SIZE ** 3) # Initialized in preprocessing
# Table for scaling and wavelet functions
scaling_table = {} 
wavelet_table = {}

def get_scaling_value(x, y, z):
    """
    Interpolate from the 8 nearest coarse mesh scaling coefficients.
    Assumes x, y, z are in grid coordinates [0, COARSE_GRID_SIZE-1].
    """
    # Determine the 8 surrounding grid points
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    
    # Get the 8 coefficients. 
    # Note: In a real implementation, boundary checks are needed.
    # The CoarseMeshNode stores the 8 nearest coefficients directly.
    # We assume the node corresponding to the 'min' corner (ix, iy, iz)
    # contains the relevant 8 coefficients for trilinear interpolation.
    node = coarse_mesh[iz * COARSE_GRID_SIZE * COARSE_GRID_SIZE + iy * COARSE_GRID_SIZE + ix]
    coeffs = node.scaling_coefficients
    
    # Trilinear interpolation formula
    # coeffs layout is assumed to be standard octant order:
    # 0: (0,0,0), 1: (1,0,0), 2: (0,1,0), 3: (1,1,0), 
    # 4: (0,0,1), 5: (1,0,1), 6: (0,1,1), 7: (1,1,1)
    c000 = coeffs[0]; c100 = coeffs[1]; c010 = coeffs[2]; c110 = coeffs[3]
    c001 = coeffs[4]; c101 = coeffs[5]; c011 = coeffs[6]; c111 = coeffs[7]
    
    # Interpolate in x
    c00 = c000 * (1 - fx) + c100 * fx
    c10 = c010 * (1 - fx) + c110 * fx
    c01 = c001 * (1 - fx) + c101 * fx
    c11 = c011 * (1 - fx) + c111 * fx
    
    # Interpolate in y
    c0 = c00 * (1 - fy) + c10 * fy
    c1 = c01 * (1 - fy) + c11 * fy
    
    # Interpolate in z
    return c0 * (1 - fz) + c1 * fz

def get_wavelet_value(wavelet_node, x, y, z):
    """
    Compute wavelet function value using table lookup and cutoff.
    """
    # Calculate distance from wavelet center to point x
    dx = x - wavelet_node.coordinates[0]
    dy = y - wavelet_node.coordinates[1]
    dz = z - wavelet_node.coordinates[2]
    
    # Normalize or scale as per the specific wavelet definition.
    # The sources mention "argument > cutoff" implies a normalized distance or coordinate.
    # Assuming the coordinates are in the same space as x,y,z and the wavelet function
    # depends on the relative position.
    # For interpolating wavelets, the argument is often the local coordinate within the support.
    # Let's assume the table lookup takes the normalized distance or the raw distance
    # scaled by the support width.
    
    # Simplified: Use a generic wavelet function lookup based on distance.
    # In practice, this requires the specific wavelet basis definition (e.g., 4th order).
    # We use a placeholder for the actual basis function evaluation which is table-looked up.
    
    dist_sq = dx*dx + dy*dy + dz*dz
    dist = math.sqrt(dist_sq)
    
    # Apply cutoff
    if dist > WAVELET_CUTOFF:
        return 0.0
    
    # Table lookup for wavelet function value
    # Key could be (wavelet_type, dist) or similar
    # Since we don't have the exact table generation code, we assume a function psi(dist)
    # The source says "compute values ... through a table lookup".
    return wavelet_table.get((wavelet_node.wavelet_type, dist), 0.0)

def compute_function_value(x, y, z):
    """
    Compute f(x) by summing coarse scaling and fine wavelet contributions.
    """
    # 1. Coarse contribution
    f_coarse = get_scaling_value(x, y, z)
    
    # 2. Fine detail contribution
    f_detail = 0.0
    
    # Determine which coarse block (x,y,z) belongs to
    ix, iy, iz = int(x), int(y), int(z)
    # Ensure bounds
    ix = max(0, min(ix, COARSE_GRID_SIZE - 1))
    iy = max(0, min(iy, COARSE_GRID_SIZE - 1))
    iz = max(0, min(iz, COARSE_GRID_SIZE - 1))
    
    node = coarse_mesh[iz * COARSE_GRID_SIZE * COARSE_GRID_SIZE + iy * COARSE_GRID_SIZE + ix]
    
    for w_node in node.wavelet_nodes:
        # Check threshold
        if abs(w_node.coefficient) < THRESHOLD_EPSILON:
            continue
            
        # Check support overlap (cutoff)
        # The cutoff check is implicitly handled in get_wavelet_value, 
        # but we can also check distance here for efficiency.
        dx = x - w_node.coordinates[0]
        dy = y - w_node.coordinates[1]
        dz = z - w_node.coordinates[2]
        dist = math.sqrt(dx*dx + dy*dy + dz*dz)
        
        if dist <= WAVELET_CUTOFF:
            psi_val = get_wavelet_value(w_node, x, y, z)
            f_detail += w_node.coefficient * psi_val
            
    return f_coarse + f_detail

def render_ray(ray_origin, ray_direction, step_size, num_steps):
    """
    Render a single ray using front-to-back compositing.
    """
    accumulated_alpha = 0.0
    accumulated_color = 0.0
    
    for i in range(num_steps):
        # Current point on ray
        t = i * step_size
        x = ray_origin[0] + ray_direction[0] * t
        y = ray_origin[1] + ray_direction[1] * t
        z = ray_origin[2] + ray_direction[2] * t
        
        # Compute function value f(x)
        f_val = compute_function_value(x, y, z)
        
        # Determine color and opacity from f(x)
        # Transfer functions C(f) and alpha(f) are application-specific
        color = transfer_function_color(f_val)
        alpha = transfer_function_opacity(f_val)
        
        # Front-to-back compositing
        # I = sum( C_i * alpha_i * d_i * product(1 - alpha_j * d_j) )
        # Here d_i is step_size.
        
        weight = (1.0 - accumulated_alpha) * alpha * step_size
        accumulated_color += color * weight
        accumulated_alpha += weight * alpha # Note: alpha is already scaled by step in some formulations, 
                                            # but standard volume rendering uses alpha as density.
                                            # The formula in source: alpha_i * d_i.
                                            # So alpha_i is opacity density.
        
        if accumulated_alpha >= 1.0:
            break
            
    return accumulated_color

def transfer_function_color(f_val):
    # Placeholder for user-defined transfer function
    return [f_val, f_val, f_val]

def transfer_function_opacity(f_val):
    # Placeholder for user-defined transfer function
    return f_val
```

### Parameters
*   **Coarse Grid Resolution:** $64^3$ is optimal for the current implementation [S1][S2, unofficial].
*   **Threshold $\epsilon$:** 0.01 is recommended as optimal for preprocessing. Rendering results do not change for values below 0.01, with an estimated image error of ~1% [S1][S2, unofficial].
*   **Cutoff Limits:** Scaling function support cutoff is 1; Wavelet function support cutoff is 2. Functions are zero for argument $> \xi_0$ [S1][S2, unofficial].

### Edge Cases and Pitfalls
*   **Cache Misses:** Accessing scaling coefficients causes one cache miss, and summing wavelet coefficients causes at least one cache miss per coarse mesh block [S1][S2, unofficial]. Mitigation: Use redundant scaling coefficients (8 nearest) and sort wavelet coefficients by level/type [S1][S2, unofficial].
*   **Redundancy:** Storing 8 nearest scaling coefficients per node is redundant but provides ~30% acceleration due to better cache coherence [S1][S2, unofficial]. Storing `wavelet_type` is redundant but saves computation time, leading to ~10% additional speedup [S1][S2, unofficial].
*   **Color Discrepancies:** Noticeable color discrepancies may appear in rendering results due to different ray integration approaches used by different renderers [S1][S2, unofficial].
*   **Hardware Acceleration:** The current implementation does not use hardware acceleration [S1][S2, unofficial].
*   **Precision:** The error in determining $f(x)$ through the truncated version of the wavelet sum with coefficients greater than the threshold is $O(\epsilon)$ [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is linear relative to the number of rays, the average number of points on a ray, and the number of wavelet coefficients in the sum [S1][S2, unofficial].
*   **Memory Complexity:** The method does not require significant memory compared to VTK-based approaches, as it avoids converting the wavelet grid into a large unstructured tetrahedral mesh [S1][S2, unofficial]. It works successfully for large data sets where VTK fails due to memory limits (e.g., > $5 \times 10^7$ tetrahedra) [S1][S2, unofficial].
*   **Performance:** On a Pentium 4 3.2 GHz, 2G RAM, Nvidia GeForce Fx5200-TD128, rendering a $500 \times 500$ image for Convection data set with threshold 0.5 took 39 seconds [S1]. *Note: The source mentions threshold 0.5 for this specific timing, but recommends 0.01 for optimal quality/error balance.*
*   **Strengths:** Better image quality than VTK's projected tetrahedra renderer; comparable quality to ZSweep; low memory footprint; suitable for large data sets; interactive coarse rendering [S1][S2, unofficial].
*   **Limitations:** Non-interactive detail addition in the current implementation; no hardware acceleration; potential color discrepancies compared to other renderers; requires table lookup for wavelet functions [S1][S2, unofficial].
*   **When to use:** When rendering wavelet-compressed data sets on systems with limited memory, or when interactive coarse visualization is required with optional fine detail [S1][S2, unofficial].
*   **When not to use:** When hardware-accelerated rendering is critical, or when the highest possible image quality (superior to ZSweep) is required and performance is not a constraint [S1][S2, unofficial].

## Background
*   **Origin:** The method was proposed in the paper "Volumetric Rendering on Wavelet-Based Adaptive Grid" [S1][S2, unofficial].
*   **History:** It was developed as an alternative to completing the wavelet-based grid to an AMR grid for VTK-based rendering. The direct summation approach allows visualization directly on the wavelet-based grid without the memory overhead of tetrahedralization [S1][S2, unofficial].
*   **Usage:** Used for rendering convection data sets (256^3 thermal convection, Rayleigh number $10^{10}$) and Hydrogen 3D Orbital data sets, generated on regular 256^3 grids and compressed via 4th order interpolating wavelet transform [S1][S2, unofficial].

## Sources
[S1] Volumetric Rendering on Wavelet-Based Adaptive Grid, https://doi.org/10.3390/fluids7070245, fetched 2026-10-26
[S2] Volumetric Rendering on Wavelet-Based Adaptive Grid - MDPI, https://www.mdpi.com/2311-5521/7/7/245, fetched 2026-10-28 (unofficial)
