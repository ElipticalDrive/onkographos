# Data Feature-Based Classification


## Overview
Data feature-based classification extends the classification domain beyond scalar intensity to include derived geometric, structural, or semantic features, enabling the distinction of materials or structures that have overlapping intensity ranges [S1, S2][S3, unofficial]. Unlike sibling methods that rely solely on 1D intensity mapping, this family utilizes multi-dimensional feature vectors—such as curvature, shape descriptors, color neighborhoods, or learned semantic embeddings—to define optical properties [S1, S2][S3, unofficial][S4].

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a scalar value $v = f(\mathbf{x})$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, S2][S3, unofficial][S4]. Advanced methods extend this domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, or texture metrics [S2, S5, unofficial].

**Output Range:**
The output is a set of optical properties, most commonly:
*   **Opacity ($\alpha$):** A value in $[0, 1]$ representing the probability of absorption or the inverse of transparency. $\alpha=0$ is fully transparent; $\alpha=1$ is fully opaque [S5, unofficial].
*   **Color ($C$):** A vector $(r, g, b)$ representing the emitted or reflected light color [S2][S4].
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

### [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md)
The Curvature-based Transfer Function (cbTF) maps volume data values and local surface curvature to optical properties. It distinguishes structural boundaries and geometric features (contours, ridges) that scalar values cannot resolve. Unlike sibling methods relying on gradient magnitude, cbTF uses second derivatives to highlight shape "bumpiness," making it effective for non-photorealistic rendering.

### [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md)
This method assigns optical properties based on 3D shape descriptors (tubiness, surfaceness, blobbiness) derived from curve-skeletons. It solves the problem of distinguishing features with overlapping intensities (e.g., vessels vs. bones) by classifying complete volumetric features rather than per-voxel values. Unlike per-voxel methods like cbTF, it allows intuitive assignment of properties to shape classes.

### [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md)
This method classifies color volume data using a high-dimensional feature vector comprising the voxel's RGB color, the colors of its six immediate neighbors, and its spatial position (up to 24 dimensions). It solves the difficulty of specifying 3D transfer functions for RGB data by allowing users to paint on 2D slices for training, treating color volumes similarly to scalar volumes. Unlike ICA-based methods, it operates directly on original RGB data.

### [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md)
This method automatically discovers material interiors and their spatial interconnectivity by generating a graph $G(M, I)$ where $M$ is the set of materials and $I$ is the set of interfaces. It sets itself apart by explicitly modeling materials as Gaussian intensity distributions with occlusion values and deducing connectivity through a greedy merging process optimized by separation energy, rather than relying solely on geometric curvature or static feature clustering.

### [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md)
This method enhances ray casting object distinction by assigning object membership ID tags to cubical vertices via a hybrid segmentation algorithm. It combines unsupervised neural network initialization with K-means clustering and statistical optimization (Fisher discriminant ratio) to avoid local optima, enabling the use of binary transfer functions for color and opacity assignment.

### [Feature Graph](8595-feature-graph.md)
The Feature Graph is a hierarchical data structure (DAG) representing classified voxel models, where nodes are features (clusters of voxels sharing boolean labels) and arcs represent set inclusion. It distinguishes itself by explicitly modeling the set-inclusion topology of classification criteria, supporting multiresolution rendering by storing voxel submodels (SOVs) at different resolution levels within the hierarchy.

### [Attribute Ranking Heuristic](11691-attribute-ranking-heuristic.md)
This is a greedy method for selecting relevant volumetric attributes to construct voxel signatures for classification. It solves the problem of high-dimensional transfer function specification by ranking candidate attributes based on their relevance (entropy and multiple correlation) to the data, allowing users to select a subset for feature vectors.

### Classifier Reuse
- **Definition:** Saving a trained neural network or SVM configuration to apply it to new data sets with similar characteristics, reducing classification time [S1].
- **Step:** Apply the saved classifier to the new data set. If misclassification occurs, the user can refine it by supplying additional training data [S1].
- **Parameter:** Saved classifier size: under 30 kilobytes [S1].
- **Pitfall:** Position information is specific to a given data set; using it reduces reusability because location changes between scans. It should be excluded if reusability is intended [S1]. Reused classifiers may misclassify materials if the new data set does not share similar characteristics (e.g., data value ranges or material locations) [S1].

### Cosine Similarity Map Computation
- **Definition:** Computes a similarity volume $S_L$ by comparing annotated feature vectors against the full feature volume $F$ using cosine similarity, enabling interactive feedback [S2].
- **Formula:** $S_L^C = \max_{a \in A_C} \frac{1}{|F|} \sum_{i \in F} \frac{F_a \cdot F_i}{\|F_a\|_2 \|F_i\|_2}$ where $A_C$ is the set of annotations for class $C$, $F_a$ is the feature vector at annotation $a$, and $F_i$ is the feature vector at voxel $i$. Resulting $S_L^C \in [0, 1]^{W' \times H' \times D'}$ [S2]. $P(x) = \max_{a \in A_C} e^{-10\tilde{p}|x-a|}$ where $x$ is the location in $R^3$ and $a$ is an annotation location [S2].
- **Parameter:** Proximity parameter $\tilde{p} \in [0, 1]$, used to scale similarity based on distance to annotations [S2].
- **Number:** Computation time: A few milliseconds on CPU or GPU [S2].

### DINO Features for SVM/RF Classification
- **Number:** SVM IoU on CT-ORG dataset with DINO features: 0.24 (liver), 0.36 (bones), 0.0 (other classes), mean 0.09. Fit time: 407s. Inference time: 2077s [S2].

### Domain Knowledge Attributes for Classification
- **Definition:** Attributes that capture domain knowledge to complement gradient-based measures. The paper specifically employs 'range weight' and 'local variance' as examples of these attributes to enable additional visualization possibilities [S3, unofficial].

### Thresholded Cosine Similarity Segmentation
- **Step:** Compute probability of a query using thresholded cosine similarity instead of softmax with negative queries [S4]. Set $p = 1$ if similarity exceeds threshold, $p = 0$ otherwise for hard decomposition [S4].

### Feature Visualization
- **Step:** Calculate 3-dimensional PCA components using teacher's feature map of first frame of training view images [S4]. Visualize features by components as RGB, normalized with min-max values with outlier removal [S4].

## Background
Data feature-based classification originated from the need to overcome the limitations of 1D intensity-based transfer functions, particularly in medical imaging where different tissues often share similar density values [S1][S3, unofficial]. Early work by Tzeng et al. introduced high-dimensional classification using support vector machines (SVMs) and neural networks to map multi-dimensional feature vectors (color, neighbors, position) to opacity [S1]. Subsequent developments incorporated geometric features like curvature [S1, S4] and shape descriptors derived from curve-skeletons [S3, unofficial] to better represent structural semantics. More recently, the field has expanded to include learned semantic features from self-supervised vision transformers (e.g., DINO) [S2] and graph-based representations of material connectivity [S1].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S3] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
[S4] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
