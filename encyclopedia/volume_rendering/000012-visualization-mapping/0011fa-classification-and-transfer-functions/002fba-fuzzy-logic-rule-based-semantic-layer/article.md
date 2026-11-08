# Fuzzy-Logic Rule Based Semantic Layer

Also called: Antecedent Evaluation, Fuzzy Logic on GPU, GPU-based fuzzy logic volume rendering, Interaction-Dependent Semantic Volume Rendering, Interaction-Dependent Semantics for Illustrative Volume Rendering, Semantic layers for illustrative volume rendering, Semantics-driven rendering with interaction-dependent rules

## Overview
This method maps volumetric data attributes and interaction-dependent parameters (e.g., mouse position, view angle, slicing plane distance) to visual rendering styles using fuzzy logic rules evaluated per frame on the GPU [S1]. It solves the problem of creating expressive, interactive illustrative volume renderings where the visual style adapts dynamically to user interaction, which is computationally too expensive for CPU-based evaluation [S1]. Unlike static transfer functions, it decouples the definition of styles from the rendering pipeline by parsing linguistic rules into shader code, allowing for local, sample-dependent application of semantic attributes like "flatness" or specific color styles based on complex logical combinations of data and interaction features [S1].

## How it works
The method operates by parsing a fuzzy logic rule base on the CPU to generate GPU shader code and texture arrays, which are then used during volume ray casting to evaluate rendering attributes for each sample point [S1].

**Inputs and Outputs:**
*   **Inputs:**
    *   Fuzzy logic rule base: A set of linguistic rules defining mappings from semantic values to rendering styles [S1].
    *   Volumetric data: Scalar field values (e.g., density) [S1].
    *   Interaction parameters: Dynamic values such as distance to mouse cursor, distance to slicing plane, distance to image plane, and viewpoint [S1].
    *   Membership functions: Defined for semantic values (e.g., "low", "middle", "high") [S1].
*   **Outputs:**
    *   Rendered image with dynamic styles (color, opacity, flatness) applied per sample based on rule evaluation [S1].

**Data Structures:**
1.  **Fuzzy Logic Expression Tree:** Constructed on the CPU from the rule base. Nodes represent logical operations (`and`, `or`, `not`) or membership function evaluations [S1].
2.  **1D Texture Array:** Stores membership functions and pre-calculated fuzzy logic functions. Each membership function is encoded as a 1D texture [S1].
3.  **Shader Code:** Generated from the expression tree. Leaf nodes are replaced with texture lookups; internal nodes are replaced with GPU-compatible operations [S1].

**Numbered Steps:**
1.  **Parse Rule Base (CPU):** The CPU parses the fuzzy logic rules to build an expression tree containing `and`, `or`, and `not` operations [S1].
2.  **Substitute Operations (CPU):** Replace expression tree nodes with corresponding GPU operations:
    *   `and` $\rightarrow$ `min`
    *   `or` $\rightarrow$ `max`
    *   `not` $\rightarrow$ `1.0 - operand` [S1]
3.  **Encode Membership Functions (CPU):** Store membership functions as 1D textures and combine them into a 1D texture array [S1].
4.  **Generate Shader Code (CPU):** Substitute leaf nodes of the expression tree with texture lookups into the 1D texture array. Expand the tree into valid shader code [S1].
5.  **Upload to GPU:** Upload the generated shader code and the 1D texture array to the GPU [S1].
6.  **Execute Ray Casting (GPU):** For each sample point during volume ray casting, call the `GetColor()` function (or equivalent shader entry point) which evaluates the fuzzy logic rules [S1].
7.  **Evaluate Antecedents (GPU):** For each rule, evaluate the antecedent by looking up membership values for the current data and interaction parameters and combining them using the substituted operations [S1].
8.  **Implication (GPU):** Apply the antecedent value $a$ to the consequent membership function $m(x)$ using truncation: $m'(x) = \min(m(x), a)$ [S1].
9.  **Aggregation and Defuzzification (GPU):** Aggregate the resulting membership functions for each style and defuzzify (e.g., using the centroid method) to derive crisp values for rendering attributes [S1].

