# Interest-Based NPR Toolkit

Also called: Non-Photorealistic Rendering (NPR) Toolkit, Variable-Detail NPR

## Overview
The Interest-Based NPR Toolkit is a flexible framework within the Volume Illustration pipeline that generates 2D non-photorealistic illustrations by associating user-defined "interest labels" with subsets of rendering techniques [S1, unofficial]. It solves the problem of programmatic bias in automated visualization by allowing variable detail levels per segment, where different labels trigger different artistic styles simultaneously [S1, unofficial]. Unlike physics-based rendering, it prioritizes communicative abstraction and artistic vision over physical accuracy [S1, unofficial].

## How it works
The method operates on an enriched model consisting of a geometric model (identified segments) and a structural model (associated interest labels) [S1, unofficial].

**Inputs:**
1.  **Geometric Model:** A set of segments derived from segmentation (e.g., seeded region growing). Each segment is a collection of voxels [S1, unofficial].
2.  **Structural Model:** A mapping from segment IDs to interest labels [S1, unofficial].
3.  **NPR Technique Library:** A set of implemented techniques (greyscale shading, simple outlines, copper plate lines, short hatching lines) [S1, unofficial].

**Outputs:**
A single 2D illustration image where each segment is rendered according to the NPR techniques associated with its interest label [S1, unofficial].

**Data Structures:**
*   **Segment Map:** A volume or list mapping each voxel to a segment ID.
*   **Label Map:** A mapping `SegmentID -> InterestLabel` (integer).
*   **Technique Map:** A mapping `InterestLabel -> Subset{NPR_Technique}`.

**Steps:**
1.  **Segmentation:** Identify segments in the volume data. The system assumes all voxels in a segment are equally interesting (Single Interest Volume assumption) [S1, unofficial].
2.  **Enrichment:** Assign an interest label to each segment. This forms the structural model [S1, unofficial].
3.  **Association:** The user defines a one-to-many mapping: each interest label is associated with a subset of available NPR techniques [S1, unofficial].
4.  **Rendering:** Render all NPR techniques associated with all interest labels simultaneously to generate the final image [S1, unofficial].

**Parameters:**
*   **Number of Interest Labels:** Default is 5. This is an arbitrary decision; the system generalizes to any number [S1, unofficial].
*   **Default Label:** All segments default to label 3 if not explicitly assigned [S1, unofficial].
*   **Technique Properties:** Techniques can be customized to portray silhouettes, surface geometry, value, light/shade, and primitive texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Define available NPR techniques
TECHNIQUES = [
    "greyscale_shading",
    "simple_outlines",
    "copper_plate_lines",
    "short_hatching_lines"
]

# User-defined mappings
# label_to_techniques: InterestLabel -> List[Technique]
# segment_to_label: SegmentID -> InterestLabel

def render_interest_based_npr(geometric_model, segment_to_label, label_to_techniques):
    """
    geometric_model: List of Segments, where each Segment is a list of Voxels
    segment_to_label: Dict mapping SegmentID to InterestLabel (int)
    label_to_techniques: Dict mapping InterestLabel to List of Technique Strings
    """
    # Initialize output image buffer
    output_image = create_blank_image()
    
    # Process each segment
    for segment in geometric_model:
        segment_id = segment.id
        interest_label = segment_to_label[segment_id]
        
        # Get techniques for this label
        techniques = label_to_techniques.get(interest_label, [])
        
        # Render techniques for this segment
        for technique in techniques:
            apply_npr_technique(
                segment.voxels, 
                technique, 
                output_image
            )
            
    return output_image

def apply_npr_technique(voxels, technique_name, image_buffer):
    """
    Applies a specific NPR technique to the geometry defined by voxels.
    Implementation details for specific techniques (e.g., edge detection for outlines)
    are handled by the underlying NPR engine.
    """
    if technique_name == "greyscale_shading":
        render_shading(voxels, image_buffer)
    elif technique_name == "simple_outlines":
        render_outlines(voxels, image_buffer)
    elif technique_name == "copper_plate_lines":
        render_copper_plate(voxels, image_buffer)
    elif technique_name == "short_hatching_lines":
        render_hatching(voxels, image_buffer)
    # [G] Other techniques would be added here based on the extensible design
```

## Subtopics

### Interest label assignment
A generalization of binary segmentation where the level of interest in each segment is quantified by assigning an interest label [S1, unofficial]. The relationship between segments and labels is many-to-one [S1, unofficial]. The user assigns one of five labels (default 3) to each segment in a synchronized structural view [S1, unofficial]. This allows variable detail levels without enforcing a specific hierarchy (e.g., 1=low, 5=high) [S1, unofficial].

## Operating characteristics
*   **Complexity:** Rendering speed must remain high for interactivity, utilizing a fast, pixel-based, geometric buffer approach [S1, unofficial].
*   **Memory:** Supports single-level undo; multiple-level undo is noted as a future improvement requiring extra memory/computation [S1, unofficial].
*   **Strengths:** High flexibility; avoids programmatic biases; allows creative artistic control [S1, unofficial].
*   **Limitations:** Requires user input for label assignment and technique mapping; not fully automatic [S1, unofficial].

## Background
Originated in the thesis "Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration" by J.L. McKinley (2002) [S1, unofficial]. It builds on the concept of enriched models (geometric + structural information) [S1, unofficial]. It follows the lead of Ebert and Rheingans [ER00] in providing a flexible NPR toolkit [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
