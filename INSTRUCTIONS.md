# PLUMED Masterclass 22.8: Modelling Concentration-driven processes with PLUMED

## Origin 

This masterclass was authored by Matteo Salvalaglio on May 20, 2022

## Aims

This Masterclass aims to introduce users to the tools available via PLUMED to analyze and perform simulations of concentration-driven processes such as nucleation, growth, and diffusion. 

## Objectives

Once this Masterclass is completed, users will be able to:

- Write a PLUMED input file to analyze a trajectory with multicolvars.
- Use the Depth-First-Search tools available in PLUMED to characterize clusters of atoms in an existing vapour condensation trajectory.
- Write a PLUMED input file to analyze the condensation of a simple liquid phase from vapour.  
- Write a PLUMED input file for CmuMD, and perform a CmuMD simulation of liquid condensation at constant driving force.
- Write a PLUMED input file for CmuMD, and perform a CmuMD simulation to model steady-state diffusive flux across a simulation box. 

## Prerequisites

We assume that you are familiar with PLUMED. However, the 2021 PLUMED Masterclass is a great place to start if you are not.

## Setting up the software 

Follow the instructions provided [here](https://urldefense.com/v3/__https://github.com/plumed/masterclass-2022__;!!JFdNOqOXpB6UZW0!5LDzfOpI0a0QUEAXjfWbM4f1ubgf4gC-ELH4qKstVB9wJwXRDKOjaCtm52xOb0YEw5tvTtPxyw$ ) to install gromacs+plumed2.8 compiled with [CmuMD](https://urldefense.com/v3/__https://github.com/mme-ucl/CmuMD__;!!JFdNOqOXpB6UZW0!5LDzfOpI0a0QUEAXjfWbM4f1ubgf4gC-ELH4qKstVB9wJwXRDKOjaCtm52xOb0YEw5sp2Fhg_A$ ).

## Background Overview 

PLUMED facilitates the calculation of functions of atom coordinates and the introduction of bias potentials in atomistic simulations. 

In this tutorial, we review how to use PLUMED functionalities for two complementary tasks: 
- Calculating collective variables that capture processes of assembly/disassembly such as the nucleation, condensation, dissolution etc. (Ex. 1, 2)
- Introducing biasing forces through CmuMD, a [method](https://urldefense.com/v3/__https://aip.scitation.org/doi/abs/10.1063/1.4917200__;!!JFdNOqOXpB6UZW0!5LDzfOpI0a0QUEAXjfWbM4f1ubgf4gC-ELH4qKstVB9wJwXRDKOjaCtm52xOb0YEw5vzGbfc4A$ ) that mimics open-boundary conditions and allows for mitigation of finite-size effects. (Ex. 3, 4)

## Resources

The data needed to complete the exercises of this Masterclass can be found on [GitHub](https://github.com/mme-ucl/PLUMED_MClass_22-08).
You can clone this repository locally on your machine using the following command:

````
git clone https://github.com/mme-ucl/PLUMED_MClass_22-08 
````

## Solution

The solution of this masterclass is available on GitHub as a [jupyter notebook](https://github.com/mme-ucl/PLUMED_MClass_22-08/blob/main/Solution/Solution.ipynb). 

## Exercise 1: analysis of a nucleation trajectory I 

We can start by using plumed to analyze a trajectory and identify the total number of atoms belonging to a liquid phase condensing from a supersaturated vapour in a long unbiased MD simulation.

To this aim, a typical choice is the implementation of a criterion inspired by the work of Ten Wolde and Frenkel: all the atoms with a threshold number of nearest neighbours larger than a threshold is considered as part of the liquid phase. 

In PLUMED this criterion can be readily implemented using the multicolvar [COORDINATIONNUMBER](https://www.plumed.org/doc-master/user-doc/html/_c_o_o_r_d_i_n_a_t_i_o_n_n_u_m_b_e_r.html): 

```plumed
# LIQUID-like atoms
lq: COORDINATIONNUMBER SPECIES=1-10000 SWITCH={CUBIC D_0=0.45 D_MAX=0.55} MORE_THAN={RATIONAL R_0=5.0 D_MAX=10.0}
PRINT ARG=lq.morethan STRIDE=1 FILE=nliquid.dat
```

The time evolution of the number of atoms in the liquid phase can be computed by analyzing the trajectory using the [driver](https://www.plumed.org/doc-master/user-doc/html/driver.html) functionality (where liquid.dat is the plumed file for this analysis). 

````
plumed driver --mf_xtc LJvap_to_liq_red.xtc --plumed liquid.dat
````

### Try on your own

Using other MultiColvar keywords can you count the number of atoms on the surface of the clusters forming in this trajectory? Can you estimate how the surface to volume ratio of the droplets forming in the system changes directly within PLUMED? 

### Available data

The MD trajectory that we will analyze can be found in the folder called `data`:
- `LJvap.gro`: reference conformation of 10000 LJ particles in a metastable vapour phase.
- `LJvap_to_liq.xtc`: trajectory.

## Exercise 2: analysis of a nucleation trajectory II

In certain contexts, it can be helpful to characterize the phase transition by following analyzing the population of clusters that form during the nucleation process. In PLUMED, this is possible by taking advantage of the [DFSCLUSTERING](https://www.plumed.org/doc-master/user-doc/html/_d_f_s_c_l_u_s_t_e_r_i_n_g.html) algorithm, which builds on the construction of a [CONTACT_MATRIX](https://www.plumed.org/doc-master/user-doc/html/_c_o_n_t_a_c_t__m_a_t_r_i_x.html) to identify clusters of atoms with specific properties. 

Here we analyse the clusters distribution emerging in the trajectory `LJvap_to_liq.xtc`, focussing on the number of clusters, and following the evolution of the four largest clusters in the system: 

```plumed
# Identify liquid-like atoms
lq: COORDINATIONNUMBER SPECIES=1-10000 SWITCH={CUBIC D_0=0.45  D_MAX=0.55} MORE_THAN={RATIONAL R_0=5.0 D_MAX=10.0}

# Define a contact matrix & perfom DFS clustering 
cm: CONTACT_MATRIX ATOMS=lq  SWITCH={CUBIC D_0=0.45  D_MAX=0.55}
dfs: DFSCLUSTERING MATRIX=cm

# Compute the size of the four largest clusters
cluster_1: CLUSTER_NATOMS CLUSTERS=dfs CLUSTER=1    
cluster_2: CLUSTER_NATOMS CLUSTERS=dfs CLUSTER=2
cluster_3: CLUSTER_NATOMS CLUSTERS=dfs CLUSTER=3
cluster_4: CLUSTER_NATOMS CLUSTERS=dfs CLUSTER=4

# Compute the number of clusters  
nclust: CLUSTER_DISTRIBUTION CLUSTERS=dfs MORE_THAN={GAUSSIAN D_0=D_0=1.95 R_0=0.01 D_MAX=1.99}

# PRINT to file
PRINT ARG=lq.morethan,nclust.*,cluster_1,cluster_2,cluster_3,cluster_4 STRIDE=1  FILE=clusters.dat

FLUSH STRIDE=1
```

### Try on your own

Modifying the setup above, can you compute the number of clusters with a size larger than 20 LJ particles? 

### Available data

The MD trajectory that we will analyze can be found in the folder called `data`:
- `LJvap.gro`: reference conformation of 10000 LJ particles in a metastable vapour phase.
- `LJvap_to_liq.xtc`: trajectory.

## Exercise 3: Steady-state diffusive flux with CmuMD

In the two examples discussed in Exercises 1 and 2 it is apparent that the process of condensation reaches steady state due to finite-size effects. However, looking at the process more carefully, one can notice that the driving force leading to phase separation is far from constant.
C$\mu$MD allows using PLUMED to apply ad-hoc forces and keep constant the composition of spatial regions of the simulation box called control regions. This feature allows to model processes driven by concentration in steady, out-of-equilibrium conditions. 

The first example we will focus on is a purely diffusive process in a LJ vapour. 

A PLUMED file that allows imposing a steady concentration difference with CmuMD looks like: 

```plumed
# Define groups of atoms
LJ: GROUP ATOMS=1-1000:1 

# Provide parameters for the CV
left:  CMUMD GROUP=lj NSV=1 FIXED=0.5 DCR=0.25 CRSIZE=0.1 WF=0.0001  ASYMM=-1 NINT=0.1 NZ=291
right:  CMUMD GROUP=lj NSV=1 FIXED=0.5 DCR=0.25 CRSIZE=0.1 WF=0.0001  ASYMM=1 NINT=0.1 NZ=291

# CmuMD is implemented as a restraint on the densities of species in CR
rleft:  RESTRAINT ARG=left AT=1.4 KAPPA=1000.0 
rright: RESTRAINT ARG=right AT=0.05 KAPPA=2000.0 

# Report the densities and bias
PRINT ...
ARG=left,right,rleft.bias,rright.bias
STRIDE=10
FILE=CMUMD_log
... PRINT
```

We can run a CMUMD simulation under these conditions as: 

````
gmx_mpi mdrun --plumed cmumd_diff.dat
````

### Try on your own 

Perform simulations at varying concentration differences.
Compute the concentration gradient across the simulation box as a function of the concentration difference between left and right controlled volumes.

### Available data 

In the folder `data`:
- `LJ_diffusion.gro`: reference conformation of 1000 LJ particles.
- `LJ_diffusion.xtc`: trajectory evolving under the effect of the stationary concentration gradient. 
 
## Exercise 4: Steady-state condensation process with C$\mu$MD

The fourth exercise combines aspects of the previous three. We will use CmuMD to control the driving force associated with the growth of a dense phase represented by a slab at the center of a simulation box. 

Similarly to the diffusion case the plumed file that can be used to perform this type of simulation reads: 

```plumed
# Define groups of atoms
LJ: GROUP ATOMS=1001-2000:1

# Provide parameters for the CV
left:  CMUMD GROUP=lj NSV=1 FIXED=0.4 DCR=0.25 CRSIZE=0.1 WF=0.0001  ASYMM=-1 NINT=0.1 NZ=291
right:  CMUMD GROUP=lj NSV=1 FIXED=0.6 DCR=0.25 CRSIZE=0.1 WF=0.0001  ASYMM=1 NINT=0.1 NZ=291

# CmuMD is implemented as a restraint on the densities of species in CR
left:  RESTRAINT ARG=left AT=3. KAPPA=2000.0 
right: RESTRAINT ARG=right AT=3. KAPPA=2000.0 

# Report the densities and bias
PRINT ...
ARG=left,right,rleft.bias,rright.bias
STRIDE=10
FILE=CMUMD_log
... PRINT
``` 

### Available data 

In the folder `data`, a CmuMD trajectory ready for analysis can be found together with the gromacs input files necessary to setup and run it. 
- `LJ_slab.gro`: reference conformation of 1000 LJ particles.
- `LJ_slab.xtc`: trajectory evolving under the effect of the stationary concentration gradient. 
- `md_input_LJ_slab`: MD input files


### Try on your own 

- Setup and run CMUMD simulations for the LJ slab system at varying CR concentrations. 
- Use the multicolvar-based approach discussed in Ex1 to follow the condensation process.  
- Use the graph-based approach discussed in Ex2 to follow the condensation process.
- Monitor the density profile across the simulation box. 
