# Defect Calculation Skill for Claude Code

A comprehensive Claude Code skill for **point defect calculations** in semiconductors and insulators using VASP. Guides users through the complete workflow from primitive cell to formation energy diagrams, with full SLURM job automation.

## Features

- **Multi-level accuracy framework**: LEVEL1 (PBE+PBE), LEVEL2 (PBE+HSE), LEVEL3 (HSE+HSE)
- **AEXX fitting**: Automatic determination of hybrid functional mixing parameter by fitting to experimental band gap
- **K-mesh convergence testing**: Mandatory convergence tests for primitive cell calculations with auto-determination for supercells
- **Automated SLURM workflow**: Background monitor (`auto_monitor.sh`) that tracks job convergence and automatically prepares/submits the next calculation phase
- **Optional AI review**: `--claude` flag enables Claude-powered review of each converged calculation before advancing
- **Finite-size corrections**: Built-in FNV (Freysoldt) correction for charged defects
- **Formation energy diagrams**: Automatic plotting of formation energy vs Fermi level with charge state labels and transition levels

## Workflow

```
Step 0: K-mesh convergence test + AEXX determination (LEVEL2/3)
Step 1: Supercell construction (nearly-cubic, 60-70 atoms)
Step 2: Bulk relaxation (ISIF=3)
Step 3: Defect structure generation (vacancies, antisites, substitutionals)
Step 4: Neutral defect relaxation (ISIF=2)
Step 5: Charge state determination (EIGENVAL analysis)
Step 6: Charged defect relaxation
Step 7: Static calculations (LVHAR, ICORELEVEL)
Step 8: Finite-size corrections (FNV)
Step 9: Formation energy calculation and diagram plotting
```

Each step has strict dependencies enforced by the automation system. All operations are logged to `defect_workflow.log`.

## Directory Structure

```
ProjectRoot/
├── Bulk/
│   ├── ktest/          # K-mesh convergence test (primitive cell)
│   ├── AEXX/           # AEXX fitting (LEVEL2/3)
│   ├── relax/          # Bulk supercell relaxation
│   └── statics/        # Bulk static calculation
├── Defect/
│   ├── Intrinsic/      # Vacancies, antisites
│   │   └── V_X/
│   │       ├── q0/relax/, q0/statics/
│   │       ├── q+1/relax/, q+1/statics/
│   │       └── ...
│   └── Impurity/       # Substitutional impurities
├── FormationEnergy/    # Output plots and logs
├── auto_monitor.sh     # SLURM job automation
├── formation_energy.py # Formation energy calculation
├── fnv_correction.py   # FNV correction script
├── prepare_defect.py   # Defect structure builder
├── defect_config.txt   # Defect definitions
└── formation_config.yaml
```

## Installation

Add to your Claude Code skills:

```bash
mkdir -p ~/.claude/skills/defect-calculation
# Copy SKILL.md and references/ into the directory
```

Or install via Claude Code:

```
/install-skill defect-calculation
```

## Usage

Invoke the skill in Claude Code:

```
/defect-calculation
```

Then describe your system, e.g.:

> "Set up V_O defect calculations for Al2O3 using LEVEL1 framework"

The skill will:
1. Ask for primitive cell source (Materials Project, file, literature)
2. Construct a nearly-cubic supercell
3. Guide through VASP input preparation
4. Set up and submit SLURM jobs
5. Monitor convergence and advance phases automatically
6. Compute corrections and plot formation energy diagrams

## Key Scripts

| Script | Purpose |
|--------|---------|
| `auto_monitor.sh` | Background SLURM monitor, auto-advances workflow phases |
| `formation_energy.py` | Reads VASP results, applies FNV corrections, plots diagrams |
| `fnv_correction.py` | Freysoldt correction for charged defects |
| `prepare_defect.py` | Builds defect POSCARs from bulk CONTCAR |
| `fit_aexx.py` | Fits AEXX to experimental band gap (LEVEL2/3) |

## Configuration

### formation_config.yaml

```yaml
level: 1                    # 1=PBE+PBE, 2=PBE+HSE, 3=HSE+HSE
bulk_static: Bulk/statics
epsilon: 10.0                # Dielectric constant
band_gap: 5.85               # Band gap (eV)
mu_O_ref: -4.9275            # 1/2 * E(O2)
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

- VASP 5.4+ (with `vasp_gam` and `vasp_std`)
- Python 3.6+ with NumPy, matplotlib, PyYAML
- SLURM workload manager
- Claude Code (for skill invocation and optional AI review)
