# Multi-Identity Embedding


## Overview
Multi-Identity Embedding is a method for identifying contributing users in collaborative Neural Radiance Field (NeRF) training by embedding customized hidden information about specific clusters of training views [S1]. It solves the problem of scalable attribution in multi-user scenarios by allowing a single NeRF model to hide distinct identities for different spatial regions without requiring model retraining for each new watermark [S1]. Unlike prior approaches that embed only a single global watermark, this method isolates identity information to specific clusters of anchor views, maintaining high visual quality by preventing false positives for unmarked identities [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A collection of user-uploaded training images (simulated via the NeRF-W dataset) [S1].
*   **Output:** A NeRF model with an embedded classifier network capable of recovering specific contributor identities based on the rendering viewpoint [S1].

**Data Structures:**
*   **Classifier Network ($F_{\phi_c}$):** A neural network component modified from the standard StegaNeRF backbone [S1]. It outputs a probability distribution over $M+1$ classes: $M$ classes corresponding to specific contributor clusters, and one additional class for "unmarked" identities [S1].

**Steps:**
1.  **Anchor Selection:** Select $M$ anchor views with distinct viewing positions from the training set [S1].
2.  **Cluster Formation:** For each anchor view, find the $K$ nearest neighbor views based on viewing position to form a cluster [S1]. Each cluster is assumed to have a common contributor identity [S1].
3.  **Network Modification:** Modify the classifier network $F_{\phi_c}$ to output $M$-class cluster predictions plus one class for identities outside the $M$ clusters [S1].
4.  **Loss Computation for Unmarked Views:** For poses outside the $M$ clusters, compute the decoding loss $L^{-dec}$ and the classification loss $L_{dec}$ to ensure no information is recovered for unmarked identities, thereby preventing false positives [S1].

**Parameters:**
*   $M$: Number of anchor views/clusters. Typical value: 3 [S1].
*   $K$: Number of nearest neighbours per anchor. Typical value: 20 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_identity_embedding(training_images, M=3, K=20):
    """
    Embeds M distinct contributor identities into a NeRF model.
    
    Args:
        training_images: List of training images with camera poses.
        M: Number of anchor views/clusters.
        K: Number of nearest neighbours per anchor.
        
    Returns:
        model: Trained NeRF model with embedded identity classifier.
    """
    # Step 1: Select M anchor views with different viewing positions
    anchors = select_distinct_views(training_images, count=M)
    
    # Step 2: Form clusters by finding K nearest neighbours for each anchor
    clusters = []
    for anchor in anchors:
        neighbors = find_k_nearest_neighbors(training_images, anchor, k=K)
        clusters.append(neighbors)
        
    # Step 3: Modify classifier network F_phi_c to output M+1 classes
    # M classes for clusters, 1 class for unmarked identities
    classifier = build_classifier(num_output_classes=M + 1)
    
    # Step 4: Train NeRF with StegaNeRF objectives
    # For poses outside the M clusters, enforce no information recovery
    for batch in training_batches(training_images):
        # Standard StegaNeRF rendering and encoding steps occur here
        # [G] Standard NeRF volume rendering and StegaNeRF encoding logic
        
        # Compute losses for unmarked views to prevent false positives
        unmarked_poses = get_unmarked_poses(batch.poses, clusters)
        if unmarked_poses:
            # L_dec ensures the classifier predicts the 'unmarked' class correctly
            loss_dec = compute_classification_loss(classifier, unmarked_poses, target_class=M)
            # L^-dec ensures no hidden information is recoverable for unmarked views
            loss_minus_dec = compute_decoding_loss(classifier, unmarked_poses)
            
            total_loss = total_loss + loss_dec + loss_minus_dec
            
        # Standard StegaNeRF losses for marked views would also be computed here
        # [G] Standard StegaNeRF L_dec and L^-dec for marked clusters
        
    return train_model(training_images, classifier, total_loss)
```

## Subtopics

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method was evaluated on the NeRF-W dataset using a network backbone similar to NeRFW [S1]. Qualitative results show successful recovery of User A, User B, and User C identities within their respective clusters, with low recovery scores for unmarked identities (e.g., 0.2368 for User A's identity in an unmarked region) [S1].
*   **Strengths:** Allows scalable, robust digital watermarking in 3D scenes by accommodating multiple distinct messages as an additional model input [S1]. Isolates watermark features to prevent entanglement with scene appearance [S1].
*   **Limitations:** Requires careful selection of anchor views and nearest neighbours to ensure cluster separation [S1]. Performance depends on the quality of the NeRF-W dataset simulation [S1].
*   **When to use:** When identifying contributing users in collaborative NeRF projects is required [S1].
*   **When not to use:** When only a single global watermark is sufficient [S1].

## Background
Multi-Identity Embedding was introduced in the paper "StegaNeRF: Embedding Invisible Information within Neural Radiance Fields" [S1]. It addresses the scenario where a collection of user-uploaded training images is used to generate a NeRF, and it is meaningful to identify the contributing users in the rendered output [S1]. The method extends the StegaNeRF framework to handle multiple identities simultaneously [S1].

## Sources
[S1] StegaNeRF: Embedding Invisible Information within Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00047, fetched 2026-10-26
