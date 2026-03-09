# Workflow Automation: SLURM Job Management

## SLURM Job Script Generation

### Template (from user's cluster)

```bash
#!/bin/sh
#SBATCH -N 2
#SBATCH -J <job_name>
#SBATCH -n 104
#SBATCH --ntasks-per-node=52
#SBATCH --partition=compute2
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH --time=24:00:00

mpirun -n 104 <vasp_executable> > vasp.out 2>&1
```

### Executable selection
- `vasp_gam` = `/opt/vasp.5.4.4/bin/vasp_gam` → use when KPOINTS = Gamma 1×1×1
- `vasp_std` = `/opt/vasp.5.4.4/bin/vasp_std` → use when multiple k-points

### POTCAR path
```
/data2/home/chensy/POT/potpaw_PBE/<Element>/POTCAR
```
Concatenate in the order matching POSCAR species line.

---

## Master Workflow Controller

### Full automation script

```bash
#!/bin/bash
#
# defect_workflow_master.sh
# Automates the complete defect calculation workflow
#
# Usage: bash defect_workflow_master.sh [config_file]
#

set -e
WORKDIR=$(pwd)
LOG="$WORKDIR/defect_workflow.log"
POTCAR_PATH="/data2/home/chensy/POT/potpaw_PBE"

# Cluster settings
NODES=2
CORES_PER_NODE=52
TOTAL_CORES=$((NODES * CORES_PER_NODE))
PARTITION="compute2"
MAX_TIME="24:00:00"
VASP_GAM="/opt/vasp.5.4.4/bin/vasp_gam"
VASP_STD="/opt/vasp.5.4.4/bin/vasp_std"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"
}

# Generate submit_job script for a directory
generate_submit_script() {
    local dir=$1
    local job_name=$2
    local vasp_exec=$3

    cat > "$dir/submit_job" << SCRIPT
#!/bin/sh
#SBATCH -N $NODES
#SBATCH -J $job_name
#SBATCH -n $TOTAL_CORES
#SBATCH --ntasks-per-node=$CORES_PER_NODE
#SBATCH --partition=$PARTITION
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH --time=$MAX_TIME

mpirun -n $TOTAL_CORES $vasp_exec > vasp.out 2>&1
SCRIPT
    chmod +x "$dir/submit_job"
}

# Check if a VASP job converged
check_converged() {
    local dir=$1
    if grep -q "reached required accuracy" "$dir/OUTCAR" 2>/dev/null; then
        return 0
    else
        return 1
    fi
}

# Wait for a SLURM job to finish
wait_for_job() {
    local job_id=$1
    local job_name=$2
    while squeue -j "$job_id" -h 2>/dev/null | grep -q "$job_id"; do
        sleep 60
    done
    log "  Job $job_id ($job_name) completed"
}

# Wait for multiple jobs
wait_for_jobs() {
    local job_ids=("$@")
    for jid in "${job_ids[@]}"; do
        while squeue -j "$jid" -h 2>/dev/null | grep -q "$jid"; do
            sleep 60
        done
    done
    log "  All jobs completed: ${job_ids[*]}"
}

# Submit a VASP job and return job ID
submit_job() {
    local dir=$1
    local dep_id=$2  # optional dependency
    cd "$dir"
    if [ -n "$dep_id" ]; then
        local jid=$(sbatch --parsable --dependency=afterok:$dep_id submit_job)
    else
        local jid=$(sbatch --parsable submit_job)
    fi
    cd "$WORKDIR"
    echo "$jid"
}

# Verify convergence after job completion
verify_job() {
    local dir=$1
    local name=$2
    if check_converged "$dir"; then
        local E=$(grep "energy  without entropy" "$dir/OUTCAR" | tail -1 | awk '{print $NF}')
        log "  ✓ $name CONVERGED: E = $E eV"
        return 0
    else
        log "  ✗ $name FAILED: not converged"
        return 1
    fi
}

# Prepare static calculation from relaxation
prepare_static() {
    local relax_dir=$1
    local static_dir=$2
    local nelect_line=$3  # optional NELECT line

    mkdir -p "$static_dir"
    cp "$relax_dir/CONTCAR" "$static_dir/POSCAR"
    cp "$relax_dir/CHGCAR" "$static_dir/" 2>/dev/null || true
    cp "$relax_dir/POTCAR" "$static_dir/"
    cp "$relax_dir/KPOINTS" "$static_dir/"

    # Static INCAR
    cat > "$static_dir/INCAR" << INCAR
NELM = 100
EDIFF = 1E-5
EDIFFG = -0.01
ISMEAR = 0
SIGMA = 0.05
GGA = PE
IALGO = 38
LORBIT = 10
PREC = Accurate
ISPIN = 2
ENCUT = 356
NPAR = 8
ICORELEVEL = 1
LVHAR = .TRUE.
ICHARG = 1
INCAR

    # Add NELECT if charged
    if [ -n "$nelect_line" ]; then
        echo "$nelect_line" >> "$static_dir/INCAR"
    fi

    generate_submit_script "$static_dir" "static" "$VASP_GAM"
}
```

