# Visualization by Example / Constructive Visual Component-based Interface

Also called: Programming-by-example approach, Spreadsheet-like interface

## Overview
This method enables users to create direct volume rendering (DVR) visualizations by combining automatically generated unit transfer functions, requiring no knowledge of the DVR process or transfer function design [S1, unofficial]. It distinguishes itself from its parent method, Histogram Peak Interpolation, by providing a constructive, spreadsheet-like interface where users select and merge pre-computed visualizations rather than manually interpolating between histogram peaks [S1, unofficial]. Unlike traditional manual transfer function design, it prioritizes speed and ease of use for inexperienced users at the cost of precise control over material boundaries and opacity [S1, unofficial].

## How it works
The interface operates as a spreadsheet-like grid where each cell represents a visualization. The core mechanism involves the automatic generation of "unit transfer functions" from the data histogram and their subsequent combination by the user.

### Inputs and Outputs
*   **Input:** A scalar volume dataset (e.g., CT scan, simulation data).
*   **Output:** A composite transfer function and the resulting volume rendering.

### Data Structures
The system is implemented within the Voreen framework [S1, unofficial].
*   **TransFuncIntensity:** Each unit transfer function is stored as a `TransFuncIntensity` object.
*   **VoreenPainter:** Each `TransFuncIntensity` is associated with a `VoreenPainter` object to handle the rendering pipeline [S1, unofficial].
*   **QtCanvas:** The user interface uses a QtCanvas to represent each cell in the spreadsheet, allowing users to interact with the visualizations directly [S1, unofficial].

### Unit Transfer Function Generation
Unit transfer functions are automatically generated to capture specific structures in the data.
1.  **Histogram Analysis:** The 1D histogram of the volume data is computed.
2.  **Peak Detection:** The Douglas-Peucker algorithm is applied to the histogram curve to identify peaks.
3.  **Epsilon Iteration:** The algorithm uses a decreasing epsilon ($\epsilon$) threshold. The process iterates until three peaks are found or $\epsilon$ reaches a minimum value of 2 [S1, unofficial].
4.  **Shape Definition:** Each identified peak corresponds to a unit transfer function with a fixed **triangle shape**.
5.  **Opacity Assignment:** Opacity values are assigned based on the peak's position or magnitude, typically increasing for higher density peaks. Users cannot modify these opacity values [S1, unofficial].
6.  **Color Assignment:** Colors are assigned randomly to each unit transfer function [S1, unofficial].

### User Operations
The user interacts with the interface through two primary functions:

#### 1. Interpolation
This function generates intermediate visualizations between two selected user visualizations.
*   **Selection:** The user selects two existing visualizations (cells) in the spreadsheet.
*   **Histogram Examination:** The system examines the section of the histogram between the peaks of the corresponding transfer functions.
*   **Peak Finding:** It attempts to find the three largest peaks within this histogram section.
*   **Fallback:** If no peaks are found in the section, the system places new transfer functions equidistantly between the two selected peaks.
*   **Algorithm:** The peak finding is achieved using the Douglas-Peucker algorithm with decreasing $\epsilon$ values until three peaks are found or $\epsilon = 2$ [S1, unofficial].

#### 2. Merge
This function combines selected visualizations into a single composite transfer function.
*   **Combination:** It combines the unit transfer functions of the selected visualizations.
*   **Overlap Removal:** It removes any overlapping sections between the combined transfer functions. This ensures that the resulting transfer function is a valid, non-redundant mapping [S1, unofficial].

### Pseudocode
The following pseudocode illustrates the core logic of the interpolation and merge operations. Note that this is a simplified representation of the logic described in the source; actual implementation relies on the Voreen framework's C++ classes and Qt UI components.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_unit_transfer_functions(histogram_data):
    """
    Generates unit transfer functions by detecting peaks in the histogram.
    Returns a list of TransFuncIntensity objects.
    """
    unit_functions = []
    # Douglas-Peucker algorithm with decreasing epsilon
    epsilon = initial_value
    while epsilon > 2:
        # Simplified Douglas-Peucker peak detection
        peaks = douglas_peucker_peaks(histogram_data, epsilon)
        
        # Select the three largest peaks if available
        if len(peaks) >= 3:
            top_peaks = sort_by_magnitude(peaks)[:3]
        else:
            top_peaks = peaks
            
        for peak in top_peaks:
            # Create a triangle-shaped unit transfer function
            # Opacity is fixed and cannot be modified by user
            tf = TransFuncIntensity(
                center=peak.value,
                width=peak.width,
                shape='triangle',
                opacity=calculate_opacity(peak) # Fixed logic
            )
            unit_functions.append(tf)
            
        epsilon -= step_size
        
    return unit_functions

