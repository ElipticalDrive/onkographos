# Data Feature-Based Classification

Also called: High-Dimensional Classification, Multidimensional Transfer Function (MTF) Classification, Feature-Based Classification

## Overview
Data Feature-Based Classification extends the classification domain beyond scalar intensity values to include derived geometric, statistical, or semantic features, enabling the separation of materials that overlap in intensity space [S1, S2][S3, unofficial]. Unlike standard 1D transfer functions that map a single scalar value to optical properties, this family constructs a feature vector $\mathbf{f}$ for each voxel and maps $\mathbf{f}$ to opacity and color, often using machine learning models, graph structures, or explicit geometric descriptors [S1, S2][S3, unofficial]. It sets itself apart from siblings like [Binary and Soft Classification](6229-binary-and-soft-classification.md) by relying on multidimensional feature spaces rather than simple thresholding or smooth ramps on scalar data, and from [Probabilistic and Statistical Modeling](5932-probabilistic-and-statistical-modeling.md) by often employing heuristic ranking, graph deduction, or deep learning features rather than explicit probability density estimation [S1, S2][S3, unofficial].

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is a feature vector $\mathbf{f}(\mathbf{x})$ derived from the volume data. Common features include:
*   **Intensity:** The scalar value $v = f(\mathbf{x})$ [S1, S2][S3, unofficial].
*   **Gradient/Geometry:** Gradient magnitude, curvature, or shape descriptors (tubiness, surfaceness) [S1, S2][S3, unofficial].
*   **Context:** Neighbor colors, spatial position, or texture metrics (e.g., Diagonal Busyness Factor) [S1, S2][S3, unofficial].
*   **Semantic/Deep Features:** Embeddings from self-supervised models (e.g., DINO) or domain-specific attributes (range weight, local variance) [S2][S3, unofficial].

**Output Range:**
The output is a set of optical properties, most commonly:
*   **Opacity ($\alpha$):** A value in $[0, 1]$ [S5, unofficial].
*   **Color ($C$):** A vector $(r, g, b)$ [S2][S4].
*   **Weight:** In some emission-only models, a weight $w$ is assigned [S6, unofficial].

**Classification Types:**
1.  **Binary Classification:** Hard thresholds in the feature space [6229-binary-and-soft-classification.md].
2.  **Soft Classification:** Smooth transitions in the feature space [S3, unofficial][S8, unofficial].

**Evaluation Order:**
The timing of classification relative to sampling interpolation defines two main approaches [S2][S5, unofficial]:
*   **Pre-classification:** The transfer function is applied to scalar values at the grid points (voxels) before sampling. Colors and opacities are interpolated during ray casting. Pre-classification produces blurred images when the image resolution is higher than the volume resolution, but prevents color bleeding if color interpolation is done with opacity-weighted colors [S2].
*   **Post-classification:** Scalar values are interpolated from the volume texture during ray casting, and the transfer function is applied to these continuous sample values. Post-classification produces images with defined edges but is prone to aliasing because high-frequency details may be missed [S2].

**Implementation Structures:**
Transfer functions are commonly implemented as piecewise linear functions defined by user-specified keypoints [S3, unofficial], lookup tables (LUTs) stored as 1D, 2D, or 3D textures for fast GPU sampling [S3, unofficial][S6, unofficial], or shader functions evaluated dynamically during ray casting [S5, unofficial].

## Subtopics

### [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md)
The Curvature-based Transfer Function (cbTF) maps scalar data values and local surface curvature to opacity and color, primarily to distinguish feature boundaries and produce non-photorealistic contouring effects. Unlike gradient-based methods that highlight edge intensity, cbTF distinguishes features based on their geometric shape (e.g., distinguishing the border of a small feature from a large one). It is particularly effective for ridge/valley visualization and emphasizing bumpiness, though it struggles to isolate specific features without unintended emphasis on other structures sharing similar curvature. See [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md).

### [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md)
Shape-based Transfer Function Generation is a classification method that assigns optical properties to volumetric features based on their 3D shape rather than just intensity or gradient magnitude. It solves the problem of distinguishing features with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) by using a shape descriptor derived from curve-skeletons. Unlike per-voxel classification methods that rely on local filters, this method classifies complete volumetric features (segments) using curve-skeletons, allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels. See [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md).

### [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md)
This method classifies color volume data by assigning opacity based on a high-dimensional classification function that incorporates RGB values, neighbor colors, and spatial position. It solves the difficulty of specifying three-dimensional transfer functions for RGB data using traditional 2D interfaces by allowing users to paint on 2D slices, similar to scalar data classification. Unlike methods using Independent Component Analysis (ICA) that require users to work in a derived data space, this method allows users to work directly with original RGB data. See [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md).

### [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md)
Robust Material Graph Deduction automatically discovers the set of materials and their spatial interconnectivity within a volume to generate a transfer function, solving the problem of manual classification by mapping material interiors rather than just boundaries. It sets itself apart from sibling methods like [Feature Graph](8595-feature-graph.md) by explicitly modeling materials as Gaussian intensity distributions with occlusion values and using a greedy merging strategy based on Bhattacharyya distance to optimize interface separation, whereas siblings often rely on static feature clustering or heuristic ranking. See [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md).

