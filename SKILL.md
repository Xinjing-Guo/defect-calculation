---
name: defect-calculation
description: Expert assistant for point defect calculations in semiconductors/insulators — supercell construction, bulk relaxation, defect structure generation, neutral/charged defect optimization, static calculations, finite-size corrections, formation energy diagrams, and SLURM job automation. Covers the full workflow with detailed operation logging.
allowed-tools: ["*"]
---

# Defect Calculation Skill

You are an expert in point defect calculations for semiconductors and insulators using VASP. You guide users through the **complete workflow** and **log every operation** to a file in the working directory.

## CRITICAL: Operation Logging

**Every operation MUST be logged** to `defect_workflow.log` in the current working directory。The log must include:
- Timestamp for each step
- **原胞来源：** 从哪获取的（数据库名+材料ID / 文献DOI / 用户文件路径），空间群、晶格参数
- Supercell matrix and size rationale
- Each defect structure: what was created, how, which atom removed/replaced/added
- **电荷态分析：** bulk EIGENVAL 的 VBM/CBM，各缺陷 EIGENVAL 中识别出的缺陷能级（band index、能量、占据数），判定逻辑，最终确定的电荷态列表和对应 NELECT 值
- NELECT values and charge state determination logic
- Job submission IDs and status
- Correction values and method details
- Formation energy results

Log format:
```
[YYYY-MM-DD HH:MM:SS] STEP: <step_name>
  Description: <what was done>
  Details: <specifics>
  Result: <outcome>
```

**日志示例 — 原胞来源：**
```
[2026-03-08 10:00:00] STEP: Primitive Cell Acquisition
  Description: 获取 CdTe 原胞结构
  Source: Materials Project, mp-406
  Space group: F-43m (216)
  Lattice: a=b=c=6.629 Å, α=β=γ=60°
  Atoms: 2 (Cd: 1, Te: 1)
  File: primitive_POSCAR
```

**日志示例 — 电荷态分析：**
```
[2026-03-08 14:00:00] STEP: Charge State Analysis - V_Cd
  Description: 分析 V_Cd 中性缺陷的 EIGENVAL，确定可能的电荷态
  Bulk reference:
    VBM: Band 286-288, E=1.790 eV (3-fold degenerate)
    CBM: Band 289, E=2.358 eV
    Band gap: 0.57 eV
  Defect levels identified:
    Band 281-283: E=1.751 eV, occ_up=0.667, occ_down=0.667
    Criteria: partial occupation + near VBM
    Electrons in defect levels: 3×0.667×2 = 4.0 e⁻
    Empty states in defect levels: 2.0
  Charge states determined: q=0, q=-1, q=-2
  NELECT values: q0=564, q-1=565, q-2=566
  Reasoning: 3-fold defect level near VBM, 4 electrons present,
             can accept 2 more → q=-1, q=-2
```

---

## Calculation Level Framework

Three accuracy levels are supported. The level determines which functional is used for relaxation and static calculations:

| Level | Relaxation | Statics | Accuracy | Cost | Typical Use |
|-------|-----------|---------|----------|------|-------------|
| **LEVEL1** | PBE | PBE | Standard | Low | Screening, trend analysis |
| **LEVEL2** | PBE | HSE06 | High | Medium | Publication-quality energetics |
| **LEVEL3** | HSE06 | HSE06 | Highest | Very High | Benchmark, wide-gap insulators |

**Key differences between PBE and HSE INCAR parameters:**

| Parameter | PBE | HSE06 |
|-----------|-----|-------|
| IALGO / ALGO | IALGO = 38 | ALGO = Damped (relax) / All (static) |
| LHFCALC | (absent) | .TRUE. |
| HFSCREEN | (absent) | 0.2 |
| AEXX | (absent) | 0.25 |
| TIME | (absent) | 0.4 |
| PRECFOCK | (absent) | Fast |
| LREAL | Auto | .FALSE. |
| POTIM | 0.5 | 0.3 (relax only, smaller for stability) |
| EDIFFG | -0.01 | -0.02 (relax only, slightly relaxed) |
| NELM | 100 | 100 (relax) / 200 (static, needs more SCF steps) |

**Important notes for HSE calculations:**
- `IALGO = 38` is **incompatible** with HSE — must use `ALGO = Damped` (relax) or `ALGO = All` (static)
- `LREAL = .FALSE.` is required for HF accuracy (no real-space projection)
- HSE relaxation is ~10-50× more expensive than PBE
- HSE static is ~10-20× more expensive than PBE
- For LEVEL2, band_gap in formation_config.yaml should use the HSE value (not PBE)
- Chemical potential references (mu_O_ref, etc.) should also be recalculated at the HSE level for LEVEL2/3

### AEXX Determination (LEVEL2/3 必须步骤)

**When user chooses LEVEL2 or LEVEL3, you MUST ask them one of the following three options:**

1. **Standard HSE06** — Use default `AEXX = 0.25`. Simplest, suitable when gap accuracy is not critical.
2. **User-specified AEXX** — User provides a custom AEXX value (e.g., from literature). Use directly.
3. **Fit to experimental band gap** — You search literature for the experimental band gap, then fit AEXX by running HSE calculations at multiple AEXX values on the **primitive cell**.

**If user chooses Option 3 (fit to experiment), follow this procedure:**

#### Step A: Find Experimental Band Gap

Use web search tools (Tavily, WebSearch) to find the experimental band gap of the bulk material. Search queries like:
- "{material_name} experimental band gap"
- "{material_name} optical band gap measurement"

Record the source (paper DOI, review article, or textbook) and the gap value. Log to `defect_workflow.log`:
```
[YYYY-MM-DD HH:MM:SS] STEP: AEXX Fitting - Experimental Band Gap
  Description: 搜索 {material} 实验带隙
  Source: {DOI or reference}
  Experimental band gap: {value} eV
  Type: {direct/indirect, optical/fundamental}
```

#### Step B: Prepare AEXX Fitting Calculations

All fitting calculations go in `Bulk/AEXX/`. Use the **primitive cell** (NOT the supercell) with a dense k-mesh for accurate band gap determination.

**Directory structure:**
```
Bulk/AEXX/
├── primitive_POSCAR          # Primitive cell (from original source or relaxed)
├── aexx_0.20/                # HSE static at AEXX=0.20
│   ├── POSCAR INCAR KPOINTS POTCAR submit_job
├── aexx_0.25/                # HSE static at AEXX=0.25
│   ├── POSCAR INCAR KPOINTS POTCAR submit_job
├── aexx_0.30/                # HSE static at AEXX=0.30
│   ├── POSCAR INCAR KPOINTS POTCAR submit_job
├── aexx_0.35/                # (optional, if needed for wide-gap materials)
│   ├── ...
├── fit_aexx.py               # Fitting script
└── aexx_fit.log              # Results
```

**INCAR for AEXX fitting (static on primitive cell):**
```
SYSTEM = AEXX fitting - primitive cell

# Electronic
ENCUT  = <same as supercell>
EDIFF  = 1E-6          # Tighter for accurate gap
PREC   = Accurate
NELM   = 200
LREAL  = .FALSE.
ISMEAR = 0
SIGMA  = 0.05
GGA    = PE

# HSE
LHFCALC  = .TRUE.
HFSCREEN = 0.2
AEXX     = <varies: 0.20, 0.25, 0.30, ...>
ALGO     = All
TIME     = 0.4
PRECFOCK = Fast

# Spin
ISPIN  = 2

# Output
LORBIT = 10
LWAVE  = .FALSE.
LCHARG = .FALSE.

# Parallelization
NPAR   = 8
```

**KPOINTS for primitive cell:** Use the converged k-mesh from `Bulk/ktest/` (see "K-mesh Determination" section). The k-mesh convergence test on the primitive cell is **mandatory** before running AEXX fitting calculations. Example for Al₂O₃ rhombohedral primitive (10 atoms, after convergence test):
```
Gamma
0
G
6 6 3
0 0 0
```

#### Step C: Extract Band Gaps and Fit

After all AEXX calculations converge, extract band gaps from EIGENVAL files and perform linear regression.

**fit_aexx.py:**
```python
#!/usr/bin/env python3
"""
Fit AEXX to reproduce experimental band gap.
Reads EIGENVAL from Bulk/AEXX/aexx_*/
Usage: python fit_aexx.py --exp_gap <value> --aexx_dir Bulk/AEXX/
"""
import numpy as np
import os
import sys
import argparse
from datetime import datetime

def get_band_gap_from_eigenval(eigenval_path):
    """Extract band gap from EIGENVAL (VBM to CBM)"""
    with open(eigenval_path) as f:
        lines = f.readlines()
    header = lines[5].split()
    nelect = int(float(header[0]))
    nkpts = int(header[1])
    nbands = int(header[2])

    vbm = -999.0
    cbm = 999.0
    idx = 7
    for k in range(nkpts):
        idx += 1  # blank line
        idx += 1  # k-point header
        for b in range(nbands):
            parts = lines[idx].split()
            e_up = float(parts[1])
            occ_up = float(parts[2])
            if len(parts) > 3:
                e_dn = float(parts[3])
                occ_dn = float(parts[4])
                if occ_up > 0.5:
                    vbm = max(vbm, e_up)
                else:
                    cbm = min(cbm, e_up)
                if occ_dn > 0.5:
                    vbm = max(vbm, e_dn)
                else:
                    cbm = min(cbm, e_dn)
            else:
                if occ_up > 0.5:
                    vbm = max(vbm, e_up)
                else:
                    cbm = min(cbm, e_up)
            idx += 1
    return cbm - vbm, vbm, cbm

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--exp_gap', type=float, required=True, help='Experimental band gap (eV)')
    parser.add_argument('--aexx_dir', default='Bulk/AEXX/', help='Directory with aexx_* subdirs')
    args = parser.parse_args()

    # Collect data points
    aexx_values = []
    gaps = []
    for d in sorted(os.listdir(args.aexx_dir)):
        if d.startswith('aexx_'):
            aexx = float(d.replace('aexx_', ''))
            eigenval = os.path.join(args.aexx_dir, d, 'EIGENVAL')
            if os.path.exists(eigenval):
                gap, vbm, cbm = get_band_gap_from_eigenval(eigenval)
                aexx_values.append(aexx)
                gaps.append(gap)
                print(f"  AEXX={aexx:.2f}: gap={gap:.4f} eV (VBM={vbm:.4f}, CBM={cbm:.4f})")

    if len(aexx_values) < 2:
        print("ERROR: Need at least 2 AEXX data points for fitting")
        sys.exit(1)

    # Linear fit: gap = a * AEXX + b
    aexx_arr = np.array(aexx_values)
    gap_arr = np.array(gaps)
    coeffs = np.polyfit(aexx_arr, gap_arr, 1)
    a, b = coeffs

    # Solve for target gap: AEXX_fit = (exp_gap - b) / a
    aexx_fit = (args.exp_gap - b) / a

    result = f"""
=== AEXX Fitting Result ===
Date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
Experimental band gap: {args.exp_gap:.4f} eV

Data points:
{''.join(f'  AEXX={av:.2f}: gap={gv:.4f} eV' + chr(10) for av, gv in zip(aexx_values, gaps))}
Linear fit: gap = {a:.4f} * AEXX + {b:.4f}
R² = {1 - np.sum((gap_arr - np.polyval(coeffs, aexx_arr))**2) / np.sum((gap_arr - np.mean(gap_arr))**2):.6f}

>>> Fitted AEXX = {aexx_fit:.4f} <<<
(to reproduce experimental gap of {args.exp_gap:.4f} eV)

Recommendation:
  Round to AEXX = {round(aexx_fit, 2):.2f}
  Verify: expected gap at AEXX={round(aexx_fit, 2):.2f} = {np.polyval(coeffs, round(aexx_fit, 2)):.4f} eV
"""
    print(result)

    log_path = os.path.join(args.aexx_dir, 'aexx_fit.log')
    with open(log_path, 'w') as f:
        f.write(result)
    print(f"Results saved to: {log_path}")

if __name__ == '__main__':
    main()
```

**Usage:**
```bash
python fit_aexx.py --exp_gap 8.8 --aexx_dir Bulk/AEXX/
```

#### Step D: Apply Fitted AEXX

After fitting, update the AEXX value in all HSE INCAR templates. The fitted value replaces the default 0.25 in:
- `write_incar_relax_hse()` (LEVEL3 only)
- `write_incar_static_hse()` (LEVEL2/3)

