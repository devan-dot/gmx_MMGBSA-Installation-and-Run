#The pipeline is fine tuned for the following system settings:
#gmx_MMPBSA 1.6.3 — Working Installation & Usage (Ubuntu 22.04, CUDA, GROMACS 2024.5)
#System
#Component	Value
#OS	Ubuntu 22.04.5 LTS (jammy)
#Kernel	6.8.0
#CPU	x86_64 (AVX2)
#GPU	NVIDIA T1000 (4 GB)
#Driver	580.95.05
#CUDA runtime	13.0
#nvcc	11.5
#Python	3.9 (inside env)
#Conda	25.11.1
#GROMACS	2024.5 (CUDA, thread_mpi)
#Edit this data based on users system specifications.


1.) Create clean environment

conda create -n gmxMMPBSA -c conda-forge python=3.9 -y
conda activate gmxMMPBSA

2.) Install AmberTools stack

conda install -c conda-forge \
  ambertools=20 \
  parmed \
  cpptraj \
  tleap \
  sander \
  -y

3.) Install MPI and mpi4py (Conda only)

conda install -c conda-forge mpi mpich mpi4py=4.1.0 -y

Verify:

which mpirun
python - << 'EOF'
import mpi4py
print(mpi4py.__version__)
EOF

Expected:

4.1.0

4.) Install gmx_MMPBSA (pip, no deps)

pip install gmx-MMPBSA==1.6.3 --no-deps

Verify:

which gmx_MMPBSA
gmx_MMPBSA --version

Expected:

gmx_MMPBSA v1.6.3

5.) Environment sanity check

which cpptraj
which tleap
which sander
which gmx
echo $AMBERHOME

Expected:

$AMBERHOME = /home/k4bioinfo/miniconda3/envs/gmxMMPBSA

6.) 
A) Running MM/GBSA

mmgbsa.in: 
&general
startframe=1,
endframe=5001,
interval=10,
/

&gb
igb=5,
saltcon=0.15,
/

&decomp
idecomp=1,
/

Command:

gmx_MMPBSA -O \
  -i mmgbsa.in \
  -cs production.tpr \
  -ct protein_only.xtc \
  -ci test_index.ndx \
  -cg 16 17 \
  -cp topol.top \
  -o FINAL_RESULTS_MMGBSA.dat \
  -eo FINAL_RESULTS_PERFRAME_GBSA.csv \
  2>&1 | tee gmx_MMGBSA_run.log
  
B) Running MM/PBSA  
  
mmpbsa.in:
&general
startframe=1,
endframe=5001,
interval=10,
/

&pb
istrng=0.15,
fillratio=4.0,
radiopt=0,
indi=1.0,
exdi=80.0,

/

&decomp
  idecomp=1,
/

Command:
mpirun -np 4 gmx_MMPBSA -O \
  -i mmpbsa.in \
  -cs production.tpr \
  -ct protein_only.xtc \
  -ci test_index.ndx \
  -cg 16 17 \
  -cp topol.top \
  -o FINAL_RESULTS_MMPBSA.dat \
  -eo FINAL_RESULTS_PERFRAME_MMPBSA.csv \
  2>&1 | tee gmx_MMPBSA_run.log
Successful run indicators->

Log should contain:

cpptraj found
tleap found
parmchk2 found
sander found
gmx found
Detected Amber/OPLS force field topology format
Starting calculations in 1 CPUs...
Total time taken: ~11 min
Finalizing gmx_MMPBSA: [ERROR] = 0; [WARNING] = 0

7.) Check Output files
File	                           Description
FINAL_RESULTS_MMGBSA/PBSA.dat	   Final average ΔG
FINAL_RESULTS_PERFRAME_GBSA/PBSA.csv   	   Per-frame energies
gmx_MMGBSA/PBSA_run.log   	           Full run log
gmx_MMGBSA/PBSA.log	                   Internal diagnostic

GUI Analysis

8.) Launch Gui version:

Command: gmx_MMPBSA_ana (launchs on its own from command)

Load:

    FINAL_RESULTS_PERFRAME_GBSA/PBSA.csv for time series

    FINAL_RESULTS_MMGBSA/PBSA.dat for summary

Create plots based on the dat files

for more information reach out to https://github.com/devan-dot
