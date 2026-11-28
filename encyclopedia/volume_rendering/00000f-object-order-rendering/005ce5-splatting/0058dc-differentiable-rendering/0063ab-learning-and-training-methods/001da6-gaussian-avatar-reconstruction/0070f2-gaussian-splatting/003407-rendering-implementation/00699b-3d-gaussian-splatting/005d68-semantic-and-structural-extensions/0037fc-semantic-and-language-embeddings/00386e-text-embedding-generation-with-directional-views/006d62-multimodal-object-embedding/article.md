# Multimodal Object Embedding

Also called: LLM-Enhanced Fusion, Multi-modal embedding fusion

## Overview
Multimodal Object Embedding creates a unified semantic representation for 3D objects by fusing vision-based CLIP embeddings with optional text-based CLIP embeddings [S1][S2, unofficial]. It solves the problem of limited semantic grounding in open-vocabulary querying by capturing both visual features and high-level attribute descriptions in a single vector space [S1]. Unlike its sibling [Text Embedding Generation with Directional Views](8239-text-embedding-generation-with-directional-views.md), which generates directional view embeddings by appending view descriptors to prompts, this method focuses on instance-level fusion of static object attributes [S1][S2, unofficial]. It differs from [TexGS-VolVis Representation and Optimization](11105-texgs-volvis-representation-and-optimization.md) and [View-Dependent Prompting](19229-view-dependent-prompting.md) by explicitly combining a language-free ontology with an LLM-generated attribute channel [S2, unofficial].

## How it works
**Inputs:**
1.  A set of isolated image crops (frames) of a 3D object instance [S1][S2, unofficial].
2.  Optional textual labels or descriptions associated with the object [S1].
3.  A Large Language Model (LLM) for generating attribute descriptions [S2, unofficial].
4.  A frozen CLIP text encoder [S1][S2, unofficial].
5.  A fusion weight parameter $\lambda \in [0, 1]$ [S2, unofficial].

**Outputs:**
*   A final multimodal object embedding $E_{object}$ (or $e_j^{fused}$) [S1][S2, unofficial].

**Data Structures:**
*   $E_{vision}$: Vision-based embedding vector.
*   $E_{text}$: Text-based embedding vector.
*   $e_j^{txt}$: CLIP-encoded description for instance $j$.
*   $e_j$: Vision embedding for instance $j$ (used in [S2, unofficial] notation).

**Steps:**

1.  **Compute Vision Embedding ($E_{vision}$):**
    *   If using the method from [S1]: Average the CLIP embeddings of $k$ selected frames.
        $$ E_{vision} = \frac{1}{k} \sum_{j=1}^{k} E_{frame_j} $$
        where $E_{frame_j}$ is the CLIP embedding of the $j$-th selected frame [S1].
    *   If using the method from [S2, unofficial]: Use the instance-specific vision embedding $e_j$ derived from the same isolated crops [S2, unofficial].

2.  **Generate and Encode Text Embedding ($E_{text}$):**
    *   **If textual labels/descriptions are available (from [S1]):**
        *   Use the provided textual description directly.
        *   Encode it using the CLIP text encoder to obtain $E_{text}$ [S1].
    *   **If textual labels are not available but LLM enhancement is used (from [S2, unofficial]):**
        1.  Pass the isolated crops to an LLM with a fixed prompt [S2, unofficial].
        2.  Obtain the attribute-rich text description $d_j$ [S2, unofficial].
        3.  Encode $d_j$ using the CLIP text encoder to obtain $e_j^{txt}$ [S2, unofficial].
        4.  Set $E_{text} = e_j^{txt}$ [S2, unofficial].