Log the decision:
```
[YYYY-MM-DD HH:MM:SS] STEP: AEXX Determination
  Description: AEXX fitting complete
  Method: Linear fit to 3 data points
  Experimental gap: 8.8 eV (source: DOI:xxx)
  Fitted AEXX: 0.32
  Used in: all HSE INCAR files
```

**IMPORTANT: The AEXX fitting (Bulk/AEXX/) can run in parallel with the supercell Bulk/relax/ step.** It uses the primitive cell, so it has no dependency on the supercell relaxation. The fitted AEXX value is only needed before writing the first HSE INCAR file (statics for LEVEL2, or relax for LEVEL3).

#### Workflow Integration

```
For LEVEL2:
  Bulk/relax (PBE) ──────────────────────────→ defect relax (PBE) → statics (HSE, AEXX fitted)
  Bulk/ktest → Bulk/AEXX (parallel, prim) ──→ fit AEXX ──────────────↗

For LEVEL3:
  Bulk/ktest → Bulk/AEXX (prim) → fit AEXX → Bulk/relax (HSE) → defect relax (HSE) → statics (HSE)
```

For LEVEL3, the AEXX fitting must complete **before** Bulk/relax starts, since Bulk/relax itself uses HSE.

---

### K-mesh Determination

**When determining k-mesh, you MUST ask the user to choose one of the following options:**

1. **Automatic (recommended)** — Auto-determine k-mesh using k-spacing formula based on cell size
2. **User-specified** — User provides a specific k-mesh (e.g., 4×4×2)
3. **Convergence test** — Run calculations at multiple k-mesh densities to verify convergence

**CRITICAL: For any calculations on small cells (primitive cells, AEXX fitting, etc.), convergence testing is MANDATORY.** Only supercell calculations (60+ atoms) can safely use Gamma-only without testing.

#### K-spacing Formula (Auto Mode)

The k-mesh is determined by maintaining consistent k-point density via k-spacing:

```
N_i = max(1, ceil(b_i / k_spacing))
```

where `b_i = 2π / a_i` is the reciprocal lattice vector length along axis i.

**Recommended k-spacing values:**

| Functional | k_spacing (Å⁻¹) | Use Case |
|---|---|---|
| PBE (standard) | 0.03–0.04 | Primitive cell PBE calculations |
| HSE (production) | 0.04–0.05 | AEXX fitting, HSE statics on primitive cell |
| Coarse | 0.06–0.08 | Quick convergence test starting point |

**Python helper function:**
```python
import numpy as np

def get_kmesh(lattice_vectors, k_spacing=0.04):
    """Determine k-mesh from lattice vectors and target k-spacing.
    lattice_vectors: 3x3 array (rows are vectors, in Å)
    k_spacing: target spacing in Å⁻¹ (smaller = denser)
    Returns: [N1, N2, N3]
    """
    recip = 2 * np.pi * np.linalg.inv(lattice_vectors).T
    kmesh = []
    for i in range(3):
        b_len = np.linalg.norm(recip[i])
        n = max(1, int(np.ceil(b_len / k_spacing)))
        kmesh.append(n)
    return kmesh
```

#### Supercell K-mesh (Automatic — No Test Needed)

For supercells with 60+ atoms (lattice vectors > ~10 Å), **Gamma-only (1×1×1)** is almost always sufficient:

```
Gamma
0
G
1 1 1
0 0 0
```

Rationale: Reciprocal vectors are very short (e.g., `2π/13 ≈ 0.48 Å⁻¹`), so a single k-point adequately samples the BZ. Use `vasp_gam` for speed.

#### Primitive Cell K-mesh (Convergence Test Required)

**For AEXX fitting, band structure, DOS, or any property calculated on the primitive cell, a k-mesh convergence test is MANDATORY.**

**Convergence test procedure:**

1. **Prepare test calculations** in `Bulk/ktest/` with increasing k-mesh density:
```
Bulk/ktest/
├── k1/    # e.g., 2×2×2 (coarse)
│   ├── POSCAR INCAR KPOINTS POTCAR submit_job
├── k2/    # e.g., 4×4×2 (medium)
│   ├── ...
├── k3/    # e.g., 6×6×3 (dense)
│   ├── ...
├── k4/    # e.g., 8×8×4 (very dense, optional)
│   ├── ...
└── ktest_result.log
```

2. **Choose test k-meshes** based on the lattice symmetry and cell dimensions:
   - Start from auto-determined k-mesh at k_spacing=0.06
   - Increase density by ~1.5× each step
   - Ensure meshes respect lattice symmetry (e.g., hexagonal: N1=N2≠N3)
   - For HSE: fewer points needed (HSE converges faster with k-mesh), use k_spacing 0.04–0.06

3. **Run PBE static** at each k-mesh (fast, even if final target is HSE). Compare total energy.

4. **Convergence criterion:** Total energy change < 1 meV/atom between successive meshes.

5. **Select the converged k-mesh** and use it for all subsequent primitive cell calculations (AEXX fitting, etc.).

**ktest INCAR (PBE static on primitive cell):**
```
NELM   = 100
EDIFF  = 1E-6
PREC   = Accurate
ISMEAR = 0
SIGMA  = 0.05
GGA    = PE
IALGO  = 38
LORBIT = 10
ISPIN  = 2
ENCUT  = <same as supercell>
NPAR   = 4
LWAVE  = .FALSE.
LCHARG = .FALSE.
```

**Log example:**
```
[YYYY-MM-DD HH:MM:SS] STEP: K-mesh Convergence Test
  Description: 对 {material} 原胞进行 k-mesh 收敛测试
  Cell: a=4.76 Å, b=4.76 Å, c=12.99 Å (rhombohedral, 10 atoms)
  Test results:
    2×2×1: E = -XXX.XXXX eV  (ΔE = --- meV/atom)
    4×4×2: E = -XXX.XXXX eV  (ΔE = X.X meV/atom)
    6×6×3: E = -XXX.XXXX eV  (ΔE = 0.X meV/atom)  ← CONVERGED
    8×8×4: E = -XXX.XXXX eV  (ΔE = 0.0X meV/atom)
  Selected k-mesh: 6×6×3
  Reasoning: Energy converged to < 1 meV/atom
```

#### Workflow Integration (K-mesh Test + AEXX Fitting)

For LEVEL2/3 with AEXX fitting (Option 3), the k-mesh convergence test should run **before** the AEXX fitting calculations:

```
Bulk/ktest/ (PBE static, fast) → determine converged k-mesh
    ↓
Bulk/AEXX/aexx_*/  (HSE static, use converged k-mesh) → fit AEXX
```

Both can run in parallel with the supercell workflow (Bulk/relax, defect relax) since they use the primitive cell.

---

## Workflow Overview

**CRITICAL: 各步骤存在严格的前后依赖关系，不能提前准备后续步骤的输入文件。**

```
Step 0 (LEVEL2/3): K-mesh Test + AEXX Determination
    询问用户 k-mesh: 自动 / 手动指定 / 收敛测试
    询问用户 AEXX: 标准HSE06(0.25) / 自定义AEXX / 拟合实验带隙
    若选拟合: k-mesh收敛测试(Bulk/ktest/)→搜索文献→AEXX拟合(Bulk/AEXX/)→确定AEXX
    LEVEL2: 可与 Step 1-4 并行（AEXX 在 statics 前确定即可）
    LEVEL3: 必须在 Step 2 前完成（relax 已使用 HSE）
    ↓
Step 1: Supercell Construction
    primitive POSCAR → nearly-cubic supercell (min_atom ~ max_atom)
    → 只准备 Bulk/relax/ 的输入文件 (POSCAR, INCAR, KPOINTS, POTCAR, submit_job)
    ↓
Step 2: Bulk Relaxation (ISIF=3)
    LEVEL1/2: PBE relax | LEVEL3: HSE relax (with fitted AEXX)
    提交 Bulk/relax/ 任务，等待收敛
    ↓ 【必须等 Bulk/relax 完成】
Step 3: Defect Structure Generation + Neutral Relaxation Setup
    用 Bulk/relax/CONTCAR 构建各缺陷 POSCAR
    → 准备所有 Defect/*/q0/relax/ 的输入文件
    ↓
Step 4: Neutral Defect Relaxation (ISIF=2, no NELECT)
    LEVEL1/2: PBE relax | LEVEL3: HSE relax
    提交所有 q0/relax/ 任务（可并行），等待收敛
    ↓ 【必须等所有 q0/relax 完成】
Step 5: Charge State Determination
    分析 q0/relax/EIGENVAL 确定各缺陷的带电态
    ↓
Step 6: Charged Defect Relaxation Setup + Submission
    LEVEL1/2: PBE relax | LEVEL3: HSE relax
    用 q0/relax/CONTCAR 作为起始结构
    → 准备所有 q±n/relax/ 的输入文件，提交任务
    ↓ 【必须等所有 charged relax 完成】
Step 7: Static Calculations Setup + Submission
    LEVEL1: PBE statics | LEVEL2/3: HSE statics
    用各 relax/CONTCAR 准备 statics/ 输入文件
    → Bulk/statics/ + 所有 Defect/*/q*/statics/
    ↓ 【必须等所有 statics 完成】
Step 8: Finite-Size Corrections
    FNV (Freysoldt) or Kumagai correction for charged defects
    ↓
Step 9: Formation Energy & Diagram
    compute E_f, plot formation energy vs Fermi level
```

### 输入文件准备时机与结构来源

| 阶段 | 准备什么 | POSCAR 来源 | 其他依赖 |
|------|---------|------------|---------|
| 初始 | `Bulk/relax/` 的全部输入 | 用户提供的原胞 → 构建超胞 | 无 |
| Bulk收敛后 | 所有 `q0/relax/` 的输入 | `Bulk/relax/CONTCAR`（移除/替换原子） | 无 |
| q0收敛后 | 所有 `q±n/relax/` 的输入 | **`q0/relax/CONTCAR`**（已弛豫的中性缺陷结构） | EIGENVAL分析确定电荷态 |
| 所有relax收敛后 | 所有 `statics/` 的输入 | 各 `relax/CONTCAR` | 各 `relax/CHGCAR`（ICHARG=1） |

**关键：带电缺陷的起始结构是中性弛豫后的 CONTCAR，不是从 bulk 重新构建的缺陷结构。**
这样做的原因是中性缺陷弛豫后原子位置已经松弛到缺陷附近的平衡位置，以此为起点加/减电子再弛豫，收敛更快、结果更可靠。

---

## Step 1: Supercell Construction

**Goal:** Build a nearly-cubic supercell with 60–70 atoms (configurable via min_atom/max_atom).

**Source:** 原胞结构的获取途径（必须记录到日志）：
1. **用户直接提供** — 已有的 POSCAR 或 CIF 文件
2. **数据库下载** — Materials Project、ICSD、COD 等，需记录材料ID（如 mp-406）
3. **文献构建** — 根据实验晶格参数手动构建，需记录文献来源
4. **其他来源** — 如从已有计算结果中提取

**Method:**
1. Read the primitive cell lattice vectors
2. Find a transformation matrix P such that P × primitive_cell ≈ cubic, with N_atoms in [min_atom, max_atom]
3. The supercell should minimize the ratio (max_lattice_length / min_lattice_length) to be as cubic as possible
4. For CdTe zincblende (FCC primitive, 2 atoms): a 4×4×4 FCC = 2×2×2 conventional cubic = 64 atoms

**Log:** Record:
- **原胞来源：** 从哪获取（数据库名+材料ID / 文献DOI / 用户提供的文件路径）
- **原胞信息：** 空间群、晶格参数、原子数
- 超胞变换矩阵、目标原子数、实际原子数
- 超胞晶格向量、cubicity ratio

**Example supercell (CdTe 64 atoms):**
```
Cubic_cell
1.0
13.258  0.000  0.000
 0.000 13.258  0.000
 0.000  0.000 13.258
Cd Te
32 32
```

---

## Step 2: Bulk Relaxation (ISIF=3)

**此步骤是整个流程的起点。初始阶段只准备 `Bulk/relax/` 目录的输入文件。**

**输入文件准备：** 在 `Bulk/relax/` 中放置 POSCAR, INCAR, KPOINTS, POTCAR, submit_job

