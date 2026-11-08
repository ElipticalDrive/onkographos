# MDTF Design Workflow

Also called: Multi-dimensional Transfer Function Design Workflow

## Overview
The MDTF Design Workflow is a semi-automatic, exploratory method for constructing multi-dimensional transfer functions (TFs) by iteratively defining attributes and selecting regions in sorted histograms [S1, unofficial]. It distinguishes itself from static histogram-based methods by allowing the user to dynamically change the second attribute of the 2D histogram and refine material definitions through direct range selection, enabling the incorporation of any classifying information in a uniform way [S1, unofficial].

## How it works
The method operates on a 2D histogram where the primary attribute is typically the data value (intensity) and the secondary attribute is selected by the user (e.g., gradient magnitude, first derivative, or second derivative) [S1, unofficial]. The workflow is iterative and consists of the following steps:

1.  **Define Attribute and Histogram Type**: The user selects a second attribute for the 2D histogram and chooses a suitable histogram type (e.g., sorted histograms are preferred as their shape remains stable when changing the second attribute) [S1, unofficial].
2.  **Refine MDTF via Selection**: The user selects an area in the histogram to define a material criterion. This is performed with one click to set the range center and mouse movements to adjust the range size in both dimensions [S1, unofficial]. The selected voxels are marked in the histogram, and the criterion boundary is displayed in the current dimensions [S1, unofficial].
3.  **Iterate**: The user concludes which attribute to study next and restarts from Step 1 if further refinement is needed [S1, unofficial].

The resulting transfer function is mathematically defined as a range selection function $\Theta(\mathbf{x})$, defined as a product of threshold functions $\theta_i(s_i(\mathbf{x}))$ with a value range of $[0.0, 1.0]$ [S1, unofficial].

$$ \Theta(\mathbf{x}) = \prod_{i} \theta_i(s_i(\mathbf{x})) $$

Where:
*   $\mathbf{x}$ is the input data point (voxel).
*   $s_i(\mathbf{x})$ is the $i$-th attribute value for $\mathbf{x}$.
*   $\theta_i$ is a threshold function corresponding to the selected range in the histogram for attribute $i$.

Typical values for color and opacity are set in a separate control interface after the range selection [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mdtf_design_workflow(data, initial_attribute, initial_histogram_type):
    """
    Iterative workflow for MDTF design.
    data: 3D array of voxel values.
    initial_attribute: The primary attribute (e.g., intensity).
    initial_histogram_type: The type of histogram to use (e.g., sorted).
    """
    # Initialize the transfer function as a product of threshold functions
    # Theta(x) = product(theta_i(s_i(x)))
    transfer_function_components = []
    
    current_attribute = initial_attribute
    current_histogram_type = initial_histogram_type
    
    while user_wants_to_continue():
        # Step 1: Define second attribute and select histogram type
        # In practice, this updates the view of the 2D histogram
        histogram = compute_2d_histogram(data, current_attribute, current_histogram_type)
        display_histogram(histogram)
        
        # Step 2: Refine MDTF by selecting an area
        # User interaction: click to set center, mouse move to set size
        selected_range = get_user_selection_from_histogram(histogram)
        
        # Create a threshold function for the selected range
        # theta_i(s_i(x)) returns 1.0 if s_i(x) is in selected_range, else 0.0
        threshold_func = create_threshold_function(selected_range)
        
        # Add to the product of components
        transfer_function_components.append(threshold_func)
        
        # Update the visual feedback
        update_histogram_markers(selected_range)
        update_color_opacity_controls()
        
        # Step 3: Conclude next attribute
        next_attribute = get_next_attribute_from_user()
        if next_attribute is None:
            break
        current_attribute = next_attribute
        # Histogram type may be updated based on user choice, defaulting to sorted
        current_histogram_type = get_histogram_type_preference()

    # Final Transfer Function Construction
    def final_tf(x):
        result = 1.0
        for theta in transfer_function_components:
            # Evaluate attribute s_i(x)
            attr_val = evaluate_attribute(x, current_attribute) # Note: attribute depends on context
            # Apply threshold
            result *= theta(attr_val)
        return result
    
    return final_tf
```

**Edge Cases and Pitfalls**:
*   **Precision**: The range selection relies on user interaction; precision depends on the resolution of the histogram and the granularity of mouse input [S1, unofficial].
*   **Boundary Definition**: The boundary of the criterion is displayed in the current dimensions, but switching attributes changes the distribution of voxels on the y-axis, requiring re-evaluation of selections [S1, unofficial].

## Subtopics

### Exploratory MDTF Design Process
A variant of the MDTF workflow that emphasizes dynamic attribute definition based on user domain knowledge [S1, unofficial]. It displays defined attributes in sorted histograms and iteratively refines the MDTF by selecting regions that define classification criteria [S1, unofficial]. Its main advantage is the uniform incorporation of any type of classifying information [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources. The iterative nature implies complexity depends on the number of iterations and the cost of histogram computation.
*   **Memory Complexity**: Not given in the sources. Requires storage for the 2D histogram and the list of selected ranges.
*   **Strengths**: Allows dynamic definition of attributes; sorted histograms maintain shape stability when changing the second attribute; enables uniform incorporation of any classifying information [S1, unofficial].
*   **Limitations**: Relies on user interaction for range selection; may require multiple iterations to converge on a satisfactory TF [S1, unofficial].
*   **When to use**: When the user has domain knowledge to guide attribute selection and needs flexibility in defining material criteria [S1, unofficial].
*   **When not to use**: When fully automatic generation is required without user interaction [S1, unofficial].

## Background
The concept of multi-dimensional transfer functions was initially proposed by Levoy in 1988, suggesting gradient magnitude as a valuable additional attribute apart from intensity [S1, unofficial]. Kindlmann and Durkin later proposed a TF model adding the first and second derivative along the gradient direction, yielding a 3D histogram that provides good separation of material boundaries [S1, unofficial]. The MDTF Design Workflow builds on these foundations by introducing an exploratory, histogram-based design process [S1, unofficial].

## Sources
[S1] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
