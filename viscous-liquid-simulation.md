# **Theoretical and Computational Framework for Viscous and Cohesive Lagrangian Fluids in Godot 4**

The realization of high-fidelity fluid dynamics within interactive 2D environments necessitates a departure from traditional, force-based simulation paradigms in favor of more robust, position-centric methodologies. For the development of gooey or highly viscous substances in Godot 4, the Position Based Fluids (PBF) framework stands as the most computationally viable approach, offering a synthesis of the stability inherent in Position Based Dynamics (PBD) and the flexible spatial discretization of Smoothed Particle Hydrodynamics (SPH).1 This report provides an exhaustive analysis of the mathematical theory, algorithmic workflows, and engine-level architectural requirements to implement a gooey fluid system that maintains physical plausibility under extreme external forces and complex boundary interactions.

## **Historical Evolution and the Shift to Position-Based Paradigms**

Traditional particle-based fluid simulations were largely dominated by the Smoothed Particle Hydrodynamics (SPH) method, originally developed for astrophysical modeling and later adapted for computer graphics.3 SPH achieves fluid-like behavior by defining a rest density and deriving pressure-based forces from an equation of state (EOS).1 However, these force-based methods are notoriously sensitive to time-step size. To maintain incompressibility—a key requirement for liquid realism—stiff equations are required, which often lead to numerical instability or "explosions" in the simulation if the time step is not sufficiently small.1

Position Based Dynamics (PBD) emerged as an alternative for game physics, prioritizing stability and controllability over strict physical accuracy. By resolving constraints directly at the position level rather than the force level, PBD avoids the typical instabilities of explicit integration.1 Position Based Fluids (PBF) adapts this logic to fluid volumes by formulating incompressibility as a set of positional constraints.1 This allows for significantly larger time steps, which is critical for maintaining high frame rates in Godot 4 games where the CPU and GPU budgets are shared with rendering and logic.6

### **Comparative Mechanics of Particle-Based Fluids**

| Feature | Smoothed Particle Hydrodynamics (SPH) | Position Based Fluids (PBF) |
| :---- | :---- | :---- |
| **Primary Variable** | Force/Acceleration | Position Displacement |
| **Stability** | Conditionally stable (requires small $\\Delta t$) | Unconditionally stable for large $\\Delta t$ |
| **Incompressibility** | Enforced via stiff EOS or pressure solvers | Enforced via iterative position projection |
| **Clumping** | High risk of tensile instability | Managed via artificial pressure terms |
| **Performance** | Computationally expensive due to small steps | Highly efficient for real-time applications |

The shift to PBF is particularly relevant for "gooey" materials, which exhibit high internal resistance and require strong cohesive forces.9 In an EOS-based SPH system, these high-viscosity parameters would further restrict the stable time step, whereas in PBF, they are integrated as stable post-processing or constraint steps.1

## **Mathematical Foundations of the 2D PBF Solver**

The fundamental goal of a 2D PBF solver is to ensure that a collection of fluid particles maintains a constant rest density $\\rho\_0$.1 For a given particle $i$, the local density $\\rho\_i$ is estimated based on the positions of its neighbors within a kernel radius $h$. This density must satisfy the constraint function $C\_i(\\mathbf{p}\_1, \\dots, \\mathbf{p}\_n) \= 0$.1

### **Smoothing Kernels and Normalization in 2D**

In a 2D environment, the smoothing kernel $W(\\mathbf{r}, h)$ must be normalized so that its integral over the 2D domain equals one. The choice of kernel is the primary mechanism for determining how smoothly or "clumpily" the gooey fluid behaves.11

For density estimation, the Poly6 (Smooth) kernel is preferred because its gradient vanishes at the origin, preventing zero-distance singularities.11 The 2D normalization factor for the Poly6 kernel is defined as:

$$W\_{smooth}(\\mathbf{r}, h) \= \\frac{35}{32\\pi h^7} (h^2 \- |\\mathbf{r}|^2)^3, \\quad 0 \\leq |\\mathbf{r}| \\leq h$$  
However, for calculating the gradient of the constraint—which dictates the direction of particle movement—a "Spiky" kernel is used to ensure a non-zero repulsive force when particles overlap.11 The 2D Spiky kernel is:

$$W\_{spiky}(\\mathbf{r}, h) \= \\frac{10}{\\pi h^5} (h \- |\\mathbf{r}|)^3, \\quad 0 \\leq |\\mathbf{r}| \\leq h$$  
The use of distinct kernels for density and gradient is a critical design choice in PBF to ensure both smooth density fields and stable particle repulsion.12

### **The Iterative Constraint Projection**