**INCAR for bulk relaxation (PBE — LEVEL1/2):**
```
NSW = 500
NELM = 100
ISIF = 3              # Full cell + ion relaxation
IBRION = 2
POTIM = 0.5
EDIFF = 1E-5
EDIFFG = -0.01
ISMEAR = 0
SIGMA = 0.05
GGA = PE              # PBE functional
IALGO = 38
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = <1.3×ENMAX>   # From POTCAR
NPAR = 8
```

**INCAR for bulk relaxation (HSE06 — LEVEL3):**
```
NSW = 500
NELM = 100
ISIF = 3              # Full cell + ion relaxation
IBRION = 2
POTIM = 0.3           # Smaller for HSE stability
EDIFF = 1E-5
EDIFFG = -0.02        # Slightly relaxed for HSE
ISMEAR = 0
SIGMA = 0.05
GGA = PE
LHFCALC  = .TRUE.     # Enable Hartree-Fock
HFSCREEN = 0.2         # HSE06 screening
AEXX     = 0.25        # 25% exact exchange
ALGO     = Damped      # Required for HF relaxation (NOT IALGO=38)
TIME     = 0.4
PRECFOCK = Fast
LREAL  = .FALSE.       # Required for HF accuracy
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = <1.3×ENMAX>
NPAR = 8
```

**KPOINTS:** Gamma-only for large supercells (64+ atoms):
```
Gamma
0
G
1 1 1
0 0 0
```

**After relaxation:**
- Use CONTCAR as the relaxed bulk structure
- Record the final energy from OSZICAR
- This CONTCAR is the basis for ALL defect structures

**Log:** Record ENCUT source, k-mesh, final energy, lattice parameters change, convergence status.

---

## Step 3: Defect Structure Generation（Bulk弛豫完成后）

**前置条件：** `Bulk/relax/` 已收敛，CONTCAR 可用。

**操作：** 用 `Bulk/relax/CONTCAR` 构建各缺陷 POSCAR，并准备所有 `q0/relax/` 目录的完整输入文件（POSCAR, INCAR, KPOINTS, POTCAR, submit_job）。

Starting from the relaxed bulk CONTCAR, create defect POSCARs:

### Vacancies
Remove one atom from the supercell:
- **V_Cd:** Remove 1 Cd atom → (N_Cd - 1) Cd + N_Te Te
- **V_Te:** Remove 1 Te atom → N_Cd Cd + (N_Te - 1) Te

### Antisites
Replace one atom with another species:
- **Te_Cd:** Replace 1 Cd with Te → (N_Cd - 1) Cd + (N_Te + 1) Te
- **Cd_Te:** Replace 1 Te with Cd → (N_Cd + 1) Cd + (N_Te - 1) Te

### Impurities (Substitutional)
Replace one host atom with an impurity:
- **Cl_Te:** Replace 1 Te with Cl → N_Cd Cd + (N_Te - 1) Te + 1 Cl
- **Na_Cd:** Replace 1 Cd with Na → (N_Cd - 1) Cd + N_Te Te + 1 Na

### Defect complexes
Create two point defects in the same supercell at various nearest-neighbor distances (nn1, nn2, nn3).

**Log for EACH defect:**
- Which atom was removed/replaced (index, species, fractional coordinates)
- Resulting composition and atom count
- Defect position in the supercell

---

## Step 4: Neutral Defect Relaxation (ISIF=2)

**输入文件已在 Step 3 中准备好（在 `q0/relax/` 目录下）。**

**INCAR for neutral defect relaxation (PBE — LEVEL1/2):**
```
NSW = 500
NELM = 100
ISIF = 2              # Fix cell, relax ions only
IBRION = 2
POTIM = 0.5
EDIFF = 1E-5
EDIFFG = -0.01
ISMEAR = 0
SIGMA = 0.05
GGA = PE
IALGO = 38
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = <same as bulk>
NPAR = 8
```

**INCAR for neutral defect relaxation (HSE06 — LEVEL3):**
```
NSW = 500
NELM = 100
ISIF = 2              # Fix cell, relax ions only
IBRION = 2
POTIM = 0.3
EDIFF = 1E-5
EDIFFG = -0.02
ISMEAR = 0
SIGMA = 0.05
GGA = PE
LHFCALC  = .TRUE.
HFSCREEN = 0.2
AEXX     = 0.25
ALGO     = Damped
TIME     = 0.4
PRECFOCK = Fast
LREAL  = .FALSE.
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = <same as bulk>
NPAR = 8
```

**Key:** No NELECT tag → VASP auto-determines from POTCAR (neutral defect).

**After relaxation:** CONTCAR becomes the starting structure for charged states.

**Log:** Final energy, convergence status, max force, number of ionic steps.

---

## Step 5: Charge State Determination (from EIGENVAL Analysis)

### Core Method: Compare Defect vs Bulk EIGENVAL

After the neutral defect relaxation (Step 4) and the bulk relaxation/static, compare their EIGENVAL files to:
1. Identify the **bulk band structure特征** — VBM, CBM, 以及价带/导带的能量分布模式
2. 在缺陷EIGENVAL中找到**缺陷能级** — 不属于bulk价带或导带的"多出来"的态
3. 分析缺陷能级的**占据情况**
4. 据此确定可能的带电态

### Step 5a: 建立bulk参考 — 提取带边和能带分布特征

EIGENVAL format (ISPIN=2): `band_index  E_up  E_down  occ_up  occ_down`

**首先从host EIGENVAL读取带边附近的完整能带结构：**
```
Host (CdTe 64-atom, NELECT=576, 288 e⁻/spin):

  --- 价带 (全部 occ=1.0) ---
  Band 276-277: E ≈ 1.081 eV   ← 价带态 (2重简并)
  Band 278-285: E ≈ 1.108 eV   ← 价带态 (8重简并)
  Band 286-288: E ≈ 1.790 eV   ← VBM (3重简并，价带顶)
  ─────────── 带隙 0.57 eV ───────────
  Band 289:     E ≈ 2.358 eV   ← CBM (导带底)
  Band 290-293: E ≈ 3.535 eV   ← 导带态 (4重简并)
  --- 导带 (全部 occ=0.0) ---
```

**关键特征要记住：**
- 价带顶(VBM): 1.790 eV (3重简并)
- 导带底(CBM): 2.358 eV
- 带隙: 0.57 eV
- 价带中的简并模式和能量间距

### Step 5b: 在缺陷EIGENVAL中识别缺陷能级

**识别方法 — 三个判据（综合使用）：**

**判据1: 部分占据 (最直接)**
占据数在0和1之间的态 (如 occ=0.667) 一定是缺陷能级。因为bulk中所有态要么完全占据(1.0)要么完全空(0.0)，部分占据只会出现在费米面附近的缺陷态上。

**判据2: 在bulk带隙内出现的态**
能量在bulk VBM (1.790) 和 CBM (2.358) 之间的态是缺陷能级。但要注意：超胞中的本体态会因缺陷影响略有偏移（~0.1 eV），所以需要一定容差。

**判据3: 偏离bulk能带分布模式的"额外"态**
对比bulk和缺陷的能带分布，找到能量位置明显不同于任何bulk能带的态。具体做法：
- bulk中价带顶是3重简并的1.790 eV，下面是8重简并的1.108 eV
- 如果在缺陷EIGENVAL中，在1.108和1.790之间出现了额外的态，或在2.358上方出现了被占据的态，这些就是缺陷态

### Step 5b 实例详解

**V_Cd 中性 (NELECT=564, 282 e⁻/spin):**
```
  Band 275-277: E ≈ 1.128 eV, occ=1.0  ← 对应bulk的 ~1.108价带态 (轻微偏移)
  Band 278-280: E ≈ 1.360 eV, occ=1.0  ← 对应bulk的 ~1.790 VBM? 不对!
                                           bulk VBM是3重简并的1.790
                                           这里变成了1.360, 差了0.43 eV
                                           → 这是价带态,只是被缺陷推移了
  Band 281-283: E ≈ 1.751 eV, occ=0.667 ←←← 缺陷能级!!!
                                           ✓ 判据1: 部分占据(0.667≠0或1)
                                           ✓ 判据2: 在bulk gap附近(1.75, 接近VBM 1.79)
                                           ✓ 判据3: bulk中此位置没有部分占据的3重简并态
  Band 284:     E ≈ 2.382 eV, occ=0.0   ← 对应bulk CBM(2.358), 导带态
```

**V_Te 中性 (NELECT=570, 285 e⁻/spin):**
```
  Band 282-284: E ≈ 1.593 eV, occ=1.0  ← 对应bulk ~1.790 VBM (下移了0.2 eV,正常范围)
  Band 285:     E ≈ 2.187 eV, occ=1.0  ←←← 缺陷能级!!!
                                           ✓ 判据2: 在bulk带隙内 (1.790 < 2.187 < 2.358)
                                           ✓ 判据3: bulk中VBM(3重简并)和CBM之间无态
                                              而这里多出了一个完全占据的态
                                           注意: occ=1.0, 但位置暴露了它
  Band 286:     E ≈ 3.320 eV, occ=0.0  ← 导带态
```

**Te_Cd 中性 (NELECT=570, 285 e⁻/spin):**
```
  Band 282-284: E ≈ 1.849 eV, occ=1.0  ← 对应bulk VBM (~1.790, 轻微上移)
  Band 285:     E ≈ 2.566 eV, occ≈0.82 ←←← 缺陷能级!!!
                                           ✓ 判据1: 部分占据
                                           ✓ 判据3: 在bulk CBM(2.358)上方, 却有电子!
  Band 286-288: E ≈ 2.64-2.72 eV, occ≈0.1 ←←← 缺陷能级!!!
                                           ✓ 判据1: 部分占据
                                           ✓ 判据3: 比CBM还高, 却有残余电子
  Band 289:     E ≈ 3.698 eV, occ=0.0  ← 导带态
```

**Cd_Te 中性 (NELECT=582, 291 e⁻/spin):**
```
  Band 288-290: E ≈ 1.643 eV, occ=1.0  ← 对应bulk VBM
  Band 291:     E ≈ 2.358 eV, occ=1.0  ←←← 缺陷能级!!!
                                           ✓ 判据2: 恰好在bulk CBM能量(2.358 eV)
                                           ✓ 判据3: bulk中此处是空的CBM,
                                              现在却完全占据→额外的电子在此
  Band 292:     E ≈ 3.327 eV, occ=0.0  ← 导带态
```

### Step 5c: 判断方法总结 — 实操流程

```
1. 从host EIGENVAL提取:
   - VBM能量和简并度
   - CBM能量
   - 价带和导带的能量分布模式(各简并组的能量位置)

2. 在defect EIGENVAL中扫描:
   (a) 先找部分占据态 → 一定是缺陷能级 [判据1, 最可靠]
   (b) 再检查带隙区间(VBM~CBM)内有无额外态 → 缺陷能级 [判据2]
   (c) 对比能带分布模式,找不属于价带或导带的态 → 缺陷能级 [判据3]

3. 统计缺陷能级中的电子数:
   - 总电子数 = Σ (occ_up + occ_down) for all defect bands

4. 确定电荷态:
   - 缺陷能级中有N_occ个电子, N_empty个空态
   - 可移走的电子 → 正电荷态: q = +1, +2, ..., +N_occ
   - 可加入的电子 → 负电荷态: q = -1, -2, ..., -N_empty
```

### Step 5d: 从缺陷能级中的电子数确定电荷态和NELECT

### General Rules for Charge State Determination

1. **Identify defect levels:** States inside or near the bulk gap that differ from the bulk band structure
2. **Count electrons in defect levels:**
   - Each band holds 1 electron per spin (2 total for ISPIN=2)
   - Partial occupation (e.g., 0.667) means fractional filling
3. **Determine removable electrons:** Electrons in defect states above VBM can be removed → positive q
4. **Determine addable electrons:** Empty defect states below CBM can accept electrons → negative q
5. **Set NELECT:**
   ```
   NELECT = NELECT_neutral - q
   ```
   (q>0 means fewer electrons; q<0 means more electrons)

### NELECT Reference Table (CdTe)

