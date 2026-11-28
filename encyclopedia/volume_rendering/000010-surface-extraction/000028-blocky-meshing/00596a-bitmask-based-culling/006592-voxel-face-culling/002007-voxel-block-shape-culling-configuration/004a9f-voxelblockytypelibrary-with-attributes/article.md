# VoxelBlockyTypeLibrary with Attributes

Also called: Minecraft-like block states, VoxelBlockyType

## Overview
`VoxelBlockyTypeLibrary` is an alternative library type that contains a list of `VoxelBlockyType` objects instead of models directly [S1]. It defines a 'block type' with attributes representing states (e.g., orientation, pressed state), similar to Minecraft block states [S1]. It solves the problem of managing complex block behaviors by automatically generating unique numerical IDs for every combination of attribute values, similar to Minecraft's block state system [S1]. It differs from the standard `VoxelBlockyLibrary` by decoupling model identity from manual ID assignment, using a combinatorial attribute system instead [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of `VoxelBlockyType` definitions [S1].
*   **Output:** A library containing a mapping from type/attribute combinations to unique numerical model IDs, with functions to resolve these IDs at runtime [S1].

**Data Structures:**
*   **VoxelBlockyType:** Defines a logical block type [S1]. Contains a `name` (StringName identifier) and a list of `VoxelBlockyAttribute` objects [S1].
*   **VoxelBlockyAttribute:** Represents a state variable [S1]. Contains a `name` (StringName identifier) and values between 0 and 255 [S1].
*   **Model ID Map:** An internal mapping associating the canonical string identifier `<type_name>[attr1=val,attr2=val,...]` to a unique integer ID [S1]. IDs are unique within the library instance but not portable across different worlds [S1].

**Numbered Steps:**

1.  **Define Block Types:** Create `VoxelBlockyType` entries [S1]. Assign a name and a list of attributes [S1]. Types should usually have very few attributes [S1].
2.  **Assign Attribute Values:** Each attribute takes integer values between 0 and 255 [S1]. Use built-in rotation attributes (`VoxelBlockyAttributeAxis`, `VoxelBlockyAttributeDirection`, `VoxelBlockyAttributeRotation`) for common geometric states [S1]. These attributes have predefined default rotations [S1]. Models are authored for the default rotation; the system generates rotated variants automatically [S1].
3.  **Generate Variants:** The system computes the Cartesian product of all attribute values for each type [S1]. For each combination, a unique model instance is created or referenced [S1]. If built-in rotation attributes are used, pre-rotated models are generated for all specified rotation axes/directions [S1]. Excluding unnecessary rotations (e.g., vertical for flat blocks) reduces memory cost, as `VoxelBlockyAttributeRotation` generates the highest number of variants [S1].
4.  **Assign Numerical IDs:** The system automatically assigns a unique integer ID to each generated variant [S1]. The ID is derived from the combination of type name and attribute values [S1]. Renaming a type or attribute changes the canonical identifier, effectively creating a new voxel type with a new ID [S1].
5.  **Runtime Lookup:** Use library functions to resolve model IDs from attribute values [S1]. Cache resolved IDs for performance if accessed frequently [S1]. Use `VoxelTool` to set the resolved ID into the voxel data [S1].

**Formulas:**
*   **Canonical Identifier Format:**
    $$ \text{ID}_{\text{key}} = \text{type\_name}[\text{attr}_1=\text{val}_1, \text{attr}_2=\text{val}_2, \dots] $$
    Where $\text{val}_i \in [0, 255]$ [S1].
