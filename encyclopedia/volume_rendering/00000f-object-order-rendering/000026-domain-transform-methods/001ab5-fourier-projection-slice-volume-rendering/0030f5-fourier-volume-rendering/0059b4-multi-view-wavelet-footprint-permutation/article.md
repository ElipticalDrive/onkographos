# Multi-view Wavelet Footprint Permutation

Also called: Multi-view arrangement, Multiview Rendering, Multiview Rendering via Wavelet Symmetry, Tensor-product multiview, Triple view rendering

## Overview
Multi-view Wavelet Footprint Permutation is a rendering technique that generates images from multiple viewing directions simultaneously by exploiting the symmetry of 3D tensor-product wavelets, allowing footprint reuse without computational overhead for splat computation [S1, unofficial]. It solves the problem of missing depth cues and occlusion information inherent in single-view X-ray-like splatting by presenting the volume from different directions in a single window [S1, unofficial]. Unlike calculating footprints for each view independently, this method reuses footprints via permutations, reducing computation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data represented by wavelet coefficients; camera parameters $\alpha$ and $\beta$ defining viewing angles; a set of 8 precomputed footprint textures for View 1 generated via the Fourier Projection-Slice (FPS) theorem [S1, unofficial].
*   **Output:** Three simultaneous rendered views (View 1, View 2, View 3) presented in a single window [S1, unofficial].

**Data Structures:**
1.  **Lookup Table for Permutations and Negations:** A table mapping basis function indices (0-7) to permutation indices and intensity negation factors for each view. The basis functions are types 1 to 7 (plus the scaling function, type 0), corresponding to tensor products of 1D basis functions $\phi$ (scaling) and $\psi$ (wavelet) along x, y, and z directions [S1, unofficial].
    *   Structure: Array of 3 entries (one per view). Each entry contains:
        *   `permutation`: Array of 8 integers (indices 0-7).
        *   `negations`: Array of 8 integers (+1 or -1).
2.  **Footprint Textures:** 8 textures (Tex0 to Tex7) representing the 2D splat footprints of the 8 basis functions for View 1, calculated using the FPS theorem [S1, unofficial].

**Lookup Table Definition:**
The specific permutations and negations for the three views are defined as follows [S1, unofficial]:

| View | Permutations (indices 0-7) | Negations (indices 0-7) |
| :--- | :--- | :--- |
| View 1 | `[0, 1, 2, 3, 4, 5, 6, 7]` | `[1, 1, 1, 1, 1, 1, 1, 1]` |
| View 2 | `[0, 1, 4, 5, 2, 3, 6, 7]` | `[1, 1, 1, -1, -1, 1, 1, -1, -1]` |
| View 3 | `[0, 4, 2, 6, 1, 5, 3, 7]` | `[1, 1, 1, 1, 1, -1, -1, -1, -1]` |

*Note: The source text lists 9 negation values for View 2 and View 3 in the notes, but the table and context imply 8 basis functions (0-7). The table in the source shows 8 negation values for View 2 and 8 for View 3. The notes' list `[1 1 1 -1 -1 1 1 -1 -1]` has 9 elements. The table `[1 1 1 -1 -1 1 1 -1 -1]` also has 9. However, standard tensor product wavelets have $2^3=8$ basis functions. The table header says "Permutations [0 1 2 3 4 5 6 7]" (8 items). The negation list in the table row for View 2 is `[1 1 1 -1 -1 1 1 -1 -1]`. This is 9 items. Let's look at the text: "multiply intensities by negation vector [1 1 1 -1 -1 1 1 -1 -1]". This is 9 items. But there are only 8 textures. The table in the source image description likely has a typo or the 9th item is a separator or error. Given 8 basis functions, we assume the negation vector has 8 elements. The table row in the source text representation might be misaligned. Let's look at View 3: `[1 1 1 1 1 -1 -1 -1 -1]` (9 items). View 1: `[1 1 1 1 1 1 1 1 1]` (9 items). It is highly likely the last '1' or '-1' is an artifact or the list includes a sign for the scaling function separately, but the permutation array has 8 items. We will use the permutation arrays as the primary guide for mapping 8 textures. For negations, we will assume the first 8 values apply to indices 0-7. If the source explicitly lists 9, we note the discrepancy. However, standard practice for 8 basis functions requires 8 coefficients. We will assume the negation vectors are length 8, derived from the first 8 values of the provided lists, or that the lists in the notes contain a typo. The table in the source [S1, unofficial] shows:
View 1 Negations: `[1 1 1 1 1 1 1 1 1]` -> 9 ones.
View 2 Negations: `[1 1 1 -1 -1 1 1 -1 -1]` -> 9 values.
View 3 Negations: `[1 1 1 1 1 -1 -1 -1 -1]` -> 9 values.
Given the permutation arrays are length 8, we assume the negation arrays are also length 8, and the 9th value is likely a typo or includes a global sign. We will use the first 8 values for implementation.

