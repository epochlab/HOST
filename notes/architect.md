# Project architecture — repo/module map

*Design overview: how `engine.md`, `agent.md`, and `neural.md`'s stages are packaged into
separate, independently-versioned repositories, and how a master project composes them*

## 1. Repo topology overview

- master — submodule pins, build/launch scripts, training/sim harness
  - engine — `engine.md`
  - agent — `agent.md`
  - neural — `neural.md` §1–2 loop wiring
  - cnn — visual cortex feature extraction (sobel filtering now lives in `engine` as an AOV — see `engine.md` §3)
  - transformer — neocortex
  - hopfield — hippocampus
  - rl — basal ganglia (action-value learning + cross-inhibition filter)
  - ring-attractor — motor output
  - mlp — proprioception encoder

Flat submodules: master pins all ten repos directly; `neural` is not a parent repo. Neural's
mechanism repos are named by computational mechanism, not biological region — `neural.md` §3 maps
each to its biological analogue, and that mapping can change without renaming the repo. `neural`
itself owns only loop-wiring (retina frame buffer, dopaminergic gate, sliding context window,
`neural.md` §2/§3) and depends on the mechanism repos as siblings, the same way `agent` depends on
`engine`.

## 2. Repo reference table

| Repo | Owns | Language/stack | Depends on | Spec |
|---|---|---|---|---|
| master | Submodule pins, build/launch, training loop, task config, sim harness | — | all repos below | this doc |
| engine | Renderer: camera → ... → framebuffer, AOV set | C++, OpenGL → CUDA/OptiX (Phase 5) | — | `engine.md` |
| agent | Retina sensor (ray-cast sampling), spatial state | C++ (shares engine's backend) | engine (BVH, AOVs) | `agent.md` |
| neural | Loop wiring: retina buffer, dopaminergic gate, reward channel, context window | Python (§6) | agent, cnn, transformer, hopfield, rl, ring-attractor, mlp | `neural.md` §1–2 |
| cnn | Visual cortex feature extraction | Python, GPU (CUDA/Apple Silicon) | — | `neural.md` §3 |
| transformer | Neocortex reasoning, working-memory context | Python, GPU (CUDA/Apple Silicon) | — | `neural.md` §3 |
| hopfield | Hippocampus storage (Hebbian, not gradient-trained) | Python, GPU (CUDA/Apple Silicon) | — | `neural.md` §3–4 |
| rl | Basal ganglia action-value learning + cross-inhibition filter | Python, GPU (CUDA/Apple Silicon) | — | `neural.md` §3 |
| ring-attractor | Motor output | Python, GPU (CUDA/Apple Silicon) | — | `neural.md` §3 |
| mlp | Proprioception encoding | Python, GPU (CUDA/Apple Silicon) | — | `neural.md` §3 |

## 3. Cross-repo interface contracts

| Interface | Producer | Consumer | Defined in |
|---|---|---|---|
| BVH, AOVs (tiered colour signal — `agent.md` §3 —, Depth, Normals) | engine | agent | `engine.md` §2–3, `agent.md` §3 |
| Packed colour-D-normal signal | agent | neural | `agent.md` §2 ("Signal packing") |
| Motor output → per-step state update | ring-attractor → neural | agent | `agent.md` §2 ("Per-step state update") |
| Position/heading → next frame's scene state | agent | engine | `agent.md` §1 |
| Reward | master (task/episode definition) | rl | `neural.md` §2–3 |

Reward and episode/reset control live in master, not `neural`/`rl` — `rl` consumes a reward scalar,
it doesn't define what earns one.

## 4. Cross-language / cross-device boundary

Engine/Agent are C++; Neural's mechanism repos are tentatively Python on GPU, targeting CUDA or
Apple Silicon (§6). Master's harness is the only repo that crosses the language boundary. Binding
mechanism is open: pybind11 (in-process, no serialization cost) vs. an IPC boundary (shared-memory
ring, ZeroMQ, gRPC — decoupled processes, a copy per step). Deferred pending neural's per-step
compute budget.

Dual GPU-backend support creates an asymmetry with Engine: Phase 5's CUDA/OptiX backend
(`engine.md` §4) is NVIDIA-only, so Engine stays on OpenGL on Apple Silicon while Neural still gets
GPU acceleration there via its portable backend. On NVIDIA this also opens GPU-resident interop for
the packed signal (§3) — e.g. CUDA/OpenGL interop, avoiding a host round-trip — with no Apple
Silicon equivalent. Unresolved; flagged in §6.

## 5. Build/versioning

- Git submodules, flat (§1). No package registry — a submodule pin is enough until a repo is
  needed outside this project tree (Occam's razor default).
- Master's `.gitmodules` pins exact commits, not branches/tags — any checkout of master reproduces
  every module's exact state.
- Each repo tags releases independently; only a pin bump in master pulls in a module's new commit.

## 6. Open questions

| Section | Open question |
|---|---|
| Neural language | Python assumed (`neural.md`'s CNN/transformer/DQN/Hopfield mix reads as a PyTorch stack) but not committed, mirroring `engine.md` §4's own deferred language choice. |
| C++↔Python boundary (§4) | pybind11 vs. IPC — settle once neural's per-step compute budget is known. |
| GPU backend (§4) | Framework must support both CUDA and Apple Silicon without per-backend forks (e.g. PyTorch's CUDA/MPS backends). Whether NVIDIA-only GPU-resident interop with Engine is worth the platform divergence is unresolved. |
| Sub-repo granularity (§1) | One repo per mechanism today; fold any that stay trivially small back into `neural` rather than keeping them as submodules. |
| Master harness scope | Owns reward/task/episode definition (§3); whether it also owns checkpointing/experiment tracking is undecided. |

## 7. Code quality standards

Cross-cutting — applies to every repo in §2, no exceptions.

Baseline (Occam's razor): correct first, simple always, fast where it matters. No new
abstraction or dependency without checking an existing one first — three similar lines beats a
premature abstraction. No dead code, no speculative generality, no defensive noise around
scenarios that can't happen. Validate at every system/security boundary (asset loads, the §4 IPC
boundary if chosen, any network input); never swallow an error. NASA/JPL's "Power of Ten" (Holzmann, 2006) 
applied where the domain allows.

| Rule | Applied here |
|---|---|
| Simple control flow, minimal recursion | Engine's recursive path tracing (`engine.md` §2) is the one deliberate exception, bounded by Russian roulette termination — provable termination satisfies the rule's intent. Elsewhere: no goto, no recursion. |
| Every loop has a fixed, provable bound | No unbounded loop without an explicit, provable exit condition. |
| No dynamic allocation after init | GPU buffers/tensors allocated once, reused per step — `engine.md`'s Memory HUD (§2) exists to catch a violation immediately. |
| One function, one screen (~60 lines) | A long function is a signal to split, not a style choice. |
| ≥2 assertions per function, on average | On invariants/preconditions, not on inputs that can't occur at that boundary. |
| Smallest possible variable scope | Applied as written. |
| Check every return value; validate every parameter | At system boundaries per the baseline above; internal calls trust already-validated invariants. |
| Minimal preprocessor use | Header guards and simple config macros only — no macro metaprogramming. |
| Restricted pointer use | C++ repos only; Python's object model isn't the indirection this rule guards against. |
| Zero warnings, static analysis every build | `-Wall -Wextra -Werror` + clang-tidy/cppcheck (C++), ruff/mypy (Python) — gating every commit, not just CI. |

## 8. References

- Git submodules — official Git documentation (`git-scm.com/book/en/v2/Git-Tools-Submodules`).
- pybind11 — seamless C++11/Python interoperability (candidate for §4's in-process binding).
- ZeroMQ / gRPC — candidates for §4's IPC boundary, if chosen over in-process binding.
- Holzmann, G.J. (2006). The Power of Ten: Rules for Developing Safety-Critical Code. IEEE Computer.