The solver seeks a displacement $\\Delta \\mathbf{p}\_i$ for each particle such that $C\_i(\\mathbf{p} \+ \\Delta \\mathbf{p}) \= 0$.5 Using a Newton-Raphson approach, the displacement is found by projecting along the constraint gradient:

$$\\Delta \\mathbf{p}\_i \= \\frac{1}{\\rho\_0} \\sum\_j (\\lambda\_i \+ \\lambda\_j) \\nabla W(\\mathbf{p}\_i \- \\mathbf{p}\_j, h)$$  
The Lagrange multiplier $\\lambda\_i$ represents the "pressure" at particle $i$ and is calculated as:

$$\\lambda\_i \= \- \\frac{C\_i(\\mathbf{p})}{\\sum\_j | \\nabla\_{\\mathbf{p}\_j} C\_i |^2 \+ \\epsilon}$$  
The term $\\epsilon$ is a constraint softening parameter (user-defined) that prevents division by zero when a particle has too few neighbors, ensuring the simulation remains robust even in sparse fluid regions.1 For gooey fluids, $\\epsilon$ can be slightly higher to allow for "compression" without explosive reactions, contributing to the soft, malleable look of viscous sludge.10

## **Engineering the "Gooey" Aesthetic: Viscosity, Cohesion, and Adhesion**

A gooey fluid is defined not just by its incompressibility, but by its resistance to shear (viscosity) and its tendency to stick to itself (cohesion) and surfaces (adhesion).9 In the PBF framework, these are implemented through specialized corrective terms.

### **Viscoelasticity and XSPH Viscosity**

Viscosity in PBF is typically handled via the XSPH method, which is a velocity-based filter applied after the position solver.2 Unlike forces that might oscillate, XSPH works by pulling each particle's velocity toward the average velocity of its neighborhood, creating a coherent, damped motion.2

The corrected velocity $\\mathbf{v}\_i^{new}$ is calculated as:

$$\\mathbf{v}\_i^{new} \= \\mathbf{v}\_i \+ c \\sum\_j \\frac{m\_j}{\\rho\_j} (\\mathbf{v}\_j \- \\mathbf{v}\_i) W(\\mathbf{p}\_i \- \\mathbf{p}\_j, h)$$  
For gooey fluids, the coefficient $c$ is set to a high value (often $0.1$ to $0.5$ in 2D contexts), which suppresses turbulence and results in a "slow-moving" flow reminiscent of molasses or heavy oil.2 This damping effectively removes high-frequency kinetic energy, which is essential for gooey substances that should not splash or break apart easily.9

### **Artificial Pressure and Surface Cohesion**

Gooey fluids must often form "strings" or bridges between surfaces.18 Standard PBF suffers from tensile instability, where a lack of neighbors causes particles to clump into small droplets due to negative pressure.1 To create a cohesive, gooey mass, an artificial pressure term $s\_{corr}$ is added to the solver's position update 1:

$$s\_{corr} \= \-k \\left( \\frac{W(\\mathbf{p}\_i \- \\mathbf{p}\_j, h)}{W(\\Delta \\mathbf{q}, h)} \\right)^n$$  
In this formulation, $k$ is a small positive constant and $n$ is typically 4\.2 This term provides a small repulsive force at very short distances that acts like a "hard shell," but when balanced with the density constraint, it allows for strong surface tension.1 By increasing $k$, the developer can make the fluid more "stretchy" and cohesive, as the particles resist being pulled apart even when the density is low.15

### **Mechanical Properties of Gooey Materials**

| Property | Physical Mechanism | PBF Implementation Method |
| :---- | :---- | :---- |
| **Viscosity** | Internal molecular friction | XSPH velocity smoothing 2 |
| **Cohesion** | Intra-fluid attraction | Artificial pressure $s\_{corr}$ 1 |
| **Adhesion** | Fluid-Surface attraction | Stickiness coefficients in boundary kernels 9 |
| **Surface Tension** | Surface minimization | Local area/mesh constraints 18 |

## **Boundary Handling: Interaction with Hard Physical Objects**

The interaction between gooey fluids and "hard" objects—represented by StaticBody2D or RigidBody2D nodes in Godot—requires a robust coupling mechanism. There are two primary schools of thought for boundary handling in PBF: the boundary particle approach and the implicit Signed Distance Field (SDF) approach.20

### **The Signed Distance Field (SDF) Approach**

SDFs are particularly powerful in Godot 4 due to the engine's baked SDF capabilities for 2D lighting and shadows.22 Mathematically, an SDF is a field where each pixel or grid cell stores the scalar distance $d$ to the nearest surface.22

