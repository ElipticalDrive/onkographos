# Interaction and Editing Interfaces

Also called: Spreadsheet-like Interface

## Overview
Interaction and Editing Interfaces provide the user-facing mechanisms for defining, modifying, and exploring transfer functions (TFs) in volume rendering. They solve the problem of mapping abstract data properties to visual attributes (color, opacity) by offering intuitive controls that bridge the gap between raw data and visual perception. This family is distinguished from its siblings by focusing on the *interaction paradigm* (e.g., sketching, widgets, spreadsheets) rather than the mathematical definition of the transfer function itself. Unlike [Automatic and Data-Driven Generation](7609-automatic-and-data-driven-generation.md) which automates the process, these interfaces require explicit or semi-explicit user input.

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a scalar value $v = f(\mathbf{x})$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, unofficial][S2][S3, unofficial][S4, unofficial]. Advanced methods extend this domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, or texture metrics [S2][S5, unofficial].

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

### [Position-dependent Transfer Function Customization](5287-position-dependent-transfer-function-customization.md)
A method for the interactive, position-dependent customization of arbitrary transfer function classification parameters in volume rendering, allowing local adaptations of visualization parameters based on voxel position without preprocessing. It solves the problem of spatial inhomogeneities in volume data (e.g., scanner artifacts) or the need to highlight specific regions of interest (ROI) by overriding or blending base transfer function parameters locally. Unlike [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) which focuses on widget placement, or [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md) which operates in 2D TF space, this method operates directly in the 3D spatial domain using geometric primitives to define a "customization volume" that modulates the classification step. It differs from simple clipping by lowering opacity outside an ROI while modifying colors inside, preserving spatial context.

### [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md)
Automated Classification Widget Placement is an interaction technique for multi-dimensional transfer functions that automatically positions and sizes classification widgets based on data probe feedback and the statistical distribution of reprojected voxels. It solves the difficulty of manually configuring effective higher-dimensional transfer functions by tying widget geometry directly to the local data distribution queried by the user. Unlike manual editing interfaces, it reduces the cognitive load of widget placement by deriving location and scale from probe state and voxel variance.

### [2D Transfer Function Widget](5735-2d-transfer-function-widget.md)
The 2D Transfer Function Widget is an interactive visualization component that presents a 2D slice of the 3D transfer function (TF) domain, allowing users to manually set and alter the transfer function while receiving immediate feedback from volume queries. It distinguishes itself from standard piecewise linear editors by integrating "re-projected voxel" visualization, which maps spatial voxel structures directly into the TF domain to help users interpret local volume characteristics like gradient and curvature. This method bridges the gap between data space and TF space, enabling users to see how specific spatial features (e.g., boundaries vs. homogeneous regions) map to TF variables.

### [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md)
Sketch-based Transfer Function Editing allows users to define volume rendering appearance by directly drawing strokes on spatial domain views (volume slices or rendered images) or via direct manipulation interfaces, which are then interpreted to generate or refine transfer functions (TFs). It solves the disconnect between abstract 1D/2D TF widgets and the visual features in the volume, enabling intuitive assignment of color and transparency based on spatial context. It sets itself apart from [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) by operating on spatial sketches rather than abstract TF graphs, and from [Image-based TF Design](7807-image-based-tf-design.md) by often employing neural networks or explicit stroke-to-TF mapping algorithms to interpret the sketches.

### [Image-based TF Design](7807-image-based-tf-design.md)
Image-based TF Design is a user-guided classification method where users draw scribbles on image slices to indicate Regions of Interest (ROIs), which are then classified using Random Forest on Implicit Neural Representation (INR) features to generate probabilistic volume rendering. It solves the problem of manual transfer function tuning by allowing sparse user supervision to automatically classify volumetric data. Unlike its parent [Interaction and Editing Interfaces](6592-interaction-and-editing-interfaces.md), which provides general tools for editing, this method specifically automates the classification step via machine learning. It differs from [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md) by operating on volumetric voxels via slice projections rather than editing 2D transfer function curves directly, and from [Position-dependent Transfer Function Customization](5287-position-dependent-transfer-function-customization.md) by deriving classification from image content features rather than spatial coordinates alone.

