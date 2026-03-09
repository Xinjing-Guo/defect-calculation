# Formation Energy Diagram & Thermodynamic Analysis

## Construction of the Formation Energy Diagram

### What the diagram shows

- **x-axis:** Fermi level E_F (from VBM=0 to CBM=band_gap)
- **y-axis:** Defect formation energy E_f (eV)
- **Each line:** One charge state of one defect, with slope = charge state q
- **Lower envelope:** The stable charge state at each Fermi level
- **Transition levels:** Kinks in the lower envelope where the stable charge state changes
- **Band edges:** Shaded regions for VB (left) and CB (right)
- **E_f = 0 line:** Dashed horizontal line — defects with E_f < 0 form spontaneously

### Mathematical construction

For each defect X in charge state q, the formation energy is linear in E_F:

```
E_f[X^q](E_F) = E_f[X^q](E_F=0) + q × E_F
```

Where E_f[X^q](E_F=0) is the formation energy at the VBM:
```
E_f[X^q](E_F=0) = (E_defect + corrections) - E_bulk + Σ(n_i × μ_i) + q × E_VBM
```

### Finding the lower envelope (stable charge states)

For each defect, at any given E_F, the stable charge state is the one with the lowest E_f. The lower envelope can be found using:

1. **Pairwise intersection:** For charge states q1 and q2, they intersect at:
   ```
   E_F* = [E_f(q1, E_F=0) - E_f(q2, E_F=0)] / (q2 - q1)
   ```
   If E_F* is in the band gap, it's a transition level ε(q1/q2).

2. **Halfspace intersection** (scipy, as used in doped): More efficient for many charge states. Each charge state defines a half-space; the lower convex hull gives the stable envelope.

### Transition level classification

| Location | Classification | Physical meaning |
|----------|---------------|-----------------|
| Near VBM (< 0.3 eV) | Shallow acceptor | Easy to ionize at room temperature |
| Near CBM (> E_gap - 0.3 eV) | Shallow donor | Easy to ionize at room temperature |
| Mid-gap (0.3 to E_gap - 0.3) | Deep level | Recombination center / trap |
| Outside band gap | Not accessible | Defect always in one charge state |

### Negative-U behavior

When a transition level ε(q1/q3) skips an intermediate charge state q2, the defect has "negative-U" character. Example: ε(0/-2) without a stable -1 state means V_Cd captures two electrons simultaneously.

---

## Plotting with doped

### Basic plot

```python
from doped.thermodynamics import DefectThermodynamics

thermo = DefectThermodynamics(
    defect_entries,
    chempots=chempots,
    vbm=vbm,
    band_gap=band_gap
)

# Plot at a specific chemical potential limit
fig = thermo.plot(limit="Te-rich")
fig.savefig("formation_energy_Te-rich.pdf")
```

### Customized plot

```python
fig = thermo.plot(
    limit="Te-rich",
    all_entries="faded",        # Show unstable states faded
    xlim=(-0.3, 1.8),          # Extend slightly beyond band edges
    ylim=(-1, 5),
    fermi_level=0.7,           # Mark a specific Fermi level
    colormap="tab10",
    chempot_table=True,        # Show chemical potential table
    auto_labels=True,          # Annotate transition levels
    filename="TL_diagram.pdf"
)
```

### Getting numerical data

```python
# Transition levels
tls = thermo.get_transition_levels()
thermo.print_transition_levels()

# Formation energies at specific conditions
df = thermo.get_formation_energies(
    limit="Te-rich",
    fermi_level=0.5  # eV above VBM
)
print(df)  # pandas DataFrame with all defect entries
```

---

## Manual Plotting (without doped)

