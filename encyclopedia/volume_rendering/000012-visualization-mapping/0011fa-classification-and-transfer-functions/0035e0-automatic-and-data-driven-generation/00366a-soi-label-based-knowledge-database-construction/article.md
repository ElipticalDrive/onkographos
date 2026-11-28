# SOI Label-Based Knowledge Database Construction

Also called: automated TIQ database generation

## Overview
This method automates the creation of a retrieval-based knowledge database for volume rendering transfer functions (TFs) by using Segment of Interest (SOI) labels from raw image volumes instead of expert-designed TFs [S1, unofficial]. It solves the problem of high labor cost and time required for manual TF design by enabling rapid, data-driven TF generation through image retrieval [S1, unofficial]. It differs from expert TF collection by relying on SOI labels, which requires significantly less effort and time to generate [S1, unofficial].

## How it works
The method constructs a database of "Two-Image Query" (TIQ) representations linked to SOI labels, which are later used to generate transfer functions.

**Inputs and Outputs:**
*   **Input:** Raw image volumes (e.g., medical CT/MRI) with associated SOI labels.
*   **Output:** A knowledge database storing TIQ representations and their corresponding SOI labels.

**Data Structures:**
*   **TIQ Representation:** A feature vector or set of features derived from rays extracted from the volume. The specific feature extraction algorithm (e.g., CNN backbone) is not fixed and can be adapted [S1, unofficial].
*   **Knowledge Database:** A storage structure mapping TIQ features to SOI labels. The text mentions hierarchical storage (e.g., using a set cover problem) for rapid retrieval [S1, unofficial], but does not specify the exact data structure layout (e.g., hash map, tree, vector database).

**Steps:**
1.  **Ray Extraction:** Automatically extract rays from the raw image volumes [S1, unofficial].
2.  **TIQ Generation:** Generate TIQ representations from the extracted rays. This involves extracting features that describe the intensity profiles or image content of the rays [S1, unofficial].
3.  **Database Storage:** Store the generated TIQ representations and their associated SOI labels in the knowledge database [S1, unofficial].

**Parameters:**
*   **CNN Backbone:** The specific neural network used for feature extraction is not constrained; recent advances in segmentation or classification CNNs can be leveraged [S1, unofficial].
*   **Dataset Specificity:** The construction procedure does not require dataset-specific parameter settings [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **False Negatives:** The retrieval process may result in false negatives (missing relevant SOIs), though the approach remains effective as users can easily add new SOIs to an existing TF [S1, unofficial].
*   **Modality Limitation:** The initial demonstration was limited to medical imaging data, though the approach is suggested for non-medical datasets [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_knowledge_database(volumes, sois):
    """
    Constructs a knowledge database from raw volumes and SOI labels.
    
    Args:
        volumes: List of raw image volume data structures.
        sois: List of SOI labels corresponding to the volumes.
        
    Returns:
        database: A structure mapping TIQ features to SOI labels.
    """
    database = {} # Placeholder for the specific storage structure (e.g., hash map, tree)
    
    for volume, sois_for_volume in zip(volumes, sois):
        # Step 1: Automatically extract rays from raw volumes
        rays = extract_rays(volume)
        
        for ray in rays:
            # Step 2: Generate TIQ representations from the rays
            # The specific feature extraction method (e.g., CNN) is not defined here
            # and depends on the chosen backbone [S1, unofficial].
            tiq_representation = extract_tiq_features(ray)
            
            # Step 3: Store TIQs and associated SOI labels in the knowledge database
            # Mapping logic (e.g., hierarchical set cover) is not specified in detail [S1, unofficial]
            store_in_database(database, tiq_representation, sois_for_volume)
            
    return database
```

## Subtopics

### SOI-Based TF Generation
This variant generates a conventional intensity-based 1D transfer function for Segments of Interest (SOIs) using the SOI labels retrieved from the knowledge database [S2]. It creates tent-shaped components for each SOI to visualize multiple SOIs in a semi-transparent manner [S2].

*   **Definition:** Generates a 1D TF where each SOI $S_m$ (for $m=1, \dots, M$) is represented by a tent shape $T_m$ in the intensity-opacity space [S2].
*   **Formula:** The tent shape $T_j$ is defined by three points:
    $$ T_j = \{(I_{l,m}, 0), (I_m, \sigma_m), (I_{h,m}, 0)\} $$
    where:
    *   $I_{l,m}$ is the lowest intensity value in SOI $S_m$.
    *   $I_{h,m}$ is the highest intensity value in SOI $S_m$.
    *   $I_m$ is the average intensity value in SOI $S_m$.
    *   $\sigma_m$ is the initial opacity of the tent peak, set to 0.3 (where 1.0 indicates full opacity) [S2].
*   **Steps:**
    1.  Retrieve SOI labels from the knowledge database.
    2.  For each SOI $S_m$, calculate $I_{l,m}$, $I_{h,m}$, and $I_m$ from the retrieved data.
    3.  Construct the tent shape $T_m$ using the formula above.
    4.  The initial TF is the union of all tent shapes: $\bigcup_{m=1}^{M} T_m$ [S2].
    5.  Assign unique colors to each tent apex using ColorBrewer [S2].
    6.  Assign black color to all tent end points to improve visual color depth perception among SOIs [S2].
*   **Parameters:**
    *   Initial opacity $\sigma_m$: 0.3 [S2].
*   **When to choose:** Use this when a semi-transparent visualization of multiple SOIs is desired, and the intensity ranges of the SOIs are known or can be estimated from the retrieved data [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The retrieval speed depends on the database structure (e.g., hierarchical set cover for rapid but approximated retrieval) [S1, unofficial].
*   **Memory Complexity:** Not given in the sources. Depends on the size of the TIQ representations and the number of volumes in the database.
*   **Strengths:** Requires less effort and time compared to expert TF collection; does not require dataset-specific parameter settings for database construction; adaptable to different CNN backbones; allows easy addition of new SOIs to existing TFs [S1, unofficial].
*   **Limitations:** Potential for false negatives in retrieval; initial implementation limited to medical imaging data (though extensible); the specific feature extraction method is not standardized [S1, unofficial].
*   **When to use:** When rapid TF generation is needed for large datasets where manual design is impractical, and SOI labels are available or can be generated.
*   **When not to use:** When high precision in TF design is critical and expert knowledge is available, or when SOI labels are not available.

## Background
The method was introduced by Y. Jung, J. Kong, and J. Kim. It builds on the concept of Case-Based Reasoning (CBR) for transfer function design, where existing approaches collected pre-designed TFs by domain experts [S1, unofficial]. This approach automates the construction procedure, including ray extraction, representation, and storage, and was demonstrated on medical imaging data (e.g., 3D-IRCADb-01 dataset) [S1, unofficial][S2].

## Sources
[S1] A Transfer Function Design Using A Knowledge Database based on Deep Image and Primitive Intensity Profile Features Retrieval, https://arxiv.org/abs/2209.06421v1, fetched 2026-10-26 (unofficial)
[S2] A Transfer Function Design for Medical Volume Data Using a Knowledge Database Based on Deep Image and Primitive Intensity Profile Features Retrieval, https://doi.org/10.1007/s11390-024-3419-7, fetched 2026-10-27