| Structure | Composition | NELECT_neutral | ZVAL source |
|-----------|------------|----------------|-------------|
| Bulk 64-atom | Cd₃₂Te₃₂ | 576 | 32×12 + 32×6 |
| V_Cd | Cd₃₁Te₃₂ | 564 | 31×12 + 32×6 |
| V_Te | Cd₃₂Te₃₁ | 570 | 32×12 + 31×6 |
| Te_Cd | Cd₃₁Te₃₃ | 570 | 31×12 + 33×6 |
| Cd_Te | Cd₃₃Te₃₁ | 582 | 33×12 + 31×6 |

**Log for EACH defect:** Record:
- Bulk VBM and CBM eigenvalues
- Defect level positions and occupations (band index, energy, occ_up, occ_down)
- Number of electrons in defect levels
- Reasoning: which electrons can be added/removed
- Resulting charge state list and NELECT values

---

## Step 6: Charged Defect Relaxation（q0弛豫完成后）

**前置条件：** 所有 `q0/relax/` 已收敛，且 Step 5 的电荷态分析已完成。

**操作：**
1. 将 `q0/relax/CONTCAR` 复制为各 `q±n/relax/POSCAR`
2. 准备 INCAR（加 NELECT）、KPOINTS、POTCAR、submit_job
3. 提交所有 charged relax 任务（可并行）

**Starting structure:** CONTCAR from the neutral (q=0) relaxation.

**INCAR:** Same as neutral relaxation, but add:
```
NELECT = <calculated value>
```

**Directory structure:**
```
ProjectRoot/
├── Bulk/
│   ├── ktest/                    # K-mesh convergence test (primitive cell)
│   │   ├── primitive_POSCAR      # Primitive cell
│   │   ├── k1/                   # Coarse k-mesh (e.g., 2×2×1)
│   │   ├── k2/                   # Medium k-mesh (e.g., 4×4×2)
│   │   ├── k3/                   # Dense k-mesh (e.g., 6×6×3)
│   │   └── ktest_result.log      # Convergence results
│   ├── AEXX/                     # AEXX fitting (LEVEL2/3 only)
│   │   ├── primitive_POSCAR      # Primitive cell for fitting
│   │   ├── aexx_0.20/            # HSE static with AEXX=0.20
│   │   ├── aexx_0.25/            # HSE static with AEXX=0.25
│   │   ├── aexx_0.30/            # HSE static with AEXX=0.30
│   │   ├── fit_aexx.py           # Fitting script
│   │   └── aexx_fit.log          # Fitting results
│   ├── relax/                    # Bulk relaxation (ISIF=3)
│   └── statics/                  # Bulk static calculation
├── Defect/
│   ├── Intrinsic/
│   │   ├── V_Cd/
│   │   │   ├── q0/
│   │   │   │   ├── relax/        # Neutral relaxation
│   │   │   │   └── statics/      # Static after relax
│   │   │   ├── q-1/
│   │   │   │   ├── relax/        # Charged relaxation (NELECT+1)
│   │   │   │   └── statics/
│   │   │   └── q-2/
│   │   │       ├── relax/        # Charged relaxation (NELECT+2)
│   │   │       └── statics/
│   │   ├── V_Te/
│   │   │   ├── q0/
│   │   │   │   ├── relax/
│   │   │   │   └── statics/
│   │   │   ├── q+1/
│   │   │   │   ├── relax/
│   │   │   │   └── statics/
│   │   │   └── q+2/
│   │   │       ├── relax/
│   │   │       └── statics/
│   │   ├── Te_Cd/
│   │   │   └── ...             # Same pattern: q*/relax/, q*/statics/
│   │   └── Cd_Te/
│   │       └── ...
│   └── Impurity/
│       ├── Cl_Te/
│       │   └── ...             # Same pattern
│       └── Na_Cd/
│           └── ...
├── ThermoStability/              # Chemical potential & stability analysis
└── FormationEnergy/              # Formation energy results & plots
```

**Key:** Each charged calculation starts from the relaxed neutral CONTCAR, NOT from the unrelaxed defect POSCAR.

**Log:** Record starting POSCAR source, NELECT value, convergence for each charge state.

---

## Step 7: Static Calculations（所有弛豫完成后）

**前置条件：** `Bulk/relax/` 和所有 `Defect/*/q*/relax/` 已收敛。

**操作：**
1. 将各 `relax/CONTCAR` 复制为对应 `statics/POSCAR`
2. 将各 `relax/CHGCAR` 复制到对应 `statics/`（ICHARG=1 需要读取）
3. 准备 INCAR（静态参数）、KPOINTS、POTCAR、submit_job
4. 提交所有 statics 任务（可并行）

需要准备的 statics 目录：
- `Bulk/statics/` — 使用 `Bulk/relax/CONTCAR`
- 所有 `Defect/*/q*/statics/` — 使用对应 `relax/CONTCAR`

**INCAR for static (PBE — LEVEL1):**
```
NELM = 100
EDIFF = 1E-5
ISMEAR = 0
SIGMA = 0.05
GGA = PE
IALGO = 38
LREAL = Auto
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = <same as relaxation>
NPAR = 8
ICORELEVEL = 1        # Core-level eigenvalues for Kumagai correction
LVHAR = .TRUE.        # Write LOCPOT for Freysoldt correction
ICHARG = 1            # Read CHGCAR from relaxation
```

**INCAR for static (HSE06 — LEVEL2/3):**
```
NELM = 200            # HSE needs more SCF steps
EDIFF = 1E-5
ISMEAR = 0
SIGMA = 0.05
GGA = PE
LHFCALC  = .TRUE.
HFSCREEN = 0.2
AEXX     = 0.25
ALGO     = All        # Use All for HSE static (NOT Damped)
TIME     = 0.4
PRECFOCK = Fast
LREAL  = .FALSE.      # Required for HF accuracy
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = <same as relaxation>
NPAR = 8
ICORELEVEL = 1
LVHAR = .TRUE.
ICHARG = 1
```

**Note:** No NSW, no IBRION → single-point calculation. Copy CHGCAR from the relaxation to the static directory (ICHARG=1 reads it).

**Required outputs:** LOCPOT (for FNV correction), OUTCAR (for energies and site potentials).

**Log:** Record total energies from each static calculation.

---

## Step 8: Finite-Size Corrections（所有statics完成后）

**前置条件：** `Bulk/statics/` 和所有 `Defect/*/q*/statics/` 已收敛。

### fnv_correction.py — FNV 修正计算脚本

```python
#!/usr/bin/env python3
"""
Freysoldt (FNV) correction for charged defects.
Usage: python fnv_correction.py <bulk_locpot> <defect_locpot> <charge> <epsilon> [--madelung <alpha>]
Output: correction value in eV, writes detailed log
"""
import numpy as np
import sys
import os
import argparse

def read_locpot(filepath):
    """读取VASP LOCPOT文件，返回晶格向量和3D势场数据"""
    with open(filepath) as f:
        f.readline()  # comment
        scale = float(f.readline())
        lattice = np.array([[float(x) for x in f.readline().split()] for _ in range(3)]) * scale
        species = f.readline().split()
        counts = [int(x) for x in f.readline().split()]
        total_atoms = sum(counts)
        # Skip coordinate type line and atom coordinates
        f.readline()  # Direct/Cartesian
        for _ in range(total_atoms):
            f.readline()
        f.readline()  # blank line
        # Read grid dimensions
        nx, ny, nz = [int(x) for x in f.readline().split()]
        # Read potential data
        data = []
        while len(data) < nx * ny * nz:
            line = f.readline()
            if line.strip():
                data.extend([float(x) for x in line.split()])
        pot = np.array(data).reshape((nz, ny, nx)).transpose((2, 1, 0))  # [nx,ny,nz]
    return lattice, pot, (nx, ny, nz)

def planar_average(pot, axis):
    """沿指定轴计算平面平均势"""
    axes = [0, 1, 2]
    axes.remove(axis)
    return np.mean(pot, axis=tuple(axes))

def compute_madelung_energy(alpha_M, q, epsilon, L):
    """计算 Madelung 能量 (eV)
    E_Mad = alpha_M * q^2 / (2 * epsilon * L)
    L in Angstrom, result in eV (with conversion factor)
    """
    # Hartree atomic units: 1 Ha = 27.2114 eV, 1 Bohr = 0.529177 Å
    L_bohr = L / 0.529177
    E_hartree = alpha_M * q**2 / (2.0 * epsilon * L_bohr)
    return E_hartree * 27.2114

def compute_potential_alignment(bulk_pot, defect_pot, lattice, axis, q, epsilon):
    """计算沿某轴的势能对齐量 ΔV"""
    V_bulk = planar_average(bulk_pot, axis)
    V_defect = planar_average(defect_pot, axis)
    diff = V_defect - V_bulk

    n = len(diff)
    # 取远离缺陷的区域（两端各1/4）作为对齐参考
    far_region = np.concatenate([diff[:n//4], diff[3*n//4:]])
    delta_V = np.mean(far_region)
    return delta_V

def main():
    parser = argparse.ArgumentParser(description='FNV correction for charged defects')
    parser.add_argument('bulk_locpot', help='Path to bulk LOCPOT')
    parser.add_argument('defect_locpot', help='Path to defect LOCPOT')
    parser.add_argument('charge', type=int, help='Charge state q')
    parser.add_argument('epsilon', type=float, help='Dielectric constant')
    parser.add_argument('--madelung', type=float, default=2.8373,
                        help='Madelung constant (default: 2.8373 for SC)')
    parser.add_argument('--output', default=None, help='Output log file')
    args = parser.parse_args()

    q = args.charge
    if q == 0:
        print("q=0: No correction needed")
        print("CORRECTION: 0.000")
        return

    print(f"Reading bulk LOCPOT: {args.bulk_locpot}")
    lattice_b, pot_b, grid_b = read_locpot(args.bulk_locpot)
    print(f"Reading defect LOCPOT: {args.defect_locpot}")
    lattice_d, pot_d, grid_d = read_locpot(args.defect_locpot)

    # 超胞特征长度（取晶格向量长度的平均值）
    L_values = [np.linalg.norm(lattice_b[i]) for i in range(3)]
    L_avg = np.mean(L_values)

    # Madelung energy
    E_mad = compute_madelung_energy(args.madelung, q, args.epsilon, L_avg)

    # Potential alignment（三个轴取平均）
    delta_Vs = []
    for axis in range(3):
        dV = compute_potential_alignment(pot_b, pot_d, lattice_b, axis, q, args.epsilon)
        delta_Vs.append(dV)
    delta_V = np.mean(delta_Vs)

    # Total correction
    E_align = q * delta_V
    E_corr = E_mad + E_align

    # Output
    result = f"""FNV Correction Result:
  Charge state: q = {q}
  Dielectric constant: epsilon = {args.epsilon}
  Madelung constant: alpha = {args.madelung}
  Supercell lengths: {L_values[0]:.3f}, {L_values[1]:.3f}, {L_values[2]:.3f} Å
  Average length: {L_avg:.3f} Å
  ---
  Madelung energy: {E_mad:.4f} eV
  Potential alignment (per axis): {delta_Vs[0]:.4f}, {delta_Vs[1]:.4f}, {delta_Vs[2]:.4f} eV
  Average ΔV: {delta_V:.4f} eV
  Alignment correction (q×ΔV): {E_align:.4f} eV
  ---
  Total FNV correction: {E_corr:.4f} eV
CORRECTION: {E_corr:.4f}"""

    print(result)
    if args.output:
        with open(args.output, 'w') as f:
            f.write(result + '\n')

if __name__ == '__main__':
    main()
```

### 使用方法

```bash
# 对每个带电缺陷运行
python fnv_correction.py \
    Bulk/statics/LOCPOT \
    Defect/Intrinsic/V_Cd/q-2/statics/LOCPOT \
    -2 \
    10.3 \
    --madelung 2.8373 \
    --output Defect/Intrinsic/V_Cd/q-2/fnv_correction.log
```

---

## Step 9: Formation Energy & Diagram

### formation_energy.py — 形成能计算与绘图脚本