**Formulas:**
*   **AND operation:** $\text{result} = \min(\text{operand}_1, \text{operand}_2)$ [S1]
*   **OR operation:** $\text{result} = \max(\text{operand}_1, \text{operand}_2)$ [S1]
*   **NOT operation:** $\text{result} = 1.0 - \text{operand}$ [S1]
*   **Implication:** Given antecedent value $a \in [0, 1]$ and membership function $m(x)$, the implication is $m'(x) = \min(m(x), a)$ [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU-side: Rule Parsing and Shader Generation
def parse_rules(rule_base):
    """
    Parses fuzzy logic rules into an expression tree.
    Returns a tree structure where nodes are operations or leaves.
    """
    tree = build_expression_tree(rule_base)
    return tree

def generate_shader_code(tree, membership_texture_array):
    """
    Translates the expression tree into GPU shader code.
    Replaces logical operators with min/max/1.0-x and leaves with texture lookups.
    """
    shader_code = ""
    for node in tree:
        if node.type == 'AND':
            shader_code += f"min({generate_shader_code(node.left)}, {generate_shader_code(node.right)})"
        elif node.type == 'OR':
            shader_code += f"max({generate_shader_code(node.left)}, {generate_shader_code(node.right)})"
        elif node.type == 'NOT':
            shader_code += f"1.0 - {generate_shader_code(node.child)}"
        elif node.type == 'LEAF':
            # Lookup membership function from 1D texture array
            # index depends on the specific semantic attribute
            shader_code += f"texture1DArray(membership_array, {node.index}, {node.value})"
    return shader_code

# GPU-side: Shader Execution (Simplified GLSL-like pseudocode)
uniform sampler1DArray membership_array;
uniform vec3 interaction_params; // e.g., distance_to_cursor, distance_to_plane, etc.
uniform vec3 data_value; // e.g., density

vec4 GetColor(vec3 position, vec3 direction) {
    // 1. Sample data and interaction parameters
    float density = sample_volume(position);
    float dist_cursor = distance(position, mouse_world_pos);
    float dist_plane = distance(position, slicing_plane);
    
    // 2. Evaluate membership functions for antecedents
    // Example: "distance to cursor is low"
    float m_low_cursor = texture1DArray(membership_array, 0, dist_cursor);
    // Example: "density is high"
    float m_high_density = texture1DArray(membership_array, 1, density);
    
    // 3. Evaluate antecedent using fuzzy logic operations
    // Rule: if (dist_cursor is low AND density is high) OR (dist_plane is low)
    float m_low_plane = texture1DArray(membership_array, 2, dist_plane);
    float antecedent = max(min(m_low_cursor, m_high_density), m_low_plane);
    
    // 4. Implication: truncate consequent membership functions
    // Example consequent: skin_style is transparent
    float m_transparent_skin = texture1DArray(membership_array, 3, 0.5); // Pre-defined style MF
    float m_transparent_skin_implicated = min(m_transparent_skin, antecedent);
    
    // 5. Aggregation and Defuzzification (simplified)
    // In practice, multiple rules contribute to multiple styles.
    // This example shows a single rule contributing to one style attribute.
    float style_value = defuzzify_centroid(m_transparent_skin_implicated);
    
    // 6. Apply style to color/opacity
    vec4 color = mix(vec3(0.0, 0.0, 0.0), vec3(1.0, 1.0, 1.0), style_value);
    float opacity = style_value;
    
    return vec4(color, opacity);
}