When a fluid particle $i$ at position $\\mathbf{p}\_i$ is checked against a hard object, the logic follows:

1. **Detection:** Sample the SDF at $\\mathbf{p}\_i$. If the value $SDF(\\mathbf{p}\_i) \< 0$, the particle has penetrated the object.14  
2. **Projection:** Move the particle along the surface normal $\\mathbf{n} \= \\nabla SDF(\\mathbf{p}\_i)$ by the distance $|SDF(\\mathbf{p}\_i)|$ to place it exactly on the surface.14  
3. **Friction/Adhesion:** Adjust the particle's velocity relative to the object's velocity to simulate surface friction or gooey "stickiness".9

SDF-based boundaries offer perfectly smooth collision planes regardless of the object's visual complexity.22 For moving rigid bodies, the SDF can be sampled in the object's local space to avoid expensive re-computation.14

### **Rigid-Fluid Coupling and Boundary Density**

To prevent fluid from "shrinking" near walls, the boundaries must contribute to the density calculation.20 Without this, a particle near a wall appears to have fewer neighbors, causing the PBF solver to "pull" it toward the wall to increase density, resulting in a visible gap or unnatural compression.20

In a consistent boundary handling scheme:

* **Volume Contribution:** The boundary is treated as a set of virtual particles with mass and density.20  
* **Pressure Mirroring:** The pressure (or $\\lambda$) of the boundary is mirrored from the nearby fluid particles to ensure a zero-velocity gradient at the wall.20  
* **SDF Weighting:** Alternatively, the density contribution of the boundary can be calculated by integrating the kernel volume over the overlapping portion of the hard object.21

For gooey liquids, high adhesion is achieved by ensuring that the XSPH step considers the zero-velocity (or prescribed velocity) of the hard object as a "neighbor," effectively sticking the fluid layers to the surface.9

## **Dynamic Forces: Gravity and External Launching**

The user requirement for fluid affected by gravity and launchable by external forces necessitates a specific integration order within the Godot physics frame.

### **Force Integration and Prediction**

At the start of the frame, the "old" positions $\\mathbf{x}\_i$ are stored. New velocities are computed by applying gravity $\\mathbf{g}$ and any external impulses $\\mathbf{I}\_{ext}$ 2:

$$\\mathbf{v}\_i^{pred} \= \\mathbf{v}\_i^{old} \+ \\Delta t (\\mathbf{g} \+ \\mathbf{f}\_{ext} / m\_i)$$  
A predicted position $\\mathbf{p}\_i \= \\mathbf{x}\_i \+ \\Delta t \\mathbf{v}\_i^{pred}$ is then used as the starting point for the PBF solver.5 This ensures that external forces—such as a player hitting the fluid or a high-pressure nozzle—are reflected in the fluid's initial momentum before the incompressibility solver corrects for clumping.25

### **Launching Mechanisms**

Launching a gooey fluid requires a "velocity injector" or a dedicated particle emitter that assigns a specific directional vector to newly spawned particles.25 Because PBF is Lagrangian, the momentum is carried by the particles themselves.1 The gooey nature ensures that a launched stream maintains its continuity (as a thick "rope" of liquid) rather than breaking into mist, provided the artificial pressure $s\_{corr}$ is tuned for high cohesion.15

## **Computational Architecture in Godot 4: The Compute Shader Pipeline**

Executing a PBF simulation on the CPU in GDScript is unfeasible for the particle counts required for visually rich fluids.6 Godot 4's RenderingDevice API allows the developer to dispatch GLSL Compute Shaders to handle the heavy lifting of neighbor finding and constraint resolution.30

### **Data Storage with SSBOs**

All simulation data must reside in Shader Storage Buffer Objects (SSBOs). In Godot 4, these are accessed via the RID (Resource ID) system.31

| Buffer Type | Data Layout (std430) | Access Pattern |
| :---- | :---- | :---- |
| **Particle Buffer** | struct { vec2 pos; vec2 old\_pos; vec2 vel; } | Read/Write 32 |
| **Grid Index Buffer** | int particle\_indices | Sorted via bitonic or counting sort 16 |
| **Cell Counter Buffer** | int cell\_offsets | Used for spatial hashing 34 |
| **Solver Data Buffer** | float lambdas | Intermediate $\\lambda$ storage 2 |

The std430 layout is essential for memory alignment, ensuring that vec2 and float types are packed tightly to minimize VRAM bandwidth.33

### **The Compute Step Workflow**

The simulation loop within a Godot \_physics\_process call follows a rigid sequence of compute shader dispatches, separated by memory barriers.37

