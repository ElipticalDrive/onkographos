# Interaction and Editing Interfaces

Also called: Spreadsheet-like Interface

## Overview
This family provides interaction mechanisms for creating, modifying, and optimizing transfer functions through direct manipulation, graphical rule specification, or automated statistical feedback [S1, unofficial][S2][S3, S4, unofficial]. It solves the usability problem of manually defining high-dimensional classification parameters, which is often counter-intuitive for domain experts [S1, unofficial][S2]. Unlike sibling methods that define specific transfer function shapes or evaluation techniques, these interfaces focus on the mechanism of user input to generate or refine the classification function [S1, unofficial][S2][S3, S4, unofficial].

## Shared principles

## Subtopics

### [Position-dependent Transfer Function Customization](5287-position-dependent-transfer-function-customization.md)
Enables interactive, position-dependent modification of classification parameters by defining seed points and interpolating effects across the volume. It solves spatial heterogeneity where a single global transfer function fails. Choose when data exhibits varying properties (e.g., scanner artifacts) requiring localized overrides without discarding data.

### [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md)
Automatically positions and sizes classification widgets in the transfer-function domain based on probe feedback and reprojected voxel distributions. It solves the difficulty of manually configuring higher-dimensional transfer functions by tying widget geometry to data characteristics. Choose when reducing user effort for multi-dimensional transfer function design is prioritized.

### [2D Transfer Function Widget](5735-2d-transfer-function-widget.md)
Maps a spatial query in the volume to the transfer function domain, displaying local data characteristics alongside current classification settings. It solves the disconnect between spatial location and material appearance by providing immediate feedback on how volume features map to transfer function variables. Choose when users need to adjust classification parameters based on local geometric context.

### [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md)
Allows users to define or refine classification functions by drawing strokes directly on spatial volume views or dedicated sketching surfaces. It solves cognitive load and precision issues of traditional widget-based editing by leveraging spatial intuition. Choose when direct manipulation of rendered images or free-form sketching is preferred over structured geometric widgets.

### [Image-based TF Design](7807-image-based-tf-design.md)
A user-guided classification method where users draw scribbles on image slices to indicate Regions of Interest, which are then classified using Random Forest on INR features to generate probabilistic volume rendering. Choose when rapid, example-based region selection is needed without manual parameter tuning.

### Graphical Rule Specification
#### definition
- [S2] A GUI-based method for constructing illustration rules using semantic properties, visual attributes, and logical operators, represented as interactive expression trees.
#### step
- [S2] Rule Creation: Drag a connection between two semantic-property widgets to specify a rule based on the two properties.
- [S2] Operator Selection: Use a design gallery rendering result images for different logical combinations (AND, OR, NOT) of the properties to select the desired operator.
- [S2] Rule Extension: Draw a connection from a semantic-property widget to an expression-tree widget to include the semantic property as an additional operand.
- [S2] Rule Combination: Draw a connection between two root nodes of different expression trees to combine them into a single rule.
#### layout
- [S2] UI Widgets: Three types: 1) Semantic data property widgets (top row) containing semantic-property and membership-function images. 2) Expression tree nodes (bottom part) where leaf nodes represent semantic properties and interior nodes represent rules/sub-rules with result images. 3) Visual attribute widget (left).

### Case-Based Reasoning Transfer Function Design
#### definition
- [S3, unofficial] A transfer function design workflow where users select SOIs directly in visualization space (2D cross-sections or 3D DVR) instead of indirect TF widgets.
#### step
- [S3, unofficial] User draws a line (ray) in 2D cross-sectional views or clicks a point in 3D DVR to select target SOIs.
- [S3, unofficial] System retrieves matching SOIs via the two-stage CBR method.
- [S3, unofficial] TFs are constructed by combining individual tent-shape components, each representing a retrieved SOI.
- [S3, unofficial] User manually eliminates TF components for false-positive SOIs if retrieval errors occur.
#### pitfall
- [S3, unofficial] False-negative SOIs (missing structures) require manual addition, but starting from scratch is avoided.

### Multi-level Interface for Volume Illustration
#### definition
- [S4, unofficial] An interface design that abstracts complex rendering parameters into levels of expertise (Novice, Expert) and levels of representation (Schematic, Complex, Realistic) to manage the large number of parameters in volume illustration.
#### step
- [S4, unofficial] Novice Level: Present high-level parameters and motifs. Users focus on attentive focus and unconscious orientation of structures without being overwhelmed by data complexity.
- [S4, unofficial] Expert Level: Present transfer functions (low level) and rendering parameters. Experts can quickly focus on specific portions of data and relevant structures.
- [S4, unofficial] Representation Levels: Define motifs with feedback from professional medical illustrators to create schematic/simplistic, complex, or realistic representations.
#### parameter
- [S4, unofficial] Interface Levels: The interface is structured into High level (Motifs), Middle level (Parameters), and Low level (Transfer functions/Rendering parameters).

### Parallel Coordinates Style Interface
#### definition
- [S1, unofficial] A parallel coordinates style interface that provides an overview of rendering options and transfer function settings, allowing the user to easily explore different parameters and backtrack the visualization history.
#### step
- [S1, unofficial] Tory et al. propose a parallel coordinates style interface that provides an overview of rendering options and transfer function settings. The user can easily explore different parameters and backtrack the visualization history using the proposed interface.

## Background
The need for intuitive interaction interfaces in volume rendering stems from the complexity of defining transfer functions in high-dimensional spaces, which is often counter-intuitive for domain experts [S1, unofficial][S2]. Early work investigated image-centric interaction, where users select regions of interest directly in visualization space rather than abstract widget spaces, receiving positive feedback from clinicians [S3, unofficial]. Subsequent developments introduced case-based reasoning to automate transfer function generation by retrieving similar structures from knowledge databases [S3, unofficial], and graphical rule specifications to allow users to construct complex illustration rules using semantic properties and logical operators via interactive expression trees [S2]. Spreadsheet-like and parallel coordinates interfaces were proposed to enable side-by-side parameter exploration and visualization history backtracking [S1, unofficial]. Multi-level interfaces abstract parameters into expertise and representation levels to manage complexity [S4, unofficial]. These methods aim to bridge the gap between domain knowledge and visualization parameters, reducing the cognitive load required for effective volume visualization [S1, unofficial][S2][S3, S4, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Semantics by analogy for illustrative volume visualization, https://doi.org/10.1016/j.cag.2011.10.006, fetched 2026-10-26
[S3] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
[S4] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
