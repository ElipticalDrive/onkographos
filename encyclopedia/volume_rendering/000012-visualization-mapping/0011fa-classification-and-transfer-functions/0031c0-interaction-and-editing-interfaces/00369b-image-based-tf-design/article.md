# Image-based TF Design

Also called: Scribble-based TF Design, User-guided TF Design

## Overview
Image-based TF Design is a user-guided classification method where users draw scribbles on image slices to indicate Regions of Interest (ROIs), which are then classified using Random Forest on Implicit Neural Representation (INR) features to generate probabilistic volume rendering [S1, unofficial]. It solves the problem of manual transfer function tuning by allowing sparse user supervision to automatically classify volumetric data. Unlike its parent [Interaction and Editing Interfaces](6592-interaction-and-editing-interfaces.md), which provides general tools for editing, this method specifically automates the classification step via machine learning. It differs from [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md) by operating on volumetric voxels via slice projections rather than editing 2D transfer function curves directly, and from [Position-dependent Transfer Function Customization](5287-position-dependent-transfer-function-customization.md) by deriving classification from image content features rather than spatial coordinates alone.

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset $V$, a set of user-drawn scribbles on 2D cross-sectional slices, and a set of target ROI classes $\{C_1, \dots, C_N\}$.
*   **Output:** A probabilistic volume rendering where each voxel $v$ has an associated probability vector $P(v)$ and corresponding optical properties (color and opacity).

**Data Structures**
*   **Scribble Data:** A set of labeled voxels $L = \{(v, c)\}$ where $v$ is a voxel index and $c \in \{1, \dots, N\}$ is the class label assigned by the user via a brush tool on a slice.
*   **Feature Vector:** For each voxel $v$, a feature vector $f(v)$ is extracted. The source specifies the use of Implicit Neural Representation (INR) features [S1, unofficial]. The exact dimensionality and bit-packing of $f(v)$ are not given in the sources.
*   **Classifier:** A Random Forest model trained on pairs $(f(v), c)$ from $L$.
*   **Probability Vector:** For each voxel $v$, a vector $P(v) = [p_1(v), p_2(v), \dots, p_N(v)]$, where $p_n(v)$ is the probability that voxel $v$ belongs to class $C_n$.

**Steps**
1.  **User Interaction:** The user selects a cross-sectional image slice from the volume. Using a brush tool, the user draws scribbles on the slice to label voxels belonging to specific ROIs [S1, unofficial]. These labels are mapped back to the corresponding 3D voxels in the volume.
2.  **Feature Extraction:** For all labeled voxels $v \in L$, extract the INR feature representation $f(v)$ [S1, unofficial].
3.  **Classification Training:** Train a Random Forest classifier using the labeled feature vectors $\{(f(v), c) | v \in L\}$. Random Forest is chosen for its robust performance and fast training time under sparse supervision [S1, unofficial].
4.  **Probability Prediction:** For every unlabeled voxel $v$ in the volume, predict the class probabilities $P(v) = [p_1(v), \dots, p_N(v)]$ using the trained Random Forest [S1, unofficial].
5.  **Rendering:** Feed the probability vectors $P(v)$ into a multi-class probabilistic volume renderer. The renderer calculates the final optical properties for each voxel using a weighted sum of the optical properties of all $N$ ROI classes.

**Formulas**
The final optical properties $C(v)$ for a voxel $v$ are calculated as:
$$ C(v) = \sum_{n=1}^{N} TF_{c,n} \cdot p_n(v) \cdot p_n(v) $$
Where:
*   $N$ is the total number of ROI classes.
*   $TF_{c,n}$ is the color and opacity (optical properties) assigned to the $n$-th ROI class in the transfer function.
*   $p_n(v)$ is the probability that voxel $v$ belongs to class $C_n$.
*   Note: The source formula includes $p_n(v)$ twice ($p_n(v) \cdot p_n(v)$) [S1, unofficial].

**Parameters**
*   **Number of Classes ($N$):** Determined by the user's defined ROIs.
*   **Random Forest Hyperparameters:** Not given in the sources.

