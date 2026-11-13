# Machine Learning and Feature Extraction

Also called: Intelligent classification, Data-driven classification, Higher-dimensional classification

## Overview
This family applies machine learning algorithms and feature extraction techniques to automatically or semi-automatically derive transfer functions for volume rendering, moving beyond manual specification. It solves the problem of efficiently classifying complex, high-dimensional data (e.g., multi-modal medical scans) by learning decision boundaries from training data or extracting invariant features. It differs from its siblings by relying on statistical learning models (SVMs, Neural Networks) or unsupervised feature analysis (ICA, DINO) rather than geometric heuristics (curvature, shape) or manual graph construction.

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a scalar value $v = f(\mathbf{x})$ representing the voxel intensity or density at position $\mathbf{x}$ [S1][S2][S3, unofficial][S4]. Advanced methods extend this domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, or texture metrics [S2, S5, unofficial].

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

### [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md)
An image processing approach that enhances ray casting object distinction by assigning object membership information (ID tags) to cubical vertices in the volume. It differs from standard clustering by combining Self-Organizing Maps (SOM) for robust initialization, K-means for preliminary grouping, and Fisher discriminant analysis for statistical optimization, specifically designed to generate binary, non-overlapping transfer functions for color and opacity assignment.

*   **When to choose:** When robust segmentation of color volume data is required, and the user needs a hybrid approach that leverages neural network initialization for stability followed by statistical optimization.
*   **Link:** [K-means-based hybrid segmentation](7597-k-means-based-hybrid-segmentation.md)

### Classifier Reuse
Saving a trained neural network or SVM configuration to apply it to new data sets with similar characteristics, reducing classification time [S1].

*   **Definition:** The trained model (neural network or SVM) is saved after training to be applied to subsequent datasets [S1].
*   **Step:** Apply the saved classifier to the new data set. If misclassification occurs, the user can refine it by supplying additional training data [S1].
*   **Parameter:** Saved classifier size: under 30 kilobytes [S1].
*   **Pitfall:** Position information is specific to a given data set; using it reduces reusability because location changes between scans. It should be excluded if reusability is intended [S1]. Reused classifiers may misclassify materials if the new data set does not share similar characteristics (e.g., data value ranges or material locations) [S1].

### DINO Features for SVM/RF Classification
Using DINO (Distillation of Knowledge in the NO-teacher) self-supervised vision transformer features for classification [S2].

*   **Number:** SVM IoU on CT-ORG dataset with DINO features: 0.24 (liver), 0.36 (bones), 0.0 (other classes), mean 0.09. Fit time: 407s. Inference time: 2077s [S2].

### ICA-based Transfer Function Design
A technique using Independent Component Analysis (ICA) to represent voxel signatures via linear transformation into a space where manual classification is simpler, often achievable using separator planes perpendicular to the new coordinate axes. ICA infers the linear transformation coefficients to simplify the signature representation [S3, unofficial].

*   **Step:** 1. Apply ICA to voxel signatures. 2. Infer appropriate linear transformation coefficients. 3. Transform voxel signatures into the new space. 4. Perform manual classification using separator planes perpendicular to the new axes [S3, unofficial].

## Background
Originating from the need to handle higher-dimensional volume data (e.g., multi-modal medical imaging), this approach was pioneered by Tzeng et al. in the mid-2000s, who implemented multidimensional transfer functions as back-propagation neural networks [S1][S3, unofficial]. The field has evolved to include self-supervised learning features, such as DINO features, for segmentation-based transfer function design [S2]. It is primarily used in medical visualization to automate the classification of complex anatomical structures [S1].

## Sources
[S1] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
[S2] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S3] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
