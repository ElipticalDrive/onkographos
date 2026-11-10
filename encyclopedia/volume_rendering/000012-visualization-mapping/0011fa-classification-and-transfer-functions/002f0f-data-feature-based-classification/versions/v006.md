# Data Feature-Based Classification

Also called: High-dimensional classification, Multidimensional transfer function classification, Feature-based classification

## Overview
Data Feature-Based Classification extends the classification domain beyond the single scalar intensity value $v$ to include derived features such as gradients, curvature, shape descriptors, or spatial context [S1, S2][S3, unofficial]. It solves the ambiguity inherent in 1D classification where distinct materials or structures share similar intensity values but differ in geometric or contextual properties [S1][S3, unofficial]. Unlike [1D Transfer Function](11719-1d-transfer-function.md) which maps a single scalar to opacity, this family maps a feature vector $\mathbf{f}$ to optical properties, allowing for the separation of features that are indistinguishable by intensity alone [S1][S3, unofficial].

## Shared principles
All methods in this family evaluate a classification function $C(\mathbf{x})$ where the input is a feature vector $\mathbf{f}(\mathbf{x})$ derived from the volume data at position $\mathbf{x}$, rather than just the scalar value $f(\mathbf{x})$ [S1, S2][S3, unofficial]. The output remains the standard optical properties: opacity $\alpha \in [0, 1]$ and color $\mathbf{C}$ [G]. The feature vector $\mathbf{f}$ may include:
*   **Geometric features:** Gradient magnitude, curvature, or shape descriptors derived from local or global structure [S1][S3, unofficial].
*   **Contextual features:** Neighbor colors, spatial position, or texture metrics [S1, S2].
*   **Semantic features:** Labels or signatures derived from machine learning models or domain-specific attributes [S1][S3, unofficial].

The classification can be performed via direct mapping (e.g., 2D/3D lookup tables), statistical models (e.g., SVMs, neural networks), or graph-based deduction [S1, S2][S3, unofficial].

## Subtopics

### [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md)
Uses the second derivative (curvature) of the scalar field, in addition to intensity, to distinguish structural transitions. It detects the *shape* of edges (convex vs. concave) rather than just their presence, allowing separation of features with identical intensity gradients but different sizes. Best for non-photorealistic rendering or highlighting ridges/valleys.

### [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md)
Assigns optical properties based on 3D shape descriptors (tubiness, surfaceness, blobbiness) derived from curve-skeletons of pre-segmented regions. It classifies complete volumetric features rather than per-voxel, solving the problem of distinguishing features with overlapping intensity ranges (e.g., vessels vs. bones). Best for anatomical structures with distinct shapes.

### [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md)
Classifies color volumes using a 24-dimensional feature vector comprising RGB color, 6 immediate neighbor colors, and 3D position. It allows users to paint on 2D slices to train the classifier, handling color data similarly to scalar data without requiring users to work in derived spaces like ICA. Best for color volume data where spatial context and neighbor relationships are critical.

### [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md)
Automatically constructs a material graph by discovering the topological structure of materials using LH histogram edge sampling and HDBSCAN clustering. It deduces material interiors and interfaces, producing a graph of materials and occlusion values for automatic background removal and interior visualization. Best for automated analysis of complex multi-material volumes.

### [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md)
Enhances object distinction by assigning object membership IDs using a hybrid of Self-Organizing Maps (SOM) and K-means clustering, incorporating a Diagonal Busyness Factor (DF) for texture. It avoids local optima via SOM initialization and uses spatial texture/color to create binary transfer functions. Best for color/texture-rich volumes where standard K-means fails due to local optima.

### [Feature Graph](8595-feature-graph.md)
Represents classified voxel data as a directed acyclic graph (DAG) based on boolean classification criteria, organizing features into an inclusion hierarchy. It supports multiresolution rendering and compact storage by storing Sets of Voxels (SOVs) only at leaf nodes. Best for managing complex, multi-label datasets with hierarchical relationships.

### [Attribute Ranking Heuristic](11691-attribute-ranking-heuristic.md)
A greedy method for selecting relevant attributes for multidimensional transfer functions by ranking them based on entropy and redundancy (multiple correlation). It presents the top $n$ attributes to the user for manual signature construction, solving high-dimensional data complexity. Best for exploratory analysis of high-dimensional datasets.

### Classifier Reuse
Saving a trained neural network or SVM configuration (under 30 KB) to apply to new datasets with similar characteristics, reducing classification time [S1]. Position information should be excluded if reusability is intended, as it is dataset-specific [S1]. Misclassification may occur if the new data does not share similar characteristics [S1].

### Cosine Similarity Map Computation
Computes a similarity volume $S_L$ by comparing annotated feature vectors against the full feature volume using cosine similarity, enabling interactive feedback [S2]. The similarity is scaled by a proximity parameter $\tilde{p}$ to select spatially local structures: $P(x) = \max_{a \in A_C} e^{-10\tilde{p}|x-a|}$ [S2]. Computation takes a few milliseconds [S2].

### DINO Features for SVM/RF Classification
Uses DINO self-supervised vision transformer features for classification. On the CT-ORG dataset, SVM IoU was 0.24 (liver), 0.36 (bones), mean 0.09, with fit time 407s and inference time 2077s [S2].

### Domain Knowledge Attributes for Classification
Attributes that capture domain knowledge to complement gradient-based measures, such as 'range weight' and 'local variance', enabling additional visualization possibilities in multidimensional transfer functions [S3, unofficial].

## Background
The concept of using features beyond intensity to improve classification in volume rendering emerged to address the limitations of 1D transfer functions [S1][S3, unofficial]. Early work by [Tzeng et al.](6327-high-dimensional-classification-with-color-data.md) introduced high-dimensional classification using SVMs and neural networks with color and position data [S1]. Later developments incorporated geometric features like curvature [S1] and shape descriptors [S3, unofficial]. Recent advancements leverage self-supervised learning (e.g., DINO) to extract semantic features for interactive classification [S2].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S3] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
