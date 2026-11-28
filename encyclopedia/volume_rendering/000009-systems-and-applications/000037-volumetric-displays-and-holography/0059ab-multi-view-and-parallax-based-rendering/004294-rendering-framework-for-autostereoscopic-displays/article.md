# Rendering Framework for Autostereoscopic Displays

Also called: Hologram Pre-processing Framework for Light Field Displays

## Overview
This framework preprocesses a computer-generated hologram (CGH) to generate a set of reconstruction images corresponding to views from a camera rotating around the scene, enabling the display of holograms on autostereoscopic light field displays [S1]. It solves the problem of rendering volumetric holographic data on displays that lack continuous depth but offer discrete multi-view parallax [S1]. It differs from standard multi-view rendering by operating directly on the holographic aperture data to preserve diffraction effects, rather than rendering geometric primitives [S1].

## How it works
The framework consists of four sequential pre-processing steps applied to the hologram to acquire a set of reconstruction images [S1]. The resolution of these images equals the 2D resolution of the screen [S1].

**Inputs and Outputs:**
*   **Input:** A computer-generated hologram (CGH) of resolution $M \times N$ [S1].
*   **Output:** A set of reconstruction images (views) corresponding to the display's angular coverage [S1].

**Data Structures:**
*   **Hologram:** A 2D array of complex values representing the holographic aperture [S1].
*   **Views:** A sequence of 2D images, each with resolution matching the screen's 2D resolution (e.g., $1280 \times 768$) [S1].

**Steps:**

1.  **View Extraction:**
    A window with the resolution of the vertical axis of the hologram moves along the horizontal axis to acquire sub-holograms that simulate a pinhole camera view [S1]. The number of views is determined by the targeted field of view (FOV) and angular resolution [S1]. Overlapping apertures can be chosen to improve rendering quality [S1].

2.  **Suppression of Aperture Diffraction (Apodization):**
    This step suppresses diffraction artifacts caused by the finite aperture of the extracted views [S1]. The specific mathematical operation is not detailed in the provided notes, but it is applied to the sub-holograms obtained in Step 1 [S1].

3.  **Back Propagation:**
    Numerical back propagation is performed on the apodized sub-holograms to reconstruct the light field at the display plane [S1]. This step requires only 1D Fast Fourier Transforms (FFTs) with a pre-computed kernel due to the constant distance [S1].

4.  **Colour Balancing:**
    Color balancing is applied to the reconstructed views to ensure correct color reproduction on the display [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_hologram_to_views(hologram, num_views, screen_resolution):
    """
    Pre-processes a hologram to generate views for an autostereoscopic display.
    
    Args:
        hologram: 2D array of complex values, shape (M, N).
        num_views: Integer, number of views to extract.
        screen_resolution: Tuple (width, height), resolution of the output views.
        
    Returns:
        list of 2D arrays, each representing a view.
    """
    views = []
    # Step 1: View Extraction
    # A window of height equal to the hologram's vertical axis moves along the horizontal axis.
    # The number of views is determined by the FOV and angular resolution.
    # Note: Exact windowing function and stride are not specified in sources [S1].
    sub_holograms = extract_views(hologram, num_views)
    
    processed_views = []
    for sub_holo in sub_holograms:
        # Step 2: Suppression of aperture diffraction (Apodization)
        # Apply apodization to suppress diffraction artifacts.
        # Note: Specific apodization function/window is not detailed in sources [S1].
        apodized_holo = apply_apodization(sub_holo)
        
        # Step 3: Back propagation
        # Numerical back propagation using 1D FFTs with pre-computed kernel.
        # Note: The exact propagation distance and kernel formula are not detailed in sources [S1].
        reconstructed_view = back_propagate(apodized_holo)
        
        # Step 4: Colour balancing
        # Apply color balancing to the reconstructed view.
        # Note: Specific color balancing algorithm is not detailed in sources [S1].
        balanced_view = apply_color_balancing(reconstructed_view)
        
        processed_views.append(balanced_view)
        
    return processed_views

def extract_views(hologram, num_views):
    """
    Extracts sub-holograms by moving a window along the horizontal axis.
    Note: Implementation details (window size, stride, overlap) are not provided in sources [S1].
    """
    # Placeholder for view extraction logic based on [S1] description.
    # The window resolution equals the vertical axis of the hologram.
    pass

def apply_apodization(sub_hologram):
    """
    Suppresses aperture diffraction.
    Note: Specific apodization method is not provided in sources [S1].
    """
    pass

def back_propagate(apodized_holo):
    """
    Performs numerical back propagation using 1D FFTs.
    Note: Specific kernel and distance parameters are not provided in sources [S1].
    """
    pass

def apply_color_balancing(view):
    """
    Applies color balancing.
    Note: Specific algorithm is not provided in sources [S1].
    """
    pass
```

**Parameters:**
*   **Number of Views:** Determined by the display's viewing angle and angular resolution. For a 70° viewing angle with 1° angular resolution, 70 views are extracted [S1].
*   **View Resolution:** Equals the 2D resolution of the screen (e.g., $1280 \times 768$ pixels) [S1].
*   **Camera Focus:** The camera is typically focused at the front part of the object [S1].

**Edge Cases and Pitfalls:**
*   **FOV and Complexity:** Larger FOVs require more views, increasing computational complexity [S1].
*   **Occlusion Handling:** The framework handles occlusion in the numerical reconstruction, but specific details are not provided [S1].

## Subtopics

### Beam Splitter (Half Mirror) Display
A reflection-based autostereoscopic display that uses a beam combiner to superimpose left/right LCD images, with field lenses directing illumination to appropriate eyes to avoid pixel registration tolerances [S2, unofficial]. A field lens is placed at the focus of a real (aerial) image to collimate rays passing through without affecting geometry [S2, unofficial]. Two LCD panels display left and right views, superimposed by a beam combiner. Field lenses direct illumination beams into the appropriate eye, making the right-view image appear dark to the left eye and vice versa [S2, unofficial]. Head-tracking illumination requires movable light sources, e.g., monochrome CRTs displaying high-contrast camera images of the left and right halves of the observer’s face [S2, unofficial]. Multiple-user access is possible using multiple independent illuminators [S2, unofficial]. Proposed by Woodgate et al. [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The computational complexity of the framework is $O(NM)$, where $N$ and $M$ are the dimensions of the hologram [S1].
*   **Performance:** On a 3.3 GHz Intel Core i7-5820 CPU with Windows 8 (x64), for the 'Dice' dataset (142,903 points), processing times per view were: View extraction 0.04s, Apodization 0.38s, Back propagation 16.33s, Colour balancing 1.19s [S1].
*   **Hardware:** Tested on an Holografika HoloVizio 722RC light field display (72 projectors, 70° viewing angle) [S1].
*   **Scaling:** The back propagation step is the most computationally expensive, requiring 1D FFTs with a pre-computed kernel due to the same distance [S1].
*   **Strengths:** Preserves diffraction effects of the hologram; suitable for high-end 3D visualization on light field displays [S1].
*   **Limitations:** High computational cost, especially for back propagation; requires pre-processing for each hologram [S1].

## Background
The framework was proposed for the three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays [S1]. It addresses the challenge of displaying holographic data on autostereoscopic displays that use an array of projectors to create a light field [S1]. The Holografika HoloVizio 722RC display, used in the experiments, is a scalable light field system design based on a specially arranged array of 72 projectors [S1].

## Sources
[S1] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
[S2] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
