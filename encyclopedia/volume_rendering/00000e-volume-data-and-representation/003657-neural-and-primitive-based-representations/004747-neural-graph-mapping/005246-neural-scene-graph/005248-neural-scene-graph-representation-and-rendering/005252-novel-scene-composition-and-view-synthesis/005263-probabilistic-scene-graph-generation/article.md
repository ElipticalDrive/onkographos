# Probabilistic Scene Graph Generation

Also called: CG3D

## Overview
Probabilistic Scene Graph Generation (CG3D) is a text-to-3D scene synthesis method that explicitly models semantic relationships and physical interactions between objects using a scene graph interpreted as a probabilistic graphical model [S1][S2, unofficial]. Unlike methods that generate individual objects or extend from a single reference image, CG3D addresses the challenge of multi-object composition by enforcing structural consistency and physical plausibility through a two-stage optimization process [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A text prompt manually deconstructed by the user into a structured scene graph [S1][S2, unofficial].
*   **Output:** A 3D scene representation (implicitly or explicitly defined, e.g., via 3D Gaussian Splatting or similar primitives, though the specific primitive type is not detailed in the provided notes) satisfying the graph constraints [S1][S2, unofficial].

**Data Structures**
The scene graph is represented as a directed graph where:
*   **Nodes:** Represent objects.
*   **Edges:** Directed edges connect an object node (tail) to an interaction node (head) [S1][S2, unofficial].
*   **Interpretation:** The textual scene graph is interpreted as a probabilistic graphical model [S1][S2, unofficial].

**Algorithm Steps**
The generation process follows an ancestor sampling strategy, divided into two distinct optimization stages [S1][S2, unofficial]:

1.  **Stage 1: Object Generation**
    *   Generate the initial 3D representations of the object nodes defined in the scene graph [S1][S2, unofficial].
    *   This stage establishes the base geometry and appearance of individual entities without enforcing inter-object physical constraints [S1][S2, unofficial].

2.  **Stage 2: Interaction and Physical Constraint Optimization**
    *   Generate the interaction nodes and their relationships to the object nodes [S1][S2, unofficial].
    *   Introduce physical forces to ensure plausible composition:
        *   **Gravity:** Applied to objects to ensure they rest on surfaces [S1][S2, unofficial].
        *   **Normal Contact Forces:** Applied to prevent interpenetration and maintain structural stability between objects [S1][S2, unofficial].
    *   Optimize the scene parameters to satisfy these physical constraints while maintaining visual fidelity to the input prompt [S1][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_scene_graph_generation(user_prompt):
    # Step 1: Parse the manually deconstructed text prompt into a scene graph
    # The graph G contains object nodes and directed edges to interaction nodes
    scene_graph = parse_scene_graph(user_prompt)
    
    # Step 2: Stage 1 - Generate Objects
    # Initialize 3D representations for all object nodes
    objects = initialize_objects(scene_graph.object_nodes)
    
    # Step 3: Stage 2 - Generate Interactions and Apply Physics
    # Define interaction nodes connected to object nodes
    interactions = define_interactions(scene_graph.edges, objects)
    
    # Introduce physical forces for stability
    # Gravity acts on all objects
    apply_gravity(objects)
    
    # Normal contact forces prevent interpenetration between interacting objects
    apply_contact_forces(objects, interactions)
    
    # Optimize the scene to satisfy both visual and physical constraints
    optimized_scene = optimize_scene(objects, interactions)
    
    return optimized_scene
```

**Edge Cases and Pitfalls**
*   **Manual Decomposition:** The method requires the user to manually deconstruct the text prompt into a scene graph, which limits automation and scalability [S1][S2, unofficial].
*   **Physical Plausibility:** The introduction of gravity and contact forces assumes a static or quasi-static scene; dynamic interactions may require additional constraints not specified in the notes [S1][S2, unofficial].

## Operating characteristics
*   **Strengths:** Explicitly models relationships and interactions between objects, addressing a key limitation of single-object generation methods [S1][S2, unofficial].
*   **Limitations:** Relies on manual scene graph construction, which is labor-intensive and not fully automated [S1][S2, unofficial].
*   **Performance:** Specific performance metrics (time, memory) are not provided in the sources [S1][S2, unofficial].

## Background
*   **Origin:** The method is referred to as CG3D in the literature [S1][S2, unofficial].
*   **Context:** It is positioned as a solution to the problem of scene generation requiring consideration of interactions and relationships between different objects, contrasting with methods that focus on individual object generation or extend from reference images (e.g., LucidDreamer, Text2Immersion) [S1][S2, unofficial].
*   **Relation to Other Methods:** It differs from LucidDreamer and Text2Immersion, which use reference images and inpainting to extend 3D scenes, by using a probabilistic graphical model approach [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