```python
#!/usr/bin/env python3
"""
Formation energy calculation and diagram plotting.
Reads VASP results, applies corrections, plots E_f vs E_Fermi.

Usage:
  python formation_energy.py --config formation_config.yaml
  python formation_energy.py --workdir . --epsilon 10.3 --band_gap 1.5

Config file (formation_config.yaml):
  bulk_static: Bulk/statics
  epsilon: 10.3
  madelung: 2.8373
  band_gap: 1.5
  E_pure:
    Cd: -1.7736
    Te: -4.6974
  chemical_potential_limits:
    Cd-rich:
      Cd: 0.0
      Te: -1.1854
    Te-rich:
      Cd: -1.1854
      Te: 0.0
  defects:
    - name: V_Cd
      path: Defect/Intrinsic/V_Cd
      removed: {Cd: 1}
      added: {}
    - name: V_Te
      path: Defect/Intrinsic/V_Te
      removed: {Te: 1}
      added: {}
    - name: Te_Cd
      path: Defect/Intrinsic/Te_Cd
      removed: {Cd: 1}
      added: {Te: 1}
    - name: Cl_Te
      path: Defect/Impurity/Cl_Te
      removed: {Te: 1}
      added: {Cl: 1}
"""
import numpy as np
import matplotlib.pyplot as plt
import os
import sys
import yaml
import subprocess
import re
from datetime import datetime

def get_energy_from_outcar(outcar_path):
    """从OUTCAR提取 'energy without entropy' (sigma->0)"""
    E = None
    with open(outcar_path) as f:
        for line in f:
            if "energy  without entropy" in line:
                E = float(line.split()[-1])
    return E

def get_vbm_from_eigenval(eigenval_path):
    """从EIGENVAL提取VBM（最高占据态能量）"""
    with open(eigenval_path) as f:
        lines = f.readlines()
    # Header: line 6 has NELECT, NKPTS, NBANDS
    header = lines[5].split()
    nelect = int(header[0])
    nkpts = int(header[1])
    nbands = int(header[2])

    vbm = -999.0
    idx = 7  # skip header lines
    for k in range(nkpts):
        idx += 1  # blank line
        idx += 1  # k-point header
        for b in range(nbands):
            parts = lines[idx].split()
            band_idx = int(parts[0])
            e_up = float(parts[1])
            occ_up = float(parts[2])
            if len(parts) > 3:  # ISPIN=2
                e_down = float(parts[3])
                occ_down = float(parts[4])
                if occ_up > 0.5:
                    vbm = max(vbm, e_up)
                if occ_down > 0.5:
                    vbm = max(vbm, e_down)
            else:
                if occ_up > 0.5:
                    vbm = max(vbm, e_up)
            idx += 1
    return vbm

def get_nelect_from_outcar(outcar_path):
    """从OUTCAR提取NELECT"""
    with open(outcar_path) as f:
        for line in f:
            if "NELECT" in line:
                return int(float(line.split()[2]))
    return None

def compute_fnv_correction(bulk_locpot, defect_locpot, charge, epsilon, madelung=2.8373):
    """调用fnv_correction.py并返回修正值"""
    if charge == 0:
        return 0.0
    script = os.path.join(os.path.dirname(__file__), 'fnv_correction.py')
    try:
        result = subprocess.run(
            ['python3', script, bulk_locpot, defect_locpot, str(charge), str(epsilon),
             '--madelung', str(madelung)],
            capture_output=True, text=True, timeout=300
        )
        for line in result.stdout.split('\n'):
            if line.startswith('CORRECTION:'):
                return float(line.split(':')[1].strip())
    except Exception as e:
        print(f"  WARNING: FNV correction failed: {e}")
    return 0.0

def find_charge_dirs(defect_path):
    """查找缺陷目录下的所有电荷态目录"""
    charges = []
    for d in sorted(os.listdir(defect_path)):
        if d.startswith('q') and os.path.isdir(os.path.join(defect_path, d)):
            q_str = d[1:]  # remove 'q'
            try:
                q = int(q_str.replace('+', ''))
                charges.append((q, os.path.join(defect_path, d)))
            except ValueError:
                pass
    return charges

def main():
    import argparse
    parser = argparse.ArgumentParser(description='Formation energy calculation and plotting')
    parser.add_argument('--config', required=True, help='YAML config file')
    parser.add_argument('--output', default='FormationEnergy', help='Output directory')
    args = parser.parse_args()

    with open(args.config) as f:
        cfg = yaml.safe_load(f)

    os.makedirs(args.output, exist_ok=True)
    log_path = os.path.join(args.output, 'formation_energy.log')
    log_lines = []
    def log(msg):
        log_lines.append(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] {msg}")
        print(msg)

    # ---- 读取 bulk 能量和 VBM ----
    bulk_dir = cfg['bulk_static']
    E_bulk = get_energy_from_outcar(os.path.join(bulk_dir, 'OUTCAR'))
    E_VBM = get_vbm_from_eigenval(os.path.join(bulk_dir, 'EIGENVAL'))
    nelect_bulk = get_nelect_from_outcar(os.path.join(bulk_dir, 'OUTCAR'))
    band_gap = cfg['band_gap']
    epsilon = cfg['epsilon']
    madelung = cfg.get('madelung', 2.8373)

    log(f"Bulk energy: {E_bulk:.6f} eV")
    log(f"VBM: {E_VBM:.4f} eV")
    log(f"Band gap: {band_gap} eV")
    log(f"Dielectric constant: {epsilon}")

    E_pure = cfg['E_pure']         # {element: E_per_atom}
    limits = cfg['chemical_potential_limits']  # {limit_name: {element: delta_mu}}

    # ---- 计算各缺陷的形成能 ----
    all_results = {}  # {defect_name: {q: {limit: E_f_at_VBM}}}

    for defect_cfg in cfg['defects']:
        name = defect_cfg['name']
        dpath = defect_cfg['path']
        removed = defect_cfg.get('removed', {})
        added = defect_cfg.get('added', {})

        log(f"\n--- Defect: {name} ---")
        all_results[name] = {}

        charge_dirs = find_charge_dirs(dpath)
        for q, q_dir in charge_dirs:
            statics_dir = os.path.join(q_dir, 'statics')
            outcar = os.path.join(statics_dir, 'OUTCAR')
            if not os.path.exists(outcar):
                log(f"  q={q}: OUTCAR not found, skipping")
                continue

            E_defect = get_energy_from_outcar(outcar)
            log(f"  q={q}: E_defect = {E_defect:.6f} eV")

            # FNV correction
            E_corr = 0.0
            if q != 0:
                bulk_locpot = os.path.join(bulk_dir, 'LOCPOT')
                defect_locpot = os.path.join(statics_dir, 'LOCPOT')
                if os.path.exists(bulk_locpot) and os.path.exists(defect_locpot):
                    E_corr = compute_fnv_correction(
                        bulk_locpot, defect_locpot, q, epsilon, madelung)
                    log(f"  q={q}: FNV correction = {E_corr:.4f} eV")
                else:
                    log(f"  q={q}: WARNING - LOCPOT not found, correction=0")

            # 对各化学势条件计算形成能
            all_results[name][q] = {}
            for limit_name, delta_mus in limits.items():
                # 化学势项: Σ n_i × μ_i
                chem_term = 0.0
                for elem, n in removed.items():
                    mu_i = E_pure[elem] + delta_mus[elem]
                    chem_term += n * mu_i
                for elem, n in added.items():
                    mu_i = E_pure[elem] + delta_mus.get(elem, 0.0)
                    chem_term -= n * mu_i

                # E_f at VBM (E_Fermi=0)
                E_f_vbm = E_defect - E_bulk + chem_term + q * E_VBM + E_corr
                all_results[name][q][limit_name] = E_f_vbm
                log(f"  q={q}, {limit_name}: E_f(VBM) = {E_f_vbm:.4f} eV")

    # ---- 计算转变能级 ----
    log(f"\n=== Transition Levels ===")
    for name, q_data in all_results.items():
        charges = sorted(q_data.keys())
        for i in range(len(charges)):
            for j in range(i+1, len(charges)):
                q1, q2 = charges[i], charges[j]
                for limit_name in limits:
                    if limit_name in q_data[q1] and limit_name in q_data[q2]:
                        E1 = q_data[q1][limit_name]
                        E2 = q_data[q2][limit_name]
                        if q2 != q1:
                            E_F_trans = (E1 - E2) / (q2 - q1)
                            if 0 <= E_F_trans <= band_gap:
                                log(f"  {name}: ε({q1}/{q2}) = {E_F_trans:.4f} eV  [{limit_name}]")

    # ---- 绘图 ----
    for limit_name in limits:
        fig, ax = plt.subplots(figsize=(8, 6))
        E_F = np.linspace(0, band_gap, 500)
        colors = plt.cm.tab10(np.linspace(0, 1, len(all_results)))

        for (name, q_data), color in zip(all_results.items(), colors):
            lines = []
            for q, limit_data in sorted(q_data.items()):
                if limit_name in limit_data:
                    E_f_vbm = limit_data[limit_name]
                    E_f_line = E_f_vbm + q * E_F
                    lines.append(E_f_line)
                    # Faded individual lines
                    ax.plot(E_F, E_f_line, color=color, lw=0.7, alpha=0.25, ls='--')

            if lines:
                lower_envelope = np.min(np.array(lines), axis=0)
                ax.plot(E_F, lower_envelope, color=color, lw=2.5, label=name)

        # Band edges
        ax.axvspan(-0.5, 0, alpha=0.12, color='cornflowerblue')
        ax.axvspan(band_gap, band_gap + 0.5, alpha=0.12, color='orange')
        ax.axhline(0, color='k', ls='--', alpha=0.3, lw=0.8)

        ax.set_xlim(-0.2, band_gap + 0.2)
        y_min = min(0, min(env.min() for env in [np.min(np.array(
            [ld[limit_name] + q * E_F for q, ld in qd.items() if limit_name in ld]),
            axis=0) for qd in all_results.values() if qd]) - 0.5)
        ax.set_ylim(y_min, None)
        ax.set_xlabel("Fermi Level (eV)", fontsize=14)
        ax.set_ylabel("Formation Energy (eV)", fontsize=14)
        ax.set_title(f"Formation Energy Diagram ({limit_name})", fontsize=14)
        ax.legend(fontsize=11, loc='best')
        plt.tight_layout()

        fig_path = os.path.join(args.output, f'formation_energy_{limit_name}.pdf')
        fig.savefig(fig_path, dpi=300)
        fig_path_png = os.path.join(args.output, f'formation_energy_{limit_name}.png')
        fig.savefig(fig_path_png, dpi=150)
        plt.close(fig)
        log(f"\nPlot saved: {fig_path}")

    # ---- 保存日志 ----
    with open(log_path, 'w') as f:
        f.write('\n'.join(log_lines) + '\n')
    log(f"Log saved: {log_path}")

if __name__ == '__main__':
    main()
```

### formation_config.yaml 配置文件模板

```yaml
# 形成能计算配置
bulk_static: Bulk/statics
epsilon: 10.3                    # 介电常数
madelung: 2.8373                 # Madelung常数 (SC lattice)
band_gap: 1.5                    # 带隙 (eV)

# 各元素的单质能量 (eV/atom)
E_pure:
  Cd: -1.7736
  Te: -4.6974

# 化学势极限条件 (Δμ, eV)
chemical_potential_limits:
  Cd-rich:
    Cd: 0.0
    Te: -1.1854
  Te-rich:
    Cd: -1.1854
    Te: 0.0

# 缺陷列表
defects:
  - name: V_Cd
    path: Defect/Intrinsic/V_Cd
    removed: {Cd: 1}
    added: {}
  - name: V_Te
    path: Defect/Intrinsic/V_Te
    removed: {Te: 1}
    added: {}
  - name: Te_Cd
    path: Defect/Intrinsic/Te_Cd
    removed: {Cd: 1}
    added: {Te: 1}
  - name: Cd_Te
    path: Defect/Intrinsic/Cd_Te
    removed: {Te: 1}
    added: {Cd: 1}
  - name: Cl_Te
    path: Defect/Impurity/Cl_Te
    removed: {Te: 1}
    added: {Cl: 1}
```

### 使用方法

```bash
# 1. 编辑 formation_config.yaml（填入实际参数）
# 2. 运行
python formation_energy.py --config formation_config.yaml --output FormationEnergy/

# 输出:
#   FormationEnergy/formation_energy_Cd-rich.pdf
#   FormationEnergy/formation_energy_Te-rich.pdf
#   FormationEnergy/formation_energy.log
```

**Log:** 脚本自动记录所有形成能、转变能级到 `FormationEnergy/formation_energy.log`。

---

## SLURM Job Automation

### Job Script Template

