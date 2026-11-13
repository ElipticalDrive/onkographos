# Interaction and Editing Interfaces

Also called: Spreadsheet-like Interface

## Overview
Interaction and Editing Interfaces provide the user-facing mechanisms for defining, modifying, and exploring transfer functions (TFs) in volume rendering. They solve the problem of mapping high-dimensional data features to optical properties by offering intuitive visual metaphors, direct manipulation tools, or automated assistance. This family is distinguished from sibling methods like [Automatic and Data-Driven Generation](7609-automatic-and-data-driven-generation.md) by its emphasis on user-guided, interactive specification rather than fully autonomous algorithmic assignment, and from [Implementation and Evaluation](7608-implementation-and-evaluation.md) by its focus on the interaction paradigm itself rather than the performance metrics of the rendering pipeline.

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a scalar value $v = f(\mathbf{x})$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, unofficial][S2][S3, unofficial][S4, unofficial]. Advanced methods extend this domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, or texture metrics [S2, S5, unofficial].

**Output Range:**
The output is a set of optical properties, most commonly:
*   **Opacity ($\alpha$):** A value in $[0, 1]$ representing the probability of absorption or the inverse of transparency. $\alpha=0$ is fully transparent; $\alpha=1$ is fully opaque [S5, unofficial].
*   **Color ($C$):** A vector $(r, g, b)$ representing the emitted or reflected light color [S2][S4, unofficial].
*   **Weight:** In some emission-only models, a weight $w$ is assigned to modulate the contribution of samples during accumulation [S6, unofficial].

**Classification Types:**
1.  **Binary Classification:** Uses hard thresholds to include or exclude data [6229-binary-and-soft-classification.md]. Values below a threshold are assigned $\alpha=0$, and values above are assigned $\alpha=1$ (or a constant high value) [G].
2.  **Soft Classification:** Uses smooth transitions (e.g., piecewise linear ramps, Gaussian bumps, or power functions) to assign intermediate opacity values [S3, unofficial][S8, unofficial].

**Evaluation Order:**
The timing of classification relative to sampling interpolation defines two main approaches [S2][S5, unofficial]:
*   **Pre-classification:** The transfer function is applied to scalar values at the grid points (voxels) before sampling. Colors and opacities are interpolated during ray casting. Pre-classification produces blurred images when the image resolution is higher than the volume resolution, but prevents color bleeding if color interpolation is done with opacity-weighted colors [S2].
*   **Post-classification:** Scalar values are interpolated from the volume texture during ray casting, and the transfer function is applied to these continuous sample values. Post-classification produces images with defined edges but is prone to aliasing because high-frequency details may be missed [S2].

**Implementation Structures:**
Transfer functions are commonly implemented as piecewise linear functions defined by user-specified keypoints [S3, unofficial], lookup tables (LUTs) stored as 1D, 2D, or 3D textures for fast GPU sampling [S3, unofficial][S6, unofficial], or shader functions evaluated dynamically during ray casting [S5, unofficial].

## Subtopics

### [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md)
Automated Classification Widget Placement is an interaction technique that positions and sizes classification widgets in the transfer-function domain based on data probe feedback and reprojected voxel distributions. It solves the difficulty of manually creating effective higher-dimensional transfer functions by tying widget geometry to the statistical distribution of data at a specific probe location. Unlike manual placement methods, it reduces user effort by automatically determining widget bounds from local data variance.

### [2D Transfer Function Widget](5735-2d-transfer-function-widget.md)
The 2D Transfer Function Widget is an interactive visualization component that presents a 2D slice of the 3D transfer function (TF) domain to facilitate manual classification and editing of volume rendering parameters. It solves the problem of interpreting high-dimensional TF data by projecting spatial volume queries (via probes or clipping planes) into the TF domain, allowing users to observe how local volume characteristics (data value, gradient, and second derivative) map to classification properties. It distinguishes itself from sibling methods by explicitly visualizing the geometric connectivity of voxel corners in the TF domain, enabling users to distinguish homogeneous regions from boundaries based on the spatial configuration of projected points.

### [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md)
Sketch-based Transfer Function (TF) Editing allows users to define or modify classification functions by drawing strokes directly on the volume rendering view or on dedicated 1D/2D parameter spaces, rather than manipulating abstract widgets. Unlike [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) which optimizes widget positions, or [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) which uses geometric handles on a 2D plot, sketch-based methods prioritize direct manipulation and intuitive mapping between visual intent and parameter space. It solves the difficulty of manually tuning numerical parameters by enabling rapid, global definition of TF shapes and local refinement through free-form drawing.