1. **Step 1: Prediction:** Apply gravity and external forces to update predicted positions.5  
2. **Step 2: Spatial Hash Update:** Calculate the grid cell for each particle and sort the particle buffer to ensure spatial locality for neighbor finding.16  
3. **Step 3: Density and Lambda:** Calculate $\\rho\_i$ and $\\lambda\_i$ for every particle.2 A memory barrier ensures all $\\lambda$ values are written before Step 4\.  
4. **Step 4: Constraint Resolution:** Calculate $\\Delta \\mathbf{p}\_i$ using the neighbors and $\\lambda$ values, then update $\\mathbf{p}\_i$.1 This step is repeated for $N$ iterations (typically 2-4 for real-time).1  
5. **Step 5: Velocity and XSPH:** Update final velocities $\\mathbf{v}\_i \= (\\mathbf{p}\_i \- \\mathbf{x}\_i) / \\Delta t$ and apply XSPH smoothing.2  
6. **Step 6: Boundary Collision:** Perform a final pass against SDFs or boundary particles to prevent penetration of hard objects.14

### **Memory Barriers and Synchronization**

In Godot 4, RenderingDevice does not automatically synchronize compute dispatches unless they are within the same render pass or explicit barriers are used.37 Since Step 3 relies on the output of Step 2, a full\_barrier() or specific buffer barrier must be called to prevent the GPU from starting calculations with stale or partially written data.37 Failure to synchronize correctly results in "jittery" fluids or ghosting artifacts where particles appear to lag behind their neighbors.38

## **Spatial Hashing: The Engine of Neighbor Discovery**

Efficiently finding neighbors within the kernel radius $h$ is the primary performance bottleneck.39 A naive search of $N$ particles against $N$ others is $O(N^2)$, which collapses at even modest particle counts.39 Spatial hashing reduces this to $O(N)$.34

### **Grid Construction in 2D**

The 2D world is divided into cells of size $h \\times h$. Each particle's position $(x, y)$ is mapped to a hash index $H(x, y)$ 34:

$$H(x, y) \= \\lfloor x / h \\rfloor \\cdot P\_1 \\oplus \\lfloor y / h \\rfloor \\cdot P\_2 \\mod \\text{HashSize}$$  
On the GPU, this is optimized by:

* **Prefix Sums:** To determine the starting memory address of each cell's particle list.16  
* **Atomic Counters:** To incrementally fill the grid buffer with particle IDs.32

When the density solver runs for particle $i$, it only checks the nine cells in the $3 \\times 3$ grid centered on $i$'s cell.39 This ensures that for a 2D gooey fluid with 10,000 particles, the number of neighbor checks per particle is limited to a constant maximum, maintaining high performance even as the fluid clumps together.34

## **Surface Reconstruction for the "Gooey" Visual Effect**

While the simulation is particle-based, a gooey fluid must be rendered as a continuous, thick liquid.1 This requires a surface reconstruction pass, typically implemented as a post-processing fragment shader or a specialized MeshInstance2D.

### **Marching Squares and Isosurfaces**

The Marching Squares algorithm is the 2D equivalent of Marching Cubes, used to reconstruct a contour from a scalar field.44

1. **Voxelization:** A screen-space or local-space grid is filled with density values $\\rho$ calculated from the fluid particles.45  
2. **Potential Field:** Each particle is treated as a "metaball"—a source of a potential field that decays with distance.43  
3. **Thresholding:** The shader identifies the "boundary" where the potential equals a specific threshold $T$. Areas where $\\rho \> T$ are rendered as liquid.43

For a gooey look, the "influence" of each metaball is increased, and a lower threshold is used, causing particles to "melt" into each other from a distance.43 This creates the characteristic thick bridges and rounded edges of viscous fluids.

### **Fragment Shader Enhancements for Gooeyness**

The visual "weight" of the fluid is achieved in the fragment shader:

* **Specular Highlights:** Gooey fluids are often highly reflective. By calculating a screen-space normal from the gradient of the density field, the shader can apply sharp specular highlights.6  
* **Chromatic Aberration/Refraction:** For substances like honey or slime, a small amount of refraction (distortion of the background) enhances the sense of volume and viscosity.6  
* **Depth-Based Color:** Darker colors in thicker regions (high density) and lighter colors at the edges provide a sense of transparency and depth.29

## **Conceptual Workflow Summary for Godot 4 Developers**

To implement this framework, a developer must synchronize the high-level Godot SceneTree with the low-level compute pipeline.