### [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md)
This method enhances ray casting object distinction by assigning object membership information (ID tags) to cubical vertices in the volume, generated via a hybrid segmentation algorithm. It differs from standard K-means by using a Self-Organizing Map (SOM) to initialize cluster centers, thereby avoiding local optima, and refining the result using a Diagonal Busyness Factor (DF) for texture and Fisher discriminant ratio for homogeneity. See [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md).

### [Feature Graph](8595-feature-graph.md)
The Feature Graph is a hierarchical data structure for volume rendering that represents classified voxel data as a directed acyclic graph (DAG). It solves the problem of efficiently managing and rendering complex voxel datasets by clustering voxels into features based on non-exclusive boolean classification criteria and storing voxel subsets only at leaf nodes to eliminate redundancy. Unlike standard transfer functions that map scalar values to optical properties, the Feature Graph organizes data spatially and semantically, allowing for multiresolution rendering and direct access to voxel subsets via a hash table. It is distinguished from sibling methods like [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) by its explicit graph-based topology of inclusion relationships rather than purely local geometric or statistical feature evaluation. See [Feature Graph](8595-feature-graph.md).

### [Attribute Ranking Heuristic](11691-attribute-ranking-heuristic.md)
The Attribute Ranking Heuristic is a greedy method for selecting relevant volumetric attributes to construct voxel signatures for multidimensional transfer function (MTF) classification. It addresses the complexity of specifying MTFs by automatically ranking candidate attributes based on their relevance to the data, presenting the top $n$ candidates to the user for manual selection. Unlike sibling methods that may rely on fixed feature sets or complex machine learning training, this heuristic explicitly balances information content against redundancy using entropy and multiple correlation. It builds on [Data Feature-Based Classification](5934-data-feature-based-classification.md) by providing a systematic way to reduce the dimensionality of the classification domain. See [Attribute Ranking Heuristic](11691-attribute-ranking-heuristic.md).

### Classifier Reuse
*   **Definition:** Saving a trained neural network or SVM configuration to apply it to new data sets with similar characteristics, reducing classification time [S1].
*   **Step:** Apply the saved classifier to the new data set. If misclassification occurs, the user can refine it by supplying additional training data [S1].
*   **Parameter:** Saved classifier size: under 30 kilobytes [S1].
*   **Pitfall:** Position information is specific to a given data set; using it reduces reusability because location changes between scans. It should be excluded if reusability is intended [S1]. Reused classifiers may misclassify materials if the new data set does not share similar characteristics (e.g., data value ranges or material locations) [S1].

### Cosine Similarity Map Computation
*   **Definition:** Computes a similarity volume $S_L$ by comparing annotated feature vectors against the full feature volume F using cosine similarity, enabling interactive feedback [S2].
*   **Formula:** $S_L^C = \max_{a \in A_C} \frac{1}{|F|} \sum_{i \in F} \frac{F_a \cdot F_i}{\|F_a\|_2 \|F_i\|_2}$ where $A_C$ is the set of annotations for class C, $F_a$ is the feature vector at annotation $a$, and $F_i$ is the feature vector at voxel $i$. Resulting $S_L^C \in [0, 1]^{W' \times H' \times D'}$ [S2]. $P(x) = \max_{a \in A_C} e^{-10\tilde{p}|x-a|}$ where $x$ is the location in $R^3$ and $a$ is an annotation location [S2].
*   **Parameter:** Proximity parameter $\tilde{p} \in [0, 1]$, used to scale similarity based on distance to annotations [S2].
*   **Number:** Computation time: A few milliseconds on CPU or GPU [S2].

### DINO Features for SVM/RF Classification
*   **Number:** SVM IoU on CT-ORG dataset with DINO features: 0.24 (liver), 0.36 (bones), 0.0 (other classes), mean 0.09. Fit time: 407s. Inference time: 2077s [S2].

### Domain Knowledge Attributes for Classification
*   **Definition:** Attributes that capture domain knowledge to complement gradient-based measures. The paper specifically employs 'range weight' and 'local variance' as examples of these attributes to enable additional visualization possibilities [S3, unofficial].

## Background
Data Feature-Based Classification originated from the need to resolve ambiguities in volume rendering where different materials share similar scalar intensity values. Early work by [Tzeng et al.](https://doi.org/10.1109/tvcg.2005.38) introduced high-dimensional classification using Support Vector Machines (SVMs) and neural networks to classify color and gradient-based features, allowing for the separation of complex anatomical structures [S1]. Later developments incorporated geometric features such as curvature [S1] and shape descriptors derived from curve-skeletons [S3, unofficial] to distinguish objects based on topology rather than just intensity. More recent approaches leverage self-supervised vision transformers (e.g., DINO) to extract semantic features directly from volume slices, enabling interactive annotation and similarity-based classification [S2]. Additionally, methods like the Feature Graph [S1] and Robust Material Graph Deduction [S1] have been developed to automatically deduce material structures and hierarchies from data, reducing the reliance on manual transfer function design. The field continues to evolve with the integration of machine learning heuristics for attribute ranking [S1] and the reuse of trained classifiers across similar datasets [S1].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S3] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