```python
import numpy as np
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 6))
band_gap = 1.5  # eV
E_F = np.linspace(0, band_gap, 200)

# Define defects: {name: [(charge, E_f_at_VBM), ...]}
defects = {
    r"$V_{\mathrm{Cd}}$": [(0, 2.8), (-1, 2.3), (-2, 1.5)],
    r"$V_{\mathrm{Te}}$": [(0, 3.5), (+1, 2.8), (+2, 1.9)],
    r"$\mathrm{Te}_{\mathrm{Cd}}$": [(0, 3.0), (-1, 2.7), (-2, 2.2)],
    r"$\mathrm{Cl}_{\mathrm{Te}}$": [(0, 1.2), (+1, 0.5)],
}

colors = plt.cm.tab10(np.linspace(0, 1, len(defects)))

for (name, charge_states), color in zip(defects.items(), colors):
    # Compute formation energy for all charge states
    E_f_all = np.array([E_f0 + q * E_F for q, E_f0 in charge_states])
    # Lower envelope
    E_f_min = np.min(E_f_all, axis=0)
    ax.plot(E_F, E_f_min, color=color, lw=2, label=name)

    # Optionally plot individual charge states (faded)
    for i, (q, E_f0) in enumerate(charge_states):
        E_fi = E_f0 + q * E_F
        ax.plot(E_F, E_fi, color=color, lw=0.8, alpha=0.3, linestyle='--')

# Band edges
ax.axvspan(-0.5, 0, alpha=0.15, color='cornflowerblue')
ax.axvspan(band_gap, band_gap + 0.5, alpha=0.15, color='orange')
ax.axhline(0, color='k', linestyle='--', alpha=0.3, lw=0.8)

ax.set_xlim(-0.2, band_gap + 0.2)
ax.set_ylim(-0.5, 5)
ax.set_xlabel("Fermi Level (eV)", fontsize=14)
ax.set_ylabel("Formation Energy (eV)", fontsize=14)
ax.legend(fontsize=11, loc='upper left')
plt.tight_layout()
plt.savefig("formation_energy_diagram.pdf", dpi=300)
```

---

## Equilibrium Fermi Level

### Charge neutrality equation

```
Σ_d [q_d × c_d(E_F, T)] + p(E_F, T) - n(E_F, T) + Q_ext = 0
```

Where:
- c_d = defect concentration of defect d with charge q_d
- p = hole concentration
- n = electron concentration
- Q_ext = external dopant charge (optional)

### Defect concentrations (Boltzmann statistics)

```
c_d = N_sites × g_d × exp(-E_f[d] / k_B T)
```

- N_sites = number of available sites per unit volume
  - For vacancies: number of host atoms of that species per volume
  - For interstitials: number of interstitial sites per volume
  - For substitutions: number of host sites of the substituted species per volume
- g_d = total degeneracy = spin_degeneracy × site_degeneracy
- k_B T ≈ 0.026 eV at 300 K

### Carrier concentrations

```
n = ∫_{E_CBM}^{∞} g(E) × f(E, E_F, T) dE

p = ∫_{-∞}^{E_VBM} g(E) × [1 - f(E, E_F, T)] dE
```

Where g(E) is the density of states and f is the Fermi-Dirac distribution.

In practice, use a calculated DOS from VASP (ISMEAR=-5, NEDOS≥3000) for accurate carrier concentrations.

### Self-consistent solution

Use root-finding (e.g., scipy.optimize.brentq) to find E_F that satisfies charge neutrality. The formation energy of each charged defect depends on E_F, which in turn depends on defect concentrations — this makes it self-consistent.

### Annealing / quenching scenario

Real materials are often synthesized at high temperature T_anneal and then cooled:
1. Compute equilibrium E_F and defect concentrations at T_anneal
2. "Freeze" defect concentrations (they can't migrate at low T)
3. Re-solve for E_F at T_measure with frozen defect concentrations but equilibrated carriers

```python
# In doped:
thermo.get_fermi_level_and_concentrations(
    annealing_temperature=800,    # K, where defects equilibrate
    quenched_temperature=300,     # K, where carriers equilibrate
    chempots=chempots,
    limit="Te-rich"
)
```

---

## Interpreting Results

### Formation energy diagram tells you:

1. **Dominant defects** — lowest E_f at the equilibrium Fermi level
2. **Conductivity type** — if E_F pins near VBM → p-type; near CBM → n-type
3. **Compensation** — if both donors and acceptors have low E_f → compensation limits doping
4. **Killer defects** — deep levels with low E_f that cause recombination
5. **Dopability** — if no defect drives E_f to mid-gap, the material can be doped

### Common patterns:

| Pattern | Meaning |
|---------|---------|
| V_cation has lowest E_f at VBM | Intrinsic p-type tendency |
| V_anion has lowest E_f at CBM | Intrinsic n-type tendency |
| Deep acceptor + deep donor both low | Fermi level pinning (hard to dope) |
| Dopant has lower E_f than compensating defect | Effective doping |
| Negative formation energy | Spontaneous defect formation (phase instability) |