1. **Initialization:** Create a RenderingDevice and allocate SSBOs for particle data and the spatial hash grid. Pre-calculate SDF textures for static level geometry.23  
2. **Physics Integration:** In \_physics\_process, collect external forces (input, explosions) and send them to the GPU.  
3. **Compute Dispatch:** Execute the PBF steps (Predict $\\to$ Hash $\\to$ Density $\\to$ Project $\\to$ Viscosity).2  
4. **Feedback Loop:** If the fluid must affect RigidBody2D objects (two-way coupling), use BufferGetDataAsync to retrieve particle velocities and apply them as forces to the rigid bodies in the next frame.28  
5. **Rendering:** Pass the particle position buffer to a MultiMeshInstance2D for raw rendering, or use a SubViewport to generate a density map for the Marching Squares shader.45

## **Second-Order Insights: Stability and Performance Optimization**

The intersection of "gooeyness" and "performance" reveals deeper insights into solver behavior. Gooey fluids, due to their high viscosity (XSPH) and cohesion ($s\_{corr}$), are inherently more stable than thin liquids like water.1 The damping of the XSPH step acts as a natural stabilizer, meaning a gooey fluid can often run with fewer PBF iterations (e.g., 2 instead of 5\) while maintaining visual plausibility.2

However, the "stickiness" of gooey fluid near hard objects can lead to a performance trap. If too many particles gather in a single spatial hash cell (e.g., a pool of thick sludge at a corner), the neighbor search becomes $O(N\_{cell}^2)$ for that specific cell.39 Developers should implement "clamping" on the number of neighbors checked per particle to ensure that even in high-density gooey regions, the frame time remains predictable.41

Finally, the launching requirement highlights the importance of the Lagrangian-Eulerian hybrid nature of PBF. While the particles move freely (Lagrangian), the boundary handling and spatial hashing rely on a fixed grid (Eulerian).3 This duality allows the gooey fluid to be "launched" into an unknown, dynamic environment with hard objects without the need for a pre-defined simulation box, provided the spatial hash is large enough or dynamically resizable.1

## **Conclusion: A Robust Path to Viscous Fluids**

The implementation of gooey fluids in Godot 4 using Position Based Fluids provides a robust, real-time solution that balances the heavy mathematical requirements of fluid dynamics with the practical constraints of game development. By leveraging the RenderingDevice for GPU-accelerated constraint projection, utilizing SDFs for smooth hard-object interaction, and carefully tuning XSPH and artificial pressure terms, developers can achieve a high level of physical "thickness" and cohesion. This framework ensures that the fluid responds naturally to gravity and external impulses while maintaining the stable, non-explosive behavior required for a consistent player experience. The synergy between PBF's stability and Godot 4's modern compute pipeline represents a powerful toolset for creating the next generation of interactive, physics-driven 2D environments.

### **Obras citadas**

