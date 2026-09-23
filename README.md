# HOST

An embodied agent simulation: a real-time physically based renderer paired with a biologically-grounded cognitive architecture.

![Sample render](sample.png)

## Overview

- **Renderer** (design doc and code in [`pathtracer/`](pathtracer/README.md)): a CPU path tracer (Intel Embree) built progressively from a direct-lighting ray tracer toward full spectral, unbiased global illumination. A GPU backend is roadmap, not shipped.
- **Agent sensor & spatial state** (`notes/agent.md`): a point-sampled retinal sensor that casts rays into the pathtracer's scene and packs RGB-D-normal signal for the agent.
- **Cognitive architecture** (`notes/neural.md`): a recurrent loop (Retina → Visual cortex → Neocortex → Basal ganglia → Motor output) modelled on anatomical analogues.

The three pieces close a loop. The pathtracer renders the environment; the agent's retina samples it; the cognitive architecture decides on an action; motor output updates the agent's state in the environment.

## Notes

Design documents live under [notes/](notes/). The pathtracer's design doc lives in [pathtracer/README.md](pathtracer/README.md) instead, alongside its code:

- [agent.md](notes/agent.md): retina sensor and spatial state
- [neural.md](notes/neural.md): cognitive architecture
- [architect.md](notes/architect.md): architecture planning notes