**Numbered Steps:**

1.  **Generate Lookup Table:** Create the lookup table for basis function types 1 to 7 (and type 0) to ensure correct footprint calculation. The table maps each basis function index $i \in [0, 7]$ to a permutation index $p_i$ and a negation factor $n_i$ for each view [S1, unofficial].
2.  **Calculate View 1 Textures:** For View 1, calculate 8 textures (Tex0 to Tex7) for the basis functions using the Fourier Projection-Slice (FPS) theorem. These textures represent the 2D footprints of the 3D basis functions projected onto the View 1 plane [S1, unofficial].
3.  **Determine View 2 Footprints:** For View 2, reuse the footprints calculated for View 1. The footprint for the basis function is equal for View 1 and View 2 due to symmetry. However, the coefficients must be permuted and negated according to the View 2 lookup table. Specifically, apply the permutation `[0, 1, 4, 5, 2, 3, 6, 7]` to the basis function indices and multiply intensities by the negation vector `[1, 1, 1, -1, -1, 1, 1, -1]` (first 8 values) [S1, unofficial].
4.  **Determine View 3 Footprints:** For View 3, use the footprint of the function (texture 4) of View 1 for certain basis functions, applying specific permutations. Apply the permutation `[0, 4, 2, 6, 1, 5, 3, 7]` to the basis function indices and multiply intensities by the negation vector `[1, 1, 1, 1, 1, -1, -1, -1]` (first 8 values) [S1, unofficial].
5.  **Apply Permutations and Negations:** For each view, iterate through the 8 basis functions. For basis function $i$, retrieve the permutation index $p_i$ and negation factor $n_i$ from the lookup table. The footprint used for splatting is Texture $p_i$. The coefficient for this footprint is multiplied by $n_i$. If intensity inversion is required, multiply the intensity by -1 [S1, unofficial].
6.  **Render:** Splat the permuted and negated footprints onto the respective projection planes for each view. The projection planes are defined by vectors $u$ and $v$ for each view [S1, unofficial].

**Formulas:**
The projection planes $P_i$ for the three views are spanned by vectors $u_i$ and $v_i$, defined by camera parameters $\alpha$ and $\beta$ [S1, unofficial]:

*   **View 1:**
    $$u_1 = [-\sin(\alpha), \cos(\alpha), 0]$$
    $$v_1 = [-\cos(\alpha)\sin(\beta), -\sin(\alpha)\sin(\beta), \cos(\beta)]$$

*   **View 2:**
    $$u_2 = [-\cos(\alpha), -\sin(\alpha), 0]$$
    $$v_2 = [-\sin(\alpha)\sin(\beta), \cos(\alpha)\sin(\beta), \cos(\beta)]$$

*   **View 3:**
    $$u_3 = [0, \cos(\alpha), \sin(\alpha)]$$
    $$v_3 = [-\cos(\beta), -\sin(\alpha)\sin(\beta), \cos(\alpha)\sin(\beta)]$$

