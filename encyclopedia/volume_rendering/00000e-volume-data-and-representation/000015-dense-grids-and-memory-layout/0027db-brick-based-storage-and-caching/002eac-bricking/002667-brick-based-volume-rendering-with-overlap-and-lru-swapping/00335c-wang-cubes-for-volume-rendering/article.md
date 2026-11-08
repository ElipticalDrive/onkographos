# Wang Cubes for Volume Rendering

Also called: 3D Wang Tiles, Wang Cube tiling, Wang Cubes Memory Optimization, Wang Cubes Texture Synthesis

## Overview
Wang Cubes is a texture synthesis method that creates non-periodic 3D textures using cubes with colored faces to overcome the memory and time limits of storing high-resolution volumetric data [S1, unofficial]. It reduces texture memory usage by storing a small set of synthesized cube templates rather than the full volume, allowing the generation of arbitrary-sized textures on the fly [S1, unofficial]. Unlike [Bricking with Boundary Voxel Duplication](4198-bricking-with-boundary-voxel-duplication.md) which duplicates boundary voxels for continuity, Wang Cubes enforce continuity through face-color matching and explicit edge-volume synthesis [S1, unofficial].

## How it works
**Inputs:** A 3D sample volume (source data) and a target resolution.
**Outputs:** A set of synthesized Wang Cubes and a tiling map, which can be composed into a large 3D texture.

**Data Structures:**
*   **Wang Cube:** A cube with 6 faces: North (N), South (S), West (W), East (E), Front (F), Back (B).
*   **Face Colors:** Each face has a discrete color identifier. Faces in the same direction (N/S, W/E, F/B) share the same color set. Colors on different axes are independent.
*   **Face Volume:** A 3D sub-volume of the same size as a Wang Cube, randomly selected from the sample volume, corresponding to a specific face color.
*   **Edge Volume:** A 3D sub-volume tied to a specific direction (NS, WE, or FB), used to enforce edge continuity.

**Parameters:**
*   $C$: Cube side length (e.g., 16).
*   $M$: Number of cube sets (minimum 16 cubes per set).
*   $V$: Dataset dimension size (e.g., 256).
*   $\pi_i, \pi_e$: Weight factors for inside and edge errors.

**Algorithm Steps:**

1.  **Initialization:** Select a random cube from the sample volume as the initial seed. Define the set of available face colors for each direction (N/S, W/E, F/B).
2.  **Face Synthesis (Inside Cube):**
    For each face of the cube:
    a. Select a random "face volume" from the sample volume corresponding to the required face color.
    b. Define the quilting region between the current cube face and the face volume using two half elliptical spheroids [S1, unofficial].
    c. Use a graph cut algorithm (max-flow/min-cut) to synthesize the texture on the face, minimizing the synthesis error $E_i$ along the cutting surface [S1, unofficial].
    d. Repeat for all faces of all cubes in the set until the accumulated inside error $E_i$ is below a threshold.
3.  **Edge Continuity Correction:**
    The initial synthesis causes discontinuities at cube edges because adjacent faces sharing a color may come from different random samples [S1, unofficial].
    a. For each direction (NS, WE, FB), select a random "edge volume" from the sample volume.
    b. For face volumes corresponding to a specific direction (e.g., WE), use the edge volumes of the other two directions (NS and FB) to modify the 4 edge regions of the face's middle plane [S1, unofficial].
    c. Search for a quilting surface within two half cylinders between the face volume and the edge volumes [S1, unofficial].
    d. Synthesize the face volumes using these edge constraints.
    e. Regenerate the cubes using the corrected face volumes.
4.  **Alternative Edge Handling (Soft Boundaries):**
    For textures without sharp boundaries, introduce an edge error factor $E_e$. The total error is $E_s = \pi_i E_i + \pi_e E_e$, where $E_e$ measures average edge discontinuities [S1, unofficial].
