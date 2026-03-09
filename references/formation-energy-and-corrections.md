# Formation Energy Calculation & Finite-Size Corrections

## Formation Energy Formula

```
E_f[X^q] = E_tot[X^q] - E_tot[bulk] + Σ(n_i × μ_i) + q × (E_VBM + E_Fermi) + E_corr
```

### Term-by-Term from VASP Static Calculations

| Term | Source | How to extract |
|------|--------|---------------|
| E_tot[X^q] | Defect static OUTCAR | `grep "energy  without entropy" OUTCAR \| tail -1` |
| E_tot[bulk] | Host static OUTCAR | Same grep command |
| n_i | Defect structure | Count atoms added/removed vs bulk |
| μ_i | Chemical potential analysis | From elemental energies + formation enthalpy |
| E_VBM | Host static EIGENVAL/OUTCAR | Highest occupied eigenvalue at Gamma |
| q | INCAR NELECT | q = NELECT_neutral - NELECT |
| E_corr | FNV correction | Madelung + potential alignment |

### Atom Count Convention (n_i)

n_i is **positive** when atoms are **removed**, negative when **added**:

| Defect | n_Cd | n_Te | Chemical potential term |
|--------|------|------|----------------------|
| V_Cd | +1 | 0 | +μ_Cd |
| V_Te | 0 | +1 | +μ_Te |
| Te_Cd | -1 | +1 | Deprecated: use → +μ_Cd - μ_Te (Cd removed, Te added) |
| Actually: Te replaces Cd → n_Cd=+1, n_Te=-1 | +1 | -1 | +μ_Cd - μ_Te |
| Cd_Te | -1 | +1 | -μ_Cd + μ_Te |
| Cl_Te | 0 (n_Te=+1, n_Cl=-1) | +1 | +μ_Te - μ_Cl |

### NELECT and Charge State

```
NELECT_neutral = Σ (N_atoms_i × ZVAL_i)   # from POTCAR
q = NELECT_neutral - NELECT_set
```

For CdTe with ZVAL(Cd)=12, ZVAL(Te)=6:
- Bulk 64-atom: NELECT = 32×12 + 32×6 = 576
- V_Cd neutral: NELECT = 31×12 + 32×6 = 564
- V_Cd q=-1: NELECT = 565 (add 1 electron)
- V_Cd q=-2: NELECT = 566 (add 2 electrons)

---

## Freysoldt (FNV) Correction — Detailed

### When to apply
Only for charged defects (q ≠ 0). Neutral defects need no correction.

### Ingredients
1. **Madelung constant** (α_M) for the supercell geometry
2. **Dielectric constant** (ε) — use ε₀ (static, including ionic screening)
3. **Potential alignment** (ΔV) from LOCPOT comparison

### Madelung Constant

Can be obtained from:
- **Analytical:** For cubic supercells, α_M ≈ 2.8373 (Madelung constant for SC lattice)
- **VASP calculation:** Place a test charge in the supercell (see SKILL.md Step 8)
- **From dasp.in:** `madelung = 2.838` for 64-atom cubic CdTe supercell

### Correction Formula

```
E_corr(FNV) = α_M × q² / (2 × ε × L) + q × ΔV_alignment
```

Where:
- L = supercell lattice parameter (Å), converted appropriately
- ε = dielectric constant (dimensionless)
- ΔV = potential alignment from LOCPOT analysis (eV)

### Practical Calculation with LOCPOT

1. Read planar-averaged electrostatic potential from bulk and defect LOCPOT files
2. Compute ΔV(z) = V_defect(z) - V_bulk(z) along each axis
3. Subtract the model charge potential (point charge screened by ε)
4. The residual should be flat far from defect → plateau value = ΔV
5. Average over the flat region

### Typical correction magnitudes

| Charge | Correction magnitude |
|--------|---------------------|
| q = ±1 | 0.1 – 0.3 eV |
| q = ±2 | 0.3 – 1.0 eV |
| q = ±3 | 0.8 – 2.0 eV |
| q = ±4 | 1.5 – 3.5 eV |

The correction scales roughly as q².

---

## Extracting E_VBM

From the host static calculation:

```bash
# Method 1: From EIGENVAL (most reliable)
# Parse the highest occupied eigenvalue

# Method 2: From OUTCAR
grep "E-fermi" Intrinsic_Defect/host/OUTCAR
# For semiconductors, E_fermi from VASP is in the gap;
# the actual VBM is the highest occupied state
```

For Gamma-only calculations, the VBM is the highest occupied eigenvalue at the Gamma point. For multi-k calculations, it's the global maximum of occupied eigenvalues across all k-points.

---

## Extracting Chemical Potentials

### From dasp.in parameters

```
E_pure = -1.7736 -4.6974      # E/atom: Cd, Te elemental bulk
p1 = 0.0 -1.1854              # Cd-rich: Δμ_Cd, Δμ_Te
p2 = -1.1854 0.0              # Te-rich: Δμ_Cd, Δμ_Te
```

The absolute chemical potentials:
```
μ_Cd = E_pure(Cd) + Δμ_Cd
μ_Te = E_pure(Te) + Δμ_Te
```

The formation enthalpy:
```
ΔH_f(CdTe) = Δμ_Cd + Δμ_Te = 0 + (-1.1854) = -1.1854 eV
```

### Computing from scratch

1. Calculate E/atom for elemental Cd (hcp metal) and Te (trigonal)
2. Calculate E/formula_unit for CdTe bulk
3. ΔH_f = E(CdTe) - E(Cd) - E(Te)
4. Cd-rich: Δμ_Cd = 0, Δμ_Te = ΔH_f
5. Te-rich: Δμ_Te = 0, Δμ_Cd = ΔH_f

---

## Complete Example: V_Cd Formation Energy

Given:
- E_tot[V_Cd, q=-2] from static = -360.123 eV
- E_tot[host] from static = -375.456 eV
- n_Cd = +1 (one Cd removed)
- μ_Cd at Cd-rich: E_pure(Cd) + 0 = -1.7736 eV
- E_VBM = 2.345 eV (from host EIGENVAL)
- q = -2
- E_Fermi = variable (x-axis)
- E_corr(FNV) = 0.594 eV (for q=-2)

```
E_f = (-360.123) - (-375.456) + 1×(-1.7736) + (-2)×(2.345 + E_Fermi) + 0.594
    = 15.333 - 1.7736 - 4.690 - 2×E_Fermi + 0.594
    = 9.464 - 2×E_Fermi
```

At VBM (E_Fermi=0): E_f = 9.464 eV
At CBM (E_Fermi=1.45): E_f = 9.464 - 2.9 = 6.564 eV

The slope is -2 (the charge state), and the line goes down as Fermi level increases (charged acceptor becomes more favorable in n-type conditions).

---

## Reporting Corrections in defect_workflow.log

For each charged defect, log:

```
[2024-01-15 14:30:00] STEP: FNV_correction
  Defect: V_Cd q=-2
  NELECT: 566 (neutral: 564, q=-2)
  Madelung constant: 2.838
  Dielectric constant: 10.3
  Supercell length: 13.258 Å
  ---
  Madelung energy: E_Mad = 2.838 × 4 / (2 × 10.3 × 13.258) = 0.0416 Ry = 0.566 eV
  Potential alignment: ΔV = 0.014 eV
  Alignment correction: q × ΔV = -2 × 0.014 = -0.028 eV
  ---
  Total FNV correction: 0.538 eV
  E_defect (uncorrected): -360.123 eV
  E_defect (corrected): -360.123 + 0.538 = -359.585 eV
```