**Parameters:**
*   $\alpha, \beta$: Camera parameters defining the viewing angles. In the example, $\alpha = 0.66, \beta = 0.91$ [S1, unofficial].
*   Number of basis function textures: 8 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The angles between individual viewing planes are not constant and depend on the initial selection of camera parameters $\alpha$ and $\beta$ [S1, unofficial].
*   Precision issues may arise when applying permutations and negations to wavelet coefficients, especially for higher-order wavelets [S1, unofficial].
*   The method assumes tensor-product wavelets. Non-tensor-product bases may not benefit from this symmetry [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_view_wavelet_footprint_permutation(volume_data, alpha, beta):
    """
    Renders volume data from three views using footprint permutation.
    
    Args:
        volume_data: 3D array of wavelet coefficients.
        alpha: Camera parameter for viewing angles.
        beta: Camera parameter for viewing angles.
        
    Returns:
        views: List of 3 rendered images (View 1, View 2, View 3).
    """
    # Step 1: Define Lookup Table for Permutations and Negations
    # Basis functions 0-7 correspond to tensor products of phi/psi
    # Permutations map index i to texture index p_i
    # Negations map index i to sign n_i
    lookup_table = {
        'View 1': {
            'permutations': [0, 1, 2, 3, 4, 5, 6, 7],
            'negations': [1, 1, 1, 1, 1, 1, 1, 1]
        },
        'View 2': {
            'permutations': [0, 1, 4, 5, 2, 3, 6, 7],
            'negations': [1, 1, 1, -1, -1, 1, 1, -1] # First 8 values from source
        },
        'View 3': {
            'permutations': [0, 4, 2, 6, 1, 5, 3, 7],
            'negations': [1, 1, 1, 1, 1, -1, -1, -1] # First 8 values from source
        }
    }
    
    # Step 2: Calculate View 1 Textures using FPS theorem
    # This step is assumed to be done by the parent Fourier Volume Rendering method
    # We assume 8 textures are available: tex0 to tex7
    view1_textures = calculate_fps_textures(volume_data)
    
    # Step 3: Define Projection Planes
    u1 = [-sin(alpha), cos(alpha), 0]
    v1 = [-cos(alpha)*sin(beta), -sin(alpha)*sin(beta), cos(beta)]
    
    u2 = [-cos(alpha), -sin(alpha), 0]
    v2 = [-sin(alpha)*sin(beta), cos(alpha)*sin(beta), cos(beta)]
    
    u3 = [0, cos(alpha), sin(alpha)]
    v3 = [-cos(beta), -sin(alpha)*sin(beta), cos(alpha)*sin(beta)]
    
    projection_planes = {
        'View 1': (u1, v1),
        'View 2': (u2, v2),
        'View 3': (u3, v3)
    }
    
    views = []
    
    for view_name in ['View 1', 'View 2', 'View 3']:
        # Get lookup table for this view
        lt = lookup_table[view_name]
        perms = lt['permutations']
        negs = lt['negations']
        
        # Get projection plane for this view
        u, v = projection_planes[view_name]
        
        # Initialize empty image for this view
        image = create_empty_image()
        
        # Step 4 & 5: Apply Permutations and Negations, then Splat
        for i in range(8):
            # Determine which texture to use
            texture_index = perms[i]
            texture = view1_textures[texture_index]
            
            # Determine sign
            sign = negs[i]
            
            # Get coefficients for basis function i
            # This assumes volume_data is structured such that we can extract
            # coefficients for basis function i. In practice, this involves
            # iterating over the wavelet decomposition levels and positions.
            # For simplicity, we assume a function get_coefficients(i) exists
            # which returns the relevant coefficients for basis function i.
            coefficients = get_coefficients_for_basis_function(i, volume_data)
            
            # Splat the texture with permuted/negated coefficients
            splat_image = splat(texture, coefficients, u, v, sign)
            
            # Add to image
            image = add_to_image(image, splat_image)
            
        views.append(image)
        
    return views

def calculate_fps_textures(volume_data):
    """
    Calculates 8 textures for View 1 using the Fourier Projection-Slice theorem.
    This is a placeholder for the actual FPS implementation.
    """
    # Implementation details depend on the specific wavelet basis and FPS method
    # This function is assumed to be provided by the parent Fourier Volume Rendering method
    return [texture0, texture1, texture2, texture3, texture4, texture5, texture6, texture7]

def get_coefficients_for_basis_function(basis_index, volume_data):
    """
    Extracts coefficients for a specific basis function from the volume data.
    This is a placeholder for the actual coefficient extraction logic.
    """
    # Implementation details depend on the wavelet decomposition structure
    return coefficients

def splat(texture, coefficients, u, v, sign):
    """
    Splats the texture onto the image plane defined by u and v,
    using the given coefficients and sign.
    """
    # Implementation details depend on the splatting method
    # This function is assumed to be provided by the parent splatting method
    return splatted_image

def create_empty_image():
    """
    Creates an empty image buffer.
    """
    return empty_buffer

def add_to_image(image, splat_image):
    """
    Adds the splat image to the main image.
    """
    return image + splat_image
```

## Subtopics

### Frequency Domain Depth Cueing
Adds depth cues to frequency domain rendering by manipulating the phase of the Fourier transform. It differs from Multi-view Wavelet Footprint Permutation by operating in the frequency domain and adding cues to a single view, rather than rendering multiple views [6400-frequency-domain-depth-cueing.md].

### Fourier Domain Approximate Shading
Simulates shading in the frequency domain by modifying the amplitude of the Fourier coefficients. It differs by focusing on shading effects rather than multi-view occlusion [6892-fourier-domain-approximate-shading.md].

### Polar Coordinate Fourier Volume Rendering
Uses polar coordinates in the frequency domain for rendering. It differs by changing the coordinate system of the Fourier transform, rather than exploiting spatial symmetry for multi-view footprint reuse [7426-polar-coordinate-fourier-volume-rendering.md].

## Operating characteristics
*   **Time Complexity:** The method reduces computation by reusing footprints. Calculating footprints for each view independently would require 3x the footprint calculation cost. By reusing View 1 footprints and applying permutations, the footprint calculation cost is reduced to 1x, with only permutation and negation overhead for Views 2 and 3 [S1, unofficial].
*   **Memory Complexity:** Requires storing 8 footprint textures, regardless of the number of views. This is constant memory overhead relative to the number of views [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales well with the number of basis functions, as the permutation table size is fixed at 8 for 3D tensor-product wavelets [S1, unofficial].
*   **Strengths:** Provides depth cues and occlusion information missing in single-view splatting; reduces computational overhead for multi-view rendering by reusing footprints [S1, unofficial].
*   **Limitations:** The angles between viewing planes are not constant and depend on camera parameters; limited to 3 views in the described implementation; requires tensor-product wavelets [S1, unofficial].
*   **When to use:** When depth cues and occlusion information are critical, and the volume data is represented by tensor-product wavelets [S1, unofficial].
*   **When not to use:** When non-tensor-product wavelets are used, or when a single view is sufficient [S1, unofficial].

## Background
Multi-view Wavelet Footprint Permutation was introduced in the context of compression domain volume rendering. It exploits the symmetry of 3D tensor-product wavelets to extend single-view splatting to a multi-view renderer without computational overhead for splat computation [S1, unofficial]. The technique is based on the Fourier Projection-Slice theorem and the properties of tensor-product functions constructed from permutations of 1D basis functions [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering for Distributed ..., https://cgl.ethz.ch/Downloads/Publications/Tutorials/1998/tu_Gro98-4.pdf, fetched 2026-10-26 (unofficial)
