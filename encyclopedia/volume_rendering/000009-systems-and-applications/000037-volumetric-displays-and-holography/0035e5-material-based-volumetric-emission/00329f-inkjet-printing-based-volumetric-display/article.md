# Inkjet Printing-Based Volumetric Display

Also called: Fluorescent Ink Volumometric Display, Volumetric Display via Inkjet Printing

## Overview
This method constructs a physical volumetric display by printing layers of photoreactive luminescent inks onto transparent films and stacking them to form a 3D volume [S1]. It solves the problem of creating full-color 3D images using commercially available printing technology, enabling the projection of multiple distinct 2D patterns from different viewing angles [S1]. Unlike sibling methods that rely on scanning beams or holography, this approach defines voxels as discrete printed layers, relying on external excitation light to trigger fluorescence [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   **3D Model Data:** A set of 3D points or cross-sectional images defining the volumetric content. For multi-pattern projection, input includes multiple 2D patterns (e.g., $512 \times 512$ pixels, 3 channels, 256 gradations each) and their desired projection directions [S1].
*   **Material Parameters:**
    *   Film thickness: $0.1$ mm [S1].
    *   Film material: Polyester transparent film (e.g., 'BG-32') with UV transmittance of $82\%$ at $365$ nm [S1].
    *   Ink composition: Europium complex (Red), $\beta$-quinophthalone (Green), Coumarin dyes (Blue) [S1].
    *   Printer resolution: Maximum $5,760 \times 1,440$ dpi [S1].
*   **Excitation Source:** Ultraviolet light peaked at $365$ nm [S1].

**Outputs:**
*   **Physical Prototype:** A stack of $N$ printed films with defined spacing, forming a volume of dimensions $35 \text{ mm} \times 35 \text{ mm} \times 12.5 \text{ mm}$ (using 20 layers) [S1].
*   **Visual Output:** A 3D image or multiple 2D patterns visible when the volume is illuminated by UV light [S1].

### Data Structures
The data structure consists of a list of layers. Each layer $i$ contains a 2D grid of pixel values.
*   **Layer:** A 2D array of size $W \times H$.
*   **Pixel:** A tuple $(R, G, B)$ where each component is an integer in $[0, 255]$ representing the intensity of the respective fluorescent ink [S1].
*   **Volume:** A list of Layers, indexed by depth $z$.

### Algorithm: Designing Multi-Pattern Projection
To create a volume that projects different 2D patterns from specific angles, the voxel values are determined by an algorithm that aggregates contributions from all desired patterns based on their projection geometry [S1].

1.  **Define Projection Geometry:**
    *   Let $P$ be the set of target patterns.
    *   For each pattern $p \in P$, define a projection direction vector $\mathbf{v}_p$.
    *   Example: Pattern B projects perpendicular to films ($\mathbf{v}_B = [0, 0, 1]$). Patterns A and C project at $\pm 30^\circ$ around the Y-axis [S1].
    *   For four patterns, rotate the Z-axis $\pm 20^\circ$ around both X- and Y-axes [S1].

2.  **Initialize Volume:**
    *   Create a 3D volume grid $V$ of size $W \times H \times D$, initialized to zero. $D$ is the number of layers (e.g., 20) [S1].

3.  **Determine Voxel Contributions:**
    *   For each target pattern $p$:
        *   Project the 2D pattern image onto the 3D volume along direction $\mathbf{v}_p$.
        *   This projection maps 2D pixel coordinates $(u, v)$ to 3D voxel coordinates $(x, y, z)$.
        *   Accumulate the RGB values of the pattern into the corresponding voxels in $V$.
        *   *Note:* The sources state that simulations did not account for light absorption by films or inks during this design phase [S1].

4.  **Quantize and Clip:**
    *   For each voxel in $V$, clamp the RGB values to $[0, 255]$ to fit the 8-bit channel representation [S1].

### Fabrication Steps
1.  **Print:** Use a commercial inkjet printer to print the 2D patterns from the designed layers onto transparent polyester films [S1].
2.  **Stack:** Stack the printed films at regular intervals. In the prototype, 20 layers were stacked at $0.5$ mm intervals [S1].
3.  **Excite:** Irradiate the stack with UV light ($365$ nm) [S1].
    *   To mitigate UV absorption by the films, use two UV light sources placed on the top and bottom of the display [S1].
4.  **Observe:** View the emitted visible light from the desired angles [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def design_volumetric_display(patterns, projection_vectors, volume_dims):
    """
    Designs a 3D volume to project multiple 2D patterns from specific directions.
    
    Args:
        patterns: List of 2D numpy arrays (H x W x 3), each representing a target image.
        projection_vectors: List of 3D vectors (x, y, z) corresponding to each pattern.
        volume_dims: Tuple (W, H, D) defining the width, height, and depth (layers) of the volume.
        
    Returns:
        volume: 3D numpy array (W x H x D x 3) containing the ink distribution.
    """
    W, H, D = volume_dims
    # Initialize volume with zeros. Shape: (W, H, D, 3) for RGB.
    volume = np.zeros((W, H, D, 3), dtype=np.float32)
    
    for pattern, vector in zip(patterns, projection_vectors):
        # Normalize the projection vector
        vec_norm = vector / np.linalg.norm(vector)
        
        # Project the 2D pattern onto the 3D volume
        # This step involves mapping 2D pixel coordinates to 3D voxel coordinates
        # along the direction of vec_norm.
        # Since the exact projection algorithm is not detailed in the source,
        # we assume a standard ray-casting or back-projection approach.
        
        # For each pixel (u, v) in the pattern:
        for y in range(pattern.shape[0]):
            for x in range(pattern.shape[1]):
                color = pattern[y, x]
                
                # Calculate 3D coordinates based on projection geometry
                # The source implies a linear projection. 
                # Let's assume the pattern is projected through the volume.
                # The exact mapping depends on the specific geometry defined by the vector.
                # For a perpendicular projection (0,0,1), the pattern is replicated across Z.
                # For angled projections, the pattern shifts across Z.
                
                # Placeholder for the specific geometric projection logic
                # which maps (x, y) from pattern to (x', y', z') in volume.
                # The source [S1] mentions rotating axes for 4-pattern projection.
                
                # Example logic for perpendicular projection:
                if np.allclose(vec_norm, [0, 0, 1]):
                    for z in range(D):
                        volume[x, y, z] += color
                else:
                    # General case: project along vector
                    # Determine which voxels are intersected by the ray from the pattern pixel
                    # This requires a ray-tracing step through the volume grid.
                    # The source does not provide the exact formula for this mapping.
                    pass

    # Quantize to 8-bit
    volume = np.clip(volume, 0, 255)
    volume = volume.astype(np.uint8)
    
    return volume

def fabricate_display(volume, layer_spacing=0.5):
    """
    Simulates the fabrication process by printing layers and stacking them.
    
    Args:
        volume: 3D numpy array (W x H x D x 3).
        layer_spacing: Distance between layers in mm.
        
    Returns:
        stack: List of printed films.
    """
    stack = []
    D = volume.shape[2]
    
    for z in range(D):
        # Extract the 2D pattern for this layer
        layer_pattern = volume[:, :, z, :]
        
        # Print to film
        film = print_to_film(layer_pattern)
        stack.append(film)
        
    # Stack films with spacing
    physical_display = Stack(stack, spacing=layer_spacing)
    return physical_display

def excite_display(display, uv_wavelength=365):
    """
    Excites the fluorescent inks with UV light.
    
    Args:
        display: The physical stack of films.
        uv_wavelength: Wavelength of UV light in nm.
        
    Returns:
        visible_emission: The resulting 3D image or projected patterns.
    """
    # The source notes that UV light is attenuated by the films.
    # Transmittance T_UV = 0.82 per film.
    # If N_UV films are between source and pattern, intensity is I = I0 * (0.82)^N_UV.
    
    # To mitigate attenuation, use top and bottom UV sources.
    # The exact optical simulation is not provided, but the result is fluorescence.
    
    emission = simulate_fluorescence(display, uv_wavelength)
    return emission
```

### Parameters and Typical Values
*   **Film Thickness:** $0.1$ mm [S1].
*   **Layer Spacing:** $0.5$ mm [S1].
*   **Number of Layers:** 20 (in the prototype) [S1].
*   **Rendering Space:** $35 \text{ mm} \times 35 \text{ mm} \times 12.5 \text{ mm}$ [S1].
*   **Pattern Resolution:** $512 \times 512$ pixels, 3 channels (RGB), 256 gradations per channel [S1].
*   **UV Wavelength:** $365$ nm [S1].
*   **UV Transmittance:** $82\%$ per film at $365$ nm [S1].

### Edge Cases and Pitfalls
*   **UV Attenuation:** UV light is significantly attenuated as it passes through multiple films. With $N_{UV} = 20$ films between the source and the pattern, UV intensity drops to $2\%$. With $N_{UV} = 25$, it drops to $<1\%$ [S1]. This reduces brightness and contrast.
*   **Viewing Angle Degradation:** Image quality degrades when viewed from diagonal directions (e.g., $\theta = \pm 30^\circ$), resulting in blurred images, although motion parallax is preserved [S1].
*   **Cross-talk:** In simulations, cross-talk (deterioration caused by other patterns) occurs, but projected patterns are still recognizable [S1].
*   **Sharpness Loss:** Increasing the number of films between the pattern and the camera ($N_{VIS}$) leads to a decrease in sharpness due to the higher transmittance in the visible region ($\sim 90\%$) compared to UV, but the physical separation causes blurring [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Fabrication Time:** Dependent on the number of layers and printer resolution. The prototype used 20 layers printed at $5,760 \times 1,440$ dpi [S1].
*   **Memory:** The volume data structure is $W \times H \times D \times 3$ bytes. For a $512 \times 512 \times 20$ volume, this is approximately $20$ MB [S1].

### Measured Performance
*   **Resolution:** The printer supports up to $5,760 \times 1,440$ dpi [S1].
*   **Brightness:** Decreases with the number of layers due to UV absorption. Top and bottom UV sources are required to maintain visibility for deep layers [S1].
*   **Viewing Angle:** Best quality at $\theta = 0^\circ$ (perpendicular). Blurring occurs at $\theta = \pm 30^\circ$ [S1].

### Strengths and Limitations
*   **Strengths:** Uses commercially available inkjet printers and transparent films. Can project multiple different 2D patterns from different directions. Full-color capability [S1].
*   **Limitations:** Image quality degrades at oblique angles. UV attenuation limits the maximum number of layers. Cross-talk between patterns can occur [S1].

## Background
This method was introduced in a study by Hirayama et al. [S1]. It represents an early prototype of a full-color volumetric display using inkjet printing technology. The work extends previous algorithms for multi-pattern projection to full-color representations [S1]. It is considered suitable for applications such as digital signage, media art, entertainment, and security [S1].

## Sources
[S1] Inkjet printing-based volumetric display projecting multiple full-colour 2D patterns., https://doi.org/10.1038/srep46511, fetched 2026-10-26