*   **Total Variants per Type:**
    $$ N_{\text{variants}} = \prod_{i=1}^{n} V_i $$
    Where $n$ is the number of attributes and $V_i$ is the number of possible values for attribute $i$ [S1]. Example: Redstone dust has 1,296 variants [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelBlockyTypeLibrary:
    def __init__(self):
        self.types = {}  # Map: StringName -> VoxelBlockyType
        self.id_map = {} # Map: CanonicalString -> int

    def add_type(self, type_name, attributes):
        """Register a block type and generate all variant IDs."""
        combinations = self._get_attribute_combinations(attributes)
        for combo in combinations:
            key_parts = [f"{attr.name}={val}" for attr, val in zip(attributes, combo)]
            canonical_key = f"{type_name}[{','.join(key_parts)}]"
            model_id = self._assign_unique_id(canonical_key)
            model = self._generate_model_for_state(type_name, attributes, combo)
            self.id_map[canonical_key] = model_id
            self._store_model(model_id, model)

    def get_model_index_with_attributes(self, type_name, attrs):
        """Get model ID for a type and attribute values. Uses StringName for type_name, standard strings for dict keys."""
        key_parts = [f"{attr_name}={val}" for attr_name, val in attrs.items()]
        canonical_key = f"{type_name}[{','.join(key_parts)}]"
        return self.id_map.get(canonical_key, -1)

    def get_model_index_single_attribute(self, type_name, attr_val):
        """Shortcut for types with exactly one attribute."""
        # Not given in the sources.
        pass

    def get_model_index_default(self, type_name):
        """Get model ID for a type with no attributes, or its default state."""
        # Not given in the sources.
        pass

    def _get_attribute_combinations(self, attributes):
        """Generate Cartesian product of attribute values (0-255)."""
        # Not given in the sources.
        pass

    def _generate_model_for_state(self, type_name, attributes, combo):
        """Generate mesh for a specific state. Handles rotation attributes."""
        # Not given in the sources.
        pass

    def _assign_unique_id(self, canonical_key):
        """Assign unique integer ID. IDs are unique within library but not portable across worlds."""
        # Not given in the sources.
        pass

    def _store_model(self, model_id, model):
        """Store model reference internally for meshing."""
        # Not given in the sources.
        pass
```

**Parameters:**
*   **Attribute Values:** Integers between 0 and 255 [S1].
*   **Attribute Count:** Should be very low [S1].
*   **Rotation Attributes:** `VoxelBlockyAttributeAxis`, `VoxelBlockyAttributeDirection`, `VoxelBlockyAttributeRotation` [S1]. Default rotations are assumed for base models [S1].

**Edge Cases and Pitfalls:**
*   **Complex States:** If a block requires complex states (objects, lists), this system fails; fallback to voxel metadata and actual nodes (entities) is required [S1].
*   **Large Variant Lists:** Many attributes or states lead to a very large list of variants (e.g., redstone dust has 1,296 variants); conditional combination of models is preferred over a list of variants [S1].
*   **Memory Cost:** `VoxelBlockyAttributeRotation` has the highest number of variants and incurs significant memory cost for pre-rotated models; exclude unnecessary rotations (e.g., vertical) to reduce cost [S1].
*   **ID Portability:** Numerical IDs are not portable across worlds; renaming types or attributes changes IDs, breaking compatibility with saved worlds [S1].

## Subtopics

### Voxel ID Management
Voxel IDs in `VoxelBlockyLibrary` act like tilemap tiles [S1]. For complex games, custom 'game block types' can be defined that map to one or multiple `VoxelBlockyModel` IDs [S1]. Minecraft examples include stairs (multiple IDs for orientations/appearances), crops (multiple IDs for growth stages), doors (2 voxels for top/bottom), and rails (many IDs for straight/slope/turn configurations) [S1]. Managing the correspondence between game block IDs and voxel IDs is handled by the developer [S1].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ after caching, or $O(k)$ where $k$ is the number of attributes for string construction [G]. Variant generation is $O(N_{\text{variants}})$ [G].
*   **Memory Complexity:** $O(N_{\text{types}} \times N_{\text{variants}})$ [G]. Memory usage scales with the number of attribute combinations; pre-rotated models increase the footprint significantly [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales poorly with high attribute counts or high value ranges per attribute; the redstone dust example (1,296 variants) illustrates the practical limit [S1].
*   **Strengths:** Automatic ID management, intuitive state representation (Minecraft-like), built-in rotation support [S1].
*   **Limitations:** Not suitable for complex states (objects/lists), high memory cost for rotations, non-portable IDs [S1].
*   **When to use:** Simple to moderately complex block states (orientation, on/off, basic connections) [S1].
*   **When not to use:** Blocks requiring complex dynamic states, or when memory is extremely constrained and rotations are not needed [S1].

## Background
Originating from the Voxel Tools library for Godot, this system was inspired by Minecraft's block state system (specifically ForgeModLoader documentation) [S1]. It was introduced as an alternative to `VoxelBlockyLibrary` to better handle the complexity of block states in voxel games [S1]. The system automates model generation and ID assignment, reducing manual configuration effort [S1].

## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
