# Structural Focus+Context Rendering

Also called: SFC rendering, structural F+C volume rendering

## Overview
Structural Focus+Context (SFC) rendering is a volume visualization technique for multiclassified data that explicitly models and renders the logical relationships between classification criteria using a directed graph [S1, unofficial]. Unlike standard volume rendering which treats classes independently, SFC constructs an auxiliary labelled voxel model and a structural graph to compute transfer functions based on boolean queries over class inclusions [S1, unofficial]. It sets itself apart from sibling methods like [Hybrid Rendering for Stress Tensor Fields](8909-hybrid-rendering-for-stress-tensor-fields.md) by focusing on the semantic structure of multiclassified scalar volumes rather than tensor field geometry, using multimodal 3D texture mapping to simultaneously render context and focus based on cluster labels [S1, unofficial].

## How it works
**Inputs:**
1.  **Original Voxel Model ($V$):** A 3D scalar field (potentially multimodal, e.g., MR/SPECT) [S1, unofficial].
2.  **Classification Criteria:** A set of independent boolean or scalar criteria used to classify voxels [S1, unofficial].
3.  **User Query:** A boolean expression defined over the initial classification criteria [S1, unofficial].

**Outputs:**
1.  **Directed Graph ($G$):** Nodes represent classification classes; edges represent logical inclusion relationships [S1, unofficial].
2.  **Labelled Voxel Model ($LV$):** A 3D volume where each voxel stores a unique integer identifier (cluster label) corresponding to the combination of classification criteria it satisfies [S1, unofficial].
3.  **Transfer Functions:**
    *   **CTF (Color/Visibility Transfer Function):** Maps cluster labels to visibility, color, and opacity [S1, unofficial].
    *   **GMTF (Gradient Modulation Transfer Function):** Maps cluster labels to a gradient modulation factor [S1, unofficial].

**Data Structures:**
*   **Directed Graph Model:** A graph $G = (N, E)$ where $N$ is the set of classification criteria (nodes) and $E$ is the set of logical implications (edges) [S1, unofficial].
*   **Auxiliary Voxel Model ($LV$):** A 3D array of integers. For a voxel at position $\mathbf{x}$, $LV[\mathbf{x}] = c$, where $c$ is the unique identifier of the cluster (combination of criteria) satisfied by $V[\mathbf{x}]$ [S1, unofficial].

**Preprocessing Steps:**
1.  **Classification:** Classify the initial voxel model $V$ according to various independent criteria. This step handles multimodal datasets if present [S1, unofficial].
2.  **Graph Construction:**
    *   Identify dependencies between classification criteria.
    *   Construct a directed graph where nodes are the classification criteria and edges represent logical inclusion relationships (implications) between them [S1, unofficial].
3.  **Clustering:**
    *   Clusterize the voxel model into subsets of voxels sharing the exact same combination of classification criteria [S1, unofficial].
    *   Construct the auxiliary voxel model $LV$. For each voxel $\mathbf{x}$, assign a unique integer identifier $c$ representing its associated cluster [S1, unofficial].
4.  **Graph Completion:** Identify dependencies between the constructed clusters to complete the graph structure, ensuring the graph accurately reflects the logical structure of the multiclassified model [S1, unofficial].
5.  **Output:** Store the directed graph $G$ and the labelled voxel model $LV$ [S1, unofficial].

**Rendering Steps:**
1.  **Query Parsing:**
    *   The user expresses a rendering query as a boolean expression of the initial classification criteria [S1, unofficial].
    *   A parser takes the boolean query and the graph structure $G$ as input [S1, unofficial].
    *   The parser computes the CTF and GMTF on the cluster label domain using structural F+C rules derived from the graph [S1, unofficial].
2.  **Multimodal 3D Texture Mapping:**
    *   Perform rendering using a strategy that simultaneously accesses the original voxel model $V$ and the labelled voxel model $LV$ [S1, unofficial].
    *   For each fragment (ray sample):
        *   Fetch the voxel value $v = V[\mathbf{x}]$ from the original model [S1, unofficial].
        *   Fetch the cluster label $c = LV[\mathbf{x}]$ from the auxiliary model [S1, unofficial].
        *   Use $c$ to look up rendering parameters from the computed CTF and GMTF [S1, unofficial].
        *   Apply structural F+C shading using the fetched parameters (visibility, color, opacity, gradient modulation) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase
def preprocess(V, criteria):
    """
    V: 3D array of voxel values (original model)
    criteria: list of classification functions or thresholds
    Returns: G (Directed Graph), LV (Labelled Voxel Model)
    """
    # Step 1: Classify voxels based on independent criteria
    # Each voxel gets a set of satisfied criteria
    voxel_classes = {}
    for x in V:
        satisfied = [c for c in criteria if c(x)]
        voxel_classes[x] = satisfied

    # Step 2: Construct Directed Graph
    # Nodes are criteria, edges are logical implications
    G = build_directed_graph(criteria)

    # Step 3: Clusterize voxels
    # Group voxels by their exact set of satisfied criteria
    clusters = {}
    cluster_id_map = {}
    next_id = 0
    
    LV = empty_3d_array_of_integers(size=V.shape)
    
    for x, satisfied_set in voxel_classes.items():
        # Create a hashable key for the combination of criteria
        key = tuple(sorted(satisfied_set))
        
        if key not in clusters:
            clusters[key] = []
            cluster_id_map[key] = next_id
            next_id += 1
            
        clusters[key].append(x)
        LV[x] = cluster_id_map[key]

    # Step 4: Identify dependencies between clusters to complete graph
    # This ensures the graph reflects the actual data structure
    complete_graph(G, clusters)

    return G, LV

def parse_query(query, G):
    """
    query: boolean expression string over classification criteria
    G: Directed Graph of classification relationships
    Returns: CTF, GMTF (transfer functions mapping cluster_id -> params)
    """
    # Parse the boolean expression using the graph structure as a grammar
    # Compute visibility, color, opacity, and gradient modulation for each cluster
    # based on structural F+C rules
    CTF = compute_ctf(query, G)
    GMTF = compute_gmtf(query, G)
    return CTF, GMTF

def render(V, LV, CTF, GMTF):
    """
    V: Original Voxel Model
    LV: Labelled Voxel Model
    CTF: Color/Visibility Transfer Function
    GMTF: Gradient Modulation Transfer Function
    Returns: Rendered Image
    """
    image = empty_image()
    
    for each fragment (ray):
        # Multimodal 3D texture mapping
        # Fetch voxel value and its label
        v = fetch_voxel_value(V, ray)
        c = fetch_cluster_label(LV, ray)
        
        # Look up parameters from transfer functions
        color, opacity, visibility = CTF[c]
        gradient_modulation = GMTF[c]
        
        # Apply structural F+C shading
        # Use visibility to determine if the cluster is rendered
        # Use opacity and color for volume integration
        # Use gradient modulation for shading effects
        if visibility:
            sample = integrate_volume_sample(v, opacity, color, gradient_modulation)
            image[fragment] = sample
        else:
            image[fragment] = background_color
            
    return image
```

**Parameters:**
*   **CTF:** Defined on the cluster label domain. Determines visibility, color, and opacity for each cluster [S1, unofficial].
*   **GMTF:** Defined on the cluster label domain. Determines the gradient modulation factor for each cluster [S1, unofficial].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Multimodal Data:** The method explicitly supports multimodal datasets (e.g., MR/SPECT, MR/PET), but the classification criteria must be defined across modalities [S1, unofficial].
*   **Boolean Expression Parsing:** The parser relies on the graph structure to resolve boolean expressions. If the graph does not accurately reflect logical implications, the transfer functions may be incorrect [S1, unofficial].
*   **Cluster Label Uniqueness:** Each cluster must have a unique identifier. If two different combinations of criteria result in the same identifier, the rendering will be ambiguous [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing involves clustering voxels, which is $O(N)$ where $N$ is the number of voxels, plus graph construction which depends on the number of criteria. Rendering is $O(F)$ where $F$ is the number of fragments, with constant-time lookups in CTF and GMTF.
*   **Memory Complexity:** Requires storing the original voxel model $V$ and the auxiliary labelled voxel model $LV$, doubling the memory footprint for voxel data. The graph $G$ is typically small relative to $V$.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Explicitly represents the structure of multiclassified datasets, allowing users to explore structural relationships between classification criteria [S1, unofficial]. Supports interactive boolean queries [S1, unofficial].
*   **Limitations:** Requires preprocessing to construct the graph and labelled model. The method is specific to multiclassified scalar volumes and may not directly apply to tensor or vector fields without adaptation.
*   **When to Use:** When the user needs to understand the logical relationships between different classification criteria in a multiclassified volume dataset.
*   **When Not to Use:** When the data is not multiclassified or when the structural relationships between classes are not of interest.

## Background
Structural Focus+Context rendering was proposed by P. Abellán, A. Puig, and D. Tost [S1, unofficial]. The method was received on 31 March 2007, accepted on 1 August 2007, and posted online on 27 October 2007 [S1, unofficial]. It addresses the limitation of existing illustrative visualization systems that do not explicitly represent the structure of multiclassified datasets [S1, unofficial]. The work is associated with the handling of multimodal and time-varying data, though prior systems lacked explicit structural representation [S1, unofficial].

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
