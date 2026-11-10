# Diffraction-Specific Approach

Also called: Hogel Vector Approach, Hogel-based CGH, Interference-Based Approach

## Overview
The Diffraction-Specific Approach (also known as the Hogel Vector Approach) is a method for generating computer-generated holograms (CGH) that avoids the direct simulation of optical interference. Instead of summing complex wavefronts from every object point to the hologram plane, it treats the hologram as a collection of subsampled functional elements called "hogels" and represents their spectral content as "hogel vectors" [S1]. This approach breaks from traditional interference-based methods by working backwards from the 3D image, rendering orthographic projections to generate the data, and decoding them using precomputed basis fringes [S1]. It sets itself apart from siblings like [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by explicitly decoupling the spatial and spectral sampling of the hologram into a vector-based representation, enabling significant bandwidth compression and faster computation at the cost of added blur [S1].

## How it works
The method consists of two main stages: encoding (generating the hogel-vector array) and decoding (recovering the fringe pattern).

### Inputs and Outputs
*   **Input:** A 3D scene representation (point cloud, mesh, or voxel grid) with color, lighting, and shading information [S1].
*   **Output:** A 2D complex field or phase map $U_h(x,y)$ suitable for display on a spatial light modulator (SLM) [S1].

### Data Structures
1.  **Hogel:** A spatial subsampling of the hologram plane. The hologram is divided into a grid of hogels.
2.  **Hogel Vector:** A spectral subsampling associated with each hogel. It represents the amplitude and phase contributions for specific viewing angles or spectral samples [S1].
3.  **Basis Fringes:** Precomputed fringe patterns corresponding to the spectral samples. These are stored to facilitate rapid decoding [S1].

### Parameters
*   **Compression Ratio (CR):** The ratio between the size of the full interference fringe and the size of the hogel-vector array. Typical values are 8:1, 16:1, and 32:1 [S1].
*   **Hogel Size:** The spatial dimensions of each hogel in pixels.
*   **Spectral Samples:** The number of orthographic projections rendered per hogel.

### Steps

#### Encoding Stage
1.  **3D Scene Preparation:** Perform a 3D filling operation to generate a list of 3D points or primitives, including color, lighting, shading, and occlusion data [S1].
2.  **Render Orthographic Projections:** Render a series of orthographic projections of the 3D scene. Each projection corresponds to a specific spectral sample (viewing angle) for the hogels [S1].
    *   The orthographic projections provide a discrete sampling of space (pixels) and spectrum (projection direction) [S1].
3.  **Construct Hogel-Vector Array:** Convert the rendered orthographic projections into a hogel-vector array. Each hogel in the array contains a vector of values derived from the corresponding pixels in the orthographic projections [S1].

#### Decoding Stage
1.  **Retrieve Basis Fringes:** Access the precomputed basis fringes corresponding to the spectral samples [S1].
2.  **Compute Inner Product:** Calculate the inner product between the array of hogel vectors and the precomputed basis fringes to recover the usable fringe [S1].
    *   This operation is performed rapidly by exploiting texture mapping functions and accumulation buffers in graphics hardware [S1].
    *   The texture mapping function multiplies a component from each hogel vector by a replicated array of a single basis fringe [S1].
    *   This operation is repeated for each hogel-vector component, accumulating the result in an accumulation buffer [S1].
3.  **Encode for Display:** Convert the recovered complex field into a format displayable on hardware (e.g., phase-only or amplitude-only) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diffraction_specific_cgh(scene, hogel_grid_size, spectral_samples, basis_fringes):
    """
    Generates a hologram fringe using the Diffraction-Specific Approach.
    
    Args:
        scene: 3D scene data (points, colors, etc.)
        hogel_grid_size: Tuple (H, W) defining the number of hogels in height and width
        spectral_samples: Number of orthographic projections to render per hogel
        basis_fringes: Precomputed array of basis fringe patterns
        
    Returns:
        fringe: 2D complex field representing the hologram
    """
    
    # Step 1: Render Orthographic Projections
    # Render 'spectral_samples' orthographic views of the scene
    # Each view corresponds to a specific spectral sample (angle)
    orthographic_projections = []
    for i in range(spectral_samples):
        # Render orthographic projection for spectral sample i
        # This involves standard 3D rasterization
        proj = render_orthographic_projection(scene, angle=i)
        orthographic_projections.append(proj)
        
    # Step 2: Construct Hogel-Vector Array
    # Divide the hologram plane into hogels
    # Each hogel contains a vector of values from the orthographic projections
    hogel_vectors = []
    hogel_height, hogel_width = hogel_grid_size
    
    for h in range(hogel_height):
        for w in range(hogel_width):
            # Extract vector for this hogel from all spectral samples
            vector = []
            for s in range(spectral_samples):
                # Get pixel value from projection s for hogel (h, w)
                # The exact mapping depends on the specific subsampling scheme
                val = extract_hogel_component(orthographic_projections[s], h, w)
                vector.append(val)
            hogel_vectors.append(vector)
            
    # Step 3: Decode using Basis Fringes
    # Initialize the fringe plane
    fringe = zeros_like(basis_fringes[0])
    
    # Compute inner product of hogel vectors and basis fringes
    # This is accelerated using texture mapping and accumulation buffer
    for h in range(hogel_height):
        for w in range(hogel_width):
            vector = hogel_vectors[h * hogel_width + w]
            # For each component in the hogel vector
            for s in range(spectral_samples):
                # Multiply component by corresponding basis fringe
                # and accumulate into the fringe plane
                fringe += vector[s] * basis_fringes[s]
                
    return fringe
```

### Edge Cases and Pitfalls
*   **Blur vs. Sharpness:** Higher compression ratios (e.g., 32:1) result in acceptable image degradation but added blur. For CR <= 8:1, the blur is invisible to the Human Visual System (HVS) [S1].
*   **Hardware Dependency:** The decoding step relies heavily on graphics hardware capabilities (texture mapping, accumulation buffers) for speed. Without specialized hardware, the computation can be slow [S1].
*   **Basis Fringe Storage:** Precomputing and storing basis fringes requires memory. The size of this storage depends on the number of spectral samples and the hogel size [S1].

## Subtopics

### Holographic Bandwidth Compression
Techniques to increase fringe computation speed and reduce data volume, including spatial subsampling, nonuniform sampling, and standard video compression algorithms [S1]. Nonuniformly sampled fringes provide lossless bandwidth compression, whereas standard MPEG algorithms are used for encoding/decoding computed fringes, implying potential loss or different characteristics [S1]. Holographic bandwidth compression increases fringe computation speed by 3000 times for the same hardware implementation [S1].

### Hogel-Vector Bandwidth Compression
A technique within the diffraction-specific approach where the fringe is subsampled spatially into hogels and spectrally into hogel vectors. It uses standard 3D computer graphics rendering (orthographic projections) to generate the data [S1]. The inner product between an array of hogel vectors and the precomputed basis fringes is performed rapidly by exploiting the texture mapping function and the accumulation buffer. The texture mapping function rapidly multiplies a component from each hogel vector by a replicated array of a single basis fringe. This operation is repeated several times, once for each hogel-vector component, accumulating the result in the accumulation buffer [S1]. Achieved a computation time of 0.9 seconds for fringes of 6-MB with CR=32:1 using Silicon Graphics RealityEngine2 (RE2) for both rendering orthographic projections and decoding [S1].

## Operating characteristics
*   **Time Complexity:** The encoding stage involves rendering orthographic projections, which is comparable to standard 3D rendering. The decoding stage involves many multiplication-accumulation calculations (MACs), which can be slow without specialized hardware [S1]. However, with specialized hardware (e.g., DSP cards, graphics subsystems), computation times can be reduced to under one second for 6-MB fringes [S1].
*   **Memory Complexity:** The method requires storing the hogel-vector array and the basis fringes. The size of the hogel-vector array is reduced by the compression ratio (CR) compared to the full fringe [S1].
*   **Measured Performance:** On a Silicon Graphics RealityEngine2 (RE2), a computation time of 0.9 seconds was achieved for fringes of 6-MB with CR=32:1 [S1]. Holographic bandwidth compression increases fringe computation speed by 3000 times for the same hardware implementation [S1].
*   **Strengths:** Increases overall computation speed and achieves bandwidth compression compared to interference-based methods [S1]. Accompanied by a loss in image sharpness (added blur), but this blur can be matched to the acuity of the human visual system (HVS) simply by choosing an appropriate compression ratio and sampling parameters [S1].
*   **Limitations:** Loss in image sharpness (added blur) [S1]. Requires specialized hardware for fast decoding [S1].
*   **When to Use:** When interactive holographic displays are required and some loss of sharpness is acceptable [S1].
*   **When Not to Use:** When maximum image sharpness is required and computation time is not a constraint [S1].

## Background
The Diffraction-Specific Approach was introduced as an alternative to the conventional Interference-Based Approach for computing holographic fringes. The Interference-Based Approach simulates optical interference by summing complex wavefronts from object elements with a reference wavefront, which requires billions of computational steps and expensive floating-point calculations [S1]. The Diffraction-Specific Approach breaks from this tradition by working backwards from the 3D image, subsampling the fringe spatially into hogels and spectrally into hogel vectors [S1]. This approach was developed to enable interactive holographic displays by reducing the computational burden and achieving bandwidth compression [S1]. The method leverages standard 3D computer graphics rendering and specialized hardware (such as graphics subsystems with texture mapping and accumulation buffers) to accelerate the computation and decoding of holographic fringes [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
