# Data Feature-Based Classification

Also called: High-dimensional classification, Multi-dimensional transfer functions, Feature-based classification

## Overview
Data Feature-Based Classification extends the classification domain beyond scalar intensity to include derived geometric, statistical, or semantic features, solving the problem of distinguishing structures that share similar intensity values but differ in shape, texture, or context [S1, S2][S3, unofficial]. Unlike [Binary and Soft Classification](6229-binary-and-soft-classification.md) which relies solely on intensity thresholds, this family maps complex feature vectors to optical properties, enabling the separation of overlapping materials and the emphasis of specific geometric contours [S1, S2][S3, unofficial].

## Shared principles
All methods in this family evaluate a classification function $C(\mathbf{x})$ where the input domain includes derived features in addition to or instead of the raw scalar value $f(\mathbf{x})$ [S1, S2][S3, unofficial].

**Feature Extraction:**
Features are computed per voxel or per segmented region. Common feature types include:
*   **Geometric:** Curvature, shape descriptors (tubiness, surfaceness, blobbiness), and distance transforms [S1][S3, unofficial].
*   **Statistical/Textural:** Local variance, range weight, and neighbor color statistics [S1][S3, unofficial].
*   **Semantic/Learned:** High-dimensional embeddings from self-supervised models (e.g., DINO) or neural network weights [S2].

**Classification Domain:**
The classification function operates in a high-dimensional space $\mathbb{R}^d$ where $d > 1$. The input vector $\mathbf{f}_i$ for voxel $i$ is constructed by concatenating relevant features [S1, S2].

**Output:**
The output remains the standard optical properties: opacity $\alpha \in [0, 1]$ and color $\mathbf{c}$ [S1, S2][S3, unofficial].

**Implementation:**
Classification is typically performed via:
1.  **Lookup Tables (LUTs):** Multi-dimensional tables (2D, 3D) indexed by feature values [S1][S3, unofficial].
2.  **Machine Learning Models:** Trained classifiers (SVM, Neural Networks, Random Forests) that map feature vectors to opacity [S1, S2].
3.  **Graph-Based Deduction:** Automatic discovery of material interfaces and topology [S1].

## Subtopics

### [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md)
Maps data value and local curvature to opacity and color. It distinguishes structural boundaries and emphasizes geometric contours (ridges, valleys) for non-photorealistic rendering, solving the limitation of gradient-based methods that cannot distinguish features of different sizes with similar intensity gradients.

### [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md)
Classifies volumetric features based on 3D geometric shape (tubiness, surfaceness, blobbiness) derived from curve-skeletons. It solves the problem of distinguishing structures with overlapping intensity ranges by assigning uniform shape descriptors to entire segmented features, rather than per-voxel local structures.

### [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md)
Classifies color volume data using a feature vector comprising RGB values, neighbor colors, and spatial position. It solves the difficulty of specifying 3D transfer functions for RGB data by allowing users to paint on 2D slices for training, working directly with original RGB data rather than derived spaces like ICA.

### [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md)
Automatically discovers material interiors and spatial interconnectivity by modeling adjacent materials as a graph $G(M, I)$. It solves the problem of manual TF creation by ensuring materials are mapped beyond boundaries, optimizing for separation energy to merge similar materials and reconstruct topology.

### [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md)
Enhances object distinction by assigning object membership IDs via a hybrid segmentation algorithm combining Self-Organized Maps (SOM) and K-means clustering with Fisher discriminant optimization. It produces binary, non-overlapping transfer functions for color and opacity assignment.

### [Feature Graph](8595-feature-graph.md)
Represents classified voxel data as a hierarchical Directed Acyclic Graph (DAG) based on boolean classification criteria. It solves the problem of efficiently managing complex voxel datasets by clustering voxels into features and organizing them into inclusion hierarchies, enabling multiresolution storage and direct access to submodels.

### [Attribute Ranking Heuristic](11691-attribute-ranking-heuristic.md)
A greedy method for selecting the most relevant volumetric attributes for classification. It ranks candidate features (e.g., gradient, curvature) based on information content (entropy) and redundancy (multiple correlation), presenting the top $n$ candidates to the user to simplify high-dimensional TF specification.

### Classifier Reuse
Saving a trained neural network or SVM configuration (under 30 KB) to apply to new datasets with similar characteristics, reducing classification time [S1].
*   **Step:** Apply the saved classifier to new data. If misclassification occurs, refine with additional training data [S1].
*   **Pitfall:** Position information is dataset-specific and should be excluded for reusability. Reused classifiers may misclassify if the new data does not share similar characteristics (e.g., data value ranges) [S1].

### Cosine Similarity Map Computation
Computes a similarity volume $S_L$ by comparing annotated feature vectors against the full feature volume $F$ using cosine similarity, enabling interactive feedback [S2].
*   **Formula:** $S_L^C = \max_{a \in A_C} \frac{1}{|F|} \sum_{i \in F} \frac{F_a \cdot F_i}{\|F_a\|_2 \|F_i\|_2}$ where $A_C$ is the set of annotations for class $C$.
*   **Proximity:** $P(x) = \max_{a \in A_C} e^{-10\tilde{p}|x-a|}$ scales similarity based on distance to annotations, where $\tilde{p} \in [0, 1]$ [S2].
*   **Performance:** Computation takes a few milliseconds on CPU or GPU [S2].

### DINO Features for SVM/RF Classification
Using DINO self-supervised features for classification on the CT-ORG dataset yields an SVM IoU of 0.24 (liver), 0.36 (bones), and 0.0 (others), with a mean of 0.09. Fit time is 407s and inference time is 2077s [S2].

### Domain Knowledge Attributes for Classification
Attributes that capture domain knowledge to complement gradient-based measures, such as 'range weight' and 'local variance', enabling additional visualization possibilities within a Multi-Dimensional Transfer Function (MDTF) [S3, unofficial].

## Background
Data Feature-Based Classification emerged to address the limitations of 1D intensity-based transfer functions, particularly in medical imaging where different tissues often have overlapping intensity ranges. Early work by [Tzeng et al. (2005)](S1) introduced high-dimensional classification using SVMs and neural networks, leveraging neighbor colors and spatial position. Later developments incorporated geometric features like curvature [S1, S2] and shape descriptors derived from skeletons [S3, unofficial]. Recent approaches utilize self-supervised vision transformers (DINO) to generate semantic feature volumes for interactive segmentation [S2], and graph-based methods to automatically deduce material topology [S1]. The field continues to evolve with hybrid segmentation techniques [S1] and heuristic attribute ranking to manage high-dimensional complexity [S1].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S3] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
