# Color Transfer Function Rescaling

Also called: Rescale Transfer Function, RescaleTransferFunctionToDataRange

## Overview
Color Transfer Function Rescaling adjusts the color and opacity transfer functions to match the range of values present in the visible data elements (cells or points) [S1]. This operation maps the full spectrum of available colors to the currently visible data, which can reveal patterns that were previously obscured by a mismatched transfer function range [S1]. It differs from static transfer function editing by dynamically adapting the mapping to the data's statistical extent rather than relying on fixed user-defined bounds.

## How it works
**Inputs and Outputs:**
*   **Input:** The current dataset (specifically the array used for scalar coloring) and the current display properties of the source.
*   **Output:** Updated color and opacity transfer functions where the domain (min/max values) is expanded or clamped to the data's range.

**Data Structures:**
The method operates on the internal representation of the transfer function within the visualization pipeline's display properties. The transfer function is defined by control points mapping data values to color and opacity. The rescaling operation modifies the domain bounds of these mappings.

**Steps:**
1.  Identify the active source in the visualization pipeline.
2.  Ensure the pipeline is updated so that the most recent data is available.
3.  Retrieve the display properties object associated with the source.
4.  Verify that the display is configured to use scalar coloring with an array that exists in the generated data. If this condition is not met, the operation fails [S1].
5.  Invoke the rescaling method on the display properties object. This resets the color and opacity maps to utilize the range of the array currently used for color mapping [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Get the active source in the pipeline
source = GetActiveSource()

# 2. Update the pipeline to ensure data is current
# [G] This step is necessary to guarantee that the data range reflects
# the latest computation or loading state.
source.UpdatePipeline()

# 3. Access the display properties for the source
display = GetDisplayProperties()

# 4. Check if scalar coloring is enabled with a valid array
# [G] The implementation must verify that display.ColorArrayName
# is not empty and corresponds to an array present in source.GetOutput().
# If not, raise an error or return early.

# 5. Rescale the transfer functions to the data range
# This expands the transfer function domain to include the min/max
# values of the selected array in the visible data.
display.RescaleTransferFunctionToDataRange()
```

**Parameters:**
*   **Scope:** The operation can be scoped to "visible elements" (cells or points in the current view) or "all timesteps" depending on the specific implementation variant or user selection [S1]. The standard `RescaleTransferFunctionToDataRange` typically targets visible elements [S1].

**Edge Cases and Pitfalls:**
*   **Missing Array:** The method requires that the display has been explicitly set to use scalar coloring using an array that is available in the data generated. If the display is set to use a non-existent array or a different coloring mode (e.g., vector coloring), the method will generate errors [S1].
*   **Performance Cost:** Rescaling across all timesteps requires reading data for all timesteps, which can be computationally costly [S1].

## Subtopics

### Automatic Rescale Range Mode
A property in the Color Map Editor that controls the automatic update behavior of the color/opacity map range relative to the data range [S1]. It determines when and how the range adapts without manual intervention [S1].
*   **Never:** The data range is never updated automatically [S1].
*   **Grow and update on Apply:** The range grows to include the current data range every time 'Apply' is hit on the Properties panel. It does not update when the timestep changes [S1].
*   **Grow and update every timestep:** The range grows to include the current data range on 'Apply' and when the timestep changes [S1].
*   **Clamp and update every timestep:** The range is clamped to the exact data range on 'Apply' and when the timestep changes [S1].
*   **Behavior Note:** 'Grow' indicates that the color/opacity map range will only increase, never shrink, to include the current data range. 'Clamp' forces the range to match the current data range exactly [S1]. The initial value for this mode is controlled by the General setting `Transfer Function Reset Mode` in the Settings dialog [S1].

## Operating characteristics
*   **Time Complexity:** Rescaling visible elements is generally efficient. However, rescaling across all timesteps is costly as it requires reading data for all timesteps [S1].
*   **Memory Complexity:** Depends on the storage of the transfer function control points and the data array being analyzed. No significant additional memory is required beyond the data itself.
*   **Strengths:** Automatically reveals data patterns by ensuring the full color map is utilized for the visible data range [S1]. Reduces the need for manual tuning of transfer function bounds.
*   **Limitations:** Can be computationally expensive for large datasets with many timesteps if rescaling across all timesteps is requested [S1]. Requires careful setup of the display properties to avoid errors [S1].
*   **When to use:** When the data range is unknown or changes dynamically, and the user wants to ensure the full color spectrum is applied to the visible data.
*   **When not to use:** When specific, fixed data ranges are required for consistent comparison across multiple views or timesteps, or when performance is critical and the dataset is large with many timesteps.

## Background
This method is part of the ParaView visualization software's color mapping and transfer function management system [S1]. It is designed to assist users in visualizing scalar data by automatically adjusting the mapping between data values and visual properties (color and opacity) [S1]. The concept of automatic range adjustment is integrated into the Color Map Editor to provide convenience during interactive exploration [S1].

## Sources
[S1] 2. Color maps and transfer functions — ParaView ..., https://cep.tacc.utexas.edu/user-guide/ReferenceManual/colorMapping.html, fetched 2026-10-29
