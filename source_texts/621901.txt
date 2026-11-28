# Benchmark interpretation and reproduction

`reported_results.csv` contains historical measurements transcribed from the provided report, including accumulated rendering times. Throughput ratio is CUDA steps/s divided by serial steps/s; rendering ratio is serial rendering time divided by CUDA rendering time. These are not new measurements.

## Observed limitations in the uploaded snapshot

1. The CPU `idle()` loop accumulates wall-clock time and advances fixed time steps, effectively pacing the simulation around 240 steps/s. The CUDA `idle()` loop launches three steps each iteration without equivalent pacing. Their steps/s ratio mixes scheduling and computation.
2. CPU ball state uses doubles; CUDA uses floats. CPU collisions mutate states sequentially; CUDA computes pair deltas from a shared state then accumulates them atomically. Matching parameters does not guarantee identical trajectories.
3. The CPU draws GLUT spheres; CUDA generates vertex geometry and uses shared VBOs. Tessellation and rendering paths differ, so rendering measurements do not isolate interoperability alone.
4. Only ten balls (45 pairs) are used. No scaling study or repeated-run dispersion is provided for this snapshot.
5. Default source duration is 50 s and `TOTAL_RUNS` is 1. Historical report durations are 100, 300, and 720 s. Raw logs and the exact source revision for those experiments were not supplied.
6. The report does not document an exact CPU/GPU model, driver/toolkit version, build flags, VSync setting, or complete original experiment environment.

## Recommended next experiment

- Add a headless benchmark running an equal, fixed number of steps without wall-clock pacing in both implementations.
- Match initial states, parameters, collision ordering policy, and precision, or document intentional differences and compare states with tolerances.
- Warm up both implementations; time CPU physics with a steady clock and CUDA work with synchronized CUDA events. Separate end-to-end measurements from kernel measurements.
- Measure rendering separately with matched geometry and frame counts. Record VSync and graphics-device selection.
- Run at least five trials and report medians plus dispersion; retain raw logs.
- Record hardware, software versions, build flags, source commit, duration, step count, and configuration.

The historical values remain useful context, but should be described as reported application throughput rather than verified pure kernel speedup.