1. Position Based Fluids \- Miles Macklin, fecha de acceso: enero 20, 2026, [https://mmacklin.com/pbf\_sig\_preprint.pdf](https://mmacklin.com/pbf_sig_preprint.pdf)  
2. (PDF) Position Based Fluids \- ResearchGate, fecha de acceso: enero 20, 2026, [https://www.researchgate.net/publication/260398844\_Position\_Based\_Fluids](https://www.researchgate.net/publication/260398844_Position_Based_Fluids)  
3. Smoothed-particle hydrodynamics \- Wikipedia, fecha de acceso: enero 20, 2026, [https://en.wikipedia.org/wiki/Smoothed-particle\_hydrodynamics](https://en.wikipedia.org/wiki/Smoothed-particle_hydrodynamics)  
4. Particle-Based Fluid Simulation for Interactive Applications \- Clemson University, fecha de acceso: enero 20, 2026, [https://people.computing.clemson.edu/\~dhouse/courses/817/papers/mueller03.pdf](https://people.computing.clemson.edu/~dhouse/courses/817/papers/mueller03.pdf)  
5. Position Based Dynamics \- GitHub Pages, fecha de acceso: enero 20, 2026, [https://matthias-research.github.io/pages/publications/posBasedDyn.pdf](https://matthias-research.github.io/pages/publications/posBasedDyn.pdf)  
6. Real-time fluid simulation using compute shaders : r/godot \- Reddit, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/godot/comments/1jlcdfp/realtime\_fluid\_simulation\_using\_compute\_shaders/](https://www.reddit.com/r/godot/comments/1jlcdfp/realtime_fluid_simulation_using_compute_shaders/)  
7. ADAPTIVE POSITION-BASED FLUIDS: IMPROVING PERFORMANCE OF FLUID SIMULATIONS FOR REAL-TIME APPLICATIONS \- arXiv, fecha de acceso: enero 20, 2026, [https://arxiv.org/pdf/1608.04721](https://arxiv.org/pdf/1608.04721)  
8. Adaptive Position-Based Fluids: Improving Performance of Fluid Simulations For Real-Time Applications \- Scribd, fecha de acceso: enero 20, 2026, [https://www.scribd.com/document/320261154/Adaptive-Position-Based-Fluids-Improving-Performance-of-Fluid-Simulations-for-Real-Time-Applications](https://www.scribd.com/document/320261154/Adaptive-Position-Based-Fluids-Improving-Performance-of-Fluid-Simulations-for-Real-Time-Applications)  
9. Understanding viscosity: adhesion vs cohesion \- Flux-pumps.co.uk, fecha de acceso: enero 20, 2026, [https://www.flux-pumps.co.uk/blog/useful-info/understanding-viscosity-adhesion-vs-cohesion/](https://www.flux-pumps.co.uk/blog/useful-info/understanding-viscosity-adhesion-vs-cohesion/)  
10. Raising stickiness in collision material makes particles spastic but not sticky \- Obi, fecha de acceso: enero 20, 2026, [https://obi.virtualmethodstudio.com/forum/thread-3376.html](https://obi.virtualmethodstudio.com/forum/thread-3376.html)  
11. Position Based Fluids \- graphics memdump, fecha de acceso: enero 20, 2026, [http://andychan.dev/notes/simulation/position-based-fluids/](http://andychan.dev/notes/simulation/position-based-fluids/)  
12. Assignment 2: Position-Based Fluids, fecha de acceso: enero 20, 2026, [https://www.cs.cmu.edu/\~scoros/cs15467-s16/assignments/A2.pdf](https://www.cs.cmu.edu/~scoros/cs15467-s16/assignments/A2.pdf)  
13. Position-Based Dynamics Simulation \- Emergent Mind, fecha de acceso: enero 20, 2026, [https://www.emergentmind.com/topics/position-based-dynamics-pbd-simulation](https://www.emergentmind.com/topics/position-based-dynamics-pbd-simulation)  
14. CS 348C Position Based Fluids in Houdini \- Stanford Computer Graphics Laboratory, fecha de acceso: enero 20, 2026, [https://graphics.stanford.edu/courses/cs348c-20-winter/HW\_PBF\_Houdini/index.html](https://graphics.stanford.edu/courses/cs348c-20-winter/HW_PBF_Houdini/index.html)  
15. 82 Cohesion and Adhesion in Liquids: Surface Tension and Capillary Action, fecha de acceso: enero 20, 2026, [https://openbooks.lib.msu.edu/collegephysics1/chapter/cohesion-and-adhesion-in-liquids-surface-tension-and-capillary-action-2/](https://openbooks.lib.msu.edu/collegephysics1/chapter/cohesion-and-adhesion-in-liquids-surface-tension-and-capillary-action-2/)  
16. Position Based Fluids | Brandon G. Nguyen, fecha de acceso: enero 20, 2026, [https://people.engr.tamu.edu/sueda/courses/CSCE450/2022F/projects/Brandon\_Nguyen/index.html](https://people.engr.tamu.edu/sueda/courses/CSCE450/2022F/projects/Brandon_Nguyen/index.html)  
17. The Sci Guys: Science at Home \- SE2 \- EP7: Viscosity of Liquids \- YouTube, fecha de acceso: enero 20, 2026, [https://www.youtube.com/watch?v=f6spBkVeQ4w](https://www.youtube.com/watch?v=f6spBkVeQ4w)  
18. Position-Based Surface Tension Flow \- Dartmouth Computer Science, fecha de acceso: enero 20, 2026, [https://www.cs.dartmouth.edu/\~bozhu/papers/pb\_st\_flow.pdf](https://www.cs.dartmouth.edu/~bozhu/papers/pb_st_flow.pdf)  
19. Real-time sim of slimy, clay-like and "brittle" fluids by tweaking a cohesion function \- Reddit, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/Simulated/comments/wej0x1/realtime\_sim\_of\_slimy\_claylike\_and\_brittle\_fluids/](https://www.reddit.com/r/Simulated/comments/wej0x1/realtime_sim_of_slimy_claylike_and_brittle_fluids/)  
20. Consistent SPH Rigid-Fluid Coupling \- Computer Animation, fecha de acceso: enero 20, 2026, [https://animation.rwth-aachen.de/media/papers/84/2023-VMV-SPH\_ConsistentBoundaryHandling.pdf](https://animation.rwth-aachen.de/media/papers/84/2023-VMV-SPH_ConsistentBoundaryHandling.pdf)  
21. Resolution-adjusted boundary particles for adaptive SPH: Resolution-adjusted boundary particles...R. Akhunov and A. Kolb \- ResearchGate, fecha de acceso: enero 20, 2026, [https://www.researchgate.net/publication/398874544\_Resolution-adjusted\_boundary\_particles\_for\_adaptive\_SPH\_Resolution-adjusted\_boundary\_particlesR\_Akhunov\_and\_A\_Kolb](https://www.researchgate.net/publication/398874544_Resolution-adjusted_boundary_particles_for_adaptive_SPH_Resolution-adjusted_boundary_particlesR_Akhunov_and_A_Kolb)  
22. Signed Distance Fields \- Render Diagrams, fecha de acceso: enero 20, 2026, [https://renderdiagrams.org/2017/12/28/signed-distance-fields/](https://renderdiagrams.org/2017/12/28/signed-distance-fields/)  
23. GPUParticlesCollisionSDF3D — Godot Engine (4.4) documentation in English, fecha de acceso: enero 20, 2026, [https://docs.godotengine.org/en/4.4/classes/class\_gpuparticlescollisionsdf3d.html](https://docs.godotengine.org/en/4.4/classes/class_gpuparticlescollisionsdf3d.html)  
24. 2D Signed Distance Field Basics | Ronja's tutorials, fecha de acceso: enero 20, 2026, [https://www.ronja-tutorials.com/post/034-2d-sdf-basics/](https://www.ronja-tutorials.com/post/034-2d-sdf-basics/)  
25. Stanford CS348C: Prog. Assignment \#1: Position Based Fluids, fecha de acceso: enero 20, 2026, [http://graphics.stanford.edu/courses/cs348c-23-winter/PA1\_PBF2016/index.html](http://graphics.stanford.edu/courses/cs348c-23-winter/PA1_PBF2016/index.html)  
26. Impulse Fluid Simulation \- CS@Dartmouth, fecha de acceso: enero 20, 2026, [https://www.cs.dartmouth.edu/\~bozhu/papers/impulse\_fluid.pdf](https://www.cs.dartmouth.edu/~bozhu/papers/impulse_fluid.pdf)  
27. Particle-Based Fluid-Fluid Interaction \- Computer Graphics Laboratory, fecha de acceso: enero 20, 2026, [https://cgl.ethz.ch/Downloads/Publications/Papers/2005/Mue05a/Mue05a.pdf](https://cgl.ethz.ch/Downloads/Publications/Papers/2005/Mue05a/Mue05a.pdf)  
28. Would Compute Shaders Be Helpful for Finite Difference PDE Solving? : r/godot \- Reddit, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/godot/comments/1mtqhg9/would\_compute\_shaders\_be\_helpful\_for\_finite/](https://www.reddit.com/r/godot/comments/1mtqhg9/would_compute_shaders_be_helpful_for_finite/)  
29. Water and lava real-time simulation using compute shaders : r/godot \- Reddit, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/godot/comments/1k9325v/water\_and\_lava\_realtime\_simulation\_using\_compute/](https://www.reddit.com/r/godot/comments/1k9325v/water_and_lava_realtime_simulation_using_compute/)  
30. Using compute shaders — Godot Engine (latest) documentation in English, fecha de acceso: enero 20, 2026, [https://docs.godotengine.org/en/latest/tutorials/shaders/compute\_shaders.html](https://docs.godotengine.org/en/latest/tutorials/shaders/compute_shaders.html)  
31. RenderingDevice — Godot Engine (stable) documentation in English, fecha de acceso: enero 20, 2026, [https://docs.godotengine.org/en/stable/classes/class\_renderingdevice.html](https://docs.godotengine.org/en/stable/classes/class_renderingdevice.html)  
32. The Godot Barn \~ Articles / A glance at Compute Shaders, fecha de acceso: enero 20, 2026, [https://thegodotbarn.com/contributions/article/219/a-glance-at-compute-shaders](https://thegodotbarn.com/contributions/article/219/a-glance-at-compute-shaders)  
33. SSBO alignment question \- OpenGL: Advanced Coding \- Khronos Forums, fecha de acceso: enero 20, 2026, [https://community.khronos.org/t/ssbo-alignment-question/75614](https://community.khronos.org/t/ssbo-alignment-question/75614)  
34. Optimizing 2D Physics Spatial Hashing \- Cristiano Politowski, fecha de acceso: enero 20, 2026, [https://cpoli.live/blog/2025/spatial-hashing/](https://cpoli.live/blog/2025/spatial-hashing/)  
35. \[Help\] Compute shaders: std430 and alignment : r/godot \- Reddit, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/godot/comments/1ok6lol/help\_compute\_shaders\_std430\_and\_alignment/](https://www.reddit.com/r/godot/comments/1ok6lol/help_compute_shaders_std430_and_alignment/)  
36. Shader Storage Buffer Objects (SSBOs) \- J Stephano, fecha de acceso: enero 20, 2026, [https://ktstephano.github.io/rendering/opengl/ssbos](https://ktstephano.github.io/rendering/opengl/ssbos)  
37. GPU synchronization in Godot 4.3 is getting a major upgrade, fecha de acceso: enero 20, 2026, [https://godotengine.org/article/rendering-acyclic-graph/](https://godotengine.org/article/rendering-acyclic-graph/)  
38. Intro to Godot 4's RenderingDevice (Compute Shaders) \- YouTube, fecha de acceso: enero 20, 2026, [https://www.youtube.com/watch?v=OR3eJxrgdAw](https://www.youtube.com/watch?v=OR3eJxrgdAw)  
39. Optimized Spatial Hashing for Collision Detection of Deformable Objects \- GitHub Pages, fecha de acceso: enero 20, 2026, [https://matthias-research.github.io/pages/publications/tetraederCollision.pdf](https://matthias-research.github.io/pages/publications/tetraederCollision.pdf)  
40. Having Difficulties With "RenderingDevice.BufferGetDataAsync" (Godot 4.4 .NET) \- Reddit, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/godot/comments/1jht4n2/having\_difficulties\_with/](https://www.reddit.com/r/godot/comments/1jht4n2/having_difficulties_with/)  
41. When to Use Spatial Hashing vs Bounding Volume Hierarchy? \- Game Development Stack Exchange, fecha de acceso: enero 20, 2026, [https://gamedev.stackexchange.com/questions/124186/when-to-use-spatial-hashing-vs-bounding-volume-hierarchy](https://gamedev.stackexchange.com/questions/124186/when-to-use-spatial-hashing-vs-bounding-volume-hierarchy)  
42. Optimizing Particle Systems with a Grid Lookup and Spatial Hashing \- Gorilla Sun, fecha de acceso: enero 20, 2026, [https://www.gorillasun.de/blog/particle-system-optimization-grid-lookup-spatial-hashing/](https://www.gorillasun.de/blog/particle-system-optimization-grid-lookup-spatial-hashing/)  
43. Metaballs / blobby objects \- Matias' Programming Blog, fecha de acceso: enero 20, 2026, [https://matiaslavik.wordpress.com/computer-graphics/metaball-rendering/](https://matiaslavik.wordpress.com/computer-graphics/metaball-rendering/)  
44. Performance analysis of different surface reconstruction algorithms for 3D reconstruction of outdoor objects from their digital images \- NIH, fecha de acceso: enero 20, 2026, [https://pmc.ncbi.nlm.nih.gov/articles/PMC4929111/](https://pmc.ncbi.nlm.nih.gov/articles/PMC4929111/)  
45. 2D Surface Reconstruction: Marching Squares with Meta-balls \- Iradicator on Games, fecha de acceso: enero 20, 2026, [https://iradicator.com/2d-surface-reconstruction-marching-squares-with-meta-balls/](https://iradicator.com/2d-surface-reconstruction-marching-squares-with-meta-balls/)  
46. I've been working on Metaballs too, though with a very different approach\! : r/Unity3D, fecha de acceso: enero 20, 2026, [https://www.reddit.com/r/Unity3D/comments/5sufr7/ive\_been\_working\_on\_metaballs\_too\_though\_with\_a/](https://www.reddit.com/r/Unity3D/comments/5sufr7/ive_been_working_on_metaballs_too_though_with_a/)  
47. Everything About Textures in Compute Shaders\! \- NekotoArts, fecha de acceso: enero 20, 2026, [https://nekotoarts.github.io/blog/Compute-Shader-Textures](https://nekotoarts.github.io/blog/Compute-Shader-Textures)  
48. How to draw textures created on the local rendering device? \- Shaders \- Godot Forum, fecha de acceso: enero 20, 2026, [https://forum.godotengine.org/t/how-to-draw-textures-created-on-the-local-rendering-device/115726](https://forum.godotengine.org/t/how-to-draw-textures-created-on-the-local-rendering-device/115726)  
49. Journey into SPH Simulation: A Comprehensive Framework and Showcase \- arXiv, fecha de acceso: enero 20, 2026, [https://arxiv.org/html/2403.11156v1](https://arxiv.org/html/2403.11156v1)  
50. Particle-Based Simulation of Fluids \- Scientific Computing and Imaging Institute, fecha de acceso: enero 20, 2026, [https://www.sci.utah.edu/\~tolga/pubs/ParticleFluidsHiRes.pdf](https://www.sci.utah.edu/~tolga/pubs/ParticleFluidsHiRes.pdf)