---

## Job Status Monitoring

### Quick status check

```bash
#!/bin/bash
# check_all_jobs.sh
# Run from the main working directory

echo "================================================"
echo "  Defect Calculation Status Report"
echo "  $(date)"
echo "================================================"

check_dir() {
    local dir=$1
    local label=$2
    if [ ! -d "$dir" ]; then
        echo "  [ ]  $label : directory not found"
        return
    fi
    if [ ! -f "$dir/OUTCAR" ]; then
        if [ -f "$dir/submit_job" ]; then
            echo "  [-]  $label : not yet submitted or running"
        else
            echo "  [ ]  $label : not set up"
        fi
        return
    fi
    if grep -q "reached required accuracy" "$dir/OUTCAR" 2>/dev/null; then
        E=$(grep "energy  without entropy" "$dir/OUTCAR" | tail -1 | awk '{print $NF}')
        NSTEPS=$(grep -c "E0=" "$dir/OSZICAR" 2>/dev/null)
        echo "  [✓]  $label : CONVERGED (E=$E eV, steps=$NSTEPS)"
    elif grep -q "Error\|ZBRENT\|VERY BAD NEWS" "$dir/vasp.out" 2>/dev/null; then
        echo "  [✗]  $label : ERROR (check vasp.out)"
    else
        NSTEPS=$(grep -c "E0=" "$dir/OSZICAR" 2>/dev/null)
        echo "  [?]  $label : NOT CONVERGED ($NSTEPS steps done)"
    fi
}

echo ""
echo "--- Bulk Relaxation ---"
check_dir "relax" "Bulk ISIF=3"

echo ""
echo "--- Host Static ---"
check_dir "Intrinsic_Defect/host" "Host static"

echo ""
echo "--- Defect Calculations ---"
for defect in Intrinsic_Defect/*/initial_structure; do
    dname=$(basename $(dirname $defect))
    [ "$dname" = "host" ] && continue
    for qdir in $defect/q*; do
        qname=$(basename $qdir)
        check_dir "$qdir" "$dname/$qname relax"
        check_dir "$qdir/static" "$dname/$qname static"
    done
done

echo ""
echo "--- Madelung ---"
check_dir "madelung/static" "Madelung constant"

echo ""
echo "================================================"
```

---

## Common Issues and Recovery

### Job didn't converge (NSW reached)
```bash
# Restart from CONTCAR
cp CONTCAR POSCAR
# Optionally increase NSW
sed -i 's/NSW = 500/NSW = 1000/' INCAR
sbatch submit_job
```

### SCF not converging
```bash
# Try different algorithm
echo "ALGO = All" >> INCAR
echo "NELM = 200" >> INCAR
echo "AMIX = 0.2" >> INCAR
echo "BMIX = 0.0001" >> INCAR
```

### Charged defect doesn't converge
```bash
# Start from neutral WAVECAR
cp ../q0/WAVECAR .
echo "ISTART = 1" >> INCAR  # Read WAVECAR
```

### Check max concurrent jobs
```bash
squeue -u $(whoami) | wc -l  # Current running + queued
# dasp.in says max_job = 4, respect this limit
```

---

## Energy Extraction Script

```bash
#!/bin/bash
# extract_energies.sh
# Extract total energies from all converged calculations

echo "=== Total Energies (eV) ==="
echo ""

# Bulk
if [ -f "Intrinsic_Defect/host/OUTCAR" ]; then
    E=$(grep "energy  without entropy" Intrinsic_Defect/host/OUTCAR | tail -1 | awk '{print $NF}')
    echo "Host (bulk):  $E"
fi

# Defects
for defect in Intrinsic_Defect/*/initial_structure; do
    dname=$(basename $(dirname $defect))
    [ "$dname" = "host" ] && continue
    echo ""
    echo "--- $dname ---"
    for qdir in $defect/q*/static; do
        qname=$(basename $(dirname $qdir))
        if [ -f "$qdir/OUTCAR" ]; then
            E=$(grep "energy  without entropy" "$qdir/OUTCAR" | tail -1 | awk '{print $NF}')
            NELECT=$(grep "NELECT" "$qdir/OUTCAR" | head -1 | awk '{print $3}')
            echo "  $qname (NELECT=$NELECT): E = $E"
        fi
    done
done
```