### Graphical Rule Specification
A GUI-based method for constructing illustration rules using semantic properties, visual attributes, and logical operators, represented as interactive expression trees [S2]. Rule creation involves dragging a connection between two semantic-property widgets to specify a rule based on the two properties [S2]. Operator selection uses a design gallery rendering result images for different logical combinations (AND, OR, NOT) of the properties to select the desired operator [S2]. Rule extension is performed by drawing a connection from a semantic-property widget to an expression-tree widget to include the semantic property as an additional operand [S2]. Rule combination is achieved by drawing a connection between two root nodes of different expression trees to combine them into a single rule [S2]. The UI consists of three widget types: semantic data property widgets (top row) containing semantic-property and membership-function images; expression tree nodes (bottom part) where leaf nodes represent semantic properties and interior nodes represent rules/sub-rules with result images; and a visual attribute widget (left) [S2].

### Case-Based Reasoning Transfer Function Design
A transfer function design workflow where users select Sources of Interest (SOIs) directly in visualization space (2D cross-sections or 3D DVR) instead of indirect TF widgets [S3, unofficial]. The user draws a line (ray) in 2D cross-sectional views or clicks a point in 3D DVR to select target SOIs [S3, unofficial]. The system retrieves matching SOIs via a two-stage Case-Based Reasoning (CBR) method [S3, unofficial]. TFs are constructed by combining individual tent-shape components, each representing a retrieved SOI [S3, unofficial]. If false-positive SOIs are retrieved, the user manually eliminates the corresponding TF components [S3, unofficial]. A pitfall is that false-negative SOIs (missing structures) require manual addition, although starting from scratch is avoided [S3, unofficial].

### Multi-level Interface for Volume Illustration
An interface design that abstracts complex rendering parameters into levels of expertise (Novice, Expert) and levels of representation (Schematic, Complex, Realistic) to manage the large number of parameters in volume illustration [S4, unofficial]. At the Novice Level, high-level parameters and motifs are presented, allowing users to focus on attentive focus and unconscious orientation of structures without being overwhelmed by data complexity [S4, unofficial]. At the Expert Level, transfer functions (low level) and rendering parameters are presented, allowing experts to quickly focus on specific portions of data and relevant structures [S4, unofficial]. Representation Levels define motifs with feedback from professional medical illustrators to create schematic/simplistic, complex, or realistic representations [S4, unofficial]. The interface is structured into High level (Motifs), Middle level (Parameters), and Low level (Transfer functions/Rendering parameters) [S4, unofficial].

### Parallel Coordinates Style Interface
A parallel coordinates style interface that provides an overview of rendering options and transfer function settings, allowing the user to easily explore different parameters and backtrack the visualization history [S1, unofficial]. Tory et al. [89] propose this interface where the user can easily explore different parameters and backtrack the visualization history using the proposed interface [S1, unofficial].

### WYSIWYG Semantic Representation
An adaptation of the WYSIWYG (What You See Is What You Get) concept to volumetric exploration, implementing a high-level semantic representation with drawing tools [S5, unofficial]. Guo, Mao and Yuan (2011) adapted this concept to volumetric exploration, implementing a high-level semantic representation that provides drawing tools to the user [S5, unofficial].

## Background
The concept of spreadsheet-like interfaces for transfer function design was proposed by Jankun-Kelly and Ma [51], allowing users to explore a range of parameter combinations side-by-side [S1, unofficial]. Parallel coordinates interfaces were proposed by Tory et al. [89] to provide an overview of rendering options [S1, unofficial]. Sketch-based editing was pioneered by Ropinski et al. (2008) using contour-based specification [S5, unofficial]. The WYSIWYG concept was adapted to volume exploration by Guo, Mao and Yuan (2011) [S5, unofficial]. Graphical rule specification was developed to facilitate direct and flexible construction of illustration rules using semantic properties [S2]. Case-Based Reasoning for TF design was investigated to allow clinicians to select SOIs directly in visualization space [S3, unofficial]. Multi-level interfaces were designed to address the overwhelm novices feel with complex data parameters [S4, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Semantics by analogy for illustrative volume visualization, https://doi.org/10.1016/j.cag.2011.10.006, fetched 2026-10-26
[S3] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
[S4] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S5] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