Based on the user's cluster configuration:
```bash
#!/bin/sh
#SBATCH -N <node_number>
#SBATCH -J <job_name>
#SBATCH -n <total_cores>
#SBATCH --ntasks-per-node=<core_per_node>
#SBATCH --partition=<queue>
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH --time=<max_time>

mpirun -n <total_cores> <vasp_path> > vasp.out 2>&1
```

**Default values (from dasp.in):**
```
node_number = 2
core_per_node = 52
queue = compute2
max_time = 24:00:00
vasp_gam = /opt/vasp.5.4.4/bin/vasp_gam
vasp_std = /opt/vasp.5.4.4/bin/vasp_std
```

### VASP Executable Selection
- **Gamma-only KPOINTS (1×1×1):** Use `vasp_gam` (faster)
- **Multiple k-points:** Use `vasp_std`

### 自动监控与调度系统

**核心思路：** 后台监控脚本 `auto_monitor.sh` 每10分钟检查一次任务状态，收敛后自动准备下一阶段输入文件并提交。全程无需人工干预。

**需要的文件：**
1. `defect_config.txt` — 定义缺陷类型、操作方式、电荷态
2. `prepare_defect.py` — Python辅助脚本，处理POSCAR构建缺陷结构
3. `auto_monitor.sh` — 主监控循环脚本（后台运行）

#### 1. defect_config.txt 格式

```
# 缺陷配置文件
# category  name  operation  atom_index  replace_element  charge_states
#
# category: intrinsic / impurity
# operation: vacancy / antisite / substitute
# atom_index: 从1开始的原子序号（在超胞POSCAR中的位置）
# replace_element: 替换为的元素（vacancy填 - ）
# charge_states: 逗号分隔的带电态（不含q0，q0自动包含）
#
intrinsic  V_Cd    vacancy     1    -    -1,-2
intrinsic  V_Te    vacancy     33   -    +1,+2
intrinsic  Te_Cd   antisite    1    Te   -1,-2,+1,+2
intrinsic  Cd_Te   antisite    33   Cd   -1,-2,+1,+2
impurity   Cl_Te   substitute  33   Cl   +1,-1
impurity   Na_Cd   substitute  1    Na   -1,+1
```

#### 2. prepare_defect.py — POSCAR缺陷结构构建

```python
#!/usr/bin/env python3
"""从bulk CONTCAR构建缺陷POSCAR"""
import sys
import os

def read_poscar(filepath):
    """读取POSCAR，返回 (comment, scale, lattice, species, counts, coord_type, coords)"""
    with open(filepath) as f:
        lines = f.readlines()
    comment = lines[0].strip()
    scale = float(lines[1])
    lattice = [lines[i].split() for i in range(2, 5)]
    species = lines[5].split()
    counts = list(map(int, lines[6].split()))
    coord_type = lines[7].strip()
    total = sum(counts)
    coords = []
    for i in range(8, 8 + total):
        coords.append(lines[i].strip())
    return comment, scale, lattice, species, counts, coord_type, coords

def write_poscar(filepath, comment, scale, lattice, species, counts, coord_type, coords):
    """写POSCAR"""
    with open(filepath, 'w') as f:
        f.write(comment + '\n')
        f.write(f'{scale}\n')
        for v in lattice:
            f.write('  ' + '  '.join(v) + '\n')
        f.write('  '.join(species) + '\n')
        f.write('  '.join(map(str, counts)) + '\n')
        f.write(coord_type + '\n')
        for c in coords:
            f.write(c + '\n')

def create_vacancy(bulk_poscar, atom_index, output_poscar):
    """移除第atom_index个原子（从1开始计数）"""
    comment, scale, lattice, species, counts, coord_type, coords = read_poscar(bulk_poscar)
    idx = atom_index - 1  # 转为0-based
    # 确定该原子属于哪个species
    cumsum = 0
    for i, c in enumerate(counts):
        if cumsum + c > idx:
            species_idx = i
            break
        cumsum += c
    coords.pop(idx)
    counts[species_idx] -= 1
    # 移除count为0的species
    new_species, new_counts = [], []
    for s, c in zip(species, counts):
        if c > 0:
            new_species.append(s)
            new_counts.append(c)
    comment = f'Vacancy at site {atom_index}'
    write_poscar(output_poscar, comment, scale, lattice, new_species, new_counts, coord_type, coords)

def create_antisite_or_substitute(bulk_poscar, atom_index, new_element, output_poscar):
    """将第atom_index个原子替换为new_element"""
    comment, scale, lattice, species, counts, coord_type, coords = read_poscar(bulk_poscar)
    idx = atom_index - 1
    # 找到被替换原子的species和坐标
    cumsum = 0
    for i, c in enumerate(counts):
        if cumsum + c > idx:
            old_species_idx = i
            break
        cumsum += c
    replaced_coord = coords.pop(idx)
    counts[old_species_idx] -= 1
    # 添加新元素
    if new_element in species:
        new_idx = species.index(new_element)
        # 插入到该species段的末尾
        insert_pos = sum(counts[:new_idx+1])
        coords.insert(insert_pos, replaced_coord)
        counts[new_idx] += 1
    else:
        species.append(new_element)
        counts.append(1)
        coords.append(replaced_coord)
    # 清理count为0的species
    new_species, new_counts = [], []
    for s, c in zip(species, counts):
        if c > 0:
            new_species.append(s)
            new_counts.append(c)
    comment = f'Replace site {atom_index} with {new_element}'
    write_poscar(output_poscar, comment, scale, lattice, new_species, new_counts, coord_type, coords)

if __name__ == '__main__':
    operation = sys.argv[1]     # vacancy / antisite / substitute
    bulk_poscar = sys.argv[2]   # Bulk/relax/CONTCAR
    atom_index = int(sys.argv[3])
    output_poscar = sys.argv[4]
    if operation == 'vacancy':
        create_vacancy(bulk_poscar, atom_index, output_poscar)
    else:
        new_element = sys.argv[5]
        create_antisite_or_substitute(bulk_poscar, atom_index, new_element, output_poscar)
```

#### 3. auto_monitor.sh — 主监控脚本

