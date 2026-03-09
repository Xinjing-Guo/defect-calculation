# Defect Calculation Skill for Claude Code

A Claude Code skill that guides users through the **complete point defect calculation workflow** in semiconductors and insulators using VASP — from primitive cell to formation energy diagrams.

## What This Skill Does

This skill covers every step of charged point defect calculations in crystalline materials:

### 1. Structure Preparation
- **Primitive cell acquisition** from Materials Project, ICSD, literature, or user-provided files
- **Supercell construction** — automatically finds a nearly-cubic supercell transformation matrix within a target atom count range (e.g., 60–200 atoms)
- **Defect structure generation** — creates vacancies (remove atom), antisites (swap species), and substitutional impurities (replace with foreign atom) from the relaxed bulk structure
- **K-mesh determination** — auto-determines k-mesh via k-spacing formula for supercells; mandatory convergence testing for primitive cell calculations

### 2. VASP Calculations
- **Bulk relaxation** (ISIF=3) — full cell + ion relaxation of the supercell
- **Neutral defect relaxation** (ISIF=2) — fix cell shape, relax ions around the defect
- **Charge state analysis** — compares defect vs bulk EIGENVAL to identify defect levels (partial occupation, in-gap states, extra states), determines possible charge states and NELECT values
- **Charged defect relaxation** — starts from neutral CONTCAR, adds/removes electrons via NELECT
- **Static calculations** — single-point with LVHAR=.TRUE. (for LOCPOT) and ICORELEVEL=1, using ICHARG=1 from relaxation CHGCAR

### 3. Post-Processing
- **Finite-size corrections** — Freysoldt (FNV) correction for charged defects: Madelung energy + potential alignment from LOCPOT comparison
- **Chemical potential analysis** — determines stability region and chemical potential limits for the host compound
- **Formation energy calculation** — computes E_f(q, E_F) = E_defect - E_bulk + chemical potential terms + q(E_VBM + E_F) + E_corr
- **Transition level identification** — finds charge-state transition levels epsilon(q1/q2) within the band gap
- **Formation energy diagram** — plots E_f vs Fermi level with lower envelope, charge state labels at correct segment positions, and band edge shading

### 4. Three-Level Accuracy Framework

| Level | Relaxation | Statics | Use Case |
|-------|-----------|---------|----------|
| LEVEL1 | PBE | PBE | Screening, trend analysis |
| LEVEL2 | PBE | HSE06 | Publication-quality energetics |
| LEVEL3 | HSE06 | HSE06 | Benchmark, wide-gap insulators |

For LEVEL2/3, the skill handles HSE-specific parameters (ALGO=Damped/All, LREAL=.FALSE., PRECFOCK, TIME) and supports **AEXX fitting** — running HSE calculations on the primitive cell at multiple AEXX values to fit the experimental band gap via linear regression.

### 5. Full Automation

- **SLURM job automation** — `auto_monitor.sh` runs in background, checks convergence every 10 minutes, automatically prepares next-phase input files and submits jobs
- **Phase-aware workflow** — enforces strict dependencies: bulk relax → defect structures → neutral relax → charge state analysis → charged relax → statics → corrections → formation energy
- **Operation logging** — every step is timestamped and logged to `defect_workflow.log` with full details (structure source, EIGENVAL analysis, NELECT values, energies, corrections)
- **Optional AI review** (`--claude` flag) — Claude reviews each converged calculation for anomalies (energy oscillation, unconverged SCF, abnormal forces, structure collapse) before advancing

## Highlights

- **Zero manual intervention** — from `sbatch` to formation energy plot, the monitor handles all intermediate steps automatically
- **Charge state determination from first principles** — no need to guess charge states; the skill analyzes EIGENVAL band structure to identify defect levels and determine which electrons can be added/removed
- **Correct formation energy diagrams** — lower envelope plotting with charge state labels positioned at the midpoint of each segment on the thermodynamic ground state line
- **Robust k-mesh handling** — convergence-tested k-mesh for small cells, Gamma-only for supercells, with user choice (auto / manual / convergence test)
- **HSE done right** — proper ALGO selection (Damped for relax, All for static), LREAL=.FALSE., adjusted POTIM/EDIFFG/NELM, and AEXX fitting workflow

## Project Structure

```
ProjectRoot/
├── Bulk/
│   ├── ktest/             # K-mesh convergence test (primitive cell)
│   ├── AEXX/              # AEXX fitting (LEVEL2/3)
│   │   ├── aexx_0.20/     # HSE static at different AEXX values
│   │   ├── aexx_0.25/
│   │   ├── aexx_0.30/
│   │   └── fit_aexx.py
│   ├── relax/             # Bulk supercell relaxation
│   └── statics/           # Bulk static calculation
├── Defect/
│   ├── Intrinsic/         # Vacancies, antisites
│   │   └── V_X/
│   │       ├── q0/relax/, q0/statics/
│   │       ├── q+1/relax/, q+1/statics/
│   │       └── ...
│   └── Impurity/          # Substitutional impurities
├── FormationEnergy/       # Output: plots (.pdf/.png) and logs
├── auto_monitor.sh        # SLURM job automation
├── formation_energy.py    # Formation energy calculation & plotting
├── fnv_correction.py      # FNV correction for charged defects
├── prepare_defect.py      # Defect structure builder
├── defect_config.txt      # Defect type definitions
└── formation_config.yaml  # Formation energy parameters
```

## Installation

```bash
mkdir -p ~/.claude/skills/defect-calculation
# Copy SKILL.md and references/ into the directory
```

## Usage

Invoke the skill in Claude Code with `/defect-calculation`, then describe your system:

> "Set up V_O defect calculations for Al2O3 using LEVEL1 framework"

The skill will guide you through each step — asking for structural source, choosing calculation level, determining k-mesh and AEXX (if applicable), preparing all input files, submitting jobs, and producing final results.

## Configuration Files

### formation_config.yaml

```yaml
level: 1                     # 1=PBE+PBE, 2=PBE+HSE, 3=HSE+HSE
bulk_static: Bulk/statics
epsilon: 10.0                 # Static dielectric constant
band_gap: 5.85                # Band gap (eV), use HSE value for LEVEL2/3
mu_O_ref: -4.9275             # 1/2 * E(O2) reference
chemical_potential_limits:
  O-rich: 0.0
  O-poor: -10.077
defects:
  - name: V_O
    path: Defect/Intrinsic/V_O
    n_O_removed: 1
```

### defect_config.txt

```
# category  name  operation  atom_index  replace_element  charge_states
intrinsic  V_O  vacancy  25  -  +1,+2
```

## Reference Documents

- `references/chemical-potentials.md` — Stability region and chemical potential determination
- `references/formation-energy-and-corrections.md` — Formation energy formalism and FNV/Kumagai corrections
- `references/formation-energy-diagram.md` — Plotting and thermodynamic analysis
- `references/workflow-automation.md` — SLURM scripts and monitoring

## Requirements

- VASP 5.4+ (`vasp_gam` and `vasp_std`)
- Python 3.6+ with NumPy, matplotlib, PyYAML
- SLURM workload manager
- Claude Code (for skill invocation and optional AI review)
