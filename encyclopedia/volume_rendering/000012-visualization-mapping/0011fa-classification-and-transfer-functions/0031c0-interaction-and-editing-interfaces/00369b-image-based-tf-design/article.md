# Image-based TF Design

Also called: Scribble-based TF Design, User-guided TF Design

## Overview
Image-based TF Design is a user-guided classification method where users draw scribbles on image slices to indicate Regions of Interest (ROIs), which are then classified using Random Forest on Implicit Neural Representation (INR) features to generate probabilistic volume rendering [S1, unofficial]. It solves the problem of manual transfer function tuning by automating the mapping from sparse user input to a full-volume classification [S1, unofficial]. Unlike its sibling [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md), which typically maps 2D sketches directly to 1D transfer function curves, this method operates in the 3D voxel space using a machine learning classifier trained on spatial and feature context [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset $V$, a set of 2D cross-sectional slices, and user-drawn scribbles (brush strokes) on these slices labeling specific ROIs.
*   **Output:** A probabilistic volume rendering where each voxel $v$ is assigned optical properties (color $C(v)$ and opacity $\alpha(v)$) based on its predicted class probabilities.

**Data Structures:**
*   **Voxel Grid:** Standard 3D grid storing scalar values and computed INR features.
*   **Feature Vector:** For each voxel, a feature vector $\mathbf{f}(v)$ is extracted. The source specifies the use of Implicit Neural Representation (INR) features [S1, unofficial].
*   **Classifier:** A Random Forest model trained on labeled voxels.
*   **Probability Vector:** For each voxel $v$, a vector $P(v) = [p_1(v), p_2(v), \dots, p_N(v)]$ where $p_n(v)$ is the probability that voxel $v$ belongs to the $n$-th ROI class.

**Steps:**
1.  **User Interaction:** Users draw scribbles on cross-sectional image slices using a brush tool to label voxels belonging to specific ROIs [S1, unofficial].
2.  **Feature Extraction:** Extract feature representations from the labeled voxels. The method uses INR features [S1, unofficial].
3.  **Classification Training:** Train a Random Forest classifier using the feature representations extracted from the labeled voxels. Random Forest is chosen for robust classification performance and fast training time even under sparse user supervision [S1, unofficial].
4.  **Probability Prediction:** Predict ROI-wise class probabilities $P(v) = [p_1(v), p_2(v), \dots, p_N(v)]$ for all unlabeled voxels, where $p_n(v)$ is the probability voxel $v$ belongs to the $n$-th ROI class [S1, unofficial].
5.  **Rendering:** Feed probability vectors into a standard front-to-back volume compositing pipeline with a multi-class probabilistic volume renderer [S1, unofficial].

**Formulas:**
The probabilistic renderer assigns optical properties according to predicted ROI-wise class probabilities. The final color $C(v)$ is calculated through a weighted sum of all $N$ ROI classes [S1, unofficial]:

$$ C(v) = \sum_{n=1}^{N} TF_{c,n} \cdot p_n(v) $$

Where:
*   $C(v)$ is the final color of voxel $v$.
*   $TF_{c,n}$ is the color defined for the $n$-th ROI class in the transfer function.
*   $p_n(v)$ is the probability that voxel $v$ belongs to the $n$-th ROI class.
*   $N$ is the total number of ROI classes.

*Note: The source text contains a typo in the formula provided: $C(v) = \sum T Fc,n pn (v) \cdot pn (v)$. The standard interpretation of probabilistic blending is a weighted sum of class colors by their probabilities. The source does not explicitly define opacity calculation in the formula, but implies it is part of the optical properties assignment [S1, unofficial].*

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_based_tf_design(volume, slices, user_scribbles):
    """
    volume: 3D array of voxel values
    slices: list of 2D slice indices and coordinates
    user_scribbles: list of (slice_index, brush_strokes) where brush_strokes are lists of (x, y, class_id)
    """
    # Step 1: Extract features for all voxels
    # Features are based on Implicit Neural Representation (INR)
    # [G] Implementation detail: INR features typically involve sampling the volume 
    # at voxel coordinates using a small neural network or basis functions.
    voxel_features = extract_inr_features(volume)

    # Step 2: Prepare training data from user scribbles
    training_data = []
    training_labels = []
    
    for slice_idx, strokes in user_scribbles:
        for stroke in strokes:
            x, y, class_id = stroke
            # Map 2D scribble to 3D voxel coordinates
            # [G] Assumes scribbles are projected to all z-depths or specific slice voxels
            voxel_coords = get_voxel_coords_from_scribble(slice_idx, x, y)
            
            for vx, vy, vz in voxel_coords:
                if 0 <= vx < volume.shape[0] and 0 <= vy < volume.shape[1] and 0 <= vz < volume.shape[2]:
                    training_data.append(voxel_features[vx, vy, vz])
                    training_labels.append(class_id)

    # Step 3: Train Random Forest Classifier
    # [G] Standard Random Forest training
    classifier = RandomForestClassifier()
    classifier.fit(training_data, training_labels)

    # Step 4: Predict probabilities for all voxels
    all_voxel_features = voxel_features.reshape(-1, voxel_features.shape[-1])
    probabilities = classifier.predict_proba(all_voxel_features)
    
    # Reshape probabilities back to 3D grid
    # probabilities shape: (num_voxels, num_classes)
    prob_grid = probabilities.reshape(volume.shape[:-1] + (probabilities.shape[1],))

    # Step 5: Render using Probabilistic Volume Compositing
    # Define transfer function colors for each class
    tf_colors = get_tf_colors_from_user_or_default()
    
    # Initialize output image
    image = np.zeros((height, width, 3))
    
    # Front-to-back compositing
    for ray in rays:
        alpha_accum = 0.0
        color_accum = np.zeros(3)
        
        for sample in ray.samples:
            v = sample.voxel
            p = prob_grid[v] # Probability vector for this voxel
            
            # Calculate voxel color based on probabilities
            voxel_color = np.dot(p, tf_colors)
            
            # [G] Opacity alpha is not explicitly defined in the source formula.
            # Standard practice: alpha might be derived from max probability or a separate TF.
            # Assuming a simple alpha model for completeness:
            alpha_voxel = np.max(p) 
            
            # Standard front-to-back blending
            weight = alpha_voxel * (1.0 - alpha_accum)
            color_accum += weight * voxel_color
            alpha_accum += weight
            
            if alpha_accum > 0.99:
                break
                
        image[ray.y, ray.x] = color_accum

    return image
```

## Subtopics

### [Slice-guided Semiautomatic Multivariate Volume Exploration](9484-slice-guided-semiautomatic-multivariate-volume-exploration.md)
GuideME replaces trial-and-error tuning with a guided, uncertainty-aware region selection process on 2D slices. It uses edge detection and shortest-path algorithms for boundary-aware selection and direct response function optimization, differing from Image-based TF Design's Random Forest classification of scribbles.

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of labeled voxels and the size of the Random Forest. The source notes "fast training time even under sparse user supervision" [S1, unofficial]. Prediction is $O(N \cdot T \cdot \log M)$ where $N$ is voxels, $T$ is trees, $M$ is feature size.
*   **Memory Complexity:** Requires storing INR features for all voxels and the Random Forest model.
*   **Strengths:** Robust classification performance; handles sparse user input well; automates TF generation [S1, unofficial].
*   **Limitations:** Requires feature extraction (INR); probabilistic blending may blur boundaries if probabilities are uncertain; dependent on the quality of INR features.
*   **When to use:** When users can identify ROIs visually on slices but struggle to define precise 1D transfer functions.
*   **When not to use:** When real-time interaction with immediate feedback on the full volume is required without retraining the classifier.

## Background
Originated in the NeuVolEx system, which uses Implicit Neural Features for volume exploration [S1, unofficial]. The method leverages Random Forest classifiers to map user scribbles to probabilistic volume rendering, addressing the difficulty of manual transfer function design for complex volumetric data.

## Sources
[S1] NeuVolEx: Implicit Neural Features for Volume Exploration, https://doi.org/10.48550/arxiv.2604.11172, fetched 2026-10-26 (unofficial)
