# Ray-Density Cloud Intersection

Also called: Density Cloud, Density Cloud Intersection, Metaball / Density Cloud Intersection, Metaball Intersection

## Overview
Ray-Density Cloud Intersection computes the intersection of a ray with a complex implicit surface defined by the fused intensity distribution of multiple atomic centers, typically modeled as Gaussian bumps [S1]. It solves the problem of rendering volumetric structures like electron clouds or metaballs where the surface is defined by a threshold of the summed density field [S1]. Unlike sibling methods that may perform culling during the rendering loop, this method incorporates culling into the enumeration phase to minimize overhead for large atom counts [S1].

## How it works
**Inputs and Outputs**
- **Input**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, a set of $N$ atoms where each atom $i$ has a center position $\mathbf{c}_i$ and a density profile function, and a threshold value $\tau$ defining the surface [S1].
- **Output**: Intersection distance $t$ along the ray, or null if no intersection exists [S1].

**Data Structures**
- **Atom**: A structure containing the 3D position $\mathbf{c}_i$ and parameters for the density function (e.g., width/sigma for Gaussian) [S1].
- **Sphere of Influence**: For each atom, a bounding sphere is precomputed based on the allowable error threshold. This sphere encloses the region where the atom's density influence is above the threshold [S1].

**Formulas**
The intensity distribution of each atom $i$ is defined as a Gaussian bump [S1]:
$$ D_i(\mathbf{p}) = A_i e^{-\frac{\|\mathbf{p} - \mathbf{c}_i\|^2}{2\sigma_i^2}} $$
where $\mathbf{p}$ is a point in space, $A_i$ is the amplitude, and $\sigma_i$ controls the spread [S1]. The total intensity at point $\mathbf{p}$ is the summation over all atoms:
$$ D(\mathbf{p}) = \sum_{i=1}^{N} D_i(\mathbf{p}) $$
The surface is defined by the isosurface $D(\mathbf{p}) = \tau$ [S1].

**Parameters**
- **Allowable error threshold**: A user-defined value that determines the sphere of influence for culling atoms. An atom is considered "close" if its density influence is above this threshold [S1].

**Steps**
1. **Preprocessing (Enumeration Phase)**:
   - For each atom, compute its sphere of influence based on the allowable error threshold [S1].
   - Enumerate voxels or spatial regions and associate them with atoms whose spheres of influence contain them [S1]. This allows culling of irrelevant atoms during ray intersection [S1].