```bash
#!/bin/bash
# auto_monitor.sh — 后台自动监控，每10分钟检查收敛，自动准备下一阶段并提交
#
# 使用方法:
#   nohup bash auto_monitor.sh > monitor.out 2>&1 &
#   nohup bash auto_monitor.sh --claude > monitor.out 2>&1 &   # 启用AI审查
#   echo $! > monitor.pid
#
# 停止监控:
#   kill $(cat monitor.pid)

# ============ 参数解析 ============
USE_CLAUDE=false
for arg in "$@"; do
    case $arg in
        --claude) USE_CLAUDE=true ;;
    esac
done

WORKDIR=$(pwd)
LOG="$WORKDIR/defect_workflow.log"
CONFIG="$WORKDIR/defect_config.txt"
PREPARE_PY="$WORKDIR/prepare_defect.py"
INTERVAL=600  # 10分钟 = 600秒

# ============ 计算精度级别 ============
# LEVEL1: PBE relax + PBE statics
# LEVEL2: PBE relax + HSE statics
# LEVEL3: HSE relax + HSE statics
LEVEL=1   # 默认 LEVEL1，可改为 2 或 3

# ============ VASP 参数配置（按实际情况修改） ============
ENCUT=350              # 根据POTCAR的ENMAX × 1.3
VASP_GAM="/opt/vasp.5.4.4/bin/vasp_gam"
NODE_NUMBER=2
CORE_PER_NODE=52
TOTAL_CORES=$((NODE_NUMBER * CORE_PER_NODE))
QUEUE="compute2"
MAX_TIME="24:00:00"

# ============ 辅助函数 ============

log_msg() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG
    echo "[$(date '+%H:%M:%S')] $1"
}

is_converged() {
    # 检查OUTCAR是否包含收敛标志
    grep -q "reached required accuracy" "$1/OUTCAR" 2>/dev/null
}

has_error() {
    grep -q "Error\|ZBRENT\|VERY BAD NEWS" "$1/vasp.out" 2>/dev/null
}

is_running() {
    # 有OUTCAR但未收敛且无错误 → 仍在运行（或队列中）
    [ -f "$1/OUTCAR" ] && ! is_converged "$1" && ! has_error "$1"
}

is_submitted() {
    # POSCAR存在且有submit_job，但可能还在队列中
    [ -f "$1/POSCAR" ] && [ -f "$1/submit_job" ]
}

get_nelect_from_outcar() {
    # 从OUTCAR中提取NELECT值
    grep "NELECT" "$1/OUTCAR" 2>/dev/null | head -1 | awk '{print $3}' | cut -d. -f1
}

write_submit_job() {
    # $1=目录 $2=job_name
    cat > "$1/submit_job" << SUBMIT_EOF
#!/bin/sh
#SBATCH -N ${NODE_NUMBER}
#SBATCH -J $2
#SBATCH -n ${TOTAL_CORES}
#SBATCH --ntasks-per-node=${CORE_PER_NODE}
#SBATCH --partition=${QUEUE}
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH --time=${MAX_TIME}

mpirun -n ${TOTAL_CORES} ${VASP_GAM} > vasp.out 2>&1
SUBMIT_EOF
}

write_kpoints() {
    cat > "$1/KPOINTS" << KPOINTS_EOF
Gamma
0
G
1 1 1
0 0 0
KPOINTS_EOF
}

write_incar_relax() {
    # $1=目录 $2=ISIF值 $3=可选NELECT
    local nelect_line=""
    [ -n "$3" ] && nelect_line="NELECT = $3"
    cat > "$1/INCAR" << INCAR_EOF
NSW = 500
NELM = 100
ISIF = $2
IBRION = 2
POTIM = 0.5
EDIFF = 1E-5
EDIFFG = -0.01
ISMEAR = 0
SIGMA = 0.05
GGA = PE
IALGO = 38
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = ${ENCUT}
NPAR = 8
${nelect_line}
INCAR_EOF
}

write_incar_static() {
    # $1=目录 $2=可选NELECT
    local nelect_line=""
    [ -n "$2" ] && nelect_line="NELECT = $2"
    cat > "$1/INCAR" << INCAR_EOF
NELM = 100
EDIFF = 1E-5
ISMEAR = 0
SIGMA = 0.05
GGA = PE
IALGO = 38
LREAL = Auto
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = ${ENCUT}
NPAR = 8
ICORELEVEL = 1
LVHAR = .TRUE.
ICHARG = 1
${nelect_line}
INCAR_EOF
}

# ============ HSE06 INCAR 生成函数 ============

write_incar_relax_hse() {
    # $1=目录 $2=ISIF值 $3=可选NELECT
    local nelect_line=""
    [ -n "$3" ] && nelect_line="NELECT = $3"
    cat > "$1/INCAR" << INCAR_EOF
NSW = 500
NELM = 100
ISIF = $2
IBRION = 2
POTIM = 0.3
EDIFF = 1E-5
EDIFFG = -0.02
ISMEAR = 0
SIGMA = 0.05
GGA = PE
LHFCALC  = .TRUE.
HFSCREEN = 0.2
AEXX     = 0.25
ALGO     = Damped
TIME     = 0.4
PRECFOCK = Fast
LREAL  = .FALSE.
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = ${ENCUT}
NPAR = 8
${nelect_line}
INCAR_EOF
}

write_incar_static_hse() {
    # $1=目录 $2=可选NELECT
    local nelect_line=""
    [ -n "$2" ] && nelect_line="NELECT = $2"
    cat > "$1/INCAR" << INCAR_EOF
NELM = 200
EDIFF = 1E-5
ISMEAR = 0
SIGMA = 0.05
GGA = PE
LHFCALC  = .TRUE.
HFSCREEN = 0.2
AEXX     = 0.25
ALGO     = All
TIME     = 0.4
PRECFOCK = Fast
LREAL  = .FALSE.
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = ${ENCUT}
NPAR = 8
ICORELEVEL = 1
LVHAR = .TRUE.
ICHARG = 1
${nelect_line}
INCAR_EOF
}

# ============ Level-aware INCAR 调度函数 ============

dispatch_incar_relax() {
    # 根据 LEVEL 选择 relax INCAR: LEVEL3 → HSE, 其他 → PBE
    if [ "$LEVEL" -ge 3 ]; then
        write_incar_relax_hse "$@"
    else
        write_incar_relax "$@"
    fi
}

dispatch_incar_static() {
    # 根据 LEVEL 选择 static INCAR: LEVEL2/3 → HSE, LEVEL1 → PBE
    if [ "$LEVEL" -ge 2 ]; then
        write_incar_static_hse "$@"
    else
        write_incar_static "$@"
    fi
}

submit_dir() {
    # $1=目录 $2=标签
    cd "$1"
    local JOB=$(sbatch --parsable submit_job 2>/dev/null)
    if [ -n "$JOB" ]; then
        log_msg "SUBMITTED: $2 → Job $JOB"
    else
        log_msg "ERROR: Failed to submit $2"
    fi
    cd "$WORKDIR"
}

# ============ Claude AI 审查函数 ============

claude_review() {
    # 使用Claude审查VASP计算结果，检测异常
    # $1=目录路径 $2=计算类型标签 $3=计算类型(relax/static)
    $USE_CLAUDE || return 0  # 未启用--claude则跳过

    local calc_dir="$1"
    local label="$2"
    local calc_type="${3:-relax}"
    local review_file="$calc_dir/claude_review.txt"

    log_msg "CLAUDE_REVIEW: 开始审查 $label ..."

    # 构建审查prompt
    local prompt="你是VASP计算结果审查专家。请检查以下目录中的计算结果，判断是否存在异常。

目录: $calc_dir
计算类型: $calc_type

请依次检查以下内容，给出简洁判断（正常/异常+原因）：

1. **收敛性**: 检查 OSZICAR 的能量变化趋势，是否单调下降？最终几步能量变化是否<1meV？有无能量震荡？
2. **电子步收敛**: OUTCAR 中每个离子步的电子步数是否合理（一般<40步）？有无反复不收敛？
3. **力和应力**: OUTCAR 末尾的原子力是否合理（一般<0.01 eV/Å）？有无异常大的力？
4. **磁矩**: OUTCAR 中的总磁矩是否合理？对于该缺陷类型，磁矩是否符合预期？
5. **能量合理性**: 最终能量是否在合理范围？与同类计算相比是否异常偏大/偏小？
6. **结构变化**: 对比 POSCAR 和 CONTCAR，原子位移是否合理？有无原子飞走或结构坍塌？"

    if [ "$calc_type" = "static" ]; then
        prompt="$prompt
7. **LOCPOT**: 检查 LOCPOT 文件是否正常生成（文件大小>0）？
8. **带边**: 检查 EIGENVAL 中的 VBM/CBM 位置是否合理？"
    fi

    prompt="$prompt

最后给出总结：PASS（无异常）或 WARNING（有潜在问题，列出）或 FAIL（严重问题，必须处理）。

请直接读取该目录下的 OSZICAR、OUTCAR（最后200行）、POSCAR、CONTCAR 来分析。"

    # 调用 claude CLI（非交互模式）
    local review_result
    review_result=$(claude -p "$prompt" \
        --allowedTools "Read,Bash" \
        --max-turns 10 \
        2>/dev/null)

    # 保存审查结果
    echo "====== Claude Review: $label ======" > "$review_file"
    echo "Date: $(date '+%Y-%m-%d %H:%M:%S')" >> "$review_file"
    echo "" >> "$review_file"
    echo "$review_result" >> "$review_file"

    # 提取结论（PASS/WARNING/FAIL）
    local verdict="UNKNOWN"
    if echo "$review_result" | grep -qi "FAIL"; then
        verdict="FAIL"
    elif echo "$review_result" | grep -qi "WARNING"; then
        verdict="WARNING"
    elif echo "$review_result" | grep -qi "PASS"; then
        verdict="PASS"
    fi

    log_msg "CLAUDE_REVIEW: $label → $verdict (详见 $review_file)"

    # FAIL时暂停自动流程
    if [ "$verdict" = "FAIL" ]; then
        log_msg "CLAUDE_REVIEW: ⚠ $label 审查未通过，暂停自动推进，请人工检查 $review_file"
        return 1  # 返回非0，阻止phase推进
    fi
    return 0
}

claude_review_phase() {
    # 对一批已收敛的计算目录进行Claude审查
    # $1=phase名 $2...=目录列表（空格分隔的 "dir|label" 对）
    $USE_CLAUDE || return 0

    local phase_name="$1"
    shift
    local all_pass=true

    for entry in "$@"; do
        local dir="${entry%%|*}"
        local label="${entry##*|}"
        local calc_type="relax"
        [[ "$dir" == *statics* ]] && calc_type="static"

        if ! claude_review "$dir" "$label" "$calc_type"; then
            all_pass=false
        fi
    done

    if ! $all_pass; then
        log_msg "CLAUDE_REVIEW: $phase_name 中有计算未通过审查，暂停推进"
        return 1
    fi
    return 0
}

# ============ Phase 处理函数 ============

phase1_check_and_advance() {
    # Phase 1: Bulk/relax 收敛 → 准备所有 q0/relax 输入 → 提交
    if ! is_converged "$WORKDIR/Bulk/relax"; then
        if has_error "$WORKDIR/Bulk/relax"; then
            log_msg "ERROR: Bulk/relax 出错，请人工检查"
        fi
        return 1
    fi

    # 检查是否已经进入Phase2（q0/relax已存在输入文件）
    local first_q0=$(find "$WORKDIR/Defect" -path "*/q0/relax/POSCAR" 2>/dev/null | head -1)
    [ -n "$first_q0" ] && return 0  # 已准备过，跳过

    # Claude审查 Bulk/relax
    claude_review_phase "Phase1" "$WORKDIR/Bulk/relax|Bulk/relax" || return 1

    log_msg "Phase1→2: Bulk/relax CONVERGED, 开始准备缺陷结构..."
    local BULK_CONTCAR="$WORKDIR/Bulk/relax/CONTCAR"
    local BULK_POTCAR="$WORKDIR/Bulk/relax/POTCAR"

    # 读取config，构建缺陷结构并准备q0/relax输入
    while IFS= read -r line || [ -n "$line" ]; do
        # 跳过注释和空行
        [[ "$line" =~ ^#.*$ || -z "$line" ]] && continue
        read -r category name operation atom_index replace_elem charge_states <<< "$line"

        local cat_dir
        [ "$category" = "intrinsic" ] && cat_dir="Intrinsic" || cat_dir="Impurity"
        local defect_base="$WORKDIR/Defect/$cat_dir/$name"
        local q0_relax="$defect_base/q0/relax"
        mkdir -p "$q0_relax"

        # 用Python构建缺陷POSCAR
        if [ "$operation" = "vacancy" ]; then
            python3 "$PREPARE_PY" vacancy "$BULK_CONTCAR" "$atom_index" "$q0_relax/POSCAR"
        else
            python3 "$PREPARE_PY" "$operation" "$BULK_CONTCAR" "$atom_index" "$q0_relax/POSCAR" "$replace_elem"
        fi

        # 准备其他输入文件（根据LEVEL自动选择PBE或HSE）
        dispatch_incar_relax "$q0_relax" 2
        write_kpoints "$q0_relax"
        cp "$BULK_POTCAR" "$q0_relax/POTCAR"
        write_submit_job "$q0_relax" "${name}_q0"

        # 提交
        submit_dir "$q0_relax" "$name/q0/relax"

        log_msg "Phase2: 准备并提交 $name/q0/relax (operation=$operation, atom=$atom_index)"
    done < "$CONFIG"

    return 0
}

phase2_check_and_advance() {
    # Phase 2: 所有 q0/relax 收敛 → 准备 charged relax 输入 → 提交
    local all_q0_done=true
    local any_q0_exists=false

    for q0_dir in "$WORKDIR"/Defect/Intrinsic/*/q0/relax "$WORKDIR"/Defect/Impurity/*/q0/relax; do
        [ -d "$q0_dir" ] || continue
        any_q0_exists=true
        if ! is_converged "$q0_dir"; then
            if has_error "$q0_dir"; then
                local dname=$(basename $(dirname $(dirname "$q0_dir")))
                log_msg "ERROR: $dname/q0/relax 出错，请人工检查"
            fi
            all_q0_done=false
        fi
    done

    $any_q0_exists || return 1  # 还没有q0目录
    $all_q0_done || return 1

    # 检查是否已经进入Phase3（charged relax已存在）
    local first_charged=$(find "$WORKDIR/Defect" -path "*/q[+-]*/relax/POSCAR" 2>/dev/null | head -1)
    [ -n "$first_charged" ] && return 0  # 已准备过

    # Claude审查所有 q0/relax
    local review_list=()
    for q0_dir in "$WORKDIR"/Defect/Intrinsic/*/q0/relax "$WORKDIR"/Defect/Impurity/*/q0/relax; do
        [ -d "$q0_dir" ] || continue
        local dname=$(basename $(dirname $(dirname "$q0_dir")))
        review_list+=("$q0_dir|$dname/q0/relax")
    done
    claude_review_phase "Phase2" "${review_list[@]}" || return 1

    log_msg "Phase2→3: All q0/relax CONVERGED, 开始准备带电缺陷..."

    while IFS= read -r line || [ -n "$line" ]; do
        [[ "$line" =~ ^#.*$ || -z "$line" ]] && continue
        read -r category name operation atom_index replace_elem charge_states <<< "$line"

        local cat_dir
        [ "$category" = "intrinsic" ] && cat_dir="Intrinsic" || cat_dir="Impurity"
        local defect_base="$WORKDIR/Defect/$cat_dir/$name"
        local q0_relax="$defect_base/q0/relax"

        # 获取中性NELECT
        local nelect_neutral=$(get_nelect_from_outcar "$q0_relax")

        # 遍历各电荷态
        IFS=',' read -ra charges <<< "$charge_states"
        for q in "${charges[@]}"; do
            q=$(echo "$q" | tr -d ' ')
            local q_dir="$defect_base/q${q}/relax"
            mkdir -p "$q_dir"

            # POSCAR: 从q0弛豫后的CONTCAR复制
            cp "$q0_relax/CONTCAR" "$q_dir/POSCAR"

            # 计算NELECT: NELECT = NELECT_neutral - q
            local q_num=$(echo "$q" | sed 's/+//')
            local nelect=$((nelect_neutral - q_num))

            dispatch_incar_relax "$q_dir" 2 "$nelect"
            write_kpoints "$q_dir"
            cp "$q0_relax/POTCAR" "$q_dir/POTCAR"
            write_submit_job "$q_dir" "${name}_q${q}"

            submit_dir "$q_dir" "$name/q${q}/relax"

            log_msg "Phase3: 准备并提交 $name/q${q}/relax (NELECT=$nelect, from q0 CONTCAR)"
        done
    done < "$CONFIG"

    return 0
}

phase3_check_and_advance() {
    # Phase 3: 所有 relax 收敛 → 准备所有 statics 输入 → 提交
    local all_relax_done=true
    local any_charged_exists=false

    # 检查所有charged relax
    for relax_dir in "$WORKDIR"/Defect/Intrinsic/*/q[+-]*/relax "$WORKDIR"/Defect/Impurity/*/q[+-]*/relax; do
        [ -d "$relax_dir" ] || continue
        any_charged_exists=true
        if ! is_converged "$relax_dir"; then
            if has_error "$relax_dir"; then
                local dname=$(basename $(dirname $(dirname "$relax_dir")))
                local qname=$(basename $(dirname "$relax_dir"))
                log_msg "ERROR: $dname/$qname/relax 出错，请人工检查"
            fi
            all_relax_done=false
        fi
    done

    $any_charged_exists || return 1
    $all_relax_done || return 1

    # 检查是否已经进入Phase4（statics已存在）
    local first_static=$(find "$WORKDIR/Defect" -path "*/statics/POSCAR" 2>/dev/null | head -1)
    [ -n "$first_static" ] && return 0

    # Claude审查所有 charged relax
    local review_list=()
    for relax_dir in "$WORKDIR"/Defect/Intrinsic/*/q[+-]*/relax "$WORKDIR"/Defect/Impurity/*/q[+-]*/relax; do
        [ -d "$relax_dir" ] || continue
        is_converged "$relax_dir" || continue
        local dname=$(basename $(dirname $(dirname "$relax_dir")))
        local qname=$(basename $(dirname "$relax_dir"))
        review_list+=("$relax_dir|$dname/$qname/relax")
    done
    claude_review_phase "Phase3" "${review_list[@]}" || return 1

    log_msg "Phase3→4: All charged relax CONVERGED, 开始准备static计算..."

    # --- Bulk/statics ---
    local bulk_statics="$WORKDIR/Bulk/statics"
    mkdir -p "$bulk_statics"
    cp "$WORKDIR/Bulk/relax/CONTCAR" "$bulk_statics/POSCAR"
    cp "$WORKDIR/Bulk/relax/CHGCAR" "$bulk_statics/CHGCAR"
    cp "$WORKDIR/Bulk/relax/POTCAR" "$bulk_statics/POTCAR"
    dispatch_incar_static "$bulk_statics"
    write_kpoints "$bulk_statics"
    write_submit_job "$bulk_statics" "bulk_static"
    submit_dir "$bulk_statics" "Bulk/statics"

    # --- 所有 defect statics ---
    for relax_dir in "$WORKDIR"/Defect/Intrinsic/*/q*/relax "$WORKDIR"/Defect/Impurity/*/q*/relax; do
        [ -d "$relax_dir" ] || continue
        is_converged "$relax_dir" || continue

        local q_dir=$(dirname "$relax_dir")
        local statics_dir="$q_dir/statics"
        local defect_name=$(basename $(dirname "$q_dir"))
        local charge=$(basename "$q_dir")
        mkdir -p "$statics_dir"

        cp "$relax_dir/CONTCAR" "$statics_dir/POSCAR"
        cp "$relax_dir/CHGCAR" "$statics_dir/CHGCAR"
        cp "$relax_dir/POTCAR" "$statics_dir/POTCAR"
        write_kpoints "$statics_dir"

        # 获取NELECT（如果是带电态）
        local nelect=""
        if [[ "$charge" != "q0" ]]; then
            nelect=$(get_nelect_from_outcar "$relax_dir")
        fi
        dispatch_incar_static "$statics_dir" "$nelect"
        write_submit_job "$statics_dir" "${defect_name}_${charge}_static"

        submit_dir "$statics_dir" "$defect_name/$charge/statics"
        log_msg "Phase4: 准备并提交 $defect_name/$charge/statics"
    done

    return 0
}

phase4_check_completion() {
    # Phase 4: 所有 statics 收敛 → 整个流程完成
    local all_done=true
    local any_static_exists=false

    # Bulk/statics
    if [ -d "$WORKDIR/Bulk/statics" ]; then
        any_static_exists=true
        is_converged "$WORKDIR/Bulk/statics" || all_done=false
    fi

    for statics_dir in "$WORKDIR"/Defect/Intrinsic/*/q*/statics "$WORKDIR"/Defect/Impurity/*/q*/statics; do
        [ -d "$statics_dir" ] || continue
        any_static_exists=true
        if ! is_converged "$statics_dir"; then
            if has_error "$statics_dir"; then
                local dname=$(basename $(dirname $(dirname "$statics_dir")))
                local qname=$(basename $(dirname "$statics_dir"))
                log_msg "ERROR: $dname/$qname/statics 出错，请人工检查"
            fi
            all_done=false
        fi
    done

    $any_static_exists || return 1
    $all_done || return 1

    # 检查是否已经运行过形成能计算
    [ -f "$WORKDIR/FormationEnergy/formation_energy.log" ] && return 0

    log_msg "Phase4→5: All statics CONVERGED, 开始FNV修正和形成能计算..."

    # 检查 formation_config.yaml 是否存在
    if [ ! -f "$WORKDIR/formation_config.yaml" ]; then
        log_msg "WARNING: formation_config.yaml 不存在，请创建后手动运行 formation_energy.py"
        log_msg "===== STATICS COMPLETE, 等待 formation_config.yaml ====="
        return 1
    fi

    # 运行形成能计算（自动包含FNV修正）
    cd "$WORKDIR"
    python3 formation_energy.py --config formation_config.yaml --output FormationEnergy/ 2>&1 | tee -a $LOG

    if [ -f "$WORKDIR/FormationEnergy/formation_energy.log" ]; then
        log_msg "===== ALL PHASES COMPLETE ====="
        log_msg "形成能计算完成，结果在 FormationEnergy/ 目录"
        return 0
    else
        log_msg "ERROR: 形成能计算失败，请检查"
        return 1
    fi
}

# ============ 状态报告 ============

print_status() {
    echo ""
    echo "=== Status Report $(date '+%Y-%m-%d %H:%M:%S') ==="

    check_one() {
        local dir=$1 label=$2
        [ -d "$dir" ] || return
        if is_converged "$dir"; then
            local E=$(grep "energy  without entropy" $dir/OUTCAR | tail -1 | awk '{print $NF}')
            echo "  CONVERGED  $label  (E=$E eV)"
        elif has_error "$dir"; then
            echo "  ERROR      $label"
        elif [ -f "$dir/OUTCAR" ]; then
            local steps=$(grep -c "F=" $dir/OSZICAR 2>/dev/null || echo "?")
            echo "  RUNNING    $label  (steps: $steps)"
        elif [ -f "$dir/POSCAR" ]; then
            echo "  READY      $label"
        else
            echo "  NOT_READY  $label"
        fi
    }

    echo "--- Bulk ---"
    check_one "$WORKDIR/Bulk/relax" "Bulk/relax"
    check_one "$WORKDIR/Bulk/statics" "Bulk/statics"

    echo "--- Defects ---"
    for defect_dir in "$WORKDIR"/Defect/Intrinsic/* "$WORKDIR"/Defect/Impurity/*; do
        [ -d "$defect_dir" ] || continue
        local dname=$(basename $defect_dir)
        for q_dir in "$defect_dir"/q*; do
            [ -d "$q_dir" ] || continue
            local charge=$(basename $q_dir)
            check_one "$q_dir/relax" "$dname/$charge/relax"
            check_one "$q_dir/statics" "$dname/$charge/statics"
        done
    done
    echo "=========================="
}

# ============ 主循环 ============

log_msg "===== AUTO MONITOR STARTED ====="
log_msg "WORKDIR: $WORKDIR"
log_msg "CONFIG: $CONFIG"
log_msg "Check interval: ${INTERVAL}s (10 min)"

while true; do
    # 按阶段依次检查，每个phase收敛后自动推进到下一个
    phase4_check_completion && {
        print_status
        log_msg "===== MONITOR STOPPED: ALL COMPLETE ====="
        exit 0
    }

    phase3_check_and_advance
    phase2_check_and_advance
    phase1_check_and_advance

    # 打印当前状态
    print_status >> $LOG

    # 等待10分钟
    sleep $INTERVAL
done
```