### [Image-based TF Design](7807-image-based-tf-design.md)
Image-based TF Design is a user-guided classification method where users draw scribbles on image slices to indicate Regions of Interest (ROIs), which are then classified using Random Forest on Implicit Neural Representation (INR) features to generate probabilistic volume rendering. It solves the problem of manual transfer function tuning by automating the mapping from sparse user input to a full-volume classification. Unlike its sibling [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md), which typically maps 2D sketches directly to 1D transfer function curves, this method operates in the 3D voxel space using a machine learning classifier trained on spatial and feature context.

### [Parallel Coordinates based Transfer Function Design](8511-parallel-coordinates-based-transfer-function-design.md)
Parallel Coordinates based Transfer Function Design (PCbTF) is a method for designing multi-dimensional transfer functions by visualizing high-dimensional data parameters and spatial attributes as polylines in a parallel coordinates plot. It solves the complexity of high-dimensional classification by allowing users to identify correlations and select parameters through polyline patterns, or by automatically reducing dimensionality using Local Linear Embedding (LLE) for k-means classification. Unlike methods that operate solely in parameter space, PCbTF integrates spatial information as explicit parameters and leverages the structural patterns of polylines to facilitate dimension reduction and transfer function primitive design.

### [VTK.js Transfer Function Interface](13014-vtk-js-transfer-function-interface.md)
The VTK.js Transfer Function Interface is a JavaScript class that provides an interactive user interface for defining and editing volume rendering transfer functions (opacity and color) using control points, which are then applied to a VTK.js `vtkVolume` actor. It distinguishes itself from other interaction methods by providing a direct, DOM-based SVG widget for manual control point manipulation, specifically designed for integration with the VTK.js visualization pipeline, rather than relying on automated placement or sketch-based editing.

### Graphical Rule Specification
**Definition:** A GUI-based method for constructing illustration rules using semantic properties, visual attributes, and logical operators, represented as interactive expression trees [S2].
**Steps:**
1.  **Rule Creation:** Drag a connection between two semantic-property widgets to specify a rule based on the two properties [S2].
2.  **Operator Selection:** Use a design gallery rendering result images for different logical combinations (AND, OR, NOT) of the properties to select the desired operator [S2].
3.  **Rule Extension:** Draw a connection from a semantic-property widget to an expression-tree widget to include the semantic property as an additional operand [S2].
4.  **Rule Combination:** Draw a connection between two root nodes of different expression trees to combine them into a single rule [S2].
**Layout:** The UI contains three widget types: 1) Semantic data property widgets (top row) containing semantic-property and membership-function images; 2) Expression tree nodes (bottom part) where leaf nodes represent semantic properties and interior nodes represent rules/sub-rules with result images; 3) Visual attribute widget (left) [S2].

### Case-Based Reasoning Transfer Function Design
**Definition:** A transfer function design workflow where users select Sources of Interest (SOIs) directly in visualization space (2D cross-sections or 3D DVR) instead of indirect TF widgets [S3, unofficial].
**Steps:**
1.  User draws a line (ray) in 2D cross-sectional views or clicks a point in 3D DVR to select target SOIs [S3, unofficial].
2.  System retrieves matching SOIs via a two-stage Case-Based Reasoning (CBR) method [S3, unofficial].
3.  TFs are constructed by combining individual tent-shape components, each representing a retrieved SOI [S3, unofficial].
4.  User manually eliminates TF components for false-positive SOIs if retrieval errors occur [S3, unofficial].
**Pitfall:** False-negative SOIs (missing structures) require manual addition, but starting from scratch is avoided [S3, unofficial].

### Parallel Coordinates Style Interface
**Definition:** A parallel coordinates style interface that provides an overview of rendering options and transfer function settings, allowing the user to easily explore different parameters and backtrack the visualization history [S1, unofficial].
**Steps:** Tory et al. [89] propose this interface where the user can easily explore different parameters and backtrack the visualization history using the proposed interface [S1, unofficial].

### WYSIWYG Semantic Representation
**Definition:** An adaptation of the WYSIWYG (What You See Is What You Get) concept to volumetric exploration, implementing a high-level semantic representation with drawing tools [S4, unofficial].

## Background
The concept of spreadsheet-like interfaces for volume rendering was proposed by Jankun-Kelly and Ma [51], allowing users to explore a range of parameter combinations simultaneously and compare results side-by-side [S1, unofficial]. Tory et al. [89] introduced a parallel coordinates style interface to provide an overview of rendering options and transfer function settings, facilitating parameter exploration and history backtracking [S1, unofficial]. Guo, Mao, and Yuan (2011) adapted the WYSIWYG concept to volumetric exploration, implementing a high-level semantic representation with drawing tools [S4, unofficial]. More recently, CBR-based approaches have enabled users to select SOIs directly in visualization space, reducing the cognitive load of indirect TF widget manipulation [S3, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Semantics by analogy for illustrative volume visualization, https://doi.org/10.1016/j.cag.2011.10.006, fetched 2026-10-26
[S3] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
[S4] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
