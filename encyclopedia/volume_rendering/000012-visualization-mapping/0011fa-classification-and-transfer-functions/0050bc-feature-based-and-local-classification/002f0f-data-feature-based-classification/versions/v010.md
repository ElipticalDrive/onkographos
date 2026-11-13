# Data Feature-Based Classification

Also called: Multi-Dimensional Transfer Function (MDTF) design, Feature-based classification

## Overview
Data Feature-Based Classification constructs transfer functions by mapping multi-dimensional feature vectors—derived from intensity, geometry, color, or topology—to optical properties, solving the ambiguity of scalar-only classification where distinct materials share similar intensity values. It differs from its siblings by explicitly leveraging diverse feature domains (e.g., curvature, shape skeletons, material graphs, or color ratios) rather than relying solely on local gradient magnitude or simple intensity thresholds, allowing for the separation of structures that are indistinguishable by scalar value alone.

## Shared principles
All methods in this family evaluate a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is a feature vector $\mathbf{f}(\mathbf{x})$ associated with position $\mathbf{x}$. While basic methods use scalar intensity $v = f(\mathbf{x})$, this family extends the domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, shape descriptors, color components, or topological graph attributes [S1, unofficial][S2][S3][S4].

**Output Range:**
The output is a set of optical properties, most commonly:
*   **Opacity ($\alpha$):** A value in $[0, 1]$ representing the probability of absorption or the inverse of transparency. $\alpha=0$ is fully transparent; $\alpha=1$ is fully opaque [S5, unofficial].
*   **Color ($C$):** A vector $(r, g, b)$ representing the emitted or reflected light color [S2][S4].
*   **Weight:** In some emission-only models, a weight $w$ is assigned to modulate the contribution of samples during accumulation [S6, unofficial].

**Classification Types:**
1.  **Binary Classification:** Uses hard thresholds to include or exclude data [6229-binary-and-soft-classification.md]. Values below a threshold are assigned $\alpha=0$, and values above are assigned $\alpha=1$ (or a constant high value) [G].
2.  **Soft Classification:** Uses smooth transitions (e.g., piecewise linear ramps, Gaussian bumps, or power functions) to assign intermediate opacity values [S3][S8, unofficial].

**Evaluation Order:**
The timing of classification relative to sampling interpolation defines two main approaches [S2][S5, unofficial]:
*   **Pre-classification:** The transfer function is applied to scalar values at the grid points (voxels) before sampling. Colors and opacities are interpolated during ray casting. Pre-classification produces blurred images when the image resolution is higher than the volume resolution, but prevents color bleeding if color interpolation is done with opacity-weighted colors [S2].
*   **Post-classification:** Scalar values are interpolated from the volume texture during ray casting, and the transfer function is applied to these continuous sample values. Post-classification produces images with defined edges but is prone to aliasing because high-frequency details may be missed [S2].

**Implementation Structures:**
Transfer functions are commonly implemented as piecewise linear functions defined by user-specified keypoints [S3], lookup tables (LUTs) stored as 1D, 2D, or 3D textures for fast GPU sampling [S3][S6, unofficial], or shader functions evaluated dynamically during ray casting [S5, unofficial].

## Subtopics

### [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md)
Maps scalar data values and local surface curvature to optical properties to distinguish features based on geometric shape rather than intensity alone. It solves the problem of distinguishing between structures with similar intensities but different curvatures, such as the borders of small versus large features, and is primarily used for non-photorealistic rendering to highlight contours, ridges, and valleys. Unlike gradient-based methods that detect edges, cbTF emphasizes the "bumpiness" or deviation from a straight aspect of the underlying isosurfaces.

### [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md)
Assigns optical properties to volumetric features based on their 3D shape rather than solely on intensity or gradient magnitude. It solves the problem of distinguishing features with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) by using a shape descriptor derived from curve-skeletons. Unlike per-voxel classification methods that rely on local filters, this method assigns a uniform shape descriptor to each entire volumetric feature (segment), allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels.

### [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md)
Classifies color volume data by assigning opacity based on a high-dimensional classification function that utilizes RGB values, neighbor colors, and spatial position, thereby avoiding the difficulty of specifying three-dimensional transfer functions for RGB data. Unlike methods using Independent Component Analysis (ICA) that require users to work in a derived data space, this method allows users to work directly with original RGB data. It builds on [Data Feature-Based Classification](5934-data-feature-based-classification.md) by extending the input domain from scalar values to multi-dimensional feature vectors derived from color and geometry.

### [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md)
An automated method for constructing transfer functions by discovering the topological structure of materials within a volume. It solves the problem of manual transfer function creation by automatically identifying material interiors and their spatial interconnectivity, producing a graph where nodes represent materials and edges represent interfaces. Unlike sibling methods that focus on local feature extraction or high-dimensional clustering, this approach explicitly models the adjacency relationships between materials to enable automatic background removal and fine control over opacity based on depth.

