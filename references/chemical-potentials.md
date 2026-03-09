# Chemical Potentials & Thermodynamic Stability Regions

## Mathematical Framework

### Binary Compound (e.g., CdTe)

For CdTe in thermodynamic equilibrium:

```
μ_Cd + μ_Te = μ_CdTe(bulk) = E_CdTe / n_formula_units
```

The formation enthalpy is:
```
ΔH_f(CdTe) = μ_CdTe - μ_Cd(elemental) - μ_Te(elemental)
```

Stability constraints (compound must not decompose):
```
Δμ_Cd ≤ 0    (otherwise metallic Cd precipitates)
Δμ_Te ≤ 0    (otherwise elemental Te precipitates)
Δμ_Cd + Δμ_Te = ΔH_f(CdTe)
```

Where `Δμ_i = μ_i - μ_i(elemental)`.

**Cd-rich limit:** Δμ_Cd = 0, Δμ_Te = ΔH_f(CdTe)
**Te-rich limit:** Δμ_Te = 0, Δμ_Cd = ΔH_f(CdTe)

### Ternary+ Compounds

For ternary (e.g., Cu₂ZnSnS₄), the stability region is a polygon in the multi-dimensional chemical potential space. Additional constraints come from all competing phases:

```
For each competing phase C_a D_b E_c:
  a×Δμ_C + b×Δμ_D + c×Δμ_E ≤ ΔH_f(C_a D_b E_c)
```

The stability region vertices are found by solving systems of linear equations where the host is in equilibrium with exactly (n_elements - 1) competing phases simultaneously.

## Competing Phase Identification

### Using Materials Project

```python
from mp_api.client import MPRester
from pymatgen.analysis.phase_diagram import PhaseDiagram, PDEntry

with MPRester(api_key) as mpr:
    entries = mpr.get_entries_in_chemsys(["Cd", "Te"])

pd = PhaseDiagram(entries)
chempots = pd.get_all_chempots(Composition("CdTe").reduced_composition)
```

### Pruning Strategy (from doped)

Not all phases in the chemical system affect the stability region. Efficient pruning:
1. Start with phases on the convex hull that border the target composition
2. For each metastable phase (within energy_above_hull tolerance, default 0.05 eV/atom):
   - Artificially lower its energy to the hull
   - Check if it would then border the target composition
   - If yes, include it as a candidate for VASP calculation
3. This reduces the number of VASP calculations needed

### Gaseous References

For elements like O, N, H, F, Cl:
- Use molecule-in-a-box calculations (O₂, N₂, etc.)
- Box size: typically 30×30×30 Å
- Use experimental bond lengths
- Apply spin polarization (O₂ is triplet)

## VASP Calculations for Competing Phases

Each competing phase needs a well-converged total energy:

```bash
# INCAR for competing phase relaxation
ENCUT  = 520        # High cutoff for accurate energies
EDIFF  = 1E-6
IBRION = 2
ISIF   = 3          # Full cell relaxation
NSW    = 100
EDIFFG = -0.01
PREC   = Accurate
ISMEAR = 0          # Gaussian for semiconductors
SIGMA  = 0.05
GGA    = PS          # PBEsol (or PE for PBE, use same as defect calcs)
```

**Critical:** Use the same functional, ENCUT, and POTCAR set for ALL phases (bulk, defects, competing phases, elemental references). Mixing functionals introduces systematic errors.

## Computing Chemical Potentials from VASP Energies

```python
import json
import numpy as np
from pymatgen.core import Composition
from pymatgen.analysis.phase_diagram import PhaseDiagram, PDEntry
from pymatgen.entries.computed_entries import ComputedEntry

# Collect all calculated energies
entries = []
entries.append(ComputedEntry("Cd", energy_Cd_per_atom * n_Cd_atoms))
entries.append(ComputedEntry("Te", energy_Te_per_atom * n_Te_atoms))
entries.append(ComputedEntry("CdTe", energy_CdTe))
# Add all competing phases...

pd = PhaseDiagram(entries)

# Get chemical potential limits
bulk_comp = Composition("CdTe").reduced_composition
all_chempots = pd.get_all_chempots(bulk_comp)

# Convert to doped format
el_refs = {str(el): pd.el_refs[el].energy_per_atom for el in pd.elements}

chempots = {"limits": {}, "elemental_refs": el_refs, "limits_wrt_el_refs": {}}
for limit_name, mu_dict in all_chempots.items():
    chempots["limits"][limit_name] = {str(el): mu for el, mu in mu_dict.items()}
    chempots["limits_wrt_el_refs"][limit_name] = {
        str(el): mu - el_refs[str(el)] for el, mu in mu_dict.items()
    }

with open("chempots.json", "w") as f:
    json.dump(chempots, f, indent=2)
```

## Interpreting the Stability Region

- **Narrow stability region** (small |ΔH_f|): Compound is marginally stable, sensitive to growth conditions
- **Wide stability region**: Compound is very stable, more forgiving growth conditions
- **Asymmetric region**: One element has tighter constraints than the other
- **For defects**: The dominant defect type changes across the stability region:
  - Metal-rich: metal interstitials and anion vacancies tend to dominate (n-type)
  - Anion-rich: metal vacancies and anion interstitials tend to dominate (p-type)