void main() {
    vec4 vecRayColor = vec4(0.0, 0.0, 0.0, 0.0);
    vec4 vecPosition = vec4(0.0, 0.0, 0.0, 0.0);
    
    // Volume Ray Casting Loop
    do {
        vec4 vecSampleColor = GetColor(vecPosition);
        if (vecSampleColor.a > 0.0) {
            // Standard volume compositing
            vecRayColor = vecSampleColor + vecRayColor * (1.0 - vecSampleColor.a);
        }
        vecPosition += vecDirection * sample_distance;
    } while (vecRayColor.a < 1.0 && vecPosition inside_volume);
    
    gl_FragColor = vecRayColor;
}
```

**Parameters:**
*   **Interaction-dependent parameters:** Distance to mouse cursor, distance to slicing plane, distance to image plane, viewpoint, user focus [S1].
*   **Flatness parameter:** A semantic parameter describing the trade-off between showing spatial relationships and showing important regions. Evaluated locally per sample [S1].
*   **Membership function resolution:** Determined by the 1D texture array size. Changes result in interactive updates [S1].
*   **Sample distance:** Controls rendering quality and performance. Typical values in examples: 1.0 [S1].

**Edge Cases and Pitfalls:**
*   **Performance:** Evaluating fuzzy logic rules for a 3D volume several times per second is computationally expensive on CPUs, requiring GPU implementation [S1].
*   **Shader Update Cost:** Regenerating shader code when rules change takes less than one second on a modern desktop PC but is the most costly operation in the system [S1].
*   **No Pre-segmentation:** The method relies on density and interaction-dependent rules rather than pre-segmentation, which may limit precision in complex anatomical boundaries compared to segmentation-based methods [S1].

## Subtopics

### [Fuzzy Set-Based Volume Selection and Intersection](6430-fuzzy-set-based-volume-selection-and-intersection.md)
Defines Selection, Ghost, and Background objects using fuzzy set arithmetic for interactive multi-object rendering.

### [Artistic Volume Rendering Template](7040-artistic-volume-rendering-template.md)
Maps volumetric attributes to artistic styles using fuzzy-logic rules, decoupling color evaluation into prioritized styles.

### [Semantics by Analogy](7439-semantics-by-analogy.md)
Specifies semantic mappings through brushing on renderings of properties of interest in an intermediate domain.

## Operating characteristics
*   **Time Complexity:** The per-frame cost is dominated by the GPU evaluation of fuzzy logic rules per sample. The CPU cost is incurred only when rules change (<1 second) [S1].
*   **Memory Complexity:** Requires storage for membership functions in 1D texture arrays and the generated shader code [S1].
*   **Measured Performance:**
    *   Visible Human dataset ($256^3$), viewport $512^2$, sample distance 1.0: 23 fps [S1].
    *   Human body dataset ($256^2 \times 415$), viewport $512^2$, sample distance 1.0: 20 fps [S1].
    *   CT-scan leg dataset ($147 \times 162 \times 429$), viewport $512^2$, sample distance 1.0: 6 fps [S1].
    *   Hardware: GeForce 8800 GTX [S1].
*   **Strengths:** Enables interactive, expression-dependent illustrations; allows local application of semantic attributes (e.g., flatness); decouples style definition from rendering code.
*   **Limitations:** Requires GPU implementation for interactivity; shader regeneration is needed for rule changes; performance scales with volume resolution and rule complexity.
*   **When to use:** Interactive illustrative volume rendering where user interaction (mouse, view, slicing) should dynamically alter visual styles.
*   **When not to use:** Static visualization where performance is critical and interaction is not required; scenes requiring pre-segmentation for accuracy.

## Background
The method was introduced by Peter Rautek, Stefan Bruckner, and M. Eduard Gröller in the paper "Interaction-Dependent Semantics for Illustrative Volume Rendering" published in the Eurographics/IEEE-VGTC Symposium on Visualization 2008 [S1]. It extends earlier work by Bruckner and Gröller on style transfer functions [S1]. The work was part of the ex vis ation project supported by the Austrian Science Fund (FWF) grant no. P18322 [S1]. It addresses the difficulty of specifying mappings between 3D data and visual representation that preserve user intention in illustrative volume visualization [S1].

## Sources
[S1] Interaction‐Dependent Semantics for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2008.01216.x, fetched 2026-11-03
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