### [Feature Graph](8595-feature-graph.md)
A hierarchical data structure that represents classified voxel models as a directed acyclic graph (DAG), where nodes correspond to non-empty clusters of voxels (features) sharing specific label values, and arcs represent the inclusion relationship of these voxel sets. It solves the problem of efficiently managing and rendering multiresolution volume data by organizing voxels into disjoint sets at leaf nodes, allowing ancestor features to be reconstructed via traversal, thereby avoiding redundancy. Unlike flat transfer functions, it explicitly models the topological inclusion of material regions, enabling focused visualization and efficient out-of-core traversal.

### [Attribute Ranking Heuristic](11691-attribute-ranking-heuristic.md)
A greedy method for selecting relevant volumetric attributes to construct voxel signatures for multidimensional transfer function (MTF) classification. It solves the problem of high-dimensional data complexity by ranking candidate attributes (e.g., intensity, gradient, curvature) based on their information content and redundancy, presenting the top $n$ candidates to the user for signature construction. It differs from sibling methods like [Attribute Selection Heuristic for Multidimensional Transfer Functions](11717-attribute-selection-heuristic-for-multidimensional-transfer-.md) by using a simpler pairwise correlation model for the initial selection, whereas its child extends this to multiple correlation with all previously selected attributes.

### [Machine Learning and Feature Extraction](12097-machine-learning-and-feature-extraction.md)
Applies machine learning algorithms and feature extraction techniques to automatically or semi-automatically determine optical properties (opacity and color) for volume rendering. It solves the problem of manually designing transfer functions for complex, high-dimensional data by learning classification boundaries from user-provided training samples. It differs from heuristic or graph-based siblings by relying on statistical learning models (e.g., SVMs, AdaBoost, K-means) rather than geometric or topological constraints.

### [Similarity-based Classification](12242-similarity-based-classification.md)
Determines optical properties by measuring the distance or correlation between a sample point's feature vector and a set of reference exemplars or annotations, rather than relying solely on raw intensity or simple gradient magnitudes. This family is distinct from its siblings because it operates in a high-dimensional feature space (range-space) to capture semantic or structural similarity, enabling the segmentation of complex, multidimensional, or spatiotemporal data where traditional scalar thresholds fail. Unlike [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) which focuses on local geometric shape, or [Machine Learning and Feature Extraction](12097-machine-learning-and-feature-extraction.md) which may use supervised labels, similarity-based methods often rely on user-provided exemplars or self-supervised proximity tasks to define classification boundaries.

### [Local Histogram-based Classification Probability](14137-local-histogram-based-classification-probability.md)
Separates overlapping tissues in volume rendering by using local histogram range weights as a second attribute alongside intensity. It allows a 2D Transfer Function to distinguish materials with identical intensity values but different neighborhood contexts, such as vessels and spongy bone in CT angiographies. Unlike standard intensity-based classification, this method computes a signed classification certainty $P$ derived from the narrowness of local intensity distributions, enabling the separation of materials that are indistinguishable by scalar value alone.

### Domain Knowledge Attributes for Classification
Attributes that capture domain knowledge to complement gradient-based measures. The paper specifically employs 'range weight' and 'local variance' as examples of these attributes to enable additional visualization possibilities [S1, unofficial].

### Feature Visualization
Calculate 3-dimensional PCA components using teacher's feature map of first frame of training view images, then visualize features by components as RGB, normalized with min-max values with outlier removal [S2].

### Color-based Feature Detectors
Feature detectors that exploit color separation in RGB image data to highlight specific tissues by computing normalized ratios of individual color channels [S3].
*   $c_R = R / (R + G + B)$, where $c_R$ is the red feature detector, and $R, G, B$ are the red, green, and blue channel values respectively [S3].
*   $c_G = G / (R + G + B)$, where $c_G$ is the green feature detector [S3].
*   $c_B = B / (R + G + B)$, where $c_B$ is the blue feature detector [S3].
*   $c_{BR} = 0.5 * c_R + 1.0 * c_G$, a brown feature detector created by a weighted linear combination of red and green feature detectors [S3].
*   Brown color composition: Brown is composed of 0.5*R + 1.0*G + 0.0*B [S3].

## Background
The concept of using attributes beyond simple intensity to define transfer functions was formalized in the context of Multi-Dimensional Transfer Function (MDTF) design. The foundational work by [S1, unofficial] introduced a framework where classifying attributes, such as range weight and local variance, are used to complement gradient-based measures, allowing users to dynamically define attributes and select classification criteria via sorted histograms. This approach enables the uniform incorporation of any type of classifying information.

Color-based feature detection was further developed for high-resolution color cryo-imaging data, where normalized ratios of RGB channels (e.g., $c_R, c_G, c_B$) and their linear combinations (e.g., for brown tissues) were used to highlight specific biological structures [S3].

More recently, the application of machine learning and feature extraction has expanded the scope of data feature-based classification. Techniques such as PCA on teacher feature maps from models like LSeg and DINO have been used to visualize high-dimensional feature spaces for editing and analysis [S2]. These developments reflect a shift from manual, heuristic-based feature selection to automated, data-driven feature extraction and classification.

## Sources
[S1] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
[S2] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S3] Enhanced volume rendering techniques for high-resolution color cryo-imaging data, https://doi.org/10.1117/12.813756, fetched 2026-11-08