def interpolate_visualizations(vis_a, vis_b, histogram_data):
    """
    Interpolates between two visualizations by finding intermediate peaks.
    """
    peak_a = vis_a.transfer_function.center
    peak_b = vis_b.transfer_function.center
    
    # Define the histogram section between the two peaks
    start_idx = index_of_value(histogram_data, peak_a)
    end_idx = index_of_value(histogram_data, peak_b)
    section = histogram_data[start_idx:end_idx]
    
    # Try to find three largest peaks in the section
    peaks_in_section = find_peaks(section)
    
    if len(peaks_in_section) >= 3:
        # Use the three largest peaks
        intermediate_peaks = sort_by_magnitude(peaks_in_section)[:3]
    else:
        # Place transfer functions equidistantly
        intermediate_peaks = equidistant_points(peak_a, peak_b, count=3)
        
    # Generate new unit transfer functions for these peaks
    new_vis = []
    for peak in intermediate_peaks:
        tf = TransFuncIntensity(
            center=peak,
            shape='triangle',
            opacity=calculate_opacity(peak)
        )
        new_vis.append(create_visualization(tf))
        
    return new_vis

def merge_visualizations(vis_list):
    """
    Merges multiple visualizations by combining their transfer functions
    and removing overlapping sections.
    """
    combined_tf = TransFuncIntensity()
    
    for vis in vis_list:
        tf = vis.transfer_function
        # Combine the transfer functions
        combined_tf = combine_tf(combined_tf, tf)
        
    # Remove overlapping sections
    final_tf = remove_overlaps(combined_tf)
    
    return create_visualization(final_tf)
```

### Parameters
*   **Douglas-Peucker Epsilon ($\epsilon$):** The threshold for peak detection. It is decreased iteratively. The process stops when three peaks are found or when $\epsilon$ reaches 2 [S1, unofficial].
*   **Unit Transfer Function Shape:** Always triangle [S1, unofficial].
*   **Opacity:** Cannot be modified by the user [S1, unofficial].

### Edge Cases and Pitfalls
*   **Color Ambiguity:** Colors are assigned randomly. If two different materials have very similar colors, ambiguity arises [S1, unofficial].
*   **Opacity Limitation:** The use of increasing opacity values for histogram peaks works well for objects where innermost structures have highest densities (e.g., CT scans) but is not suitable for general applications [S1, unofficial].
*   **Histogram Limitation:** Using a one-dimensional histogram and restricting peak identification is insufficient for identifying all structures in a data set, particularly those not corresponding to histogram peaks or enhanced by shading [S1, unofficial].
*   **Spatial Unrelatedness:** Neighboring peaks in the histogram can correspond to spatially completely unrelated structures, leading to confusion about spatial relationships [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given. However, usability tests suggest that meaningful visualizations can be constructed in less than a minute [S1, unofficial].
*   **Memory Complexity:** Not explicitly given. The system stores `TransFuncIntensity` objects and associated `VoreenPainter` objects for each visualization cell [S1, unofficial].
*   **Measured Performance:**
    *   **NCAT Phantom (8 Bit, $128^3$ voxels):** Inexperienced user took less than 30 seconds; experienced user took about 90 seconds with the original Voreen interface [S1, unofficial].
    *   **Nucleon Dataset (8 Bit, $4^{13}$ voxels):** Inexperienced student created meaningful visualization with both interfaces [S1, unofficial].
    *   **Bonsai CT Dataset (8 Bit, $256^3$ voxels):** Visualized using the spreadsheet-like interface [S1, unofficial].
    *   **CT Scan of Two Feet (16 Bit, $256 \times 256 \times 250$ voxels):** Provided by Osirix, visualized using the interface [S1, unofficial].
*   **Strengths:** Faster than traditional transfer function design; requires no knowledge of DVR; intuitive for inexperienced users [S1, unofficial].
*   **Limitations:** Less accurate results due to lack of direct control; manual transfer functions result in sharper material boundaries and more appropriate opacity values, making shape perception slightly easier [S1, unofficial]. Users struggled with datasets where there is no clear relationship between data set features and histogram features (e.g., CT feet) [S1, unofficial].

## Background
*   **Origin:** Introduced in GRAPP 2010 - International Conference on Computer Graphics Theory and Applications [S1, unofficial].
*   **Implementation:** Implemented in Voreen (Meyer-Spradow et al., 2009), a rapid-prototyping environment for ray-casting-based volume visualizations, using C++ and Qt UI framework [S1, unofficial].
*   **History:** The approach combines a "programming-by-example" method with automatic transfer function design. It builds on the concept of spreadsheet-like interfaces for visualization exploration [S1, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28441/, fetched 2026-10-29 (unofficial)
