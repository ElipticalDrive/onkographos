# Contour Filtering

Also called: Isosurface Extraction via Contour, vtkContourFilter

## Overview
Contour Filtering is a method that generates isocontours (lines or surfaces) from scalar field data at specified scalar values [S1, S2, unofficial]. It differs from general-purpose isosurface extraction algorithms like Marching Cubes by being implemented as a processing filter in visualization pipelines, often coupled with normal generation and stripification to optimize rendering [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A dataset containing scalar values (e.g., `vtkImageData`, `vtkUnstructuredGrid`) [S2, unofficial].
- Output: A `vtkPolyData` mesh representing the isosurface [S2, unofficial].

**Data Structures:**
The method relies on the VTK pipeline architecture. The core data structure is `vtkPolyData`, which stores vertices, lines, and polygons. The input scalar field is accessed via the filter's input connection [S2, unofficial].

**Parameters:**
- **Isovalues:** A list of scalar values at which to extract contours. Typical usage involves setting specific values or generating a range [S1, S2, unofficial].
- **FeatureAngle:** An angle threshold (in degrees) used by the subsequent normal generation step to determine whether to smooth normals across edges. A typical value is 60.0 degrees [S2, unofficial].

**Steps:**
1. Instantiate the contour filter object.
2. Connect the input scalar data to the filter.
3. Set the desired contour value(s) using `SetValue(index, value)` or `GenerateValues` [S2, unofficial].
4. (Optional but typical) Connect the output to a `vtkPolyDataNormals` filter to compute vertex normals. Set the `FeatureAngle` parameter [S2, unofficial].
5. (Optional but typical) Connect the result to a `vtkStripper` to convert polygons into triangle strips for efficient rendering [S2, unofficial].
6. Map the resulting polydata to a renderer [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Create the contour filter
contour_filter = VTKContourFilter()

# 2. Set input data (assumed to be a scalar field dataset)
contour_filter.SetInputConnection(input_data_port)

# 3. Set contour value. Index 0 corresponds to the first isovalue.
# Example: Extract surface where scalar value is 51.
contour_filter.SetValue(0, 51)

# 4. Generate normals for shading
normals_filter = VTKPolyDataNormals()
normals_filter.SetInputConnection(contour_filter.GetOutputPort())
normals_filter.SetFeatureAngle(60.0)

# 5. Stripify for rendering efficiency
stripper = VTKStripper()
stripper.SetInputConnection(normals_filter.GetOutputPort())

# 6. The output is now a vtkPolyData ready for mapping
output_polydata = stripper.GetOutput()
```

**Edge Cases and Pitfalls:**
- If the input data lacks scalar arrays, the filter may fail or produce empty output.
- The `FeatureAngle` setting affects the visual smoothness of the resulting mesh; incorrect values may lead to faceted or overly smoothed appearances [S2, unofficial].

## Subtopics

### [Marching Cubes](30-isosurface-extraction.md)
The underlying geometric algorithm often used by contour filters to compute the surface geometry within each cell.

### [vtkPolyDataNormals](8434-visualization-and-rendering-techniques.md)
A common companion filter for computing surface normals.

### [vtkStripper](8434-visualization-and-rendering-techniques.md)
A filter for optimizing polygon data for rendering.

## Operating characteristics
- **Time Complexity:** Depends on the number of cells intersected by the isosurface and the complexity of the underlying cell geometry (e.g., hexahedral vs. tetrahedral).
- **Memory Complexity:** Proportional to the size of the output mesh (vertices and polygons).
- **Strengths:** Integrates easily into visualization pipelines; provides built-in tools for normal generation and stripification [S2, unofficial].
- **Limitations:** The resulting unstructured mesh may not be conforming if downstream filters (like shrink filters) are applied, as noted in specific AMR contexts [S1, unofficial].

## Background
The method is implemented as `vtkContourFilter` in the Visualization Toolkit (VTK). It is part of the standard VTK filter hierarchy for extracting isosurfaces from volumetric data [S2, unofficial]. It is widely used in medical imaging and scientific visualization for skin extraction and other surface modeling tasks [S2, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
[S2] researchxr/3DMR, https://github.com/researchxr/3DMR, fetched 2026-10-27 (unofficial)
