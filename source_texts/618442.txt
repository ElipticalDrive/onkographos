# Task Design Rationale

The idea came from ordinary life. When I was younger, I watched a Brain Games episode where contestants had to fit irregularly shaped objects into a car trunk under a time limit. It stuck with me because the reasoning involved was so clearly spatial: looking at shapes, imagining rotations, and working out what nests inside what.

I keep running into the same kind of thinking in everyday situations. Unloading a dishwasher, you naturally stack cups inside cups and position bowls so their cavities are used. Helping a friend move, I had to load a rolling chair, a shoe rack, a dustbin, and several other bulky items into one trip, which meant trying configurations mentally until one felt stable enough to actually drag.

That made me curious whether frontier models can do this kind of reasoning at all. Humans develop spatial intuition through embodiment and experience. LLMs are trained to predict plausible text sequences, not to build accurate internal models of geometry or physical scenes.

That is why I built the task around 3D object packing. Objects are represented as voxelized point sets in an xyz coordinate system. The goal is to place all of them without overlap while minimizing the total bounding box volume. I liked this setup because it removes unnecessary ambiguity while preserving the real difficulty. The model still has to reason about shape, rotation, translation, nesting, and efficient use of space. It cannot succeed by sounding plausible.

# QC Methodology

I did not treat this as a one-shot text puzzle. The model is given multiple turns and an explicit action space: it can place objects, rotate them along any axis, check for collisions, and iteratively revise its arrangement. The agent is not handicapped; it has all the tools a human solver would want.

The end state is fully objective. A valid solution must place every object from the input, preserve each object exactly without deformation, and avoid any overlapping voxels. Among valid arrangements, the score is determined by how tightly the bounding box fits the reference volume. Pass and fail are unambiguous, and the verifier operates on geometry, not text.

I also did not anchor on my own first solution. I asked a couple of friends to check whether my arrangement was actually optimal, and that turned out to matter - one of them identified a better configuration I had missed. That more optimal arrangement was later independently found by Gemini, which gave me additional confidence that the task was grounded in geometry rather than my personal intuition.

To make this process tractable, I built a 3D visualizer. Since model inputs and outputs are lists of coordinates, inspecting solutions purely by reading text is unreliable. The visualizer let me understand my own arrangements and see exactly what each model was producing, which made it much easier to distinguish genuine reasoning failures from formatting or parsing issues.

The task is fair because object fitting is a real and common problem. Packing luggage, stacking dishes, loading a car trunk — these are normal human activities. The task is not trying to trick the model. It is trying to isolate a capability that actually matters and that current models still struggle with.

# Training Signal Value

At CMU, one of my course projects involves kernel optimization. My intuition is that many hard optimization problems require an ability to reason concretely about structure and arrangement, not just pattern-match on surface features. Kernel optimization is not literally object packing, but both tasks demand the ability to mentally manipulate structure under hard constraints.

I think this task targets a real and underexplored weakness in current language models: explicit spatial reasoning over structured objects under constraints. Models that improve on this kind of task would likely become better at practical problems in robotics, manipulation planning, and logistics. Warehouse automation is one obvious example.

There are many axes along which a harder version of this task could be built:
- Physical properties like weight distribution and rigidity for stability
- Richer object representations beyond voxels
- Object interactions like friction, magnetism, or chemical compatibility

More generally, this task forces a model to track multiple interacting structures, simulate the effect of transformations, and search over combinatorial arrangements while respecting hard constraints. That is a fundamentally different skill from recalling facts or generating fluent prose. It is much closer to the kind of mental simulation humans use for physical problems — the same kind I found valuable for visualizing physics olympiad problems in high school. Spatial reasoning datasets of this kind could improve model performance on structured, constraint-heavy tasks well beyond packing.

# Time Estimate

This took about 15 to 20 hours in total, spread across roughly three days. A meaningful portion of that time went into hand-crafting examples, checking fairness, and building the 3D visualizer from scratch so I could properly inspect voxel arrangements. That tool turned out to be important both for finding a correct human solution and for understanding what the models were actually producing.

# Oracle Trajectory

The human solution (validated by me) is in `harbor-task/solution/`. The entry point is `solve.sh`, which calls the same `place_object.py` tools the agent uses. To validate it end-to-end, run:

```bash
bash harbor-task/solution/verify.sh
```

This builds the task Docker image, executes `solve.sh` inside it, then runs the verifier and asserts `score=1.0000`. The script exits non-zero on any failure, so a clean exit is a reliable indicator of a correct solve.

# Model Results

All trial results are in `evaluation/results/3d-voxel-packing-frontier-eval/`. Each subdirectory contains the agent trajectory, the artifact (`solution.json`), and the verifier output including `reward.txt`. The pass@2 results across two independent runs per model (temperature=0) are:

| Model | Run 1 | Run 2 | pass@2 |
|---|---|---|---|
| GPT-5.4 | 0.8259 | 0.8500 | 0 / 2 |
| Claude Opus 4.6 | 0.9444 | 0.8947 | 0 / 2 |
| Gemini 3.1 Pro | **1.0000** | 0.8500 | 1 / 2 |

Gemini solved the task once but not consistently. GPT and Opus both produced spatially reasonable arrangements but left meaningful volume slack in both runs. No model reliably achieved score=1.0000, which matches the intended difficulty target.
