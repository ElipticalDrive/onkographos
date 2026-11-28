# Diffraction-Specific Approach

Also called: Hogel Vector Approach, Hogel-based CGH, Interference-Based Approach

## Overview
The Diffraction-Specific Approach is a method for calculating Computer-Generated Holograms (CGH) that avoids the direct simulation of optical interference between object and reference waves [S1]. Instead, it treats the holographic fringe as a signal that can be subsampled spatially into functional elements called "hogels" and spectrally into an array of "hogel vectors" [S1]. By working backwards from the 3D image and leveraging standard 3D computer graphics rendering techniques (orthographic projections), this approach significantly increases computation speed and achieves bandwidth compression compared to conventional interference-based methods [S1]. It is distinguished from its sibling [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by its explicit decomposition of the wavefield into hogel vectors and its reliance on texture-mapping hardware for decoding [S1].

## How it works
The method consists of two main stages: encoding (generation of the hogel-vector array) and decoding (reconstruction of the fringe) [S1].

### Inputs and Outputs
*   **Inputs:** A 3D scene defined by primitives (points, polygons) with attributes including color, lighting, shading, and occlusion information [S1].
*   **Outputs:** A holographic fringe pattern (complex amplitude or phase-only data) suitable for display on a Spatial Light Modulator (SLM) [S1].

### Data Structures
1.  **Hogel:** A spatially subsampled functional holographic element that partitions the hologram plane [S1].
2.  **Hogel Vector:** A spectral subsampling of the wavefront information associated with a single hogel, storing values for each projection direction [S1].
3.  **Basis Fringes:** A precomputed set of reference fringe patterns corresponding to the spectral samples, used during the decoding step [S1].

### Parameters
*   **Compression Ratio (CR):** The ratio between the size of the fringe and the hogel-vector array. Typical values are 8:1, 16:1, and 32:1 [S1].
*   **Spatial Subsampling:** Determines the hogel grid resolution [S1].
*   **Spectral Subsampling:** Determines the number of components in each hogel vector [S1].

### Steps

#### 1. Encoding: Generating the Hogel-Vector Array
1.  **3D Scene Preparation:** Perform a 3D filling operation to generate a list of 3D points or primitives, including color, lighting, shading, and occlusion data [S1].
2.  **Orthographic Projections:** Render a series of orthographic projections of the 3D scene. Each projection corresponds to a specific spectral sample (direction) of the hogels, providing a discrete sampling of space (pixels) and spectrum (projection direction) [S1].
3.  **Vector Conversion:** Convert the rendered orthographic projections into a hogel-vector array by mapping the pixel values from the projections to the components of the hogel vectors [S1].

#### 2. Decoding: Recovering the Fringe
1.  **Basis Fringe Retrieval:** Retrieve the precomputed basis fringes corresponding to the spectral samples used during encoding [S1].
2.  **Inner Product Calculation:** Compute the inner product between the array of hogel vectors and the basis fringes to reconstruct the interference fringe [S1].
    *   *Implementation Note:* This is accelerated using graphics hardware. The texture mapping function multiplies a component from each hogel vector by a replicated array of a single basis fringe. This operation is repeated for each hogel-vector component, and the results are accumulated in an accumulation buffer [S1].
3.  **Output Generation:** The accumulated result forms the final holographic fringe [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diffraction_specific_cgh(scene, hogel_grid, spectral_directions, basis_fringes):
    """
    Generates a holographic fringe using the Diffraction-Specific Approach.
    
    Args:
        scene: List of 3D primitives with color/lighting data.
        hogel_grid: Grid definition for spatial subsampling (hogels).
        spectral_directions: List of viewing directions for spectral subsampling.
        basis_fringes: Precomputed array of basis fringe patterns.
        
    Returns:
        fringe: The reconstructed holographic fringe.
    """
    
    # Step 1: Encoding - Generate Hogel-Vector Array
    # Render orthographic projections for each spectral direction
    projections = []
    for direction in spectral_directions:
        # Standard 3D rendering of scene along direction
        proj = render_orthographic(scene, direction)
        projections.append(proj)

    # Convert projections into hogel-vector array
    # Each hogel in the grid stores a vector of values, one per spectral direction
    hogel_vectors = create_empty_hogel_array(hogel_grid, len(spectral_directions))
    for idx, proj in enumerate(projections):
        # Map projection pixels to hogel grid cells
        for cell in hogel_grid.cells():
            # Aggregate projection data for this cell into the vector component
            hogel_vectors[cell][idx] = aggregate_projection_data(proj, cell)

    # Step 2: Decoding - Recover Fringe
    # Initialize output fringe buffer
    fringe = create_zeros_like(basis_fringes[0])

    # Compute inner product between hogel vectors and basis fringes
    # Exploits texture mapping: multiply each vector component by its basis fringe
    for cell in hogel_grid.cells():
        hv = hogel_vectors[cell]
        for comp_idx in range(len(hv)):
            # Multiply hogel vector component by corresponding basis fringe
            contribution = hv[comp_idx] * basis_fringes[comp_idx]
            # Accumulate result in buffer
            fringe += contribution

    return fringe

def render_orthographic(scene, direction):
    # Standard computer graphics rendering of scene using orthographic projection
    return render(scene, projection_type="orthographic", view_direction=direction)

def create_empty_hogel_array(grid, num_components):
    # Initialize array where each grid cell holds a list of num_components values
    return {cell: [0.0] * num_components for cell in grid.cells()}

def aggregate_projection_data(projection, cell):
    # Sample or average pixel values from the projection corresponding to the cell
    return mean(projection[cell.region])

def create_zeros_like(array):
    # Create a new array of the same shape filled with zeros
    return [[0.0] * len(row) for row in array]
```

### Edge Cases and Pitfalls
*   **Image Sharpness vs. Compression:** Higher compression ratios (e.g., 32:1) result in acceptable but degraded image quality due to added blur [S1]. For CR $\le$ 8:1, the blur is invisible to the Human Visual System (HVS) [S1].
*   **Hardware Dependency:** The decoding step relies heavily on multiplication-accumulation (MAC) operations, making performance significantly tied to the availability of hardware acceleration (e.g., texture mapping and accumulation buffers) [S1].
*   **Basis Fringe Precomputation:** The method requires precomputed basis fringes, and reconstruction accuracy depends on their fidelity [S1].

## Subtopics

### Holographic Bandwidth Compression
Techniques to increase fringe computation speed and reduce data volume, including spatial subsampling, nonuniform sampling, and standard video compression algorithms [S1]. Nonuniformly sampled fringes provide lossless bandwidth compression, whereas standard MPEG algorithms can be used for encoding/decoding computed fringes [S1]. This approach increases fringe computation speed by 3000 times for the same hardware implementation [S1].

### Hogel-Vector Bandwidth Compression
A technique within the diffraction-specific approach where the fringe is subsampled spatially into hogels and spectrally into hogel vectors [S1]. It uses standard 3D computer graphics rendering (orthographic projections) to generate the data [S1]. The inner product between hogel vectors and precomputed basis fringes is performed rapidly by exploiting texture mapping and accumulation buffers [S1]. This achieved a computation time of 0.9 seconds for fringes of 6-MB with CR=32:1 using Silicon Graphics RealityEngine2 (RE2) [S1].

### Fringelets
A variant that further subsamples the hogel in the spatial domain [S1]. Each hogel is encoded as a spatially smaller "fringelet" [S1]. Using a simple sample-replication decoding scheme, fringelets provide the fastest method of fringe computation [S1]. Complex images have been generated in under one second for 6 MB fringes [S1]. Additionally, a "fringelet display" can optically decode fringelets to produce a CR-times greater image volume without increased electronic bandwidth [S1].

## Operating characteristics
*   **Computational Complexity:** The encoding stage involves standard 3D rendering, which is efficient [S1]. The decoding stage involves many multiplication-accumulation (MAC) calculations, which can be slow on general-purpose CPUs but are accelerated by specialized hardware (DSP cards, graphics subsystems) [S1].
*   **Performance:** For a 6-MB fringe with CR=32:1, computation time was 0.9 seconds on Silicon Graphics RealityEngine2 (RE2) [S1]. With specialized DSP hardware, computation times of about one second were achieved for similar fringe sizes [S1].
*   **Bandwidth Compression:** Achieves significant bandwidth reduction [S1]. Compression ratios of 8:1, 16:1, and 32:1 are typical [S1].
*   **Image Quality:** Accompanied by a loss in image sharpness (added blur) [S1]. For CR $\le$ 8:1, blur is invisible to the HVS [S1]. For CR 16:1 or 32:1, good images are achieved with acceptable degradation [S1].
*   **Strengths:** Fast computation for interactive displays, bandwidth compression, leverages existing 3D graphics hardware [S1].
*   **Limitations:** Loss of image sharpness at high compression ratios, reliance on precomputed basis fringes, decoding complexity without hardware acceleration [S1].

## Background
The Diffraction-Specific Approach was introduced as an alternative to the conventional Interference-Based Approach for computing holographic fringes [S1]. It was developed to address the high computational cost of simulating optical interference, which requires billions of complex arithmetic operations (trigonometric functions and square roots) for small simple holographic images [S1]. The approach was pioneered by researchers who recognized that the fringe could be treated as a subsampled signal (hogels and hogel vectors) and reconstructed using standard 3D graphics techniques [S1]. It was presented in the context of interactive three-dimensional holographic displays, aiming to achieve computation speeds sufficient for real-time or near-real-time holographic video [S1].

## Sources
[S1] Interactive three-dimensional holographic displays, https://doi.org/10.1145/271283.271312, fetched 2026-11-05