5.  **Tiling:**
    Place the synthesized cubes in a grid such that adjacent cubes have matching face colors. This creates a non-periodic 3D texture [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synthesize_wang_cubes(sample_volume, cube_size, num_cubes, target_resolution):
    """
    Synthesize a set of Wang Cubes and tile them to create a large texture.
    """
    # 1. Define color sets for each axis direction
    # Colors are indices into the sample volume patches
    color_sets = {
        'NS': generate_color_set(sample_volume, cube_size),
        'WE': generate_color_set(sample_volume, cube_size),
        'FB': generate_color_set(sample_volume, cube_size)
    }
    
    # 2. Initialize cube set
    cubes = []
    for _ in range(num_cubes):
        cube = {
            'colors': {
                'N': random.choice(color_sets['NS']),
                'S': random.choice(color_sets['NS']),
                'W': random.choice(color_sets['WE']),
                'E': random.choice(color_sets['WE']),
                'F': random.choice(color_sets['FB']),
                'B': random.choice(color_sets['FB'])
            },
            'texture': None
        }
        cubes.append(cube)
        
    # 3. Synthesize face volumes and cubes
    # Step 3a: Initial face synthesis using elliptical spheroids
    for cube in cubes:
        for face in ['N', 'S', 'W', 'E', 'F', 'B']:
            face_vol = get_random_patch_from_sample(sample_volume, cube_size)
            # Define quilting region using half elliptical spheroids
            quilting_region = define_elliptical_spheroid_region(cube, face, face_vol)
            # Use graph cut (max-flow/min-cut) to synthesize
            synthesized_face = graph_cut_synthesis(cube[face], face_vol, quilting_region)
            cube['texture'] = combine_cube_texture(cube['texture'], synthesized_face, face)
            
    # 3b: Edge continuity correction
    edge_volumes = {
        'NS': get_random_patch_from_sample(sample_volume, cube_size),
        'WE': get_random_patch_from_sample(sample_volume, cube_size),
        'FB': get_random_patch_from_sample(sample_volume, cube_size)
    }
    
    for cube in cubes:
        # Modify WE faces using NS and FB edge volumes
        modify_face_edges(cube, 'WE', edge_volumes['NS'], edge_volumes['FB'])
        # Modify NS faces using WE and FB edge volumes
        modify_face_edges(cube, 'NS', edge_volumes['WE'], edge_volumes['FB'])
        # Modify FB faces using NS and WE edge volumes
        modify_face_edges(cube, 'FB', edge_volumes['NS'], edge_volumes['WE'])
        
    # 4. Tile the cubes
    # Create a tiling map where adjacent cubes have matching colors
    tiling_map = generate_tiling_map(cubes, target_resolution)
    
    # 5. Compose final texture
    final_texture = compose_texture(cubes, tiling_map, target_resolution)
    return final_texture

def modify_face_edges(cube, target_face, edge_vol_1, edge_vol_2):
    """
    Modify the 4 edge regions of a face using edge volumes.
    Uses quilting within half cylinders.
    """
    face_vol = get_face_volume_from_cube(cube, target_face)
    # Define quilting surface within half cylinders between face and edge volumes
    quilting_surface = define_cylindrical_quilting_surface(face_vol, edge_vol_1, edge_vol_2)
    # Synthesize the face with edge constraints
    new_face_texture = graph_cut_synthesis(face_vol, quilting_surface)
    update_cube_texture(cube, target_face, new_face_texture)

def define_elliptical_spheroid_region(cube, face, face_vol):
    """
    Define the quilting region between a cube face and a face volume
    using two half elliptical spheroids.
    """
    # Implementation of elliptical spheroid geometry for quilting
    # Returns a mask or region definition for the graph cut algorithm
    return region_mask

def graph_cut_synthesis(source, target, region):
    """
    Perform 3D texture synthesis using max-flow/min-cut graph cut algorithm.
    Minimizes error within the defined region.
    """
    # Construct graph with source and target patches
    # Define edge weights based on pixel/voxel differences
    # Run max-flow algorithm
    # Return synthesized texture
    pass
```

**Edge Cases and Pitfalls:**
*   **Edge Discontinuity:** Textures are discontinuous on cube edges along cube faces because adjacent faces share colors but come from randomly selected samples [S1, unofficial]. This is mitigated by the edge volume modification step.
*   **Repetitive Appearance:** Since a set of Wang Cubes has a limited number of samples, they will inevitably repeat themselves somewhere in the volume [S1, unofficial]. This can be improved by using larger cube sets [S1, unofficial].
*   **Minimum Cube Set Size:** To ensure continuity for the whole space, the minimum cube set size increases to 128 cubes if corner colors are also constrained [S1, unofficial].

## Operating characteristics
*   **Memory Reduction:** The required texture memory with Wang Cubes is significantly lower than without. For a dataset of size $256 \times 256 \times 128$ and cubes of size $16^3$, the required texture memory is $1/128$ of that required without Wang Cubes [S1, unofficial].
*   **Formula:** The space requirement difference ratio is $16C^3 / V^3$, where $M$ is the number of cube sets, $C$ is the cube side length, and $V$ is the dataset size dimension [S1, unofficial].
    *   Texture memory with Wang Cubes: $M \times 16 \times C^3$ [S1, unofficial].
    *   Texture memory without Wang Cubes: $M \times V^3$ [S1, unofficial].
*   **Performance:** Synthesis time is limited by the graph cut algorithm and the number of cubes. However, once synthesized, composing arbitrary-sized 3D textures is fast [S1, unofficial].
*   **Strengths:** Reduces texture memory usage; allows non-periodic textures; combines volume data and segmentation masks into one texture unit [S1, unofficial].
*   **Limitations:** Repetitive appearance; edge discontinuity if not properly corrected; requires careful tuning of error thresholds and edge volumes [S1, unofficial].

## Background
Wang Cubes are the 3D extension of 2D Wang Tiles, which are square tiles with colored edges [S1, unofficial]. The method was introduced to address the memory and time limitations of high-resolution 3D textures in volume rendering applications [S1, unofficial]. It is particularly useful for medical datasets (e.g., abdomen, lung, liver, kidneys) and can be used in both surface-based methods and direct volume rendering [S1, unofficial]. The technique allows for the generation of illustrative styles and high-resolution textures from small sample volumes [S1, unofficial].

## Sources
[S1] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