2. **Ray Intersection Calculation**:
   - Since the equation $D(\mathbf{o} + t\mathbf{d}) = \tau$ cannot be solved algebraically due to the summation of Gaussian functions, numerical methods are required [S1].
   - Use **Regula Falsi** and **Newton iteration** to find the root $t$ [S1].
   - During the evaluation of $D(\mathbf{o} + t\mathbf{d})$, only include atoms whose spheres of influence contain the current point $\mathbf{o} + t\mathbf{d}$ [S1]. This culling is performed during enumeration, not in the inner rendering loop [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_density_cloud_intersection(ray_origin, ray_direction, atoms, threshold, tolerance=1e-6, max_iter=50):
    """
    Computes the intersection of a ray with a density cloud surface.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray (normalized).
        atoms: List of Atom objects, each with position 'center' and 'sigma'.
        threshold: Float, the density threshold defining the surface.
        tolerance: Float, convergence tolerance for numerical solver.
        max_iter: Int, maximum iterations for Newton's method.
        
    Returns:
        Float: Intersection distance t, or None if no intersection.
    """
    
    # Precompute spheres of influence for culling
    # An atom is included if its density at the point is above a cutoff.
    # The sphere of influence radius R_i is derived from the condition D_i(p) >= cutoff.
    # For Gaussian: A_i * exp(-R_i^2 / (2*sigma_i^2)) = cutoff
    # R_i = sigma_i * sqrt(-2 * ln(cutoff / A_i))
    # Note: The specific cutoff value is the "allowable error threshold" [S1].
    # For implementation, we assume a global cutoff or per-atom threshold is provided.
    # Here we assume a simple distance check against a precomputed radius for efficiency.
    
    def get_influencing_atoms(point):
        """Return atoms whose sphere of influence contains the point."""
        influencing = []
        for atom in atoms:
            # Check if point is within the sphere of influence
            # This check relies on precomputed radii based on the allowable error threshold [S1]
            if distance_sq(point, atom.center) <= atom.influence_radius_sq:
                influencing.append(atom)
        return influencing

    def total_density(point, influencing_atoms):
        """Calculate total density at point using only influencing atoms."""
        density = 0.0
        for atom in influencing_atoms:
            dist_sq = distance_sq(point, atom.center)
            # Gaussian bump formula [S1]
            density += atom.amplitude * exp(-dist_sq / (2 * atom.sigma**2))
        return density

    def density_derivative(point, influencing_atoms):
        """Calculate derivative of density along ray direction."""
        # d/dt D(o + t*d) = grad(D) . d
        # grad(D_i) = D_i * (-2(p - c_i) / (2*sigma^2)) = -D_i * (p - c_i) / sigma^2
        deriv = 0.0
        for atom in influencing_atoms:
            diff = point - atom.center
            dist_sq = dot(diff, diff)
            d_val = atom.amplitude * exp(-dist_sq / (2 * atom.sigma**2))
            # Gradient contribution
            grad_contrib = -d_val * diff / (atom.sigma**2)
            deriv += dot(grad_contrib, ray_direction)
        return deriv

    # Initial guess and bounds for Regula Falsi / Newton
    # We need to find t such that density(o + t*d) - threshold = 0
    
    # Simple initial search to bracket the root or find a starting point
    # This part is not explicitly detailed in sources, so we use a standard approach [G]
    t_min = 0.0
    t_max = 1000.0 # Arbitrary large value, should be scene-dependent [G]
    
    # Check if ray starts inside or outside
    start_atoms = get_influencing_atoms(ray_origin)
    start_density = total_density(ray_origin, start_atoms)
    
    if abs(start_density - threshold) < tolerance:
        return 0.0
        
    # Find a bracketing interval [t_a, t_b]
    t_a = 0.0
    t_b = 1.0
    density_a = start_density
    density_b = total_density(ray_origin + t_b * ray_direction, get_influencing_atoms(ray_origin + t_b * ray_direction))
    
    # Expand interval if no sign change found
    while (density_a - threshold) * (density_b - threshold) > 0 and t_b < 10000.0:
        t_a = t_b
        density_a = density_b
        t_b *= 2
        density_b = total_density(ray_origin + t_b * ray_direction, get_influencing_atoms(ray_origin + t_b * ray_direction))
        
    if (density_a - threshold) * (density_b - threshold) > 0:
        return None # No intersection found in range
        
    # Newton-Raphson iteration with Regula Falsi fallback or hybrid
    # Sources specify Regula Falsi and Newton iteration [S1]
    
    current_t = t_b
    for i in range(max_iter):
        current_point = ray_origin + current_t * ray_direction
        influencing = get_influencing_atoms(current_point)
        f_val = total_density(current_point, influencing) - threshold
        f_deriv = density_derivative(current_point, influencing)
        
        if abs(f_val) < tolerance:
            return current_t
            
        if abs(f_deriv) < 1e-10:
            # Fallback to Regula Falsi if derivative is near zero
            # Regula Falsi: t_new = t_b - f(t_b) * (t_a - t_b) / (f(t_b) - f(t_a))
            # We need to track f(t_a) and f(t_b)
            pass 
            
        # Newton step
        next_t = current_t - f_val / f_deriv
        
        # Clamp to interval to ensure convergence
        if next_t < t_a or next_t > t_b:
            next_t = (t_a + t_b) / 2.0 # Bisection step as safe fallback
            
        # Update interval
        next_point = ray_origin + next_t * ray_direction
        next_density = total_density(next_point, get_influencing_atoms(next_point))
        
        if (next_density - threshold) * (density_a - threshold) < 0:
            t_b = next_t
            density_b = next_density
        else:
            t_a = next_t
            density_a = next_density
            
        current_t = next_t
        
    return current_t if abs(total_density(ray_origin + current_t * ray_direction, get_influencing_atoms(ray_origin + current_t * ray_direction)) - threshold) < tolerance else None

def distance_sq(p1, p2):
    return (p1.x - p2.x)**2 + (p1.y - p2.y)**2 + (p1.z - p2.z)**2

def dot(v1, v2):
    return v1.x * v2.x + v1.y * v2.y + v1.z * v2.z

def exp(x):
    import math
    return math.exp(x)
```

**Subtopics**
- [Kalra & Barr Implicit Surface Intersection](1278-kalra-barr-implicit-surface-intersection.md): Uses marching cubes for isosurface extraction rather than ray intersection. Choose when rasterization is preferred over ray tracing.
- [SDF Raycast Collision Detection](2417-sdf-raycast-collision-detection.md): Uses Signed Distance Functions for collision detection. Choose when exact distance to surface is needed for physics, not just visual intersection.
- [Ray Tracing Implicit Surfaces](2814-ray-tracing-implicit-surfaces.md): General category for ray-implicit intersection. This method is a specific instance for density clouds.
- [LevelSetHDDA](2875-levelsethdda.md): Uses Hierarchical Depth-First Approach for level sets. Choose for hierarchical data structures.
- [Ray-Ellipsoid Intersection](3463-ray-ellipsoid-intersection.md): Analytical intersection for ellipsoids. Choose for simple primitive shapes.
- [RayImplicitIntersection](3591-rayimplicitintersection.md): General implicit surface intersection. This method is a specific implementation for density clouds.
- [Edge Point Re-parametrization](10270-edge-point-re-parametrization.md): Technique for refining intersection points. Use as a post-processing step for accuracy.
- [Logit-based Root Finding (L-Tracing)](18793-logit-based-root-finding-l-tracing.md): Alternative root-finding method. Choose for better stability in certain density profiles.

## Operating characteristics
- **Time Complexity**: The intersection calculation is expensive because it requires numerical iteration (Regula Falsi and Newton) for each ray [S1]. The cost depends on the number of influencing atoms at each iteration step [S1].
- **Memory Complexity**: Depends on the storage of atom positions and precomputed spheres of influence [S1].
- **Measured Performance**: For a model with 64 atoms, culling time is less than 30% of total rendering time [S1]. For a model with 7,011 atoms (Brain density cloud), rendering took 45 minutes on a VAX 11/785, with 8 seconds of preprocessing [S1]. Enumeration time for the 7,011 atom model was less than 10 seconds, a negligible fraction of the total rendering time [S1].
- **Strengths**: Culling is performed during the enumeration phase, which is more efficient than Blinn's scheme that performs culling in the inner rendering loop [S1]. This makes it scalable to larger atom counts [S1].
- **Limitations**: Numerical solution is required, which is slower than algebraic solutions [S1]. The Gaussian function spreads to infinity, requiring careful culling [S1].
- **When to use**: When rendering metaballs or density clouds where visual fidelity of fused surfaces is required [S1].
- **When not to use**: When performance is critical and the number of atoms is very large, or when an algebraic solution is possible (e.g., using polynomial functions like biquadratic) [S1].

## Background
The metaball or density cloud primitive was first introduced by Nishimura in the Links ray-tracing system [S1]. It was later independently introduced by Blinn in English-language literature [S1]. The method was implemented in the ART (Accelerated Ray-Tracing) system, where it was used to render complex structures like the "Brain" density cloud based on a DNA model [S1].

## Sources
[S1] Arts: Accelerated ray-tracing system, https://ieeexplore.ieee.org/abstract/document/4056861/, fetched 2026-10-25