**Edge Cases and Pitfalls**
*   **Sparse Supervision:** The method relies on Random Forest's ability to generalize from sparse labels. If scribbles are too sparse or ambiguous, classification accuracy may degrade [S1, unofficial].
*   **Feature Representation:** The quality of classification depends on the INR features. The specific extraction method for INR features is not detailed in the provided text [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_based_tf_design(volume, user_scribbles, num_classes):
    """
    volume: 3D array of voxel data
    user_scribbles: list of (voxel_index, class_label) tuples from user input
    num_classes: int N
    """
    # Step 1 & 2: Extract features for labeled voxels
    # Note: INR feature extraction details are not provided in sources [S1, unofficial]
    labeled_features = []
    labeled_labels = []
    for voxel_idx, class_label in user_scribbles:
        f_v = extract_inr_features(volume, voxel_idx) # [G] Placeholder for INR extraction
        labeled_features.append(f_v)
        labeled_labels.append(class_label)

    # Step 3: Train Random Forest
    # Note: Specific RF implementation details are not provided [S1, unofficial]
    classifier = train_random_forest(labeled_features, labeled_labels)

    # Step 4: Predict probabilities for all voxels
    all_voxels = get_all_voxel_indices(volume)
    all_probs = []
    for v in all_voxels:
        f_v = extract_inr_features(volume, v) # [G] Placeholder for INR extraction
        p_v = classifier.predict_proba(f_v) # Returns array of size N
        # Ensure p_v has length num_classes
        if len(p_v) < num_classes:
            p_v = np.pad(p_v, (0, num_classes - len(p_v)))
        all_probs.append(p_v)

    # Step 5: Render using probabilistic renderer
    # Note: TF_c,n values are assumed to be provided or defined by user
    # C(v) = sum(TF_c,n * p_n(v) * p_n(v))
    rendered_volume = render_probabilistic(volume, all_probs, num_classes)
    return rendered_volume
```

## Subtopics

### [Slice-guided Semiautomatic Multivariate Volume Exploration](9484-slice-guided-semiautomatic-multivariate-volume-exploration.md)
GuideME is an interactive volume visualization workflow that replaces trial-and-error transfer function tuning with a guided, uncertainty-aware region selection process on 2D slices. It solves the problem of exploring high-dimensional, multivariate volumetric data by allowing users to define Regions of Interest (ROIs) via a lasso that snaps to detected feature boundaries, automatically extracting features to generate a probabilistic transfer function. Unlike its parent [Image-based TF Design](7807-image-based-tf-design.md), which relies on Random Forest classification of user scribbles, GuideME utilizes edge detection and shortest-path algorithms for boundary-aware selection and direct response function optimization for transfer function generation.

## Operating characteristics
*   **Time Complexity:** Training time is fast due to the Random Forest algorithm, even with sparse user supervision [S1, unofficial]. Prediction time is linear with respect to the number of voxels and the number of trees in the forest.
*   **Memory Complexity:** Requires storage for the volume data, the trained Random Forest model, and the probability vector for each voxel.
*   **Strengths:** Robust classification performance; fast training; allows expert users to perform finer TF adjustments for subtle structures (e.g., small vessels, lung bronchus) [S1, unofficial].
*   **Limitations:** Relies on the quality of INR features; requires user interaction to provide labels; probabilistic rendering may introduce blending artifacts if probabilities are not well-calibrated.
*   **When to use:** When exploring multivariate volumetric data with complex structures where manual TF tuning is difficult, and sparse user guidance is acceptable.
*   **When not to use:** When real-time interaction with immediate feedback on every stroke is required without retraining; when INR features are not applicable or too expensive to compute.

## Background
Image-based TF Design is part of the NeuVolEx system, which uses Implicit Neural Features for volume exploration [S1, unofficial]. The method was introduced to address the challenges of volume exploration tasks, specifically the need for efficient and accurate classification of ROIs based on user input. It builds on the concept of using machine learning (Random Forest) to bridge the gap between user intent (scribbles) and volumetric classification.

## Sources
[S1] NeuVolEx: Implicit Neural Features for Volume Exploration, https://doi.org/10.48550/arxiv.2604.11172, fetched 2026-10-26 (unofficial)
