# Quantum-Inspired Optimization for Logistics Routing

### An honest benchmark of QAOA and VQE against classical solvers on the Traveling Salesman Problem

A reproducible Jupyter notebook that formulates a 5-city TSP as a QUBO / Ising Hamiltonian, solves it with **QAOA** and **VQE** on a noiseless Qiskit Aer simulator, and measures both against exact and classical baselines: **brute force, Held-Karp dynamic programming, a Genetic Algorithm (DEAP), Simulated Annealing, and Nearest-Neighbor + 2-opt / Or-opt local search**.

The project deliberately favors rigor over hype. It reports multi-seed variance, a hyperparameter sweep, an optimizer comparison, and significance tests, and states plainly where the quantum methods fall short.

---

## Table of Contents

- [Headline Findings](#headline-findings)
- [Why Only 5 Cities?](#why-only-5-cities)
- [What's Inside](#whats-inside)
- [Results](#results)
- [How the Problem Is Encoded](#how-the-problem-is-encoded)
- [Notebook Map](#notebook-map)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Outputs](#outputs)
- [Limitations and Caveats](#limitations-and-caveats)
- [Extending the Project](#extending-the-project)
- [License](#license)

---

## Headline Findings

1. **Classical methods solve this instance optimally, almost instantly.** GA, Simulated Annealing, Held-Karp and NN + 2-opt + Or-opt all reach the true optimum (tour length 20.4936) in milliseconds, on every one of 5 seeds.
2. **Baseline QAOA does not reliably find a feasible tour.** With 16 qubits, `reps=3` and a 60-iteration COBYLA loop, only 1 of 4,096 final shots satisfied the constraints. Across 5 seeds, only 2 produced any feasible tour.
3. **Feasibility sits at the random-sampling level.** Of the 2^16 = 65,536 bitstrings, only 4! = 24 encode valid tours, so uniform random sampling would be feasible about 0.037% of the time (24 / 65,536). Baseline QAOA's observed 0.02% valid-shot fraction is in that same range, i.e. the penalty-based QUBO landscape is not being steered toward feasibility at this depth and budget.
4. **Structure helps more than depth.** A feasibility-preserving **XY mixer** raised the valid-shot fraction to 99.8% (versus about 0% for baseline QAOA) and ran in 12 s, though on this run its best tour was still 13% above optimal.
5. **Warm-starting needs careful reading.** Warm-started QAOA reached the optimum, but its initial state is the classical optimal tour, so the result reflects the classical input rather than a quantum speedup.
6. **Classical solvers scale; the quantum approach does not (yet).** At 50 cities the QUBO would need 49^2 = 2,401 qubits, beyond any simulator and current hardware. The classical pipeline solves 50 cities in milliseconds.

## Why Only 5 Cities?

QAOA needs one binary variable per (city, position) pair. Fixing the start city reduces this to `(N-1)^2` qubits:

| Cities (N) | Qubits needed | Statevector size |
|---|---|---|
| 5 | 16 | 65,536 amplitudes (practical limit on a laptop) |
| 10 | 81 | ~2.4 x 10^24 amplitudes |
| 50 | 2,401 | not simulable |

The notebook is scoped to the largest size where QAOA can be run exactly, so that the comparison is real rather than extrapolated. Larger instances (10, 20, 50 cities) are covered with classical methods only.

## What's Inside

**Exact methods**
- Brute-force enumeration (24 tours at N = 5), used as ground truth
- Held-Karp dynamic programming, validated against brute force and run on a 13-city instance (589,824 DP states instead of 479,001,600 permutations)

**Classical heuristics**
- Genetic Algorithm (DEAP: ordered crossover, shuffle mutation, tournament selection)
- Simulated Annealing (2-swap neighborhood, geometric cooling)
- Nearest-Neighbor construction with 2-opt and Or-opt local search

**Quantum variational methods (Qiskit Aer, noiseless statevector)**
- Baseline QAOA with a hand-written optimization loop (full control over logging and the simulator backend)
- VQE with a hardware-efficient `TwoLocal` ansatz (128 parameters)
- Warm-started QAOA (bit-flip initial state from a classical tour)
- XY-mixer QAOA (feasibility-preserving ring mixer with one-hot initial state)
- Optimizer comparison: COBYLA vs SPSA vs L-BFGS-B
- A guarded template for running on real IBM Quantum hardware (not executed in this notebook)

**Rigor and reporting**
- Multi-seed robustness sweep (5 seeds per method)
- One-at-a-time QAOA hyperparameter sensitivity (`reps`, `shots`, `maxiter`, `penalty`)
- Paired significance tests (Wilcoxon signed-rank, paired t-test, Cohen's d)
- Asymmetric cost-matrix experiment (one-way / directional costs)
- Interactive `ipywidgets` dashboard and an automated report export (CSV, PNG, JSON, HTML)

## Results

All numbers below come from the saved run of the notebook on a **5-city synthetic instance** with coordinates `(0,0), (2,4), (5,2), (6,6), (1,5)`. The optimal tour is `(0, 1, 4, 3, 2)` with length **20.4936**.

### Main comparison

| Method | Tour length | Gap to optimal | Runtime | Notes |
|---|---|---|---|---|
| Brute force | 20.4936 | 0.0% | n/a | 24 permutations |
| Held-Karp (exact DP) | 20.4936 | 0.0% | 0.0004 s | |
| NN + 2-opt + Or-opt | 20.4936 | 0.0% | 0.0004 s | plain NN alone: 14.6% gap |
| Simulated Annealing | 20.4936 | 0.0% | 0.003 s | |
| Genetic Algorithm | 20.4936 | 0.0% | 0.18 s | |
| QAOA (baseline) | 22.7272 | 10.9% | 70.1 s | 1 / 4,096 valid shots |
| VQE (TwoLocal) | 22.7272 | 10.9% | 116.7 s | 0.6% valid shots |
| QAOA, warm-started | 20.4936 | 0.0% | 46.6 s | 7.1% valid shots; seeded with the classical optimum |
| QAOA, XY mixer | 23.1664 | 13.0% | 12.0 s | 99.8% valid shots |

### Multi-seed robustness (5 seeds)

| Method | Feasible runs | Mean gap | Mean runtime |
|---|---|---|---|
| GA | 5 / 5 | 0.0% | 0.16 s |
| Simulated Annealing | 5 / 5 | 0.0% | 0.002 s |
| QAOA | 2 / 5 | 19.7% (std 14.7, over feasible runs) | 68.8 s |

Three of five QAOA seeds never sampled a single constraint-satisfying bitstring in 4,096 shots.

### QAOA hyperparameter sensitivity (one parameter varied at a time)

| Parameter | Values tested | Observation |
|---|---|---|
| `reps` | 1, 2, 3, 5 | Gap 13.0%, 9.3%, 9.3%, 0.0% |
| `shots` | 100, 300, 600 | No feasible tour, 9.3%, 0.0% |
| `maxiter` | 20, 60, 100 | No feasible tour, 9.3%, 9.3% |
| `penalty` | 10, 25, 50 | No feasible tour, 9.3%, 30.1% |

Valid-shot fractions stayed at or below 0.07% in every configuration, so the cases that reached the optimum did so from a handful of feasible samples among thousands. They are single-seed results and should not be read as a reliable trend.

### Optimizer comparison (same ansatz and budget)

| Optimizer | Best tour | Gap | Runtime | Evaluations |
|---|---|---|---|---|
| COBYLA | 22.3988 | 9.3% | 49.5 s | 48 |
| SPSA | none feasible | n/a | 181.5 s | 171 |
| L-BFGS-B (finite-difference) | 22.7272 | 10.9% | 7.3 s | 7 |

### Classical scalability (QAOA/VQE excluded by design)

| Cities | NN + 2-opt | GA | SA |
|---|---|---|---|
| 10 | 58.06 | 58.06 | 66.94 |
| 20 | 77.33 | 111.70 | 117.96 |
| 50 | 113.62 | 315.48 | 267.91 |

With the fixed 100-iteration budgets used here, GA and SA degrade at larger sizes while NN + 2-opt stays strong, a useful reminder that metaheuristics need budgets that scale with the instance.

### Asymmetric costs

With directional cost differences up to 2.14, the optimal tour changes to `(0, 2, 3, 4, 1)` with length 17.5055, and a tour is no longer interchangeable with its reverse. This also means 2-opt's segment reversal no longer applies directly, whereas Or-opt still does.

## How the Problem Is Encoded

Binary variable `x[c,p] = 1` if city `c` is visited at position `p`. City 0 is fixed as the start and end.

- **Objective:** start leg + middle legs `x[c1,p] * x[c2,p+1] * D[c1,c2]` + end leg
- **Constraints:** each free city appears in exactly one position; each free position holds exactly one city
- **Penalty method:** `QuadraticProgramToQubo(penalty=25.0)` folds the constraints into the objective
- **Hamiltonian:** `to_ising()` yields 16 qubits and 100 Pauli terms
- **QAOA circuit:** `QAOAAnsatz`, 6 parameters at `reps=3`, transpiled depth 190, run on `AerSimulator(method="statevector")`
- **Read-out:** 4,096 final shots, post-selecting the best valid tour; infeasible bitstrings (a position with 0 or 2+ cities, or a repeated city) are discarded

## Notebook Map

| Section | Topic |
|---|---|
| 1-2 | Setup, instance loading (tries TSPLIB `burma14`, falls back to synthetic) |
| 3-4 | Brute-force ground truth, QUBO / Ising formulation |
| 5-9 | QAOA, GA, Simulated Annealing, results table, convergence curves |
| 10-11 | Discussion, tour visualization |
| 12-13 | Multi-seed robustness, hyperparameter sensitivity |
| 14 | VQE with a hardware-efficient ansatz |
| 15-16 | Classical local search (NN, 2-opt, Or-opt), Held-Karp exact DP |
| 17 | Statistical significance testing |
| 18-21 | Warm-started QAOA, XY-mixer QAOA, optimizer comparison |
| 22 | IBM Quantum hardware template (guarded, not executed) |
| 23-24 | Classical scalability to 50 cities, asymmetric cost matrix |
| 25-28 | Master config reference, automated report export, summary, dashboard |

## Getting Started

### Prerequisites

- Python 3.10 or newer
- About 8 GB of RAM recommended (16-qubit statevector simulation)

### Installation

```bash
git clone https://github.com/<your-username>/quantum-inspired-routing.git
cd quantum-inspired-routing

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install qiskit qiskit-aer qiskit-algorithms qiskit-optimization \
            deap networkx matplotlib numpy scipy pandas requests ipywidgets jupyter
```

The first notebook cell also installs any missing core package automatically.

### Run

```bash
jupyter notebook quantum_inspired_routing_tsp5_new.ipynb
```

Run the cells top to bottom. Approximate cost on a laptop:

| Section | Approximate time |
|---|---|
| Single QAOA run (Section 5) | about 1 minute |
| Multi-seed sweep (Section 12, 5 seeds) | about 6 minutes |
| Hyperparameter sweep (Section 13) | about 10 minutes |
| SPSA in the optimizer comparison (Section 21) | about 3 minutes |

Reduce `N_SEEDS` or the sweep grids for a quick pass. Sections that were not run are simply omitted from the final report instead of raising errors.

### Optional: real quantum hardware

Section 22 contains a template that submits the optimized QAOA circuit through Qiskit Runtime's Sampler to the least-busy IBM Quantum backend with enough qubits. It is disabled by default. Set `RUN_ON_REAL_HARDWARE = True` and configure your IBM Quantum account to use it. Note that the results in this repository come only from the noiseless simulator.

## Configuration

Key parameters (each defined in its own section; Section 25 lists them all):

| Group | Parameter | Value |
|---|---|---|
| Instance | `N_CITIES` | 5 |
| QAOA | `reps` / `shots` / `maxiter` / `final_shots` / `penalty` | 3 / 300 / 60 / 4096 / 25.0 |
| VQE | `reps` / `shots` / `maxiter` | 3 / 300 / 60 |
| GA | `generations` / `pop_size` | 100 / 30 |
| SA | `iterations` | 100 |
| Robustness | `n_seeds` | 5 |
| Sweep grid | `reps`, `shots`, `maxiter`, `penalty` | [1,2,3,5], [100,300,600], [20,60,100], [10,25,50] |
| Scalability | sizes | 10, 20, 50 |

## Outputs

Section 26 writes a timestamped folder, `tsp5_report_<timestamp>/`, containing:

- 7 CSV tables: `main_results`, `robustness_sweep`, `robustness_summary`, `hyperparameter_sweep`, `significance_tests`, `optimizer_comparison`, `scalability`
- A headline comparison figure (PNG)
- `summary.json` with the key numbers
- `report.html` tying everything together

## Limitations and Caveats

- **Instance source.** The saved run used the built-in synthetic 5-city instance because the TSPLIB download timed out. Reconnecting to the network switches to the first 5 cities of `burma14`.
- **Single small instance.** Conclusions apply to one 5-city problem. A larger set of random instances would be needed for general claims.
- **Simulator only.** Everything quantum ran noiseless on Aer. Real NISQ hardware adds noise that would further hurt constraint satisfaction.
- **Statistical power.** With 5 seeds, and with QAOA seeds that produced no feasible tour excluded, the significance tests had only 2 usable pairs for QAOA vs GA/SA (Wilcoxon p = 0.5, paired t-test p = 0.31). Treat them as a demonstration of the method. Use 20-30+ seeds before citing significance.
- **Run-to-run variance.** QAOA results differ between runs; for example, baseline COBYLA gave 22.7272 in Section 5 and 22.3988 in the sensitivity sweep.
- **Warm start is not a speedup.** Its initial state is the classical optimum, so matching that optimum is expected.
- **XY mixer scope.** The mixer protects position-uniqueness only; city-uniqueness is not guaranteed by the mixer alone. In the saved run it still reached 99.8% valid shots, but with a 13% gap, from a single seed.
- **Fixed budgets.** GA and SA used a fixed 100-iteration budget even at 50 cities, which understates what tuned versions can do.
- **No quantum advantage is claimed or observed.**

## Extending the Project

- Increase `N_SEEDS` and test on many random 5-city instances for stronger statistics
- Try deeper QAOA (`reps` > 5), multi-start optimization, or CVaR-based objectives
- Add a mixer that also protects city-uniqueness (full permutation-preserving mixer)
- Tune the penalty weight per instance, or use constraint-aware encodings
- Run the template on IBM Quantum hardware, then add noise models to the simulator for comparison
- Investigate quantum-inspired evolutionary algorithms or cluster-decomposed QAOA for larger routing problems
- Port the notebook into a script or package driven by a `config.yaml`

## License

Distributed under the MIT License. See `LICENSE` for details.
