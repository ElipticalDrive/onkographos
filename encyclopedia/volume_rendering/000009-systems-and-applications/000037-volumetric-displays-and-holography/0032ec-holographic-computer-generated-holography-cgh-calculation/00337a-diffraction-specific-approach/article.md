# Diffraction-Specific Approach

Also called: Hogel Vector Approach, Hogel-based CGH, Interference-Based Approach

## Overview
The Diffraction-Specific Approach (also known as the Hogel Vector Approach) is a method for generating computer-generated holograms (CGH) that bypasses the direct simulation of optical interference. Instead of summing complex wavefronts from every object point to the hologram plane, it treats the hologram as a collection of subsampled functional elements called "hogels," represented spectrally as "hogel vectors" [S1]. This approach breaks from traditional interference-based methods by working backwards from the 3D image, rendering orthographic projections to generate the data, and decoding them using precomputed basis fringes [S1]. It sets itself apart from siblings like the [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by explicitly utilizing spatial and spectral subsampling to achieve significant bandwidth compression and computation speedups, at the cost of added image blur [S1].

## How it works
The method consists of two main phases: encoding (generation of the hogel-vector array) and decoding (reconstruction of the fringe pattern).

### Inputs and Outputs
*   **Input:** A 3D scene description (list of 3D points/primitives with color, lighting, shading, and occlusion data) [S1].
*   **Output:** A 2D complex field or phase-only pattern (the hologram fringe) suitable for display on an SLM [S1].

### Data Structures
1.  **Hogel:** A spatial subsample of the hologram plane. The hologram is divided into a grid of these functional elements [S1].
2.  **Hogel Vector:** A spectral representation of a hogel. It is an array of coefficients corresponding to a set of precomputed "basis fringes" [S1].
3.  **Basis Fringes:** Precomputed reference patterns used during the decoding step to reconstruct the fringe from the hogel vectors [S1].

### Parameters
*   **Compression Ratio (CR):** The ratio between the size of the full interference fringe and the size of the hogel-vector array. Typical values are 8:1, 16:1, and 32:1 [S1].
*   **Sampling Parameters:** Determine the spatial and spectral subsampling rates. These must be chosen to match the acuity of the Human Visual System (HVS) [S1].

### Steps

#### Phase 1: Encoding (Hogel-Vector Generation)
1.  **3D Scene Preparation:** Perform a 3D filling operation to generate a list of 3D points or primitives, including color, lighting, shading, and occlusion information [S1].
2.  **Orthographic Projection Rendering:** Render a series of orthographic projections. Each projection corresponds to a specific spectral sample (viewing direction) of the hogels [S1].
    *   These projections provide a discrete sampling of space (pixels) and spectrum (projection direction) [S1].
3.  **Hogel-Vector Array Construction:** Convert the rendered orthographic projections into a hogel-vector array. The orthographic projections are easily mapped to the hogel-vector structure [S1].

#### Phase 2: Decoding (Fringe Reconstruction)
1.  **Basis Fringe Retrieval:** Retrieve the set of precomputed basis fringes corresponding to the hogel structure [S1].
2.  **Inner Product Calculation:** Compute the inner product between the array of hogel vectors and the precomputed basis fringes to recover the usable fringe [S1].
    *   This step involves multiplying components of each hogel vector by replicated arrays of single basis fringes and accumulating the results [S1].

### Pseudocode
The following pseudocode illustrates the encoding and decoding process. It assumes standard 3D graphics primitives are available for rendering orthographic projections.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diffraction_specific_cgh(scene_primitives, hogel_grid_size, num_spectral_samples, basis_fringes):
    """
    Generates a hologram fringe using the Diffraction-Specific Approach.
    
    Args:
        scene_primitives: List of 3D points/primitives with color/lighting data.
        hogel_grid_size: Dimensions of the hogel grid (H, W).
        num_spectral_samples: Number of orthographic projection directions.
        basis_fringes: Precomputed array of basis fringe patterns.
        
    Returns:
        fringe: The reconstructed hologram fringe pattern.
    """
    
    # Phase 1: Encoding
    # 1. Render orthographic projections for each spectral sample
    # Each projection is an image of size (hogel_height, hogel_width)
    orthographic_projections = []
    for view_index in range(num_spectral_samples):
        # Render scene from a specific orthographic direction
        # This captures the spatial distribution of light for that spectral component
        projection = render_orthographic_projection(scene_primitives, view_index)
        orthographic_projections.append(projection)
        
    # 2. Convert projections to Hogel-Vector Array
    # The array shape is (H, W, num_spectral_samples)
    # Each element (h, w, s) is the intensity/value of the hogel at (h,w) for spectral sample s
    hogel_vectors = convert_projections_to_hogel_vectors(orthographic_projections, hogel_grid_size)
    
    # Phase 2: Decoding
    # 3. Recover the fringe using basis fringes
    # The fringe is reconstructed by summing the contributions of each hogel vector
    # weighted by its corresponding basis fringe.
    
    # Initialize fringe with zeros
    fringe = initialize_zero_fringe(hogel_grid_size)
    
    # Decode each hogel
    for h in range(hogel_grid_size[0]):
        for w in range(hogel_grid_size[1]):
            # Get the hogel vector for this spatial location
            hv = hogel_vectors[h, w, :]
            
            # Compute inner product with basis fringes
            # basis_fringes[h, w, :] contains the precomputed basis patterns for this hogel
            # The inner product reconstructs the local fringe pattern for this hogel
            local_fringe = inner_product(hv, basis_fringes[h, w, :])
            
            # Add to the total fringe
            fringe[h, w, :] = local_fringe
            
    return fringe

def render_orthographic_projection(primitives, view_index):
    """
    Renders an orthographic projection of the 3D scene from a specific direction.
    This is a standard 3D graphics operation.
    """
    # Implementation depends on the graphics API
    # Returns a 2D array representing the intensity/color at each pixel
    pass

def convert_projections_to_hogel_vectors(projections, hogel_grid_size):
    """
    Converts the rendered orthographic projections into the hogel-vector array.
    """
    # Map each projection to the corresponding spectral component in the hogel vector
    # This involves arranging the pixel data from the projections into the (H, W, S) structure
    pass

def inner_product(vector, basis_fringes):
    """
    Computes the inner product between a hogel vector and its basis fringes.
    This is the core decoding step, often accelerated by hardware texture mapping.
    """
    # Sum of element-wise multiplication
    result = sum(v * b for v, b in zip(vector, basis_fringes))
    return result
```

### Edge Cases and Pitfalls
*   **Blur vs. Resolution:** The added blur from subsampling is acceptable if the Compression Ratio (CR) is chosen appropriately. For CR $\le$ 8:1, the blur is invisible to the HVS. For CR = 16:1 or 32:1, good images are achieved with acceptable degradation [S1].
*   **Hardware Dependency:** The decoding step is computationally intensive due to many multiplication-accumulation (MAC) operations. It relies heavily on specialized hardware (e.g., graphics subsystems with texture mapping and accumulation buffers) for interactive speeds [S1].
*   **Loss of Sharpness:** Unlike interference-based methods, this approach inherently loses image sharpness due to subsampling. This must be balanced against the need for computation speed [S1].

## Subtopics

### Holographic Bandwidth Compression
Techniques to increase fringe computation speed and reduce data volume, including spatial subsampling, nonuniform sampling, and standard video compression algorithms [S1]. Nonuniformly sampled fringes provide lossless bandwidth compression, whereas standard MPEG algorithms are used for encoding/decoding computed fringes, implying potential loss or different characteristics [S1]. This approach increases fringe computation speed by 3000 times for the same hardware implementation [S1].

### Hogel-Vector Bandwidth Compression
A technique within the diffraction-specific approach where the fringe is subsampled spatially into hogels and spectrally into hogel vectors. It uses standard 3D computer graphics rendering (orthographic projections) to generate the data [S1]. The inner product between an array of hogel vectors and the precomputed basis fringes is performed rapidly by exploiting the texture mapping function and the accumulation buffer. The texture mapping function rapidly multiplies a component from each hogel vector by a replicated array of a single basis fringe. This operation is repeated several times, once for each hogel-vector component, accumulating the result in the accumulation buffer [S1]. Achieved a computation time of 0.9 seconds for fringes of 6-MB with CR=32:1 using Silicon Graphics RealityEngine2 (RE2) for both rendering orthographic projections and decoding [S1].

## Operating characteristics
*   **Time Complexity:** The encoding phase involves rendering orthographic projections, which is efficient using standard 3D graphics hardware. The decoding phase involves many multiplication-accumulation (MAC) calculations, which can be slow on general-purpose CPUs but fast on specialized hardware (DSPs, graphics subsystems) [S1].
*   **Memory Complexity:** The method achieves significant bandwidth compression. The size of the hogel-vector array is smaller than the full interference fringe by the Compression Ratio (CR) [S1].
*   **Measured Performance:** On a Silicon Graphics RealityEngine2 (RE2), a computation time of 0.9 seconds was achieved for fringes of 6-MB with CR=32:1 [S1]. Holographic bandwidth compression techniques can increase fringe computation speed by 3000 times for the same hardware implementation [S1].
*   **Strengths:** Increases overall computation speed and achieves bandwidth compression compared to interference-based methods. It is fast enough for interactive holographic displays when specialized hardware is used [S1].
*   **Limitations:** Accompanied by a loss in image sharpness (added blur). The blur must be managed by choosing appropriate CR and sampling parameters [S1].
*   **When to use:** When interactive or near-interactive holographic display is required, and the human visual system's acuity limits can be exploited to reduce data volume and computation time [S1].
*   **When not to use:** When the highest possible image sharpness is required, or when specialized hardware for decoding is not available [S1].

## Background
The Diffraction-Specific Approach was introduced as an alternative to the conventional Interference-Based Approach for computing holographic fringes. It was developed to address the high computational cost of simulating optical interference, which requires billions of complex arithmetic operations (trigonometric functions and square roots) for small simple holographic images [S1]. The approach was pioneered by researchers who recognized that the resolution produced by interference-based methods is often finer than can be utilized by the human visual system (HVS) [S1]. By subsampling the fringe spatially and spectrally, the method leverages standard 3D computer graphics rendering and specialized hardware (like DSPs and graphics subsystems) to achieve interactive speeds [S1]. The concept of "hogels" and "hogel vectors" was central to this development, allowing for efficient encoding and decoding of holographic data [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