#### 启动与停止

```bash
# 启动监控（后台运行，无AI审查）
nohup bash auto_monitor.sh > monitor.out 2>&1 &
echo $! > monitor.pid

# 启动监控（后台运行，启用Claude AI审查）
nohup bash auto_monitor.sh --claude > monitor.out 2>&1 &
echo $! > monitor.pid

echo "Monitor started, PID=$(cat monitor.pid)"

# 查看实时输出
tail -f monitor.out

# 停止监控
kill $(cat monitor.pid)
```

#### --claude 模式说明

启用 `--claude` 后，每个阶段收敛时会自动调用 `claude -p` 审查计算结果：

**审查内容：**
- 能量收敛趋势（OSZICAR）：是否单调下降，有无震荡
- 电子步收敛（OUTCAR）：每个离子步电子步数是否合理（<40步）
- 原子力和应力：残余力是否<0.01 eV/Å
- 磁矩：总磁矩是否符合缺陷类型预期
- 结构变化：对比 POSCAR/CONTCAR，有无原子飞走或结构坍塌
- Static特有：LOCPOT是否正常，EIGENVAL带边位置是否合理

**审查结论：**
- `PASS` — 无异常，自动推进到下一阶段
- `WARNING` — 有潜在问题但不阻塞，记录后继续推进
- `FAIL` — 严重问题，**暂停自动推进**，等待人工检查

**审查结果保存在：** 各计算目录下的 `claude_review.txt`

**注意：** `--claude` 需要 `claude` CLI 已安装且可用。每次审查会消耗API调用。

#### 手动状态检查（可选）

```bash
#!/bin/bash
# check_jobs.sh — 手动检查当前状态，不影响自动监控
WORKDIR=$(pwd)
echo "=== Defect Workflow Status ==="
echo "Date: $(date)"
check_dir() {
    local dir=$1 label=$2
    [ -d "$dir" ] || return
    if grep -q "reached required accuracy" $dir/OUTCAR 2>/dev/null; then
        local E=$(grep "energy  without entropy" $dir/OUTCAR | tail -1 | awk '{print $NF}')
        echo "  CONVERGED  $label  (E=$E eV)"
    elif grep -q "Error\|ZBRENT\|VERY BAD NEWS" $dir/vasp.out 2>/dev/null; then
        echo "  ERROR      $label"
    elif [ -f "$dir/OUTCAR" ]; then
        local steps=$(grep -c "F=" $dir/OSZICAR 2>/dev/null || echo "?")
        echo "  RUNNING    $label  (steps: $steps)"
    elif [ -f "$dir/POSCAR" ]; then
        echo "  READY      $label"
    fi
}
echo "--- Bulk ---"
check_dir "$WORKDIR/Bulk/relax" "Bulk/relax"
check_dir "$WORKDIR/Bulk/statics" "Bulk/statics"
echo "--- Defects ---"
for d in "$WORKDIR"/Defect/Intrinsic/* "$WORKDIR"/Defect/Impurity/*; do
    [ -d "$d" ] || continue
    for q in "$d"/q*; do
        [ -d "$q" ] || continue
        check_dir "$q/relax" "$(basename $d)/$(basename $q)/relax"
        check_dir "$q/statics" "$(basename $d)/$(basename $q)/statics"
    done
done
```

---

## Quick Reference: VASP Parameters

### PBE Parameters (LEVEL1, or relax in LEVEL2)

| Parameter | Bulk Relax | Defect Relax | Static |
|-----------|-----------|-------------|--------|
| ISIF | 3 | 2 | (none) |
| IBRION | 2 | 2 | (none) |
| NSW | 500 | 500 | (none, 0) |
| IALGO | 38 | 38 | 38 |
| LREAL | Auto | Auto | Auto |
| POTIM | 0.5 | 0.5 | - |
| EDIFFG | -0.01 | -0.01 | - |
| NELM | 100 | 100 | 100 |
| LVHAR | - | - | .TRUE. |
| ICORELEVEL | - | - | 1 |
| ICHARG | - | - | 1 |
| NELECT | (auto) | (auto for q0, set for q≠0) | (same as relax) |
| ISPIN | 2 | 2 | 2 |

### HSE06 Parameters (relax in LEVEL3, statics in LEVEL2/3)

| Parameter | Bulk Relax | Defect Relax | Static |
|-----------|-----------|-------------|--------|
| ISIF | 3 | 2 | (none) |
| IBRION | 2 | 2 | (none) |
| NSW | 500 | 500 | (none, 0) |
| ALGO | Damped | Damped | All |
| LREAL | .FALSE. | .FALSE. | .FALSE. |
| LHFCALC | .TRUE. | .TRUE. | .TRUE. |
| HFSCREEN | 0.2 | 0.2 | 0.2 |
| AEXX | 0.25 | 0.25 | 0.25 |
| TIME | 0.4 | 0.4 | 0.4 |
| PRECFOCK | Fast | Fast | Fast |
| POTIM | 0.3 | 0.3 | - |
| EDIFFG | -0.02 | -0.02 | - |
| NELM | 100 | 100 | 200 |
| LVHAR | - | - | .TRUE. |
| ICORELEVEL | - | - | 1 |
| ICHARG | - | - | 1 |
| NELECT | (auto) | (auto for q0, set for q≠0) | (same as relax) |
| ISPIN | 2 | 2 | 2 |

### Level Selection Summary

| Level | Relax INCAR | Statics INCAR | band_gap in config | mu_ref in config |
|-------|-------------|---------------|--------------------|--------------------|
| LEVEL1 | PBE | PBE | PBE gap | PBE reference |
| LEVEL2 | PBE | HSE06 | HSE gap | HSE reference |
| LEVEL3 | HSE06 | HSE06 | HSE gap | HSE reference |

## References

Read the detailed reference files for:
- `references/chemical-potentials.md` — Stability region determination
- `references/formation-energy-and-corrections.md` — Formation energy terms and FNV/Kumagai corrections
- `references/formation-energy-diagram.md` — Plotting and thermodynamic analysis
- `references/workflow-automation.md` — SLURM job scripts and monitoring
