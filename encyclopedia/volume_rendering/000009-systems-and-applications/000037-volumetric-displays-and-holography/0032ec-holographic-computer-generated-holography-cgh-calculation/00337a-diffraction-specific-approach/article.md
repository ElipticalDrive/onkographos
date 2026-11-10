# Diffraction-Specific Approach

Also called: Hogel Vector Approach, Hogel-based CGH, Interference-Based Approach (Note: Source contrasts this with the "Interference-Based Approach, " but the provided aliases list includes "Interference-Based Approach." This is likely a misclassification in the alias list or refers to the fact that it models diffraction rather than simulating interference directly. The text explicitly states it "breaks from the traditional simulation of optical holographic interference." We list the aliases as provided but clarify the distinction in the Overview.)

## Overview
The Diffraction-Specific Approach is a method for calculating Computer-Generated Holograms (CGH) that avoids the direct simulation of optical interference between object and reference waves. Instead, it treats the holographic fringe as a signal that can be subsampled spatially into functional elements called "hogels" and spectrally into an array of "hogel vectors" [S1]. By working backwards from the 3D image and leveraging standard 3D computer graphics rendering techniques (orthographic projections), this approach significantly increases computation speed and achieves bandwidth compression compared to conventional interference-based methods [S1]. It is distinguished from its sibling [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by its explicit decomposition of the wavefield into hogel vectors and its reliance on texture-mapping hardware for decoding, whereas volume rendering typically sums contributions voxel-by-voxel.

## How it works
The method consists of two main stages: encoding (generation of the hogel-vector array) and decoding (reconstruction of the fringe).

### Inputs and Outputs
*   **Inputs:** A 3D scene defined by primitives (points, polygons) with attributes including color, lighting, shading, and occlusion information [S1].
*   **Outputs:** A holographic fringe pattern (complex amplitude or phase-only data) suitable for display on a Spatial Light Modulator (SLM) [S1].

### Data Structures
1.  **Hogel:** A spatially subsampled functional holographic element. The hologram plane is divided into a grid of hogels [S1].
2.  **Hogel Vector:** A spectral subsampling of the wavefront information associated with a single hogel. It represents the frequency components or projection directions contributing to that hogel [S1].
3.  **Basis Fringes:** A precomputed set of reference fringe patterns corresponding to the spectral samples. These are used during the decoding step [S1].

### Parameters
*   **Compression Ratio (CR):** The ratio between the size of the full fringe data and the size of the hogel-vector array. Typical values are 8:1, 16:1, and 32:1 [S1].
*   **Hogel Size:** Determines the spatial subsampling.
*   **Spectral Samples:** Determines the number of components in the hogel vector.

### Steps

#### 1. Encoding: Generating the Hogel-Vector Array
1.  **3D Scene Preparation:** Perform a 3D filling operation to generate a list of 3D points or primitives, including color, lighting, shading, and occlusion data [S1].
2.  **Orthographic Projections:** Render a series of orthographic projections of the 3D scene. Each projection corresponds to a specific spectral sample (direction) of the hogels. This provides a discrete sampling of space (pixels) and spectrum (projection direction) [S1].
3.  **Vector Conversion:** Convert the rendered orthographic projections into a hogel-vector array. The pixel values from the projections are mapped to the components of the hogel vectors [S1].

#### 2. Decoding: Recovering the Fringe
1.  **Basis Fringe Retrieval:** Retrieve the precomputed basis fringes corresponding to the spectral samples used during encoding [S1].
2.  **Inner Product Calculation:** Compute the inner product between the array of hogel vectors and the basis fringes. This step reconstructs the interference fringe [S1].
    *   *Implementation Note:* This can be accelerated using graphics hardware. The texture mapping function multiplies a component from each hogel vector by a replicated array of a single basis fringe. This operation is repeated for each hogel-vector component, and the results are accumulated in an accumulation buffer [S1].
3.  **Output Generation:** The accumulated result forms the final holographic fringe [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diffraction_specific_cgh(scene_primitives, hogel_grid, spectral_samples, basis_fringes):
    """
    Generates a holographic fringe using the Diffraction-Specific Approach.
    
    Args:
        scene_primitives: List of 3D primitives with color/lighting data.
        hogel_grid: Grid definition for spatial subsampling (hogels).
        spectral_samples: List of viewing directions for spectral subsampling.
        basis_fringes: Precomputed array of basis fringe patterns.
        
    Returns:
        fringe: The reconstructed holographic fringe.
    """
    
    # Step 1: Render Orthographic Projections
    # Render the scene from each direction in spectral_samples
    # using orthographic projection.
    projections = []
    for direction in spectral_samples:
        # Render orthographic view of scene_primitives along 'direction'
        # Result is a 2D image (pixel values representing intensity/color)
        proj = render_orthographic(scene_primitives, direction)
        projections.append(proj)
        
    # Step 2: Convert Projections to Hogel-Vectors
    # Map the pixel data from projections into hogel-vector components.
    # Each hogel in the grid will have a vector of values, one for each spectral sample.
    hogel_vectors = initialize_hogel_vectors(hogel_grid, len(spectral_samples))
    
    for idx, proj in enumerate(projections):
        # Map projection pixels to hogel vector components
        # This involves subsampling the projection to match the hogel grid
        # and assigning the values to the corresponding component of each hogel vector
        map_projection_to_hogels(proj, hogel_vectors, spectral_samples[idx])
        
    # Step 3: Decode Fringe using Basis Fringes
    # Initialize the output fringe buffer
    fringe = zeros_like(basis_fringes[0]) # Assuming all basis fringes have same spatial size
    
    # Accumulate contributions from all hogel vectors and basis fringes
    # This is the inner product step, accelerated by texture mapping in hardware
    for hogel_idx in hogel_grid.indices():
        hv = hogel_vectors[hogel_idx] # The hogel vector for this spatial location
        
        # For each component of the hogel vector (corresponding to a spectral sample)
        for comp_idx, weight in enumerate(hv):
            # Retrieve the corresponding basis fringe
            bf = basis_fringes[comp_idx]
            
            # Multiply the weight by the basis fringe
            # In hardware, this is done via texture mapping:
            # weight is the texture coordinate/value, bf is the texture
            contribution = weight * bf
            
            # Accumulate into the fringe buffer
            fringe += contribution
            
    return fringe
```

### Edge Cases and Pitfalls
*   **Image Sharpness vs. Compression:** Higher compression ratios (e.g., 32:1) result in acceptable but degraded image quality due to added blur. For CR $\le$ 8:1, the blur is invisible to the Human Visual System (HVS) [S1].
*   **Hardware Dependency:** The decoding step relies heavily on multiplication-accumulation (MAC) operations. Performance is significantly tied to the availability of hardware acceleration (e.g., texture mapping and accumulation buffers) [S1].
*   **Basis Fringe Precomputation:** The method requires precomputed basis fringes. The accuracy of the reconstruction depends on the fidelity of these precomputed patterns [S1].

## Subtopics

### Holographic Bandwidth Compression
Techniques to increase fringe computation speed and reduce data volume, including spatial subsampling, nonuniform sampling, and standard video compression algorithms [S1]. Nonuniformly sampled fringes provide lossless bandwidth compression, whereas standard MPEG algorithms can be used for encoding/decoding computed fringes (implying potential loss or different characteristics) [S1]. This approach increases fringe computation speed by 3000 times for the same hardware implementation [S1].

### Hogel-Vector Bandwidth Compression
A technique within the diffraction-specific approach where the fringe is subsampled spatially into hogels and spectrally into hogel vectors. It uses standard 3D computer graphics rendering (orthographic projections) to generate the data. The inner product between hogel vectors and precomputed basis fringes is performed rapidly by exploiting texture mapping and accumulation buffers. This achieved a computation time of 0.9 seconds for fringes of 6-MB with CR=32:1 using Silicon Graphics RealityEngine2 (RE2) [S1].

### Fringelets
A variant that further subsamples the hogel in the spatial domain. Each hogel is encoded as a spatially smaller "fringelet." Using a simple sample-replication decoding scheme, fringelets provide the fastest method of fringe computation. Complex images have been generated in under one second for 6 MB fringes. Additionally, a "fringelet display" can optically decode fringelets to produce a CR-times greater image volume without increased electronic bandwidth [S1].

## Operating characteristics
*   **Computational Complexity:** The encoding stage involves standard 3D rendering, which is efficient. The decoding stage involves many multiplication-accumulation (MAC) calculations, which can be slow on general-purpose CPUs but are accelerated by specialized hardware (DSP cards, graphics subsystems) [S1].
*   **Performance:** For a 6-MB fringe with CR=32:1, computation time was 0.9 seconds on Silicon Graphics RealityEngine2 (RE2) [S1]. With specialized DSP hardware, computation times of about one second were achieved for similar fringe sizes [S1].
*   **Bandwidth Compression:** Achieves significant bandwidth reduction. Compression ratios of 8:1, 16:1, and 32:1 are typical [S1].
*   **Image Quality:** Accompanied by a loss in image sharpness (added blur). For CR $\le$ 8:1, blur is invisible to the HVS. For CR 16:1 or 32:1, good images are achieved with acceptable degradation [S1].
*   **Strengths:** Fast computation for interactive displays, bandwidth compression, leverages existing 3D graphics hardware.
*   **Limitations:** Loss of image sharpness at high compression ratios, reliance on precomputed basis fringes, decoding complexity without hardware acceleration.

## Background
The Diffraction-Specific Approach was introduced as an alternative to the conventional Interference-Based Approach for computing holographic fringes. It was developed to address the high computational cost of simulating optical interference, which requires billions of complex arithmetic operations (trigonometric functions and square roots) for small simple holographic images [S1]. The approach was pioneered by researchers who recognized that the fringe could be treated as a subsampled signal (hogels and hogel vectors) and reconstructed using standard 3D graphics techniques. It was presented in the context of interactive three-dimensional holographic displays, aiming to achieve computation speeds sufficient for real-time or near-real-time holographic video [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
