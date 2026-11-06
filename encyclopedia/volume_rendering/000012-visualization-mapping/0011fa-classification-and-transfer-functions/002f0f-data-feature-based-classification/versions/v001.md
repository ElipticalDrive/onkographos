# Data Feature-Based Classification


## Overview
Data feature-based classification assigns opacity and color to volume data by mapping one or more derived data features (such as texture, distance, shape, or curvature) to visual properties, rather than relying solely on raw scalar values. This approach solves the ambiguity inherent in 1D transfer functions where different materials may share similar intensity values but possess distinct structural or textural characteristics. It is distinguished from sibling methods like [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by its explicit use of higher-dimensional feature spaces (e.g., gradient, texture metrics, or spatial relationships) to drive the classification decision, often enabling semi-automatic or automatic segmentation of complex structures.

## Shared principles
Data feature-based classification relies on the construction of a feature vector $\mathbf{f}_i$ for each voxel $i$ in the volume. Unlike simple scalar mapping, this vector $\mathbf{f}_i$ contains multiple components derived from the raw data $V$ and its local neighborhood. The classification function $C(\mathbf{f}_i)$ maps this high-dimensional input to a scalar confidence or probability score, which is then used to determine opacity $\alpha$ and color $\mathbf{c}$.

The general pipeline consists of:
1.  **Feature Extraction**: Computing local or global descriptors for each voxel. Common features include:
    *   Raw data value $V_i$.
    *   Gradient magnitude $\|\nabla V_i\|$.
    *   Curvature $\kappa_i$ (second derivative).
    *   Texture metrics (e.g., from local histograms or co-occurrence matrices).
    *   Distance to a seed or segmented region $d_i$.
    *   Spatial position $\mathbf{p}_i = (x, y, z)$.
    *   Neighbor colors or values.
2.  **Classification**: Applying a classifier (e.g., SVM, Random Forest, Neural Network, or heuristic rule) to the feature vector.
3.  **Transfer Function Application**: Mapping the classifier output to visual properties.

The choice of features determines the discriminative power of the classification. For instance, texture features are essential for distinguishing materials with similar intensities but different microstructures, while distance features are useful for isolating specific anatomical structures relative to a known landmark.

## Subtopics

### [Texture-based Transfer Function](5450-texture-based-transfer-function.md)
A transfer function that uses data value and 20 texture metrics as input. It is typically automatic or semi-automatic. It is chosen when materials have similar intensity values but distinct textural patterns (e.g., distinguishing tissue types in medical imaging or geological strata). It differs from scalar-based methods by incorporating local statistical variations.

### [Distance-based Transfer Function](5451-distance-based-transfer-function.md)
A transfer function that uses data value and distance to a previously segmented structure as metrics. It is chosen when the goal is to isolate a specific object or region relative to a known reference point. It differs from other methods by explicitly using spatial proximity as a discriminative feature, allowing for the selection of structures based on their location relative to a seed or boundary.

### [Shape-based Transfer Function Generation](5941-shape-based-transfer-function-generation.md)
Shape classification assigns to each skeleton region a degree of membership in supported shape classes. Only voxels in the pre-segmentation are considered; background voxels are ignored (descriptor set to all zero). Classes are computed independently. It is chosen when the geometric shape of a structure is a primary identifier (e.g., distinguishing between tubular and spherical organs). It differs from intensity-based methods by focusing on global or local geometric properties rather than raw values.

### Curvature-based Transfer Function
**Definition**
A transfer function that uses data value and curvature (second derivative) to distinguish transitions in tissue [S1]. It maps data and curvature to color and opacity, often in combination with camera position. It is used to display contours around features, ridge/valley visualization, or emphasize bumpiness [S1].

**Step**
Computation of curvature requires at least a 26-point neighborhood [S1].

**Parameter**
Metrics: Data value and curvature [S1].

**Layout**
Data structure: 2D look-up table [S1].

**Pitfall**
Many features share the same curvature, making it difficult to emphasize a targeted feature without unintended emphasis. Noise amplifies this behavior [S1].

**Number**
Real-time capability: Yes, with a minimum of 27 3D texture lookups per sample [S1].

**Usage**
Real-time GPU variants are found in GPU Gems 2 [PF05] and [HSS∗05] for iso-surfaces [S1].

### Classifier Reuse
**Definition**
Saving a trained neural network or SVM configuration to apply it to new data sets with similar characteristics, reducing classification time [S2].

**Step**
Apply the saved classifier to the new data set. If misclassification occurs, the user can refine it by supplying additional training data [S2].

**Parameter**
Saved classifier size: under 30 kilobytes [S2].

**Pitfall**
Position information is specific to a given data set; using it reduces reusability because location changes between scans. It should be excluded if reusability is intended [S2]. Reused classifiers may misclassify materials if the new data set does not share similar characteristics (e.g., data value ranges or material locations) [S2].

### High-Dimensional Classification with Color Data
**Definition**
A method for classifying color volume data by assigning opacity based on a classification function that uses RGB values, neighbor colors, and position, avoiding the difficulty of specifying 3D transfer functions for RGB data [S2].

**Step**
Paint on 2D slices passing through the volume to provide training data [S2]. Construct classification features from the RGB color values of a voxel, the colors of its six neighbors, and its position [S2].

**Parameter**
Input dimensions: Up to 24 dimensions (RGB color + 6 neighbors * 3 colors + position components, though position dimensionality is not explicitly broken down, the total is stated as up to 24) [S2].

**Comparison**
Unlike methods using Independent Component Analysis (ICA) that require users to work in a derived data space, this method allows users to work directly with original RGB data [S2].

### Cosine Similarity Map Computation
**Definition**
Computes a similarity volume $S_L$ by comparing annotated feature vectors against the full feature volume F using cosine similarity, enabling interactive feedback [S3].

**Formula**
$$S_L^C = \max_{a \in A_C} \frac{1}{|F|} \sum_{i \in F} \frac{F_a \cdot F_i}{\|F_a\|_2 \|F_i\|_2}$$
where $A_C$ is the set of annotations for class C, $F_a$ is the feature vector at annotation $a$, and $F_i$ is the feature vector at voxel $i$. Resulting $S_L^C \in [0, 1]^{W' \times H' \times D'}$ [S3].
$$P(x) = \max_{a \in A_C} e^{-10\tilde{p}|x-a|}$$
where $x$ is the location in $R^3$ and $a$ is an annotation location [S3].

**Parameter**
Proximity parameter $\tilde{p} \in [0, 1]$, used to scale similarity based on distance to annotations [S3].

**Number**
Computation time: A few milliseconds on CPU or GPU [S3].

### DINO Features for SVM/RF Classification
**Number**
SVM IoU on CT-ORG dataset with DINO features: 0.24 (liver), 0.36 (bones), 0.0 (other classes), mean 0.09. Fit time: 407s. Inference time: 2077s [S3].

## Background
Data feature-based classification emerged as a response to the limitations of 1D transfer functions, which rely solely on scalar data values. Early work by Levoy [Lev88] and Kindlmann et al. [KKH02] introduced gradient-based transfer functions, which added the first derivative to the feature set. Subsequent research expanded this to include curvature [KWTM03], texture [CR08], and distance [TPD06] metrics. The survey by [S1] categorizes these methods and compares their real-time capabilities and user interaction requirements. Later developments, such as those by Tzeng et al. [S2], introduced high-dimensional classification using machine learning techniques (SVMs, Neural Networks) to handle complex feature spaces, including color and spatial position. More recently, self-supervised vision transformers (DINO) have been leveraged to extract semantic features for transfer function design [S3].

## Sources
[S1] A Survey of Transfer Functions Suitable for Volume Rendering, https://doi.org/10.2312/vg/vg10/077-083, fetched 2026-10-26
[S2] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S3] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