3.  **Fuse Embeddings:**
    *   **Case A: No textual information available:**
        $$ E_{object} = E_{vision} $$
        [S1]
    *   **Case B: Textual information available (Simple Average from [S1]):**
        $$ E_{object} = \frac{E_{vision} + E_{text}}{2} $$
        [S1]
    *   **Case C: LLM-Enhanced Fusion (Weighted from [S2, unofficial]):**
        $$ e_j^{fused} = \lambda e_j + (1 - \lambda) e_j^{txt} $$
        where $\lambda \in [0, 1]$ is a hyperparameter [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multimodal_embedding(crops, textual_labels=None, llm=None, clip_text_encoder=None, lambda_weight=0.5):
    """
    Computes the multimodal object embedding.
    
    Args:
        crops: List of image crops (tensors) of the object.
        textual_labels: Optional list of text strings describing the object.
        llm: Optional LLM instance for generating descriptions.
        clip_text_encoder: Frozen CLIP text encoder.
        lambda_weight: Weight for vision vs text in fusion (lambda in [S2, unofficial]).
        
    Returns:
        E_object: The final fused embedding vector.
    """
    
    # Step 1: Compute Vision Embedding
    # Assuming a helper function to get CLIP visual embedding for a crop
    vision_embeddings = [get_clip_visual_embedding(crop) for crop in crops]
    
    # Method from [S1]: Average frame embeddings
    # Method from [S2, unofficial]: Uses instance embedding e_j (assumed pre-computed or same as average)
    # Here we follow [S1] averaging for E_vision, or [S2, unofficial] e_j if provided.
    # For consistency with [S1] primary definition:
    E_vision = sum(vision_embeddings) / len(vision_embeddings)
    
    # Step 2: Compute Text Embedding
    E_text = None
    
    if textual_labels is not None and len(textual_labels) > 0:
        # Use provided labels [S1]
        # Assuming textual_labels is a single string or list to be encoded
        text_input = textual_labels[0] if isinstance(textual_labels, list) else textual_labels
        E_text = clip_text_encoder.encode(text_input)
        
    elif llm is not None and clip_text_encoder is not None:
        # LLM-Enhanced Fusion [S2, unofficial]
        # 1. Pass crops to LLM with fixed prompt
        description = llm.generate_description(crops, prompt="fixed_prompt")
        # 2. Encode description
        E_text = clip_text_encoder.encode(description)
        
    # Step 3: Fuse
    if E_text is None:
        # No textual information [S1]
        E_object = E_vision
    else:
        # Check if using [S1] simple average or [S2, unofficial] weighted fusion
        # [S1] uses (E_vision + E_text) / 2
        # [S2, unofficial] uses lambda * e_j + (1-lambda) * e_txt_j
        # We implement [S2, unofficial] weighted fusion as it is more general, 
        # but note [S1] sets lambda=0.5 implicitly if E_vision and E_text are normalized similarly.
        
        # Note: [S1] and [S2, unofficial] may have different normalization assumptions.
        # [S1] explicitly divides by 2. [S2, unofficial] uses lambda.
        # To be safe and cover both, we check if lambda is explicitly meant for [S2, unofficial] context.
        # Given the alias "LLM-Enhanced Fusion" is tied to [S2, unofficial], we prioritize [S2, unofficial] formula.
        
        E_object = lambda_weight * E_vision + (1 - lambda_weight) * E_text
        
    return E_object
```

**Parameters:**
*   $\lambda$: Fusion weight. Typical value not given in [S1]. [S2, unofficial] specifies $\lambda \in [0, 1]$.
*   $k$: Number of selected frames for averaging. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Cosine similarity computation for querying requires normalized vectors. Ensure $E_{object}$ and $E_{query}$ are normalized before computing $S(E_{query}, E_{object})$ [S1].
*   **Boundary:** If no crops are available, the method cannot compute $E_{vision}$ [S1][S2, unofficial].
*   **Degenerate Input:** If the LLM fails to generate a description, fallback to $E_{vision}$ [S2, unofficial].

## Subtopics

### Text Embedding Generation with Directional Views
Encodes text prompts into latent embeddings using a frozen CLIP text encoder, and generates directional view embeddings by appending view descriptors to prompts. Choose when view-dependent semantic cues are needed. [Link](8239-text-embedding-generation-with-directional-views.md)

### TexGS-VolVis Representation and Optimization
Uses a volume-based representation for optimization. Choose when volumetric consistency is prioritized over splatting efficiency. [Link](11105-texgs-volvis-representation-and-optimization.md)

### View-Dependent Prompting
Uses view-dependent prompts for segmentation. Choose when appearance changes significantly across views. [Link](19229-view-dependent-prompting.md)

## Operating characteristics
*   **Time Complexity:** $O(k)$ for averaging $k$ frame embeddings [S1]. $O(1)$ for LLM inference per instance (assuming fixed prompt) and CLIP encoding [S2, unofficial]. Retrieval is $O(|I_{stable}|)$ cosine computations per prompt [S2, unofficial].
*   **Memory Complexity:** Stores one embedding vector per object instance [S1][S2, unofficial].
*   **Strengths:** Enables accurate and efficient open-vocabulary querying by capturing visual and semantic cues in a single vector space [S1]. No second voting pass or re-optimization is required for LLM-enhanced fusion [S2, unofficial].
*   **Limitations:** Relies on the quality of the LLM-generated descriptions [S2, unofficial]. Requires isolated crops for LLM input [S2, unofficial].
*   **When to use:** When open-vocabulary querying of 3D objects is required and textual attributes are beneficial [S1][S2, unofficial].
*   **When not to use:** When computational resources for LLM inference are unavailable or when only visual cues are sufficient [S1].

## Background
The method originated from research into natural language interaction for volume visualization and open-vocabulary 3D segmentation [S1][S2, unofficial]. [S1] introduced the basic fusion of vision and text CLIP embeddings for NLI4VolVis. [S2, unofficial] extended this with LLM-enhanced fusion for DIGS, demonstrating gains in retrieval accuracy without re-optimization [S2, unofficial].

## Sources
[S1] NLI4VolVis: Natural Language Interaction for Volume Visualization via LLM Multi-Agents and Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3633888, fetched 2026-10-26
[S2] DIGS: Distribution-Informed Gaussian Splatting for Training-Free Open-Vocabulary 3D Segmentation, https://doi.org/10.48550/arxiv.2610.16177, fetched 2026-10-26 (unofficial)
