# Interaction and Editing Interfaces

Also called: Spreadsheet-like Interface

## Overview
Interaction and Editing Interfaces provide the user-facing mechanisms for defining, modifying, and exploring transfer functions (TFs) in volume rendering. They solve the problem of mapping high-dimensional data features to optical properties (opacity and color) by offering intuitive controls that abstract the complexity of mathematical function definition. This family is distinct from the mathematical definitions of TFs themselves (e.g., [1D Transfer Function](11719-1d-transfer-function.md)) by focusing on the *interface* and *interaction paradigm* rather than the evaluation kernel. It sets itself apart from [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) by emphasizing direct user manipulation of parameters or spatial features, whereas automated placement focuses on UI layout optimization.

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

### [2D Transfer Function Widget](5735-2d-transfer-function-widget.md)
[2D Transfer Function Widget](5735-2d-transfer-function-widget.md) is an interactive visualization component that maps a spatial query in a volume to a 2D slice of the multi-dimensional transfer function (TF) domain, allowing users to inspect local data characteristics and adjust classification parameters. It solves the ambiguity of how specific voxel structures map to TF variables by displaying the queried point and its eight voxel corners as points in the TF space, connected by edges reflecting spatial connectivity. Unlike standard point probes, it visualizes the local gradient and curvature structure, enabling users to distinguish between homogeneous regions and boundaries based on the geometric configuration of the projected points.

### [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md)
[Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md) allows users to define volume rendering appearance by drawing strokes directly on spatial views (volume slices or rendered images) or via dedicated sketching interfaces, which are then automatically converted into mathematical transfer functions. It solves the difficulty of manually adjusting numerical parameters in traditional interfaces by leveraging the user's intuitive understanding of spatial features. It sets itself apart from [2D Transfer Function Widget](5735-2d-transfer-function-widget.md) by mapping interaction to the spatial domain (pixels/voxels) rather than the abstract parameter domain, and from [Image-based TF Design](7807-image-based-tf-design.md) by explicitly generating the underlying classification function from the sketch geometry rather than just using the image as a reference.

### [Image-based TF Design](7807-image-based-tf-design.md)
[Image-based TF Design](7807-image-based-tf-design.md) is a user-guided classification method where users draw scribbles on image slices to indicate Regions of Interest (ROIs), which are then classified using Random Forest on Implicit Neural Representation (INR) features to generate probabilistic volume rendering. It solves the problem of manual transfer function tuning by automating the mapping from sparse user input to a full-volume classification. Unlike its sibling [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md), which typically maps 2D sketches directly to 1D transfer function curves, this method operates in the 3D voxel space using a machine learning classifier trained on spatial and feature context.

### [Parallel Coordinates based Transfer Function Design](8511-parallel-coordinates-based-transfer-function-design.md)
[Parallel Coordinates based Transfer Function Design](8511-parallel-coordinates-based-transfer-function-design.md) is a method for designing multi-dimensional transfer functions by visualizing high-dimensional data parameters and spatial attributes as polylines in a parallel coordinates plot. It solves the complexity of high-dimensional classification by allowing users to identify correlations and select parameters through polyline patterns, or by automatically reducing dimensionality using Local Linear Embedding (LLE) for k-means classification. Unlike methods that operate solely in parameter space, PCbTF integrates spatial information as explicit parameters and leverages the structural patterns of polylines to facilitate dimension reduction and transfer function primitive design.

### Graphical Rule Specification
A GUI-based method for constructing illustration rules using semantic properties, visual attributes, and logical operators, represented as interactive expression trees [S2]. Rule creation involves dragging a connection between two semantic-property widgets to specify a rule based on the two properties [S2]. Operator selection uses a design gallery rendering result images for different logical combinations (AND, OR, NOT) of the properties to select the desired operator [S2]. Rule extension is performed by drawing a connection from a semantic-property widget to an expression-tree widget to include the semantic property as an additional operand [S2]. Rule combination is achieved by drawing a connection between two root nodes of different expression trees to combine them into a single rule [S2]. The UI consists of three widget types: semantic data property widgets (top row) containing semantic-property and membership-function images; expression tree nodes (bottom part) where leaf nodes represent semantic properties and interior nodes represent rules/sub-rules with result images; and a visual attribute widget (left) [S2].

### Case-Based Reasoning Transfer Function Design
A transfer function design workflow where users select SOIs directly in visualization space (2D cross-sections or 3D DVR) instead of indirect TF widgets [S3, unofficial]. The user draws a line (ray) in 2D cross-sectional views or clicks a point in 3D DVR to select target SOIs [S3, unofficial]. The system retrieves matching SOIs via a two-stage CBR method [S3, unofficial]. TFs are constructed by combining individual tent-shape components, each representing a retrieved SOI [S3, unofficial]. If retrieval errors occur, the user manually eliminates TF components for false-positive SOIs [S3, unofficial]. A pitfall is that false-negative SOIs (missing structures) require manual addition, although starting from scratch is avoided [S3, unofficial].

### Multi-level Interface for Volume Illustration
An interface design that abstracts complex rendering parameters into levels of expertise (Novice, Expert) and levels of representation (Schematic, Complex, Realistic) to manage the large number of parameters in volume illustration [S4, unofficial]. The Novice Level presents high-level parameters and motifs, allowing users to focus on attentive focus and unconscious orientation of structures without being overwhelmed by data complexity [S4, unofficial]. The Expert Level presents transfer functions (low level) and rendering parameters, allowing experts to quickly focus on specific portions of data and relevant structures [S4, unofficial]. Representation Levels define motifs with feedback from professional medical illustrators to create schematic/simplistic, complex, or realistic representations [S4, unofficial]. The interface is structured into High level (Motifs), Middle level (Parameters), and Low level (Transfer functions/Rendering parameters) [S4, unofficial].

### Parallel Coordinates Style Interface
A parallel coordinates style interface that provides an overview of rendering options and transfer function settings, allowing the user to easily explore different parameters and backtrack the visualization history [S1, unofficial]. Tory et al. [89] propose this interface where the user can easily explore different parameters and backtrack the visualization history using the proposed interface [S1, unofficial].

### WYSIWYG Semantic Representation
An adaptation of the WYSIWYG (What You See Is What You Get) concept to volumetric exploration, implementing a high-level semantic representation with drawing tools [S5, unofficial]. Guo, Mao and Yuan (2011) adapted the concept WYSIWYG to volumetric exploration, implementing a high-level semantic representation that provides drawing tools to the user [S5, unofficial].

## Background
The concept of a "spreadsheet-like interface" for exploring parameter combinations side-by-side was proposed by Jankun-Kelly and Ma [51] [S1, unofficial]. Tory et al. [89] proposed a parallel coordinates style interface for overviewing rendering options and transfer function settings [S1, unofficial]. Tzeng and Ma [92] proposed using ISODATA clustering on a subset of data voxels to classify the volume, displaying classified features in a cluster-space user interface to allow TF design without knowing the transfer function space [S1, unofficial].

## Sources

[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Semantics by analogy for illustrative volume visualization, https://doi.org/10.1016/j.cag.2011.10.006, fetched 2026-10-26
[S3] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
[S4] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S5] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
